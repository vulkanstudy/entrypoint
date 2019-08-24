# 初期化

少しづつVulkanのプログラムを作っていきましょう

今回のプログラムを入力して、[最終的に作られるコード](https://github.com/vulkanstudy/4_setup)はこちら。

# アプリケーションクラスの導入

メイン関数をシンプルにするために、アプリケーションクラスを導入していきましょう。

## メイン関数

アプリケーションクラスを導入することで、メイン関数はシンプルにできます。
具体的には、アプリケーションクラスのインスタンスを作成して、`run()` メソッドを呼び出します。

```cpp:src/main.cpp
#include <iostream>
#include "MyApplication.h"

int main() 
{
	MyApplication app;

	try {
		app.run();
	}
	catch (const std::exception& e) {
		std::cerr << e.what() << std::endl;
		return EXIT_FAILURE;
	}

	return EXIT_SUCCESS;

	return 0;
}
```

## アプリケーションクラス

アプリケーションクラスは、次のクラスを拡張していくものとなります。

```cpp:src/MyApplication.h 
#pragma once

class MyApplication
{
public:
	MyApplication();
	~MyApplication();

	void run();
};
```

# ウィンドウの生成
Windows のアプリケーションは、画面の一部分の「窓」で実行されます（全画面かもしれませんが）。

GLFWは、OSのAPI呼び出しを大分サポートしてくれるので、かなり簡単にウィンドウを生成することができます。
この処理だけを追加してみましょう。

## 宣言の追加

GLFWでは、GLFWwindowへのポインタとして、ウィンドウを管理することができます。
こちらは、アプリケーションクラスのメンバーとして持たせましょう。
後は、セッティングとしての「初期化」、「後片付け」のメソッドと、初期化が終わった後の基本ループである「通常処理」の呼び出しでアプリケーションを実行させ続けることができます。


```cpp:src/MyApplication.h 
#pragma once

#include <GLFW/glfw3.h>

class MyApplication
{
private:
	GLFWwindow* window_;
public:
	MyApplication(): window_(nullptr){}
	~MyApplication() {}

	void run()
	{
		// 初期化
		initializeWindow();

		// 通常処理
		mainloop();

		// 後片付け
		finalizeWindow();
	}
};
```

## 生成処理の実体

では、それぞれのメソッドの中を見てみましょう。

GLFWは、``glfwInit()``関数で初期化して、``glfwCreateWindow()``関数でウィンドウを生成します。
glfwCreateWindowの返り値が、GLFWwindowのインスタンスへのポインタです。
引数は、ウィンドウ内の解像度とタイトル名などを指定します。``nullptr``になっているパラメータは、
後から登場するので、気にしないで進んでいきましょう。

ウィンドウ生成時のオプションは、``glfwWindowHint``で指定します。
今回は、Vulaknを使うので、OpenGL やOpenGL ES でない ``GLFW_NO_API``の指定と、
ユーザーによるウィンドウのサイズ変更を禁止する設定を行いました。
他のオプションは、GLFWの「Window guide」の[「Window creation hints」](https://www.glfw.org/docs/latest/window_guide.html#GLFW_CLIENT_API_hint)の項目を見てみてください。

```cpp:src/MyApplication.h 
	constexpr static char APP_NAME[] = "Vulkan Application";
	// 表示ウィンドウの設定
	void initializeWindow()
	{
		const int WIDTH = 800;
		const int HEIGHT = 600;

		glfwInit();

		glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);// OpenGL の種類の設定
		glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);// ユーザーはウィンドウサイズを変更できない

		window_ = glfwCreateWindow(WIDTH, HEIGHT, APP_NAME, nullptr, nullptr);
	}
```

後片付けは、初期化のそれぞれに対応する関数`glfwTerminate()`、`glfwDestroyWindow()`を、逆順で呼び出していきます。


```cpp:src/MyApplication.h 
	void finalizeWindow()
	{
		glfwDestroyWindow(window_);
		glfwTerminate();
	}
```

メインループは、GLFWが終わらないか確認しながら、イベントが起きる様なら(例えば、キーボード入力)それを処理していきます。

```cpp:src/MyApplication.h 
	// 通常の処理
	void mainloop()
	{
		while (!glfwWindowShouldClose(window_))
		{
			glfwPollEvents();
		}
	}
```

# Vulkanの導入

GLFWの頑張りで、ウィンドウを表示することができたので、次は、Vulkanが画面を表示するようにしていきます
（といっても、実際に意味のある画面を表示するのは、まだまだ先なのですが…）。

Vulkanに指示を出すのは、「VkInstance」のインスタンス(Vulkanのメモリ的な意味での実体)を通して行います。
![Vulkanのインスタンス](4/instance.png "Vulkanのインスタンス")

## 宣言の追加

Vulkanの初期化と後片付けのコードをまず追加してきましょう。

```cpp:src/MyApplication.h 
	VkInstance instance_;// ★追加！
	void run()
	{
		// 初期化
		initializeWindow();
		initializeVulkan();// ★追加！

		// 通常処理
		mainloop();

		// 後片付け
		finalizeVulkan();// ★追加！
		finalizeWindow();
	}
```

## インスタンス生成の実体

初期化と片付けの関数では、インスタンスを生成するメドッドと、インスタンスを破棄する関数``vkDestroyInstance``を呼び出します。

```cpp:src/MyApplication.h 
	// Vulkanの設定
	void initializeVulkan()
	{
		createInstance(&instance_);
	}

	void finalizeVulkan()
	{
		vkDestroyInstance(instance_, nullptr);
	}
```

ここで、``createInstance`` は、MyApplicationのprivateメソッドです。
ここから、Vulkanの面倒くさいところが始まってきます。

``createInstance``では、``VkInstanceCreateInfo``の構造体を渡して``vkCreateInstance``を呼び出します。

``VkInstanceCreateInfo``構造体では、タイトル名などのアプケーション情報を記録した``VkApplicationInfo``構造体の実体のアドレスを記録することで、
アプリケーションの設定情報をVulkanに伝えます。
![vkCreateInstanceのイメージ](4/instance_info.png "vkCreateInstanceのイメージ")


```cpp:src/MyApplication.h 
	static void createInstance(VkInstance *dest)
	{
		// アプケーション情報を定めるための構造体
		VkApplicationInfo appInfo = {};
		appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;    // 構造体の種類
		appInfo.pApplicationName = APP_NAME;                   // アプリケーション名
		appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0); // 開発者が決めるバージョン番号
		appInfo.pEngineName = "My Engine";                     // ゲームエンジン名
		appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);      // ゲームエンジンのバージョン
		appInfo.apiVersion = VK_API_VERSION_1_0;               // 使用するAPIのバージョン

		// 新しく作られるインスタンスの設定の構造体
		VkInstanceCreateInfo createInfo = {};
		createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO; // 構造体の種類
		createInfo.pApplicationInfo = &appInfo;                    // VkApplicationInfoの情報

		// valkanの拡張機能を取得して、初期化データに追加
		std::vector<const char*> extensions = getRequiredExtensions();
		createInfo.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
		createInfo.ppEnabledExtensionNames = extensions.data();

		// インスタンスの生成
		if (vkCreateInstance(&createInfo, nullptr, dest) != VK_SUCCESS) {
			throw std::runtime_error("failed to create instance!");
		}
	}

```

なお、``VkInstanceCreateInfo`` や ``VkApplicationInfo`` は共に``sType``という名前の「種類」を指定するためのメンバーが存在しています。
これは、Vulkanの内部構造に関係しています。
Vulkanでは、GPUへの指示を「コマンド」の形で送信します。コマンドは、リングバッファに格納された指示の種類と必要なパラメータです。
GPUは、リングバッファのデータを随時読みだして、種類ごとの指示を実行します。
さて、コマンドの「指示」を判断するための情報が必要です。その情報こそが``sType``になります。
``VK_STRUCTURE_TYPE_***``ですと、単なる情報の塊として、GPUが使いやすいメモリのどこかにコピーされるだけですが
(実際にインスタンスを「生成」するコマンドは、``vkCreateInstance``の中で作り出されます)、
それにしても、後から使う上で送られてきたデータをどのように扱えば良いか判断するための情報をGPUに伝えなければならず、
それを``sType``を通じて行うことになっています。

![Vulkanのコマンドを送るイメージ](4/command.png "Vulkanのコマンドを送るイメージ")

ということで、``VkInstanceCreateInfo``などの型の情報があれば、それを利用してどのようなデータなのか周知させることが
できるように思いますが、Vulkanのデータでは、いちいち``sType``にデータの種類を「正確」に記録する必要があります。

面倒くさい。

これが、いわゆるモダンな3D APIの面倒くさいところです。
手動で種類を指定したり、細々とした作業が必要になってきます（しかも、いろいろなパラメータを正しく設定する責任はアプリケーション側にあります）。

GPUの性能を最大限に活かすために、人間は奴隷となって下働きしなくてはなりません。

頑張りましょう…

## 拡張機能
先ほどのコードでは、``VkInstanceCreateInfo`` に対して、``enabledExtensionCount``と``ppEnabledExtensionNames``を通して拡張機能を追加していました。
その中身を見てみましょう。

``glfwGetRequiredInstanceExtensions`` を使うと、現在使っているシステムが対応しているウィンドウシステムの拡張機能（エクステンション）を取得することができます。
引数には、返ってきた結果の個数を格納するための変数を指定し、返り値は結果文字列のポインタ配列になります。

```cpp:src/MyApplication.h 
	static std::vector<const char*> getRequiredExtensions()
	{
		// 拡張の個数を検出
		uint32_t glfwExtensionCount = 0;
		const char** glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

		std::vector<const char*> extensions(glfwExtensions, glfwExtensions + glfwExtensionCount);

		if (enableValidationLayers) {
			extensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
		}

		return extensions;
	}
```

なお、実際にどのような拡張機能が読みだされるのかは、試しに表示してみるのが良いでしょう。
下のようなコードで、拡張機能を列挙することができます。

```cpp:src/MyApplication.h 
#ifdef _DEBUG
		// 有効なエクステンションの表示
		std::cout << "available extensions:" << std::endl;
		for (const auto& extension : extensions) {
			std::cout << "\t" << extension << std::endl;
		}
#endif
```

これを、手元にあったRadeon RX 560 で実行してみると、次の結果が得られました。

![拡張機能の表示](4/extensions.png "拡張機能の表示")

全て``VK_``で始まっていることがわかります。``VK_KHR_``は、Vulkanの仕様を決めているKhronos Groupが標準化したものになります。
``VK_KHR_surface``は、クロスプラットフォームのサーフェス（画面）が使え、``VK_KHR_win32_surface``としてWindows用のサーフェスも使えることが読み取れます。

``VK_EXT_``は、まだ標準化まで至っていない機能となります。
``VK_EXT_debug_utils``ということで、デバッグ用の機能が追加されたシステムであることが読み取れます。


# 検証レイヤーの追加

# デバッグメッセージ表示

[dummy](URL)

![dummy](URL "comment")


* [戻る](./)

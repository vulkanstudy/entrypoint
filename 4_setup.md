# 初期化

少しづつVulkanのプログラムを作っていきましょう

今回のプログラムを入力して、[最終的に作られるコード](https://github.com/vulkanstudy/4_setup)はこちら。

# アプリケーションクラスの導入

メイン関数をシンプルにするために、アプリケーションクラスを導入していきましょう。

## メイン関数

アプリケーションクラスを導入することで、メイン関数はシンプルにできます。
具体的には、アプリケーションクラスのインスタンスを作成して、`run()` メソッドを呼び出します。

```src/main.cpp
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

```src/MyApplication.h 
#pragma once

class MyApplication
{
public:
	MyApplication();
	~MyApplication();

	void run();
};
```

## ウィンドウクラスの導入
Windows のアプリケーションは、画面の一部分の「窓」で実行されます（全画面かもしれませんが）。

GLFWは、OSのAPI呼び出しを大分サポートしてくれるので、かなり簡単にウィンドウを生成することができます。
この処理だけを追加してみましょう。

GLFWでは、GLFWwindowへのポインタとして、ウィンドウを管理することができます。
こちらは、アプリケーションクラスのメンバーとして持たせましょう。
後は、セッティングとしての「初期化」、「後片付け」のメソッドと、初期化が終わった後の基本ループである「通常処理」の呼び出しでアプリケーションを実行させ続けることができます。

```src/MyApplication.h 
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

では、それぞれのメソッドの中を見てみましょう。

GLFWは、``glfwInit()``関数で初期化して、``glfwCreateWindow()``関数でウィンドウを生成します。
glfwCreateWindowの返り値が、GLFWwindowのインスタンスへのポインタです。
引数は、ウィンドウ内の解像度とタイトル名などを指定します。``nullptr``になっているパラメータは、
後から登場するので、気にしないで進んでいきましょう。

ウィンドウ生成時のオプションは、``glfwWindowHint``で指定します。
今回は、Vulaknを使うので、OpenGL やOpenGL ES でない ``GLFW_NO_API``の指定と、
ユーザーによるウィンドウのサイズ変更を禁止する設定を行いました。
他のオプションは、GLFWの「Window guide」の[「Window creation hints」](https://www.glfw.org/docs/latest/window_guide.html#GLFW_CLIENT_API_hint)の項目を見てみてください。

```src/MyApplication.h 
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


```src/MyApplication.h 
	void finalizeWindow()
	{
		glfwDestroyWindow(window_);
		glfwTerminate();
	}
```

メインループは、GLFWが終わらないか確認しながら、イベントが起きる様なら(例えば、キーボード入力)それを処理していきます。

```src/MyApplication.h 
	// 通常の処理
	void mainloop()
	{
		while (!glfwWindowShouldClose(window_))
		{
			glfwPollEvents();
		}
	}
```



# 

[dummy](URL)

![dummy](URL "comment")


* [戻る](./)

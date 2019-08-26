# 物理デバイス

インスタンスを作成しました。これは、アプリケーションから見たVulkan全体のオブジェクトです。
次に描画するハードウェアに対応したオブジェクトの物理デバイスのオブジェクトを作ります。


物理デバイスは、GPUをイメージするのが良いでしょう。
最近では、CPUチップの中にGPUの機能が入っていたり、別にビデオカードを指したりと、一つのPCが複数のGPUを持つことがよくあります
（ハイエンドのゲーマーになると一つのPCに複数のGPUを指したりしますし、eGPUと呼ばれる外部の外部のビデオカードが入ったボックスにUSBで接続することで主にノートPCの性能を上げるシステムも見られます）。

![物理デバイスのイメージ](5/phys_device.png "物理デバイスのイメージ")

今回のプログラムを入力して、[最終的に作られるコード](https://github.com/vulkanstudy/5_device)はこちら。

# 物理デバイスのオブジェクト

物理デバイスは、``VkPhysicalDevice``としてオブジェクト化されます。
``VkPhysicalDevice``は、``VkInstance``を生成するときに内部で作られるので、開発者が生成する必要はありません。
しかし、複数の物理デバイスが存在するので、その中で適切なものを選ばなくてはなりません
（複数のGPUに指示を出す場合は、複数の物理デバイスを同時に制御します）。

ひとまず、一つだけ物理デバイスを使うことにして、物理デバイスのオブジェクトをアプリケーションに追加してみましょう。

```cpp:src/MyApplication.h 
class MyApplication
{
private:
	constexpr static char APP_NAME[] = "Vulkan Application";

	GLFWwindow* window_;
	VkInstance instance_;
	VkPhysicalDevice physicalDevice_ = VK_NULL_HANDLE;// ★追加
	VkDebugUtilsMessengerEXT debugMessenger_;// デバッグメッセージを伝えるオブジェクト
```

このオブジェクトは、``VkInstance``が生成されたら使うことができるので、Vulkanの初期化でオブジェクトを選択する処理を入れてみましょう。

```cpp:src/MyApplication.h 
	// Vulkanの設定
	void initializeVulkan()
	{
		createInstance(&instance_);
		initializeDebugMessenger(instance_, debugMessenger_);
		physicalDevice_ = pickPhysicalDevice(instance_);// ★追加
	}
```

# 物理デバイスの列挙

それでは、``pickPhysicalDevice``の中身を見ていきましょう。

デバイスを選ぶ方法はいくつか考えられます。
GPU名を取得できるので、その名前からユーザーが選ぶこともできますし、最初に見つかったものを使うという手もあるでしょう。
ここでは、GPUの性能よって得点をつけて、最も良い得点のGPUを使うことにしてみます。

物理デバイスを取ってくるのは``vkEnumeratePhysicalDevices``を使います。
他の同様の関数のように、保存先を与えないで実行すると、物理デバイスの数を取得することができるので、
最初に物理デバイスの数を数えて、もう一度、今度は保存先の配列を与えて呼び出して、実体をまとめて引っ張ってきます。

```cpp:src/MyApplication.h
	/*** デバイスの選択 ***/
	static VkPhysicalDevice pickPhysicalDevice(const VkInstance &instance) 
	{
		// デバイス数の取得
		uint32_t deviceCount = 0;
		vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
		if (deviceCount == 0) throw std::runtime_error("failed to find GPUs with Vulkan support!");

		// デバイスの取得
		std::vector<VkPhysicalDevice> devices(deviceCount);
		vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());

		// 適切なデバイスを選出(最高得点のデバイスを使用する)
		VkPhysicalDevice best_device = VK_NULL_HANDLE;
		int best_score = -1000000000;
		for (const auto& device : devices) {
			int score = rateDeviceSuitability(device);// 得点計算
			if (best_score < score) {
				best_device = device;
				best_score = score;
			}
		}

		// 使える物理デバイスがなければ大問題
		if(best_device == VK_NULL_HANDLE) throw std::runtime_error("failed to find a suitable GPU!");

		return best_device;
	}
```

さて、得点付けですが、物理デバイスから``VkPhysicalDeviceProperties``型の特性と、
``VkPhysicalDeviceProperties``型の機能の利用の有無を取得することができます。
これらの情報を基に、アプリケーションで必要な機能から使える物理デバイスを制限したり、
GPUの性能の高さを評価することができます。

ここでは、「そとつけGPUなら+1000」、「最大テクスチャ解像度を得点にそのまま加える」、
「再分割機能に対応していないと却下(コメントアウトで無効)」という条件で得点付けをしてみました。
この評価は、使用する機能で変わってくるので、実際にアプリケーションを作る際は、リリース前に再確認しましょう。

```cpp:src/MyApplication.h 
	static int rateDeviceSuitability(const VkPhysicalDevice device)
	{
		// デバイスに関する情報を取得
		VkPhysicalDeviceProperties deviceProperties;
		VkPhysicalDeviceProperties deviceFeatures;
		vkGetPhysicalDeviceProperties(device, &deviceProperties);
		vkGetPhysicalDeviceFeatures(device, &deviceFeatures);

		int score = 0;

		// 外付けGPUなら高評価
		if (deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) score += 1000;

		// 最大テクスチャ解像度を性能の評価値に加える
		score += deviceProperties.limits.maxImageDimension2D;

		// テッセレーションシェーダに対応していないと問題外(テッセセレーションを使う場合)
//		if (!deviceFeatures.tessellationShader) return 0;

#ifdef _DEBUG
		// デバイス名の表示
		std::cout << "Physical Device: " << deviceProperties.deviceName
			<< " (score: " << score << ")" << std::endl;
#endif // _DEBUG
		return score;
	}
```


```cpp:src/MyApplication.h 
```


* [戻る](./)

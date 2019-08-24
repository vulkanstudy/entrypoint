# 初期化

少しづつVulkanのプログラムを作っていきましょう

[今回のコード](https://github.com/vulkanstudy/4_setup)はこちら。

# アプリケーションクラスの導入

メイン関数をシンプルにするために、アプリケーションクラスを導入していきましょう。

[アプリケーションクラス導入までのコミット](https://github.com/vulkanstudy/4_setup/commit/624f416b68888fb60a83810db9a80e3d46f657f9)はこちら。

## メイン関数

アプリケーションクラスを導入することで、メイン関数はシンプルにできます。
例えば、

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

# 

[dummy](URL)

![dummy](URL "comment")


* [戻る](./)

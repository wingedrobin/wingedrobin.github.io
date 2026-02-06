+++
title = "AWS GameLift的Server Side SDK編譯"
date = "2017-07-24T01:50:07+08:00"
draft = false
tags = []
+++

由於最近工作可能會用到AWS的[GameLift](https://aws.amazon.com/tw/gamelift/)(用GameLift來管理&擴展Unreal的Game Server)，所以稍微研究了一下，然後先下載了它的[伺服器開發套件](https://aws.amazon.com/tw/gamelift/getting-started/)(2017/06/29號的版本)並試著Build出lib跟dll。

編譯過程當中遇到了一些問題，在這邊作個記錄，讓有碰到相同問題的人能夠有個參考方向。 \
(吐槽：你大AWS就不能把Binary檔先編好再放出來嗎？)

首先，將下載下來的壓縮檔解開後，會是像下圖一樣的資料夾結構：

![](https://lh3.googleusercontent.com/pw/AP1GczPqf8IhRXXWygCXyXuqiqX6MoknZma7ackiE5j5WyuIhZOJ6MOjV2XcEWWJDHF_vvPYnuLoLLxMjy-aIGAN5QNSjC6zeJ59LCA57gHDsoOPBr2Op4k)

因為是要用在Unreal上，所以先只看**GameLift-Cpp-ServerSDK-3.1.6**這個資料夾就好。參考官方[將Amazon GameLift添加到Unreal Engine遊戲伺服器項目](http://docs.aws.amazon.com/gamelift/latest/developerguide/integration-engines-setup-unreal.html)這篇。

首先，編譯的過程會用到下列工具，所以建議可以先行安裝，然後將安裝路徑加到環境變數裡：

1. cmake
2. msbuild
3. git

開啟console，並移動到GameLift-Cpp-ServerSDK-3.1.6資料夾下，接著根據上面那篇文件的內容，輸入下面幾行指令：

``` BAT
mkdir out
cd out
cmake -G "Visual Studio 14 2015 Win64" -DBUILD_FOR_UNREAL=1 ..
msbuild ALL_BUILD.vcxproj /p:Configuration=Release
```

然後在編譯過程中會發生下圖的錯誤

![](https://lh3.googleusercontent.com/pw/AP1GczMJRvBBwn1QxT8-CHjB1mzsnZEifI-3ApT7mT5ArJWHPS8iNTCa5PSPFba8DkiKWExYPW8IKXM-NnVxwnnbuyf9BhpbzsQ37kh8X_HX6hiYx4TTlH4)

訊息當中出現了**No CMAKE_C_COMPILER could be found.** 跟 **No CMAKE_CXX_COMPILER could be found.** 這兩句，所以我花了一些時間確認cmake的compiler要如何設定。但是不管怎麼調整，就算在cmake gui裡的Configure是正常的，但還是無法正確編譯。只好從錯誤訊息當中再找找有沒有其它更有幫助的資訊。結果發現了紅字的上面有一行：

{{< admonition type=info title="Info" open=true >}}
See also "C:/Users/User/Documents/UnrealProjects/ThirdPartyPlugin/Gamelift_06_29_2017/GameLift-SDK-Release-3.1.6/GameLift-Cpp-ServerSDK-3.1.6/out/thirdparty/protobuf-prefix/src/protobuf-build/CMakeFiles/CMakeError.log".
{{< /admonition >}}

該log的內容如下：

{{< admonition type=info title="Info" open=true >}}
Compiling the C compiler identification source file "CMakeCCompilerId.c" failed.
Compiler:  
Build flags: 
Id flags:  

The output was:
1
Microsoft (R) Build Engine version 14.0.25123.0
Copyright (C) Microsoft Corporation. 著作權所有，並保留一切權利。

已經開始建置於 2017/7/23 下午 12:55:16。
節點 1 (預設目標) 上的專案 "C:\Users\User\Documents\UnrealProjects\ThirdPartyPlugin\Gamelift_06_29_2017\GameLift-SDK-Release-3.1.6\GameLift-Cpp-ServerSDK-3.1.6\out\thirdparty\protobuf-prefix\src\protobuf-build\CMakeFiles\3.9.0\CompilerIdC\CompilerIdC.vcxproj"。
PrepareForBuild:
  正在建立目錄 "Debug\"。
  正在建立目錄 "Debug\CompilerIdC.tlog\"。
C:\Program Files (x86)\MSBuild\Microsoft.Cpp\v4.0\V140\Microsoft.CppBuild.targets(312,5): error MSB3491: 無法將行寫入檔案 "Debug\CompilerIdC.tlog\CompilerIdC.lastbuildstate"。指定的路徑、檔名，或是兩者都太長。完整的檔名必須少於 260 個字元，並且目錄名稱必須少於 248 個字元。 [C:\Users\User\Documents\UnrealProjects\ThirdPartyPlugin\Gamelift_06_29_2017\GameLift-SDK-Release-3.1.6\GameLift-Cpp-ServerSDK-3.1.6\out\thirdparty\protobuf-prefix\src\protobuf-build\CMakeFiles\3.9.0\CompilerIdC\CompilerIdC.vcxproj]
專案 "C:\Users\User\Documents\UnrealProjects\ThirdPartyPlugin\Gamelift_06_29_2017\GameLift-SDK-Release-3.1.6\GameLift-Cpp-ServerSDK-3.1.6\out\thirdparty\protobuf-prefix\src\protobuf-build\CMakeFiles\3.9.0\CompilerIdC\CompilerIdC.vcxproj" (預設目標) 建置完成 -- 失敗。

建置失敗。

"C:\Users\User\Documents\UnrealProjects\ThirdPartyPlugin\Gamelift_06_29_2017\GameLift-SDK-Release-3.1.6\GameLift-Cpp-ServerSDK-3.1.6\out\thirdparty\protobuf-prefix\src\protobuf-build\CMakeFiles\3.9.0\CompilerIdC\CompilerIdC.vcxproj" (預設目標) (1) ->
(InitializeBuildStatus 目標) -> 
  C:\Program Files (x86)\MSBuild\Microsoft.Cpp\v4.0\V140\Microsoft.CppBuild.targets(312,5): error MSB3491: 無法將行寫入檔案 "Debug\CompilerIdC.tlog\CompilerIdC.lastbuildstate"。指定的路徑、檔名，或是兩者都太長。完整的檔名必須少於 260 個字元，並且目錄名稱必須少於 248 個字元。 [C:\Users\User\Documents\UnrealProjects\ThirdPartyPlugin\Gamelift_06_29_2017\GameLift-SDK-Release-3.1.6\GameLift-Cpp-ServerSDK-3.1.6\out\thirdparty\protobuf-prefix\src\protobuf-build\CMakeFiles\3.9.0\CompilerIdC\CompilerIdC.vcxproj]

    0 個警告
    1 個錯誤

經過時間 00:00:00.15


Compiling the C compiler identification source file "CMakeCCompilerId.c" failed.
Compiler:  
Build flags: 
Id flags:  

The output was:
1
Microsoft (R) Build Engine version 14.0.25123.0
Copyright (C) Microsoft Corporation. 著作權所有，並保留一切權利。

已經開始建置於 2017/7/23 下午 12:55:16。
節點 1 (預設目標) 上的專案 "C:\Users\User\Documents\UnrealProjects\ThirdPartyPlugin\Gamelift_06_29_2017\GameLift-SDK-Release-3.1.6\GameLift-Cpp-ServerSDK-3.1.6\out\thirdparty\protobuf-prefix\src\protobuf-build\CMakeFiles\3.9.0\CompilerIdC\CompilerIdC.vcxproj"。
PrepareForBuild:
  正在建立目錄 "Debug\"。
  正在建立目錄 "Debug\CompilerIdC.tlog\"。
C:\Program Files (x86)\MSBuild\Microsoft.Cpp\v4.0\V140\Microsoft.CppBuild.targets(312,5): error MSB3491: 無法將行寫入檔案 "Debug\CompilerIdC.tlog\CompilerIdC.lastbuildstate"。指定的路徑、檔名，或是兩者都太長。完整的檔名必須少於 260 個字元，並且目錄名稱必須少於 248 個字元。 [C:\Users\User\Documents\UnrealProjects\ThirdPartyPlugin\Gamelift_06_29_2017\GameLift-SDK-Release-3.1.6\GameLift-Cpp-ServerSDK-3.1.6\out\thirdparty\protobuf-prefix\src\protobuf-build\CMakeFiles\3.9.0\CompilerIdC\CompilerIdC.vcxproj]
專案 "C:\Users\User\Documents\UnrealProjects\ThirdPartyPlugin\Gamelift_06_29_2017\GameLift-SDK-Release-3.1.6\GameLift-Cpp-ServerSDK-3.1.6\out\thirdparty\protobuf-prefix\src\protobuf-build\CMakeFiles\3.9.0\CompilerIdC\CompilerIdC.vcxproj" (預設目標) 建置完成 -- 失敗。

建置失敗。

"C:\Users\User\Documents\UnrealProjects\ThirdPartyPlugin\Gamelift_06_29_2017\GameLift-SDK-Release-3.1.6\GameLift-Cpp-ServerSDK-3.1.6\out\thirdparty\protobuf-prefix\src\protobuf-build\CMakeFiles\3.9.0\CompilerIdC\CompilerIdC.vcxproj" (預設目標) (1) ->
(InitializeBuildStatus 目標) -> 
  C:\Program Files (x86)\MSBuild\Microsoft.Cpp\v4.0\V140\Microsoft.CppBuild.targets(312,5): error MSB3491: 無法將行寫入檔案 "Debug\CompilerIdC.tlog\CompilerIdC.lastbuildstate"。指定的路徑、檔名，或是兩者都太長。完整的檔名必須少於 260 個字元，並且目錄名稱必須少於 248 個字元。 [C:\Users\User\Documents\UnrealProjects\ThirdPartyPlugin\Gamelift_06_29_2017\GameLift-SDK-Release-3.1.6\GameLift-Cpp-ServerSDK-3.1.6\out\thirdparty\protobuf-prefix\src\protobuf-build\CMakeFiles\3.9.0\CompilerIdC\CompilerIdC.vcxproj]

    0 個警告
    1 個錯誤

經過時間 00:00:00.14
{{< /admonition >}}

裡面寫到**error MSB3491: 無法將行寫入檔案 “Debug\CompilerIdC.tlog\CompilerIdC.lastbuildstate”。指定的路徑、檔名，或是兩者都太長。完整的檔名必須少於 260 個字元，並且目錄名稱必須少於 248 個字元。**，於是我試著將整個資料夾移動到*C:*底下後再編譯一次，路徑、檔名太長的Error message都沒有再出現，看起來應該是解決了，但是再繼續編譯時又出現了不同的錯誤，如下圖：

![](https://lh3.googleusercontent.com/pw/AP1GczPGGu78LmpyRu-2ma0F0-gZX1OPVthe_F_S9_a5dmq5Yb8u2p2mFdES5Gl_PhpP_M5fuz9zS-2cLP09sR1gcC3tkCaZjMc-BVZOygCTl3GhM_Bcfg0)

Google了一下，在GitHub上找到一篇相關的[問題](https://github.com/google/googletest/issues/1115)，依照該issue討論的[內容](https://github.com/google/googletest/commit/c2d90bddc6a2a562ee7750c14351e9ca16a6a37a)修改**178**行的減號後，再重新編譯一次，就能順利完成GameLift Server端SDK的編譯。

最後還是要再吐槽一次，AWS 你把它編好再放出來不難吧？

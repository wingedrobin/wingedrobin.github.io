+++
title = "社群版UE4.27-HTML5-ES3在Ubuntu上建置與打包的問題及處理"
date = "2026-01-19T16:32:52+08:00"
lastmod = "2026-01-21T14:07:58+08:00"
draft = false
tags = ["UE4", "HTML5", "Ubuntu"]
+++

由於在官方版本4.23後中止維護HTML5功能，因此如果需要4.24以後的功能或修正，只能想辦法自行實做或繞過。\
經過考量之後，決定使用社群版引擎，以下紀錄在引擎編譯及專案打包過程中發生的問題與其解決方式。

> 系統版本： Ubuntu 24.04 LTS及Xubuntu 25.10 \
> 測試專案： 2D Platform樣板專案
>
> 注意： 以下內容僅能確保引擎建置及專案打包可以成功，不保證所有引擎功能皆正確無誤。

# 下載引擎

社群版引擎原始碼Repo可由此[**README**](https://github.com/SpeculativeCoder/UnrealEngine-HTML5-ES3)中找到。

# 執行Setup.sh

{{< admonition type=failure title="錯誤訊息" open=true >}}
Failed to download `'https://cdn.unrealengine.com/dependencies/UnrealEngine-43104418/010ea6faee2d21055f0ebd1d2ee7a201041d913c'`: Error: SecureChannelFailure (The authentication or decryption has failed.) (WebException)
{{< /admonition >}}

解決方式：

根據官方的[**說明**](https://forums.unrealengine.com/t/upcoming-disruption-of-service-impacting-unreal-engine-users-on-github/1155880)，需要從官方Repo中的[**Releases**](https://github.com/EpicGames/UnrealEngine/releases/tag/4.27.2-release)下載修改過的Commit.gitdeps.xml，並覆蓋 */Engine/Build/* 路徑下的檔案。

# 執行HTML5Setup.sh

(腳本位於 */Engine/Platforms/HTML5/*)

[註] 此步驟與官方版不同，[**官方版建置流程**](https://dev.epicgames.com/documentation/en-us/unreal-engine/linux-quick-start?application_version=4.27#3-buildingue4onlinux)在執行Setup腳本後，應為執行GenerateProjectFiles腳本。

{{< admonition type=failure title="錯誤訊息" open=true >}}
bash: ./HTML5Setup.sh: Permission denied
{{< /admonition >}}

解決方式：

使用chmod或透過界面修改，以增加執行權限。

``` BASH
chmod +x HTML5Setup.sh
```

---

{{< admonition type=failure title="錯誤訊息" open=true >}}
/HTML5Setup.sh: line 24: cmake: command not found
{{< /admonition >}}

解決方式：

安裝cmake。\
原本在執行GenerateProjectFiles.sh的時候會安裝cmake，但由於社群版引擎的建置流程在Setup與GenerateProjectFiles之間還多了一個HTML5Setup，因此需要事先安裝cmake以避免錯誤。

---

{{< admonition type=failure title="錯誤訊息" open=true >}}
./HTML5Setup.sh: line 33: python: command not found
{{< /admonition >}}

由於Ubuntu已內建python3，所以安裝python-is-python3即可。

---

{{< admonition type=failure title="錯誤訊息" open=true >}}
./HTML5Setup.sh: line 388: ./Build_All_HTML5_libs.sh: Permission denied \
./Build_All_HTML5_libs.sh: line 93: ./build_html5_zlib.sh: Permission denied \
./Build_All_HTML5_libs.sh: line 94: ./build_html5_libPNG.sh: Permission denied \
./Build_All_HTML5_libs.sh: line 96: ./build_html5_Ogg.sh: Permission denied \
./Build_All_HTML5_libs.sh: line 97: ./build_html5_Vorbis.sh: Permission denied \
./Build_All_HTML5_libs.sh: line 98: ./build_html5_libOpus.sh: Permission denied \
./Build_All_HTML5_libs.sh: line 101: ./build_html5_ICU.sh: Permission denied \
./Build_All_HTML5_libs.sh: line 103: ./build_html5_HarfBuzz.sh: Permission denied \
./Build_All_HTML5_libs.sh: line 105: ./build_html5_FreeType2.sh: Permission denied \
./Build_All_HTML5_libs.sh: line 112: ./build_html5_PhysX3.sh: Permission denied
{{< /admonition >}}

Build_All_HTML5_libs腳本位於 */Engine/Platforms/HTML5/Build/BatchFiles/*，其餘腳本則位於下一層的 */ThirdParty/*。

解決方式：

可以將上述列出的腳本個別增加執行權限，或者使用下面的指令批次把特定目錄階層下的所有腳本(包含但不限於上述)一併處理，是否採用可自行斟酌。

``` BASH
find [PATH/TO/YOUR/SEARCH/DIR] -name "*.sh" -exec chmod +x {} +
```

---

此外，其中build_html5_Ogg.sh除了權限問題，還會碰到下面問題，需要調整其腳本內容。

錯誤訊息：

{{< admonition type=failure title="錯誤訊息" open=true >}}
\+ EMFLAGS=\
\+++ which emcc.py\
\++ dirname\
dirname: missing operand\
Try 'dirname --help' for more information.\
\+ EMPATH=
{{< /admonition >}}

原因為透過which在環境變數$PATH的目錄中無法找到emcc.py，因此將腳本調整為if-else結構，處理搜尋結果為空的狀況。

解決方式：

將39行開始修改如下。

``` BASH
if which emcc.py > /dev/null 2>&1; then
	EMPATH=$(dirname `which emcc.py`)
else
	EMPATH="$EMSDK/upstream/emscripten"
fi
export CFLAGS="-I\"$EMPATH/system/lib/libc\"" # 1.37.36 needs this...
```

[註] 由於腳本執行過程中，會編譯*/Engine/Platforms/HTML5/Build/BatchFiles/ThirdParty/*下的第三方函式庫，建議將上述所有調整一併處理之後再執行HTML5Setup腳本，以節省處理錯誤時的來回次數。

# 執行 GenerateProjectFiles.sh

{{< admonition type=failure title="錯誤訊息" open=true >}}
WARNING: Library '[/PATH/TO/YOUR/ENGINE]/Engine/Plugins/Media/BinkMedia/Source/BinkMediaPlayerSDK/lib/BinkUnrealLinux.a' was not resolvable to a file when used in Module 'BinkMediaPlayerSDK', assuming it is a filename and will search library paths for it. This is slow and dependency checking will not work for it. Please update reference to be fully qualified alternatively use PublicSystemLibraryPaths if you do intended to use this slow path to suppress this warning.
{{< /admonition >}}

(如果確定專案不會用到Bink，那麼忽略這個警告應該不會造成影響)

解決方式：

修改`/Engine/Plugins/Media/BinkMedia/Source/BinkMediaPlayerSDK/BinkMediaPlayerSDK.Build.cs`的第15行。

``` C#
protected virtual string LibDirectory { get { return Path.Combine(Path.GetDirectoryName(SdkBaseDirectory), "SDK/lib"); } }
```

# **make** (編譯引擎)

{{< admonition type=failure title="錯誤訊息" open=true >}}
In file included from [/PATH/TO/YOUR/ENGINE]/Engine/Intermediate/Build/Linux/B4D820EA/UnrealFrontend/Development/HTML5/HTML5TargetPlatform/Module.HTML5TargetPlatform.cpp:4: \
[/PATH/TO/YOUR/ENGINE]/Engine/Platforms/HTML5/Source/Developer/HTML5TargetPlatform/Private/HTML5TargetPlatformModule.cpp:10:9: error: 'LOCTEXT_NAMESPACE' macro redefined \
[-Werror,-Wmacro-redefined] \
#define LOCTEXT_NAMESPACE "FHTML5TargetPlatformModule" \
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ^ \
[/PATH/TO/YOUR/ENGINE]/Engine/Platforms/HTML5/Source/Developer/HTML5TargetPlatform/Private/HTML5TargetPlatform.h:21:9: note: previous definition is here \
#define LOCTEXT_NAMESPACE "FHTML5TargetPlatform" \
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ^ \
1 error generated.
{{< /admonition >}}

解決方式：

將`/Engine/Platforms/HTML5/Source/Developer/HTML5TargetPlatform/Private/HTML5TargetPlatformModule.cpp`的第10行給刪除(或註解掉)。

此階段只有碰到這個錯誤，解決之後應能順利將引擎編譯成功。

# 專案打包時期


{{< admonition type=failure title="錯誤訊息" open=true >}}
UATHelper: Packaging (HTML5):   [/PATH/TO/YOUR/ENGINE]/Engine/Binaries/ThirdParty/Python/Linux/bin/python2.7: error while loading shared libraries: libpython2.7.so.1.0: cannot open shared object file: No such file or directory
{{< /admonition >}}

因為後續打包流程會使用到emsdk-4.0.22，而emsdk-4.0.22會用到python3.10以上版本的功能，即便把找不到shared libraries libpython2.7.so.1.0的問題給解決掉，也沒有辦法正常完成打包流程，所以直接修改為使用python3。

解決方式：

修改`/Engine/Platforms/HTML5/Source/Programs/UnrealBuildTool/HTML5SDKInfo.cs`。

將第194到197行修改如下，即可。\
(第204到207行是否刪除或註解掉應無影響)

``` C#
if (/* BuildHostPlatform.Current.Platform == UnrealTargetPlatform.Mac || */ BuildHostPlatform.Current.Platform == UnrealTargetPlatform.Linux)
{	// TODO: find python3
	return "/usr/bin/python";
}
// use UE4's bundled python executable
string UE4PythonPath = FileReference.Combine(UnrealBuildTool.EngineDirectory, "Binaries", "ThirdParty", "Python").FullName;
if (BuildHostPlatform.Current.Platform == UnrealTargetPlatform.Mac)
{
	return Path.Combine(UE4PythonPath, "Mac", "bin", "python2.7");
}
// if (BuildHostPlatform.Current.Platform == UnrealTargetPlatform.Linux)
// {
// 	return Path.Combine(UE4PythonPath, "Linux", "bin", "python3");
// }
```

經過以上修改之後，即可成功建置社群版引擎，並且可以正確打包HTML5的遊戲了。

---

# 參考資料

* [社群版引擎說明文件](https://github.com/SpeculativeCoder/UnrealEngine-HTML5-ES3)
* [Command ‘python’ not found！解決 python 是 python3](https://dsalearning.github.io/linux/linux-xrdp-install/)
* [UE4官方文件](https://dev.epicgames.com/documentation/en-us/unreal-engine/linux-quick-start?application_version=4.27#3-buildingue4onlinux)

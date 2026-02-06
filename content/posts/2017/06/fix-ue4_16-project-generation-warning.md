+++
title = "解決UE由4.15升級到16後產生專案檔時出現的警告訊息"
date = "2017-06-04T21:13:48+08:00"
draft = false
tags = ["UE4"]
+++

當引擎升級到 4.16 之後，在將 .uproject 切換到新版本、或重新產生專案檔時會出現下列的警告訊息：

{{< admonition type=warning title="Warning" open=true >}}
*[PATH\TO\PROJECT\PROJECT_NAME]*\Source\\*[PROJECT_NAME]*\\*[PROJECT_NAME]*.Build.cs: warning: Module constructors should take a ReadOnlyTargetRules argument (rather than a TargetInfo argument) and pass it to the base class constructor from 4.15 onwards. Please update the method signature.

*[PATH\TO\PROJECT\PROJECT_NAME]*\Source\\*[PROJECT_NAME]*.Target.cs: warning: SetupBinaries() is deprecated in the 4.16 release. From the constructor in your .target.cs file, use ExtraModuleNames.Add(“Foo”) to add modules to your target, or set LaunchModuleName = “Foo” to override the name of the launch module for program targets.

*[PATH\TO\PROJECT\PROJECT_NAME]*\Source\\*[PROJECT_NAME]*Editor.Target.cs: warning: SetupBinaries() is deprecated in the 4.16 release. From the constructor in your .target.cs file, use ExtraModuleNames.Add(“Foo”) to add modules to your target, or set LaunchModuleName = “Foo” to override the name of the launch module for programtargets.
{{< /admonition >}}

解決方式：

{{< admonition type=tip title="[PROJECT_NAME].build.cs" open=true >}}
將函式 `public PROJECT_NAME(TargetInfo Target)`
修改為 `public PROJECT_NAME(ReadOnlyTargetRules Target) : base (Target)` 
{{< /admonition >}}

{{< admonition type=tip title="[PROJECT_NAME]Editor.Target.cs" open=true >}}
將函式 `public PROJECT_NAMEEditorTarget(TargetInfo Target)`
修改為 `public PROJECT_NAMEEditorTarget(TargetInfo Target) : base(Target)` \
並且在 `Type = TargetType.Editor;` 的下一行加入 `ExtraModuleNames.Add("PROJECT_NAME");` \
再刪掉 `public override void SetupBinaries` 整個函式
{{< /admonition >}}

{{< admonition type=tip title="[PROJECT_NAME].Target.cs" open=true >}}
與 **PROJECT_NAME**Editor.Target.cs 的修改內容相同
將函式 `public PROJECT_NAMETarget(TargetInfo Target)`
修改為 `public PROJECT_NAMETarget(TargetInfo Target) : base(Target)` \
並且在 `Type = TargetType.Editor;` 下一行加入 `ExtraModuleNames.Add("PROJECT_NAME");` \
最後刪掉 `public override void SetupBinaries` 整個函式
{{< /admonition >}}

Reference
* [C++ 4.16 Transition Guide](https://forums.unrealengine.com/showthread.php?145757-C-4-16-Transition-Guide)


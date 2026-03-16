+++
title = "不著痕跡收集UE專案的PSO快取"
date = "2026-03-13T17:49:43+08:00"
lastmod = "2026-03-16T20:49:27+08:00"
draft = false
tags = ["UE4", "UE5"]
+++

# 緣由

24年的10月，我們在新品節期間發布[咒](https://store.steampowered.com/app/2328540/)的Demo版後，發現即便玩家有高於建議配備的硬體，仍然會出現lag的狀況，與團隊內部效能測試時的反饋不同。經過同事的提點，才認識PSO(Pipeline State Object)這個東西，將其導入遊戲後也確實解決了lag的問題。

至於這篇文章並不是在講述UE跟PSO的內容，而是分享我在實務中如何簡化團隊內部測試時收集PSO cache的手法，後續也繼續沿用在[伊藤潤二狂熱:無盡的囹圄](https://store.steampowered.com/app/3633250/)專案上。

[註1] 本篇文章內容以Windows環境為主。 \
[註2] 在製作咒的時候是使用UE4.27，當時引擎尚未提供[PSO Precaching](https://dev.epicgames.com/documentation/en-us/unreal-engine/pso-precaching-for-unreal-engine)機制；而伊藤案則是使用UE5.3，但由於當時專案時程緊迫，並無多餘時間研究該功能，或許這個新機制能夠更好的解決PSO在收集上的一些問題。不過至少在UE5.3(含)以前，引擎原本提供的PSO收集方式與這篇文章提出的手法都還適用。

# 詞彙說明

| 詞彙 | 說明 |
| --- | --- |
| Bootstrap Executable | 打包後遊戲根目錄的.exe檔，名稱為*PROJECT_NAME*.exe |
| Main Game Executable | 打包後位於遊戲目錄*PROJECT_NAME\Binaries\Win64\\* 下的.exe檔 |

# PSO Caching啟用方式選擇

引擎提供的PSO收集方式有兩種方式(建議先了解官方設計的[流程](https://dev.epicgames.com/documentation/en-us/unreal-engine/manually-creating-bundled-pso-caches-in-unreal-engine#collectionflow))：

1. 透過config
{{< admonition type=info title="Config/DefaultEngine.ini" open=true >}}
[ConsoleVariables] \
r.ShaderPipelineCache.Enabled=1 \
r.ShaderPipelineCache.LogPSO=1 \
r.ShaderPipelineCache.SaveBoundPSOLog=1
{{< /admonition >}}

2. 或是在啟動遊戲時，帶入-logpso參數

因為我希望之後打包的Shipping版可以同時用在內部測試並推上Steam，免去因為修改ini後重新需要重新打包版本的狀況，所以選擇使用啟動遊戲時帶入參數的方式。

# 痛點

如下面的流程圖所示，我們在測試遊戲版本的原始流程是由NAS上複製最新的版本後，在本地打開遊戲資料夾，執行bootstrap executable以進行遊戲測試。

但現在為了收集測試時產生的PSO cache，需要讓同仁們在每次測試打包版時，

1. 用terminal開啟遊戲並帶入參數，或是建立捷徑後在屬性加入參數。
2. 測試結束後再由同仁自行把PSO cache檔(.upipelinecache)上傳到NAS[^1]。

這些行為與原本熟悉的測試流程不同，即便寫成教學文件或是反覆口頭提醒，還是可以預期一定有人會忘記，所以不是太合適的方法(或者說這種方案的執行成本過高)，需要一個能夠在維持現有測試流程下、又能完成收集PSO的方案。

{{< mermaid >}}
flowchart LR
    begin --> A[由NAS複製新版本到本地]
    A --> |原始流程| B[透過bootstrap executable執行遊戲]
    A --> |terminal| C[透過terminal執行遊戲並帶入-logpso參數]
    A --> |shortcut| D[為bootstrap executable建立捷徑並在其屬性內容加入-logpso參數]
    B --> E[進行遊戲]
    C --> E
    D --> E
    E --> F[結束測試]
    F --> |原始流程| stop
    F --> G[將PSO cache上傳到NAS]
    G --> stop
    style C fill:#f9f,color:#fff
    style D fill:#f9f,color:#fff
    style G fill:#f9f,color:#fff
{{< /mermaid >}}

# 簡化手法

## python腳本

由一隻獨立的python腳本啟動遊戲，同時帶入`-logpso`參數，並在遊戲結束後把產生的pso檔案上傳到NAS。 \
演算法大致如下：

1. 找出*PROJECT_NAME\Binaries\Win64*路徑下的main game executable
2. 檢查main game executable的路徑中是否帶有**Shipping**字串，如果有，則為Shipping版，以一個布林值做紀錄。
3. 用os.system加上main game executable的路徑以執行遊戲，同時帶入`-logpso`參數(如果需要指定產出的Shader Model版本，可以再加上`-sm5`或`-sm6`參數)
4. 遊戲結束後腳本繼續往下執行
5. 檢查NAS能否連接，如果不能，則結束腳本
6. 檢查**CollectedPSOs資料夾**[^2]是否存在以及是否有產生PSO cache檔案，如果為否，則結束腳本
7. 檢查NAS上是否有存放PSO cache的資料夾，沒有的話就建立一個
8. 把本地剛才生成的所有PSO cache上傳到NAS上
9. 刪除本地PSO cache，避免之後測試相同版本後會再次上傳相同檔案
10. 結束python程式

{{< mermaid >}}
flowchart TD
    begin --> A[找到main game executable]
    A --> B[判斷是否為Shipping版]
    B --> C[執行遊戲並帶入參數]
    C --> |遊戲結束後| D[檢查NAS能否連線]
    D --> |Valid| E[檢查是否產生PSO cache]
    D --> |Invalid| stop
    E --> |Yes| F[上傳PSO cache到NAS]
    E --> |No| stop
    F --> G[刪除本地PSO cache]
    G --> stop
{{< /mermaid >}}

## 腳本用法

使用PyInstaller將腳本打包成exe檔，並換上與遊戲相同的執行檔圖示(.ico)，偽裝成原本遊戲的bootstrap executable。

此後，每次打包完成後都用它替換掉原本的bootstrap executable，再將遊戲放到NAS上，給團隊成員下載、測試即可。如此就可以讓其他人以原來熟悉的方式測試遊戲，不需要關心或擔心是否遺漏掉什麼步驟。

而把PSO cache統一放在NAS上，也便於打包流程一併將所有的PSO cache包進最新的版本內。

## 簡化效果比較

| | 原始測試流程 | PSO cache收集 + 未簡化 | PSO cache收集 + 簡化後 |
| :---: | :---: | :---: | :---: |
| 啟動遊戲 | 執行bootstrap executable | 需透過terminal或建立捷徑手動輸入參數 | 執行偽裝的bootstrap executable |
| PSO cache收集 | - | 測試後，需手動找到檔案並上傳至NAS | 遊戲關閉後自動上傳NAS |
| 風險 | - | 高(容易發生忘記加參數或忘記上傳) | 零 |

# 後記

這篇的內容不是什麼很正規的方法，畢竟原本的bootstrap executable有其作用(可參考reference)，不過當時在公司內部並沒有出現什麼狀況，也能解決PSO收集上的痛點，所以就連續用在兩個專案上了，如果讀者有什麼更適合的方案也歡迎留言討論。

我自己把這個手法稱作『**輕薄的假象**』，雖然其實一點也不輕薄(原本的Bootstrap executable大約是2xx~3xxKB左右，而python腳本包裝成的exe大約是6MB左右)。經過兩個專案的試驗，沒有任何團隊同仁主動向我回報發現執行檔大小有異狀，應該可以算是偷渡成功了吧？

# Reference

* [[Python] 使用PyInstaller將程式碼轉換成執行檔](https://vocus.cc/article/64dbaa1afd897800014214cb)
* [Why does packaging my project result in 2 builds?](https://forums.unrealengine.com/t/why-does-packaging-my-project-result-in-2-builds/2230088/2)
* [Difference between the two .exe files once packaged?](https://forums.unrealengine.com/t/difference-between-the-two-exe-files-once-packaged/396720/2)
* [Change the details of a game’s shipping executable](https://forums.unrealengine.com/t/change-the-details-of-a-games-shipping-executable/455748/12)
* [Unreal – BootstrapPackagedGame has dual EXEs after packaging](https://www.myredstone.top/en/archives/2761)

[^1]: 需要把每個人測試產生的upipelinecache檔收集起來，重新打包進新的版本，才會有最新的PSO的效果。
[^2]: 該資料夾會在有啟用PSO caching的遊戲被測試後才建立，並將.upipelinecache檔置於此處。Shipping版路徑為 *C:\Users\USER_NAME\AppData\Local\PROJECT_NAME\Saved\CollectedPSOs*，而Development版則是 *PROJECT_NAME\Saved\CollectedPSOs*

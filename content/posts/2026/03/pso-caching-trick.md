+++
title = "Pso Caching Trick"
date = "2026-03-13T17:49:43+08:00"
draft = true
tags = ["UE4", "UE5"]
+++

# 緣由

24年的10月，我們在新品節期間發布[咒](https://store.steampowered.com/app/2328540/)的Demo版後，發現即便玩家有高於建議配備的硬體，仍然會出現lag的狀況，與團隊內部效能測試的反饋不同。經過同事的提點，才認識到PSO(Pipeline State Object)這個東西，後續將其導入遊戲後也確實解決了lag的問題。

至於這篇文章並不是在講述UE跟PSO的內容，而是分享我在實務中如何簡化團隊內部測試時收集PSO的手法，後續也有繼續沿用[伊藤潤二狂熱:無盡的囹圄](https://store.steampowered.com/app/3633250/)專案上。

[註1] 咒在製作的時候是使用UE4.27，當時引擎尚未提供[PSO Precaching](https://dev.epicgames.com/documentation/en-us/unreal-engine/pso-precaching-for-unreal-engine)機制；而伊藤則是使用UE5.3，但因為當時專案時程緊迫，並無多餘時間研究該功能，或許它能夠更好的解決PSO在收集上的一些問題。不過這篇文章提出的方案與引擎提供的PSO收集方式至少在UE5.3以前都還適用。 \
[註2] 本篇文章內容以Windows環境為主。

# 用語說明

| 用語 | 說明 |
| --- | --- |
| Bootstrap Executable | 打包後遊戲根目錄的.exe檔，名稱為*PROJECT_NAME*.exe |
| Main Game Executable | 打包後位於遊戲目錄*PROJECT_NAME*\Binaries\Win64\下的.exe檔 |

# PSO Caching流程選擇

引擎提供的PSO收集方式有兩種方式(建議先了解官方設計的[流程](https://dev.epicgames.com/documentation/en-us/unreal-engine/manually-creating-bundled-pso-caches-in-unreal-engine#collectionflow))：

1. 透過config
{{< admonition type=info title="Config/DefaultEngine.ini" open=true >}}
[ConsoleVariables] \
r.ShaderPipelineCache.Enabled=1 \
r.ShaderPipelineCache.LogPSO=1 \
r.ShaderPipelineCache.SaveBoundPSOLog=1
{{< /admonition >}}

2. 或是在啟動遊戲時，帶入-logpso參數

CollectedPSOs在啟用、測試後會產生該資料夾，.upipelinecache會置於此處 (dev/shipping路徑不同)

遊戲測試結束後.upipelinecache檔會寫入到(shipping版在`C:\Users\USER_NAME\AppData\Local\PROJECT_NAME\Saved\CollectedPSOs`，而Development版在`PROJECT_NAME\Saved\CollectedPSOs`)
需要把每個人測試產生的upipelinecache檔收集起來，重新打包進新的版本，才會有最新的PSO的效果。


因為我希望之後打包的Shipping版可以同時用在內部測試及推上Steam，不需要修改ini後重新打包一個版本，所以選擇使用啟動遊戲時帶入參數的方式。

# 痛點

讓團隊成員每次測試打包版都用console開啟遊戲並帶入參數，或是建立捷徑後在屬性加入參數，這兩種方式都不可行；測試結束後再自行把pipeline cache檔上傳到NAS，這個也不可行，因為與原本習慣的測試流程不同，需要事先教育、提醒，但一定會有人忘記，所以不可行，需要找其他方式進行。

前
透過terminal啟動遊戲並代入參數。或是建立bootstrap executable的捷徑，並對捷徑的內容屬性加入參數。
後
測試結束後由成員自行把pipeline cache檔上傳到NAS。

都與原本熟悉的測試流程不同，即便寫成教學文件、提醒，可以預期一定還是會有人忘記，所以不是可行的方法，需要一個能夠在維持現有測試流程下、又能完成收集PSO的方案。

{{< mermaid >}}
flowchart LR
    begin --> |原始流程| A[透過bootstrap executable執行遊戲]
    begin --> |terminal| B[terminal]
    begin --> |shortcut| C[shortcut]
    A --> D[進行遊戲]
    B --> D
    C --> D
    D --> E[結束測試]
    E --> |原始流程| stop
    E --> F[pipeline cache]
    F --> stop
    style B fill:#f9f,color:#fff
    style C fill:#f9f,color:#fff
    style F fill:#f9f,color:#fff
{{< /mermaid >}}

輸入指令、建立捷徑

# 簡化手法

## python腳本

由一隻獨立的python腳本啟動遊戲，同時代入`-logpso`參數，並在遊戲結束後把產生的pso檔案上傳到NAS。 \
演算法大致如下：

常數定義
* 專案名稱
* pipeline cache檔副檔名 `.upipelinecache`
* 遠端NAS資料夾路徑
* 執行檔路徑 `PROJECT_NAME\Binaries\Win64`
* 上傳路徑

1. 找出*PROJECT_NAME\Binaries\Win64*路徑下的main game executable
2. 檢查main game executable的路徑中是否帶有`Shipping`字串，如果有，則為Shipping版，以一個布林值紀錄。
3. 用os.system加上main game executable的路徑以執行遊戲，並帶入`-logpso`參數(如果需要指定產出的Shader Model版本，可以再加上`-sm5`或`-sm6`參數)
4. 遊戲結束後腳本繼續往下執行
5. 檢查NAS能否連接，如果不能，則結束腳本
6. 檢查**CollectedPSOs資料夾**[^1]是否存在，如果不存在，則結束腳本
7. 由*Saved\CollectedPSOs*中取得PSO cache，如果沒有PSO cache檔案，則結束腳本
8. 檢查NAS上是否有存放pipeline cache的資料夾，沒有的話就建立一個
9. 把本地剛才生成的所有pipeline cache上傳到NAS上
10. 刪除本地pipeline cache，避免之後重複上傳相同檔案
11. 結束python程式

## 腳本用法

使用PyInstaller將腳本打包成exe檔，並換上與遊戲相同的執行檔圖示(.ico)，偽裝成原本遊戲的bootstrap executable。

此後，每次打包完成後都用它替換掉原本的bootstrap executable，再將遊戲放到NAS上，給團隊成員下載、測試即可。 \
如此就可以讓其他人以原來熟悉的方式測試遊戲，不需要關心或擔心是否有漏掉什麼步驟。

而把PSO cache統一放在NAS上，也便於打包流程一併將所有的PSO cache包進最新的版本內。

# 後記

這篇的內容不是什麼很正規的方法，畢竟原本的bootstrap executable有其作用(可參考reference)，不過當時在公司內部並沒有出現什麼狀況，也能解決PSO收集上的痛點，所以就連續用在兩個專案上了。 \
如果讀者有什麼更適合的方案也歡迎留言討論。

[註] 我自己把這個手法稱作『輕薄的假象』，雖然其實一點也不輕薄(原本的Bootstrap executable大約是2, 3XXKB左右，而python做成的exe大約是6MB左右)。 \
經過兩個專案的試驗，沒有任何團隊同仁主動向我回報發現執行檔大小有異狀，應該可以算是偷渡成功了吧？

# Reference

* [[Python] 使用PyInstaller將程式碼轉換成執行檔](https://vocus.cc/article/64dbaa1afd897800014214cb)
* [Why does packaging my project result in 2 builds?](https://forums.unrealengine.com/t/why-does-packaging-my-project-result-in-2-builds/2230088/2)
* [Difference between the two .exe files once packaged?](https://forums.unrealengine.com/t/difference-between-the-two-exe-files-once-packaged/396720/2)
* [Change the details of a game’s shipping executable](https://forums.unrealengine.com/t/change-the-details-of-a-games-shipping-executable/455748/12)
* [Unreal – BootstrapPackagedGame has dual EXEs after packaging](https://www.myredstone.top/en/archives/2761)

[^1]: Shipping版路徑為 *C:\Users\USER_NAME\AppData\Local\PROJECT_NAME\Saved\CollectedPSOs*，而Development版則是 *PROJECT_NAME\Saved\CollectedPSOs*

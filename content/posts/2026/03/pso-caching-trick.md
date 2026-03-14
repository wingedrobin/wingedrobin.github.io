+++
title = "Pso Caching Trick"
date = "2026-03-13T17:49:43+08:00"
draft = true
tags = ["UE4", "UE5"]
+++

在咒的Demo上架後碰到玩家lag的狀況才開始

後續用在咒跟狂熱兩個案子上

讓團隊的每個成員每次測試打包版都用console開啟遊戲並帶入參數，或是建立捷徑後在屬性加入參數，這兩種方式都不可行；測試結束後再自行把pipeline cache檔上傳到NAS，這個也不可行，因為與原本習慣的測試流程不同，需要事先教育、提醒，但一定會有人忘記，所以不可行，需要找其他方式進行。

輸入指令、建立捷徑

前置常數設定
1. 專案名稱
2. pipeline cache檔副檔名 `.upipelinecache`
3. 遠端NAS資料夾路徑
4. 執行檔路徑 `{PROJECT_NAME}\\Binaries\\Win64`
5. 上傳路徑

1. 找出`{PROJECT_NAME}\Binaries\Win64`路徑下的.exe檔
2. 檢查.exe檔路徑中是否帶有`Shipping`字串，如果有，則為Shipping版，以一個布林值紀錄。
3. 用os.system加上.exe檔的路徑以執行遊戲，並帶入`-logpso`參數 (如果需要產出SM5，可以再加上`-sm5`)
4. 遊戲結束後腳本繼續往下執行
5. 檢查NAS能否連接，如果不能，則結束python程式
6. 檢查CollectedPSOs資料夾是否存在
7. 由`Saved\CollectedPSOs`中取得pipeline cache(shipping版在`C:\Users\{user_name}\AppData\Local\{PROJECT_NAME}\Saved\CollectedPSOs`，而Development版在`{PROJECT_NAME}\Saved\CollectedPSOs`)
8. 檢查NAS上是否有存放pipeline cache的資料夾，沒有的話就建立一個
9. 把本地剛才生成的所有pipeline cache上傳到NAS上
10. 刪除本地pipeline cache，避免之後重複上傳相同檔案
11. 結束python程式

使用PyInstaller將此python程式打包成.exe檔，同時換上與遊戲相同的執行檔圖示(.ico)

之後只要將此.exe檔替換掉原本專案打包出來的.exe
再放到NAS上，給其他人下載、測試
就不需要讓其他人下-logpso的指令、也不需要在測試結束後，手動把pipeline cache複製到NAS上了

而把pipeline cache統一放在NAS上
也便於之後打包流程，一併將所有的pipeline cache包進最新的版本內

我自己把這個手法稱作『輕薄的假象』，雖然一點也不輕薄就是了 (原本的Bootstrap executable大約是2, 3XXKB左右，而python做成的.exe大約是6MB左右)。
可能不是什麼很正規的方法，不過當時在公司內部並沒有出現什麼狀況，也能解決....的狀況，所以就連續用在兩個專案上了。

# Reference

* [[Python] 使用PyInstaller將程式碼轉換成執行檔](https://vocus.cc/article/64dbaa1afd897800014214cb)
* [Why does packaging my project result in 2 builds?](https://forums.unrealengine.com/t/why-does-packaging-my-project-result-in-2-builds/2230088/2)
* [Difference between the two .exe files once packaged?](https://forums.unrealengine.com/t/difference-between-the-two-exe-files-once-packaged/396720/2)
* [Change the details of a game’s shipping executable](https://forums.unrealengine.com/t/change-the-details-of-a-games-shipping-executable/455748/12)
* [Unreal – BootstrapPackagedGame has dual EXEs after packaging](https://www.myredstone.top/en/archives/2761)

+++
title = "在Code Blocks上使用gtest"
date = "2015-02-28T19:42:09+08:00"
lastmod = "2026-02-05T11:57:49+08:00"
draft = false
tags = ["C++", "Unit Test"]
+++

# Step 0. 事前準備

下載Google Test : [gtest-1.7.0](https://code.google.com/p/googletest/downloads/list) \
下載CMake : [cmake-3.2.0-rc2](http://www.cmake.org/download/)

# Step.1 用CMake編譯Google Test

1. 將CMake安裝或解壓縮至適當路徑
2. 執行\bin下的cmake-gui.exe
3. 在”Where is the source code:”中填入gtest-1.7.0的路徑
4. 在”Where to build the binaries:”中填入欲放置bulid完成的資料, 這邊我在gtest-1.7.0路徑下建立了一個build-codeblocks-mingw資料夾
5. 按下Configure

![gtest_01](https://lh3.googleusercontent.com/pw/AP1GczPAM9St2Iuziyg7dGUDCNBeHPcsqjAgpWpXbAYxdBehlflHQGKe8KrwEjtJHtjYKwCVTVbf2SR_z5h5eDCQ6riaLlbNubxN7BobQItu0_rErXQh1_s)

6. 選擇CodeBlocks-MinGW MakeFiles的generator

![gtest_02](https://lh3.googleusercontent.com/pw/AP1GczMNU1f9YL3XyS0BwgvNTnNGFFU50GxYwMwWIIMvn4qUd7NHWyJzmbPSZ7PlFKfJ5p37u9QgDO9f_U9J_Jzoe60nwbm2EB9fy88PEa1dZRPhZy0EVJE)

7. 按下Finish並等待建置完成
8. 按下Generate
(有部份參考網站寫道必需勾選gtest_disable_pthreads核取方塊, 但沒有勾選也好像沒有影響, 可能是gtest版本的差異?)

![gtest_03](https://lh3.googleusercontent.com/pw/AP1GczNVxrAa9rWagGouEESVVe3bDy4ZIveorXEFtesqtxuztG4sHbuxiLDBdoCrjC2XWtBiD_3t0HreFym-ycrRv9mQaL1WAcLkXDGTa9FKCRgyiYi7-bg)

9. 開啟..\gtest-1.7.0\build-codeblocks-mingw\中由CMake產生的gest project file(.cbp檔)

![gtest_04](https://lh3.googleusercontent.com/pw/AP1GczMxXaKLORXyhq5hsNY4JHyjisUcOH5H3u10ZqRN_T9HI0pihqv2A8iZWo98csNicpG63j5vd1OlcWw3ld4PcI6IjpwlpPI2l-A5BGGik4fw3tVa-dY)

10. 按下Build即可完成編譯

![gtest_05](https://lh3.googleusercontent.com/pw/AP1GczP3Es90TPKT4e4pX_bVyy0GMCDZUabTlvtDhUE_8ys9fpacIIOOXkbRS-sfuz2LAhdtkOnOXQz_d3_8d8Ff8GMYk4TNMBaF9NYnP-dDKxeXlpWlGFM)

# Step 2. 在Code Blocks上設定gtest

1. 首先建立一個專案
2. 在Project選單中, 點選Build Options

![gtest_06](https://lh3.googleusercontent.com/pw/AP1GczNzXkFppOOkNrLswYTAodzwFxBiPmxvgTbgGYTMXF-BwhRmpkziZalvEMY1K3TotuDNAEusBFRAWX6N_1-JYH4RJ9Dpe42oGH8SEvEgrQ0ANX-cu4I)

3. 點選Linker Settings頁籤, 並在Other linker options下填入-lgtest

![gtest_06](https://lh3.googleusercontent.com/pw/AP1GczOPGGvuUWhGIRgfT2nGCbiEHNG_6dd4GViYZYNfK-O3PcuO6LVUr63cWAB_-lNsH-egTUNwcvcdolUV-xindcyQCJfZOH8OPMxot_yU4M88sbdBbCU)

4. 換到Search directories頁籤, 在Compiler子頁籤中點選Add, 並填入..\gtest-1.7.0\include的完整路徑

![gtest_07](https://lh3.googleusercontent.com/pw/AP1GczMp-YyHrU8k0FSlhOaqz4Ca_zB37ca0fX5pNt9qH8JdOhLiNgTWt6authA2Wexybh8mRs9uM5TAxooqvGPmRxtflBc1dWMKiuErHWXByOJXyo97j5A)

5. 同樣在Linker子頁籤中點選Add, 加入..\gtest-1.7.0\build-codeblocks-mingw的完整路徑

![gtest_08](https://lh3.googleusercontent.com/pw/AP1GczPLEFLVa37Jw9gz_tzudSsFteGKlPd_J4YaL2t5ZldoSGbUilbSfo2Bdbm0v5ZFJGJCCJ0elA601ECGB8PLwzkV4Vz52euJFG7djHoiV_CMeA9JUE0)

6. 按下OK即可完成設定

# Step 3. 用簡單範例測試是否完成

在專案中簡單輸入下面的程式碼:

``` C++
#include <cstdlib>
#include <gtest/gtest.h>

int mul(int a, int b)
{
    return a * b;
}

TEST(multest, HandleNoneZeroInput)
{
    EXPECT_EQ(21, mul(3, 7));
    EXPECT_EQ(-24 ,mul(-6, 4));
}

int main(int argc, char **argv)
{
    testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

經過編譯並執行, 可以得到unit test的結果

![gtest_09](https://lh3.googleusercontent.com/pw/AP1GczOPolFS84k5fxD1FID4B3JdXaMUWiT8dz0HdUGHy3DU6U2HPe3lm27U1MUJr8OCHCwGw4ff9tksrQGdPaoO0s4pbdY-5E0kv6_3IFSCsqHrDQulKhs)

參考資料來源: [JustDoIT博客](http://www.cnblogs.com/TenosDoIt/p/3412721.html)

+++
title = "videoInput Library除錯"
date = "2015-03-04T17:21:39+08:00"
draft = false
tags = []
+++

因為碩論研究中有用到[videoInput](http://www.muonics.net/school/spring05/videoInput/)這個Library，當時在用的時候要先取得PTZ攝影機的Device ID，不過透過Device Name怎麼樣都無法取得正確的ID，所以只好翻出它的的原始碼來看 結果就抓到Bug啦!! 害我為了這個問題搞了好久…

在videoInput原始程式碼當中，第808行開始的`getDeviceIDFromName()`函式：

``` C++
int videoInput::getDeviceIDFromName(char * name)
{
    if (listDevices(true) == 0) return -1;

    int deviceID = -1;

    for (int i = 0; i < VI_MAX_CAMERAS; i++) {
        if (deviceNames[i] == name) {
            deviceID = i;
            break;
        }
    }

    return deviceID;
}
```

問題出在第815行的`if (deviceNames[i] == name)`。首先：

1. 函式內的name變數宣告為指向char的pointer
2. deviceName則是一個static char型態的二維陣列，那麼deviceName[i]所存的內容即是一個char所組成的字串。

如果依照原始程式這種==的寫法，會變成判斷deviceNames[i]變數所存的字串是否等於name變數所指的記憶體位置，不合程式邏輯，所以我將這行程式修改如下：

`if(!strcmp(deviceName[i], name))`

修改之後再將Library重新build一次，就可以正確地使用啦。

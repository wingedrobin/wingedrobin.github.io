+++
title = "行動遊戲開發分享會"
date = "2016-05-29T12:27:04+08:00"
draft = false
tags = []
+++

上個月15號參加資策會主辦的『行動遊戲開發分享會-以AWS Lumberyard遊戲引擎為例研討會』，分別由Amazon、Bonio幫你優、Twitch跟FourDesire四合願帶來不同主題的分享。

[註]：由於是資策會與AWS Taiwan合辦的分享會，所以文章內容會像業配文一樣，所以不喜歡的朋友還請多見諒摟 :-)

# 1 - AWS Lumberyard遊戲引擎及相關遊戲服務的介紹 Solutions Architect

講者：KJ Wu, Solutions Architect, AWS

這個場次主要分享目前世界上遊戲開發公司如何使用AWS的服務，而這些使用方法所形成的architecture pattern。還有介紹一些近期AWS上推出的gaming工具及服務。

## Gloable Gaming狀況

首先談到目前global gaming的市場狀況大約在910億美金左右。

### 以區域作區分

![](https://newzoo.com/wp-content/uploads/2015/04/Newzoo_Global_Games_Market_2015_Per_Region_US_China_V1_Transparent.png)
<center><a href="https://newzoo.com/">圖片出處</a></center>

* 亞洲全球佔比約47%
    * 主要為中國
* 第二大市場：北美
* 台灣排名15

### 以裝置作區分

![](https://newzoo.com/wp-content/uploads/2011/06/Newzoo_Global_Games_Market_2015_Per_Screen_Segment_V1_Transparent1.png)
<center><a href="https://newzoo.com/">圖片出處</a></center>

台灣市場當中：

* 42%為Mobile(包含watch)
    * 加上floating screen(pad)則總共佔60%
* PC佔30%
* VR及OTT(Over The Top)僅10%

## AWS Gaming Architecture Pattern

這段談到的是一些遊戲公司利用AwS建置backend服務時的實際best practice，如果我們要建置自己的backend時可以作為參考。

### AWS Global Infrastructure

![](https://d0.awsstatic.com/global-infrastructure/Global-Infrastructure_2.15.png)
<center><a href="https://aws.amazon.com/tw/about-aws/global-infrastructure/">圖片來源</a></center>

即將新加入的region:

* 亞洲
    * 韓國(2016年初上線)
    * 寧夏(中國)
    * 印度
* 歐洲
    * 英國
* 北美
    * 蒙特婁
    * 俄亥俄

### 案例說明

對於不熟悉雲端服務的人，講者先以跑跑薑餅人為例，說明Cloud兩大特點：

1. 用多少、花多少
2. Virtualing

遊戲在Launch之後的24小時內，根據流量的增減，instance數從2台擴增到60台，最後回到2台。那麼當使用的instance為2台時，就算2台的價錢，為60台時，則是60台的價錢。

### Game Backend

首先，backend的部份會有一個Central(Backend) Server提供Client資訊：
* User Registeration
* User Login
* Game List
* Leader Board

所以一般而言，在Backend端會先選擇region。原則是選擇離大部份客戶較近的region，其他要跨region的，則有另外一些solution來降低latency。在一個region裡面有多個Availability Zone(可想成cluster of data center)，建議後台的虛擬機器建置在兩個不同的AZ上。用意是達到high availability design，如果當一個AZ斷掉了(實體機器或網路)，而另外一個AZ還活著，那麼服務就不會受到影響。

當客戶連到backend時，通常會需要DB服務，AWS提供RDS(Relational Database Service)，可以協助在DB上面自動enable一個standby instance，會自動進行syncronice replication，並且同時進行監控，當master DB當掉的時候，會自動切換，再啟動另一個新的standby，並sync過去，所以在任一時間點上，資料會存在兩個地方(至少同步在兩個地方)。

靜態檔案資料則是放在S3上。

#### 在這之後…?

可以在backend API server作Auto Scaling，根據你要的metrics去驅動何時增減機器。透過監控工具(cloud watch)，定義metrics。透過自訂cloud watch上的custom log作為驅動auto scaling的alert point，例如使用者連線到一定數量/快到達上線的時候，再加開機器，不用再以標準的cpu、memory、disk容量當標準。

另外AWS也提供了Cache Service - ElastiCache

* Memory Cache
* Redis

如果有跨region的使用者，可以用CDN加快他對靜態檔案的下載速度，來協助edge的cache。

#### Database Scaling

Database Auto Scaling困難之處：

* 資料同步寫入多台機器
* 成本高

克服方式：

1. Database Sharding
    註冊資料放在一個DB，Gaming Asset Data放另一個DB。
    缺點：Aplication複雜度增加，因為資料寫入位置不一樣，因此要有一個分辨寫入位置的Layer。一旦Sharding的DB數量越多，則越painfunl，更動也會更困難。

2. 不要用relation DB，改用NoSQL DB(DynamoDB in AWS)
    特點：
        manage no sql DB服務，當IR需要增減時，可以動態調整。
        support json document
    限制：因為是以key-value架構，所以不適合複雜的sql query。

#### Cross Region

將在同一region內的common service拆分出來，分別作部署。跨region的部份，則是讓使用者先到central common服務如login。login後，再到local region的機器做連線。

優點：透過central作控管，開始進行遊戲的時候，會找到最近的region。

如果要將resource replicate一份到另一個region，AWS提供[Cloud Formation](https://aws.amazon.com/tw/cloudformation/)，將資源package起來成json template，拿到另一個region再run一遍，上面json定義用的EMOB、MySQL、EC2、Auto Scaling都會設定好，會在另一個region上長出一份replicate。

相當適用於環境上需要replication或是快速建立一個 Disaster Recovery(DR)。

#### Mobile Back-end architecture on AWS

* Authenticate & sync -> **Cognito** \
    如果Mobile App是利用Facebook/Google+/Amazon等帳號當作登入的驗證來源，AWS的Cognito可以在後台整合好。可以先定義好例如由FB登入進來的使用者具有何種授權，可以對aws的resource做什麼？

    [流程](http://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html)：
    Mobile App呼叫FB API做帳號認證，拿到OpenID token回來傳給Cognito，Cognito會自動根據先前設定的權限，回傳一個AWS的Authrized token給客戶端，客戶端就可以拿token去呼叫AWS API，做權限內的事。

* Analyze user behavior -> **Analytics** \
    特點： push上去的data可以auto explore到s3，s3下來的json file，後面big data、BI的分析可以接著自己的solution。

* Run business logic -> **Lambda** \
    Serverless Computing Resource，例如將Java code或壓縮檔上傳到Lambda，就可以直接進行呼叫，並在執行完就關掉；也可以是Event Driven的功能。可以把它想成EC2上的computing resource，只是不用開機器，只要把code upload上去就可以執行。

* Store contnet -> **S3**
    儲存靜態檔案

* Send push notifications -> **SNS**
* Store data -> **DynamoDB**
* Amazone API Gateway
    建置於CDN edge層面的API，把後台的東西在API Gateway服務這層再包起來變成restful api。可以把後台藏在API Gateway後面，可以從edge端限制/擋掉進來的request，回復client一個Error Message。

### Mobile HUB

如果使用者(指使用AWS服務的開發者)不熟悉各種功能、不了解應該要選擇哪些服務的話，AWS把多個服務，用一個integrate的console包成一個新的服務([Mobile HUB](https://aws.amazon.com/tw/mobile/))，使用者只要在console上點選需要的功能，Mobile HUB會在後台把相對應AWS的服務根據config做設定完畢，同時可以下載一個simple程式、source回來，再根據這份source code去改你要的東西。

* Testing lab服務，可以把App或Game上傳，用上面的physical手機做test。

#### 分析

* Mobile Analytics
* 可以call AWS的Machine learning做預測
* 透過[Kinesis](https://aws.amazon.com/tw/kinesis/)即時接收串流資料，例如大量push message、廣告。

---

* 實際案例：

<iframe src="https://www.slideshare.net/slideshow/embed_code/key/4JXxS1G8QS8Xf6?startSlide=45" width="100%" height="515" frameborder="0" marginwidth="0" marginheight="0" scrolling="no"></iframe>

需求：希望能夠呈現一個即時的view，來了解目前玩家的狀況。

因為玩家數量多，每秒丟進來的資料量大，所以用Kinesis把in game activity資訊收回來，接著後台在EC2上有一些負責分析邏輯的Backend server，分析想要看的資訊，經過處理後放到near real time的dashoard，所以可以在後台即時監控玩家狀況。同時把這些處理完的資料(history data)存到S3，可以透過batch方式作data mining，最後push到Redshift(relational database的data ware house服務裡面)，在透過BI的工具JDBC/ODBC connect到data ware house裡面，去做較複雜的Query運算或AD-HOC。

#### AWS Mobile SDK for Game

* New support Language
    * GO Language
    * C/C++(Dev. Preview)
* SDK for Unity
* SDK for Xamarin

## Amazon Lumberyard

### What do we hear from Game Developers?

<iframe src="https://www.slideshare.net/slideshow/embed_code/key/fjM15QHSQEnVpE?startSlide=3" width="100%" height="515" frameborder="0" marginwidth="0" marginheight="0" scrolling="no"></iframe>

* 應用雲端服務Scalability的特性來應付玩家數量的變化
* 遊戲中的Community
* BI分析
* Client端開發速度

因此AWS推出了Lumberyard

<iframe src="https://www.slideshare.net/slideshow/embed_code/key/fjM15QHSQEnVpE?startSlide=4" width="100%" height="515" frameborder="0" marginwidth="0" marginheight="0" scrolling="no"></iframe>

> Lumberyard = Cloud (AWS) + Client (Lumberyard) + Community (twitch)

<iframe src="https://www.slideshare.net/slideshow/embed_code/key/fjM15QHSQEnVpE?startSlide=6" width="100%" height="515" frameborder="0" marginwidth="0" marginheight="0" scrolling="no"></iframe>

Lumberyard是個開源引擎，底層為Cry-Engine (C++)，所以如果有需要的話可以取得原始碼自行修改。它還融入Doublehelix的Game Assets(prebuild的一些function、primitive moudle)。

* Client-side Platform
    目前主要支援的平台：
    * PC
    * PS4/XBox One
    * Mobile
        * Limit Support
    * VR
        * Limit Support

* Develope Platform: Windows Only

### Cloud Side

在開發完成之後，如果有用到cloud的服務，可以將整個遊戲的build部署到cloud端，在cloud端上跑game。

#### GameLift
<iframe src="https://www.slideshare.net/slideshow/embed_code/key/fjM15QHSQEnVpE?startSlide=12" width="100%" height="515" frameborder="0" marginwidth="0" marginheight="0" scrolling="no"></iframe>
<iframe src="https://www.slideshare.net/slideshow/embed_code/key/fjM15QHSQEnVpE?startSlide=14" width="100%" height="515" frameborder="0" marginwidth="0" marginheight="0" scrolling="no"></iframe>

流程：把build透過GameLift的command line API上傳到AWS GameLift服務，接著可以根據build去啟動一個fleet(多台會在後面運行的機器)。

能夠解決後台server管理的部份，不用自己處理機器，只要把build上傳，而client端用GameLift API跟fleet作連線；可以把fleet想像成由AWS GameLift幫你管理的cluster，不用自己負責處理。會自動根據user上線的數量做增減(auto scaling的設定可以自由調整)。

* 適合multiplayer online game。

#### 使用方法

1. 透過command line上傳build
2. 在GameLift的console會看到上傳後的遊戲
3. 根據build建立fleet
4. fleet backend設定

#### Game Type

* 適合Session base game，client一定要跟server保持在同一個session，等多人即時互動類型。
* 不適合async非即時互動

| 適合 | 不適合 |
| :---: | :---: |
| First Person Shooters | MMOs(Massively multiplayer online game) |
| MOBA(Multiplayer Online Battle Arena) | Mobile/Social |
| Racing | Peer to Peer |
| Sports | |

### Community

整合Twitch

<iframe src="https://www.slideshare.net/slideshow/embed_code/key/fjM15QHSQEnVpE?startSlide=23" width="100%" height="515" frameborder="0" marginwidth="0" marginheight="0" scrolling="no"></iframe>

透過2個function：

1. Twitch ChatPlay
2. Twitch JoinIn

## Summary

1. Game backend在雲端上best practice的architecture pattern
2. Mobile Game Service
3. Lumberyard

Game的lifecycle很短，time to market相對重要，應該善用現成的服務，很多東西可以不用再自己動手，直接利用cloud的管理環境，用最少的錢去測試你的idea是否work。一旦這個game真的take off、玩家數量開始成長的時候，可以再進行後續的optimization，也不會需要擔心。

# 2 - 微積分大賽的驚奇之旅─Bonio如何透過AWS做到程式快速部署及全球化

講者：Bonio幫你優- Jason Ho, Bonio幫你優 CEO

這個section針對topic所談到的內容其實不多，比較多是一些舉辦微積分大賽的心得還有將Pagamo推到中國/美國市場的一些分享，技術含量相對較低。

## Target Audience

對一般的遊戲而言，所有玩家即是該遊戲的end-user。但Pagamo的vision卻不同，它的user是2-tier的，一個是玩遊戲的學生，屬於end-user，另外一個則是老師。那麼對於Pagamo來說，老師這個tier的user是比較重要的，原因在於一旦有老師採用了Pagamo來作為他的教學工具，那麼這個老師的所有學生也都會成為Pagamo的user。

面臨的困難：控制遊戲性與教育性之間的平衡，不能因為對遊戲性要求的增加，反而降低了老師這個tier的user對於產品的接受度。

## 市場

台灣local的市場太小，要成功的話，必需以global市場為目標，但要先克服自己台灣的市場。但Pagamo嚴格說來只能算是個平台，它並沒有自身的內容(即題目)，所以最大的困難是內容的收集，因此Pagamo在台灣選擇與康軒出版社合作。

* 投入中國市場的策略：

如果對某個地區的市場不熟，不要自己踩進去，花上很多的人力、時間及金錢，結果只是讓自己一開始就走在不可能成功的路上。所以先找一些特定地點作為pilot project，並且找了解該市場的合作夥伴，把marketing、運營都交給合作夥伴負責，自己則是以technology跟product。

## Q&A

Q: 如何防止山寨? \
A: 不要擔心山寨，因為產業中有人作類似/相同東西是必然的，否則must be something wrong，應該思考怎樣跑再對手前面。

Q: (承上題)中國市場可能會為了保護本國企業而做手段上的封殺? \
A: 慎選能夠幫忙負責處理這些事情合作對象，並且不要在意要跟別人分享成功。

Q: 中國市場Content問題? \
A: 一樣是合作對象負責；美國市場不好談合作，但出版商願意談；反而中國不太肯談。 \

# 3 - 深入了解Twitc如何成為遊戲開發者們的好工具

講者：胡晉華

## 社群影響力

Twitch雖然只是個實況遊戲的平台，但由於聚集了大量的使用人數，加上實況主、聊天室內觀眾之間能夠及時的互動，所以其在遊戲玩家圈內的社群影響力是具相當程度的。

> 實況主的觀眾數量 = 看到遊戲的人數。

因此，具有人氣的實況主直播的遊戲，能夠直接或間接影響到觀眾(玩家)加入的意願。

## Developer Chat

除了實況遊戲內容以外，Twitch對作為遊戲開發者角色的我們有什麼可利用之處呢？

* ArenaNet - Guild Wars 2

實況內容是遊戲在Alpha階段的Developer Chat，由遊戲製作人、程式設計師、場景設計師與主持人等，從剛完成的第一個關卡、第一種職業開始玩給觀眾們看，而觀眾們也能夠即時給予設計團隊反饋。

很多人會擔心遊戲在還沒完成時就公開內容，會讓自己的idea被偷走等等。但Guild Wars 2成功的原因卻是在開發前期、尚未投入大量資金的時候，就已經藉由這些事前收集到的feedback知道市場對於這款遊戲基本元素的接受度(例如打擊感、魔法特效、職業設計等)，透過社群力量把遊戲設計成開發團隊心目中想要的樣子，卻又不失去跟真正受眾所期望的水準。除此之外，這也能讓觀眾對作品的開發產生參與感，同時也是宣傳遊戲的好方法。

## Interaction

* Twitch Play

實例：[Twitch Play Pokemon](https://zh.wikipedia.org/wiki/Twitch_Plays_Pok%C3%A9mon)

當Twitch Play Pokemon爆紅之後，遊戲開發者發現觀眾有時不只是想要看實況主玩，也不是單純想玩遊戲，而是想整實況主。因此有獨立開發團隊開發了一款遊戲[Choice Chamber](http://www.choicechamber.com/)，當實況主進入關卡時，聊天室可以投票決定關卡怪物強度、玩家的裝備，讓實況主有不同難度的挑戰。

[補充參考資料]：

1. [群眾參與: Twitch Plays Pokemon的啟示](http://vikingbar.org/2014/03/%E5%8F%83%E8%88%87%EF%BC%9Atwitch-plays-pokemon%E7%9A%84%E5%95%9F%E7%A4%BA/)
2. [獨樂樂不如眾樂樂？玩家的遊戲實況是否能激起觀看者的遊玩意圖](http://ccs.nccu.edu.tw/word/HISTORY_PAPER_FILES/7587132015.pdf)(論文PDF)
3. [Twitch plays everything: How livestreaming is changing game design](http://arstechnica.com/gaming/2015/10/twitch-plays-everything-how-livestreaming-is-changing-game-design/)

# 4 - 小型團隊也能出頭天─看Walkr如何滿足玩家的遊戲體驗

講者：FourDesire四合願- Taco Chen, 四合願CEO

[FourDesire(現名SPARKFUL)](https://sparkful.app/zh-TW)是獨立開發團隊，目前有發行幾款App，這次分享的是他們在2014年的得獎作品『[Walkr](https://sparkful.app/zh-TW/walkr)』，一款計步器類型的輕遊戲。利用玩家步行距離來為飛行船補充能量，藉此探索各個星球。那麼分享內容就是這款Walkr利用AWS服務來達到由單機遊玩轉型成具有社交功能的多人服務。

講者說原先Walkr是單機的遊戲，所以並沒有設計後台服務，上市後玩家數量逐漸成長，同時也收到玩家希望能夠增加社群功能的反饋，因此選用對應的AWS服務來架構出滿足不同需求的後台。

![](https://lh3.googleusercontent.com/pw/AP1GczOEwzP4t2vEGf90Idr1yAQtW-aSiBq3HR6B9gyL0affsXaIwydoIB-2X1LwT-ef29QP-xRke0RY1AoCOakI1C_VQo0DQ286q4B7O51WO-ckTsxw1oA)

雖然Walkr是以計步器作為的主要gameplay，能夠以離線單機的方式進行，不過一旦需要更新星球資料或是上傳成績來增加社群間的互動時，都會需要穩定的網路控管，來減輕後端負責運算的EC2的壓力，所以在客戶端與EC2之間加上了ELB服務。玩家上傳的成績等資料則在EC2運算後儲存於RDS中，而玩家需要用到的資源則是存於S3當中，並且以CloudFront服務提升傳輸資源給玩家的效率；最後為了加速EC2在運算時的速度，會將常用到的資源、資料等暫存於ElasticCache中，降低存取RDS或S3的次數。

## 推播

當為遊戲加上連線功能之後，那麼就可以利用推播的功能來將訊息傳送給玩家，例如活動公告、老玩家回流或是商品特價等等。

![](https://lh3.googleusercontent.com/pw/AP1GczNqZD8XTjTIfZp0BrRH8tL75eynrgoy_k_GGYL8rxKDKmRiPAPlBq4nSh-S5iG0Sgo1LC4Zr_q7j921fd1KEGKXSOUqaJl4gZJNLT4uIKticyDr25Q)

但是當玩家數量增加後，如果還是由原本負責運算遊戲內容的EC2來處理推播訊息的話，勢必會造成效能上額外的負擔。講者為了讓遊戲後台的服務功能單純化，藉此提高推播的效率，所以將要推播給玩家的訊息內容資料，儲存在一台Redis上，再由另外一台新的EC2專門負責處理推播功能，將訊息分別傳遞給APNS及GCM，以達到資訊傳送給所有玩家的目的。

## 監控

當所有服務都在雲端了之後，確實省錢省時間，但機器出狀況了怎麼辦？

![](https://lh3.googleusercontent.com/pw/AP1GczPhxACDnBLCTXPNJcY26vGGi0Bq9exziFR8gbT3gH-5HY2NdLH-9F8AiSsAWfwyB-Xchvx-eeI1L6hmD0WbUfPO6d-4qg_z9fVENE-_mdbw66LUu7c)

雖然雲端的優點相當多，有auto scaling、即時備援等等，但機器/系統的特性還是不變，一樣會壞掉或當機的可能，那麼如何管理、監控這些雲端上的機器呢？必竟不像架設在公司內部的設備，可以派人負責處理。所以為了這個需要，便在整個後台系統當中加入了Cloudwatch服務，負責監控整個遊戲後台當中最重要、會影響gameplay的幾個部份，同時設定一些警示條件，一旦系統出現狀況或異常，透過郵件、簡訊，甚至是電話來通知開發者，讓開發者能夠及早處理。

## Domain Polling

將遊戲推向中國市場時，雲端後台服務的設計需要克服哪些困難？

![](https://lh3.googleusercontent.com/pw/AP1GczM7OhDcWhcmn5-zi_5YDuKyTCyGLqLjLIGz-EFHLYSQXTTnexwB2b954CKEKEOnvLizNKJV6shPLRnzOkQtswLrSo6KSkZzhCau5XVI3GxfoXltpV8)

由於Walkr後台服務是架設在日本伺服器上，而中國玩家在連線時會被擋在長城之內，是由於長城會擋住特定domain的連線，造成無法與遊戲正常連線，那麼講者開設多個AWS的Route53服務，讓中國的玩家透過沒有被長城擋掉的domain與後台連線，萬一某個Route53的domain又被長城擋掉時，那麼就再換一台新的Route53，利用新的domain保持能讓玩家正常連線到後台服務。

特別的是，這種解決方法是中國玩家推薦給講者他們使用的，在牆內的人們總是嚮往自由的啊(誤)。

## 維護控制台

![](https://lh3.googleusercontent.com/pw/AP1GczPF33glLTk089yHRpNfjE-GRphq5DrC3JB7CMZTBum9D7U0coQsJX3JuNp_JClBNrQApRSDLeMOJdjDC6tZw1JCvz2_HSD5EkqmIij2aOatZp7H9Vo)

最後，對於儲存於RDS的資料，利用一台EC2以及AWS Dashboard服務進行管理、維護。

上述的全部功能利用了大量不同的AWS服務來組成一個完整的多人遊戲後台，達到了由單機遊戲轉換成具多人社群功能的目的，也剛好與第一場的分享前後呼應。

[註]：本文內容中所用參考資料皆取自公開資源，若侵犯到您的權益還煩請留言或來信告知，本人會立刻作出修正，謝謝。

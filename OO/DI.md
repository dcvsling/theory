# 依賴注入 Dependency Injection 

## Intradution

此篇文章是針對於 之前[直播](https://www.facebook.com/119279178101235/videos/240049847482134)時
我沒有注意到還有一個討論區 同時也可能沒法即時做回應
所以除了介紹一些 DI 相關資訊
也包含我感興趣的問題 以及認同的觀點

## 依賴注入簡介

### 什麼是依賴注入

依賴注入是實現控制反轉的其中一種解決方案
在 C# 上常見的作法為
透過對於每一個可能被初始化物件的型別
為其註冊一個初始化的方法 也就是實作工廠模式
並且在之後需要取用時 透過這個方法來初始化
在初始化的過程中如果需要參數
則會從註冊表中找出其型別相同的註冊資料並重複上述的行為
所以 DI 以 design pattern 來描述的話就是一個 Aggregate Factory
Aggregate 就是指前面說的反覆累進的行為 (與 Linq 上的 Aggregate 作用相同)

#### 依賴注入的解耦合

透過第三方初始化的作法 之所以可以做為控制反轉的解決方案
勢因為這個第三方初始化功能 通常僅會存在於真正使用服務的地方 (例如: 負責啟動的 Web Project)
其餘的時候都是對於 DI 的抽象層來操作 (註冊服務與服務容器都是)
這就可以讓所有可能造成複雜依賴 甚至[依賴地獄](https://en.wikipedia.org/wiki/Dependency_hell)
因為所有的依賴都會依賴到 DI 的抽象 而 DI 的實作層會依賴必提供所有有註冊的對象

### 依賴注入額外的好處

這裡列出的額外好處 除了公認的好處外
還有一些我個人觀察到的好處
有的實現比較困難 所以知道可以就好了

#### 通常 DI 都有的好處

##### 依賴注入的易於抽換

依賴注入的抽換是指 基於註冊時所註冊為 Key 的型別
可以在註冊時期替換他實際產生的實作物件
範例如下:

```csharp
// 於系統內
services.AddSingleton<IMailService, SmtpMailService>();
// 於測試中
services.Replace<IMailService, FakeMailService>();
```

##### 作為非同步與多執行序的 Context

以 C# Web 開發上 現在越來越多人支持 非同步開發的 Web
但在過去想要實現非同步開發通常都會遇上 遺失 HttpContext 的問題
原因也是因為當初 HttpContext.Current 也是服務定位器
而在不同執行序下的 HttpContext.Current 就會變成 null (如現在已改變請告訴我一聲)
依照執行序的特性我們必須要提供一個 Context 物件以提供該執行序所需要的工作
而作為匯集容器恰好非常合適擔任這個角色
如此一來只要從容器中取得 HttpContext 就不會有遺失的問題了
取得方式請參考 [IHttpContextAccessor官方文件](https://docs.microsoft.com/zh-tw/aspnet/core/fundamentals/http-context?view=aspnetcore-5.0)

##### 簡化 SRP 所造成的瑣碎類別的服務建構與替換

藉由 DI 自動累進建構的特性
讓因為 SRP 而變得瑣碎的類別每一個都只透過一次的註冊就能夠完成
而不需要另外建立一個難以調整的工廠 (至少可維護性與可擴展性很難突破)
需要做的只有當你實作類別前或後記得註冊就好

#### .Net Core 內建 DI 的好處

##### Build Time (此名稱是我自己取的 不確定是否有正確的名稱)

除了一般所知道的 Design Time (程式設計階段) 與 Runtime (執行階段) 之外
DI 創造了一個可以特別被區別出來的階段 也就是 DI 建構服務的時機
我們可以透過於這個階段來解決一些只想在最一開始執行一次的功能
ex: 實現高效能的 AOP, 讓 Attribute 產生作用, 提前建立資料通道 (pipeline) 等等

##### Open Generic Type 註冊

這個優勢 .Net Core 已經透過 ```Microsoft.Extensions.Options``` 這個套件發揮的淋漓盡致
光從透過 ```IOptions<T>``` 就能產生任何沒有被註冊進來的無參數建構子物件
就已經表達了 並不是什麼都得要注入才能被產生 XD
不過這個套件本身的技術 又是足以寫另一大篇技術文的玩意
所以這裡就不贅述了 那個套件的原始碼讓我覺得完全是一種藝術 XD
也是我後來決定只用內建的 DI 的原因之一

## 使用依賴注入要避免的事情

### 反模式 - 避免變成服務定位器模式 (Service Locator Pattern)

服務定位器模式常見的案例像是 ```DateTime.Now``` 這種透過全域定位讓大家都可以存取的模式
在DI技術剛開始盛行的時候 以這種方式實現 DI 人還滿多的 (包括我也有嘗試過)
而這之所以該避免以及被稱為反模式(anti-pattern)
是因為 DI 作為 IOC(依賴反轉) 的解決方案
其服務定位類別本身就已經創造了極為強大的黏著性
導致如果沒有參考宣告該服務定位類別的依賴就會導致無法運作
而現在的做法 可以讓每個類別都剛好僅需要該類別所需要的依賴
並解決當初要解決的問題

### 避免直接從容器取得服務

在 .Net Core 中內建的 DI 容器型別為 ```IServiceProvide```
但是在 MSDN 上其實幾乎沒有直接使用或是注入這個服務
其主要的原因在於 直接從 ```IServiceProvider``` 取得服務可能會破壞物件的留存期
其原因為接下來要講的問題

### 避免圈養依賴 Captive Dependency

[Captive Dependency](https://blog.ploeh.dk/2014/06/02/captive-dependency/) 這個名稱出自於 [Mark Seemann](https://blog.ploeh.dk/tags/#Dependency%20Injection-ref)的部落格
意思是指在 ServiceLifetime 較長的服務中注入較短的服務時會造成的問題
例如一個單例的服務中如果注入了每次 request (Scoped) 產生或是 每次都必須產生新的服務
就會導致位於單例服務中的這些留存其較短的服務 留存時間與單例相同 (不會因為 Scoped 改變而更新或取代)
這其實在 DI 的使用上有時候不容易被發現 而且不會造成任何錯誤
但所幸 .Net Core 內建 DI 會在建構服務時同時驗證是否有此類問題 如果有的話就會發出例外
所以請盡量不要因為看到相關錯誤就把驗證關掉

### 避免循環依賴

另一個 DI 難以避免的是循環依賴 而這件事情會導致在執行階段發生 stackoverflow exception
而.Net Core 內建 DI 建構服務時的驗證也會驗證這件事情
所以再次強調 請盡量不要關閉此驗證設定

## Q & A

這部分我從留言中盡可能地去蒐集所有人的提問 與 回應
但因為主軸是問題 而不是誰問的
所以接下來僅會出現討論區的留言
不管是提問還是回應的相關補充或回答
但不會標記是誰提或誰回的
以及如果前面提過的事情可能就不會在後續提及

### Architecture

#### 三層式架構

##### 當系統變大時遭遇到的問題

我個人覺得這個問題屬於價購問題 而不是 DI 問題
所以如果希望可以改善專案狀況 個人通常會且推薦考慮[Vertical Slice Architecture](https://www.youtube.com/watch?v=5kOzZz2vj2o&t=2022s)
也就是相對於三層切法為橫切再加上縱切系統的概念
縱切系統通常就不會是架構上的層級 而是功能上的分界
這樣做的時候就可以逐漸變得更接近近代的架構規劃

#### Domain Driven Design & Clean Architecture

關於這個專案中因為 DI 使用情境特殊(像是: 包含兩個不同套件的 DI Container)及其他
所以 暫時就不再此對這個架構評論

### Design Pattern

#### Repository

Repository 我現在已經盡可能不去使用了 所以 我僅能提供再我放棄他前的最好狀況與難解問題

##### Generic Repository

關於 泛型的 Repository 怎麼做會最好 已我曾經實作過的案例中
泛型型別(以下稱為Entity) 理想的情況最好是一個 Domain Object
並且該Domain Object 及期子樹物件或集合
用同一個獨立的 DbContext
所以我乾脆就建立一個 ```DbContext<TEntity>```
取得 ```DbSet<T>``` 的方法如同 Clean Architecture 的 EfRepository 直接下 ```context.Set<T>()```
並且配合既有提供的慣例(ex: 自動產生relation)以及加上一些自定義的通用慣例(ex: 產生允許ValueGenerator與Id欄位)
同樣的 Controller 也是 ```Controller<T>``` 透過 aspnetcore 官方提供的做法來實現
至於商務邏輯可以透過一些擴展實現 或是從 Entity 結構上著手
但即使這樣仍然覺得不太行

###### 困境: EF Core vs Db View

曾經跟一位朋友討論這個架構與 EF Core Code First 時發現一件事情
事實上 EF Core 與 Db View 其實扮演著相同的角色 也就是 Data View
所以在一個資料庫被獨立規劃的情況 並且使得 APP 的存取受限於 Db 時
EF 幾乎是可有可無 而且 完全無法發揮其優勢
所以 上述的做法還必須追加一個條件 就是不能適用於一個被 dba design 過的資料庫
尤其是要求只能讀 view 的那種

###### 困境: Get or Query 該回傳 IEnumerable 還是 IQueryable

這裡簡單提及問題 畢竟主軸是 DI 不是 Repository
IEnumerable 表示已經查完 IQueryable 表示尚未開始查
有些資料集可以整個拉下 有的資料集不允許一次拉全部
這是最後我棄用 Repository 作為 Data Access 的主要原因

###### 困境: TEntity and TKey

我發現有人分享了這個文章[Gereric Repository With TKey](https://docs.abp.io/en/abp/latest/Repositories#generic-repositories)
也就是關於 ```IRepository<TEntity, TKey>``` 這樣的宣告
會這樣做通常是為了例如像是 ```GetById(TKey id)``` 或是 於方法中在泛型的情況下需要取得 Id 時
即便我甚至動用 ```System.Linq.Expression``` 的方式來動態建立相關的存取方法也都不慎理想
所以最後我並沒有採用 TKey
乾脆就直接 ```Get(TEntity entity)``` 並且只有帶入 Id
而 controller 那裏 Action 的參數都只使用 TEntity 而不是透過 Id (包含前端的配合)

###### 困境: 注入過多 Repository

如果你發現你需要注入很多 Repository 來實現當下的類別
通常表示你在當下的類別做了過多的事情
舉一個例子
假設在部落格相關的系統中
你發現你需要注入 User(使用者資訊) Role(權限) Post(文章) 這三個 Repository
在我個人的實作上 User 搭配 Role 作為 Post 可視權限上
可以在 Post 查詢出來之後才處理
所以此處可以僅對於 Post 本身如何查詢來寫
一般來說 User, Role, Post 應該會是三個不同的 service 兩兩來交互運作產生結果來解決

### 周邊套件相關

#### Singleton 的 DbContext 或 Controller

通常 DbContext 與 Controller 都會是 Scoped
因為每一個 Scoped 的 DI 容器各自是獨立的 如此能夠保證執行序安全及其相關問題
如果你的 Singleton 服務真的需要 DbContext 或同等留存期的服務
你可以不要用注入的 而是透過服務的方法以參數帶入 
讓其他可以注入 DbContext 的類別來注入並且同時讓他注入此 Singleton 服務
並且在執行該服務方法時來執行 Singleton 服務的方法並且帶入 DbContext 

#### 多個 DI Container 案例

##### 多個不同套件的 DI

以 C# 來說 不同套件的 DI 現階段應該都可以與 .Net Core 內建的 DI 一起使用
這是為了解決當初 DI 剛開始盛行時 各家 DI 都打造自己的套件而導致彼此間難以合併的問題

##### 多個相同套件的 DI

一般來說會導致必須建立兩個以上的 DI 容器的機率真的很低
我能想到的多半是為了做一個臨時的 容器為了註冊最後產生的容器

##### 容器中的容器

例如像是 EF Core 之所以能透過參考不同的包 而適用於不同的 Sql Provider 就是屬於在容器中在建立獨立的子容器
並且於註冊時用這個子容器來提供服務

##### 不同留存期的容器

事實上 DI 只有兩種留存期 每次都建立 與 只建立一次
而 Scoped 的留存期的產生 就是透過建立一個與自己相同註冊內容的容器
這個新的容器就會有 每一次都建立(Transient) 與只建立一次 (Scoped) 和 從父容器取得 (Singleton)
所以其實每一次 Request 都會建立一個新的容器

#### SOLID

DI 與 SOLID 於各方面的相性非常合

#### 建議明確一一註冊 而不是透過統一檢索的方法註冊

我在直播上所提到的明確知道自己註冊了什麼 不只是單純的註冊確認
用統一檢所的方式其實很難確認註冊所用的方法以及所需要的留存期
如果你覺得這些可以透過 Attribute 解決
那為何不用擴充方法來讓註冊變成可選的
抑或是參考 裝載組件(HostingStartup) 的做法來避免檢索
因為每個 domain 不是每一次都需要 以及 同樣不用總是得依賴專用的檢索功能才能被引用
尤其是用裝載組件的做法 可以讓你完全實現 不用參考一樣能使用的能力
並且擁有相同的自由度

#### 不同型別使用相同的實例

要實現這件事情的方法並不是用組件檢索才能實現
實現的條件是註冊成 Singleton (包含先初始化後直接註冊該物件)
如果是 Scoped 內的不同型別使用相同實例則為

```csharp
[Fact]
public void WhenRegisteredAsForwardedSingleton_InstancesAreTheSame()
{
    var services = new ServiceCollection();

    services.AddSingleton<Foo>(); // We must explicitly register Foo
    services.AddSingleton<IFoo>(x => x.GetRequiredService<Foo>()); // Forward requests to Foo
    services.AddSingleton<IBar>(x => x.GetRequiredService<Foo>()); // Forward requests to Foo

    var provider = services.BuildServiceProvider();

    var foo1 = provider.GetService<Foo>(); // An instance of Foo
    var foo2 = provider.GetService<IFoo>(); // An instance of Foo
    var foo3 = provider.GetService<IBar>(); // An instance of Foo

    Assert.Same(foo1, foo2); // PASSES
    Assert.Same(foo1, foo3); // PASSES
}
```

[參考來源](https://andrewlock.net/how-to-register-a-service-with-multiple-interfaces-for-in-asp-net-core-di/)

#### 應該要註冊哪些服務

要說不要什麼都注入也沒錯 認為幾乎都要注入也沒有錯
我覺得大概可以這樣區分
對於當下 domain 而言包含邏輯或是負責處理資料的服務的都會需要註冊
對於當下 domain 而言屬於要處理的 data 的類別就不需要註冊
而當下這個 domain 認為是 data 的物件
可能是另一個 domain 用來處理物件資料的邏輯必須要的東西
所以就會產生上述都需要被註冊的狀況
下面是對於直播上提到的各種大家提過可能是註冊進 DI 的東西

- Controller
  - MVC Core 不會主動幫你註冊你的 Controller 進 DI 除非 ControllerAsService 被執行
- View
  - 這部分雖然我沒有仔細去確認 但是 View 在我們寫的程式中是透過字串(Path, Naming)檢索 這不符合 DI Key 的型別  
而且僅透過字串要產生相對應的型別名稱最少都要有一個 Mapping Table 那基於 Razor 作為 Html Generator Compiler 的角色也不需要在多一手去注入
- Routing
  - Routing 對於 Controller 的 Mapping 是透過 Route Data 與 Controller 上所定義的 Route Data 做比對 與 DI 無關  
Route Data 是多組 Key Value 的集合 預設 key 會有 controller(controller name) 與 action(action name)  
Route Data 的作用與應用案例為 Area  
Area 功能就是建立一個新的 RouteData, 其 key 為 area, value 為 AreaAttribute 的參數值(字串)  

所以整體來說 .Net Core 其實只有在 DI 註冊他們用來處理我們所撰寫的東西的那些服務
我們自己寫的東西我們得自己註冊進去
當然啦 各種預設值 通常也可能會被註冊進去以實現 Null Object Pattern

#### Aggregate Composition Root

這是一個對於 DI 容器的 Design Pattern 方式的描述
也就是 DI 指

- Aggregate: 累進建構的
- Composition: 從任何 Index 取出都是服務
- Root: Container 介面為整個結構的主要進入根

如果用圖片表示的化就會像是具有單一根的多元樹
因此在開發上就會有較為理想的開發模式
也就是依循相同的結構把自己的東西接到樹上的節點
就能夠與 DI 高度結合
在 .Net Core 內建的 DI 上就是透過對於 ```IServiceCollection``` 的擴充方法來提供自己功能的註冊表
至於範例 你可以從任何客製化的註冊方式去查看該方法的原始碼就會發現很多 且幾乎都是相同的模式 (ex: services.AddMvc())

#### Register and Resolver

在第三方套件的 DI 會很常見這兩個字
在 .Net Core 內建 DI 中沒有 原因我記得 github issue 有解釋
Register 就是 IServiceCollection
Resolver 就是 IServiceProvider

#### 如何註冊 ValueType 以及透過 ValueType 的值來建立 Scoped Service

在 .Net Core DI 中你不能注入 ValueType 成為一個 Service
你只能用一個類別去包裹你的 Value 進行註冊
並且該值也不能隨意臨時決定
該值必須要在註冊時就必須決定他的來源
對於每一個 Scoped 都必須遵守一定程度上的不變性(Immutable)
以識別其相同或不同的 Scoped

## 結語

雖然內容很多 但大多都不是需要真的記得的事情
事實上就算不瞭解這些一樣可以使用DI
(或是 被使用DI 但這就很考驗導入者的能力了)
只需要看過了解 有興趣歡迎自行驗證與研究 (還有很多可以玩啊 XD)

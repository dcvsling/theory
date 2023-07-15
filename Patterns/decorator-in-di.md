# Decorator with DI

## Intradution

關於[微軟.Net的一篇文章](https://devblogs.microsoft.com/dotnet/migrating-realproxy-usage-to-dispatchproxy/)
裡面介紹了關於 ReadProxy, DispatchProxy, DynamicProxy
三種代理並且介紹透過代理來實現關於裝飾器模式與 AOP
先前看到有不少為了透過裝飾器模式實現 Logging, Caching 等等的作法
透過上述的代理方式來實現
不僅用了反射(Reflection) 也遇到非同步的困境
導致效能變差 系統變得複雜 甚至難以除錯
我對於裝飾器模式也稍微做過一些研究
而我覺得這件事情如果能夠從架構上來解決
就能夠保證效能, 偵錯, 非同步 等等各種困境來取代使用代理

## Decorator Pattern

### 個人研究

關於前面提到對於裝飾器的研究 我幹過一些很誇張的事情
像是透過 Reflection.Emit 來動態產生型別繼承要被裝飾的類別
或是用 Mono.Cecil 改變要被裝飾的類別的繼承父類 (包括 Object)
用 T4 自動產生每個 Property 的裝飾類別並繼承
上述都有成功(有比較深刻印象的是這些 失敗的就不提了 XD)
但他們都是非常不理想的實現
原因各有不同
至於代理我當時也有嘗試過幾個
像是透過 MarshByRefObject 的 RealProxy
Castle 的 Dynamic Proxy
新的 Dispatch Proxy
這些在效能上最好的雖然是 Dynamic Proxy
但仍然會減損效能
嘗試到最後我發現大多數常見的裝飾器實現都不是很理想~~(好啦~~我知道我很挑~~XD)~~
而且真正會需要我使用裝飾器模式的地方
沒有足夠多或是重要到要去犧牲每一次的執行過程的效能
所以最後我覺得最理想的裝飾器模式就是簡單實現每一個要裝飾的對象
而且對於裝飾對象也額外追加了一些規則

### 最原始的裝飾及其特性與規則

```csharp
public interface IWriter
{
    void Write(string text);
}

public class Writer : IWriter
{
    public void Write(string text) => Console.WriteLine(text);
}

public class WriterDecorator : IWriter
{
    private readonly IWriter _writer;

    public WriterDecorator(IWriter writer)
    {
        _writer = writer;
    }
    public void Write(string text)
    {
        _writer.Write(text + " decorator ok!");
    }
}
IWriter writer = new WriterDecorator(new Writer());
```

事實上大多數的裝飾器模式都是想用通用的方式來實現 ```WriterDecorator``` 這個裝飾類別
而這個類別具有一些規則

- 實現與要裝飾對像相同的介面或類別
- 透過建構子來帶入額外要用於裝飾的其他物件

而這樣做可以獲得什麼哪些好處

- 效能: 經過裝飾的物件在執行時沒有額外效能耗損
- OCP 原則: 可以在不破壞裝飾對象的符號與不對其做任何修改行為來實現
- 強型別: 在裝飾過程中可以始終維持強型別 以減少型別的判斷邏輯
- 非同步: 可以以非常簡單的方式支持非同步的裝飾器或裝飾對象
- 易於擴展: 這個後面會提到

其餘重複的優勢與特性就略過了
但這樣也不是沒有缺點

- 要實作太多裝飾器: 可能會有 DRY 問題
- 暫時想不到了 有人有想到的話可以補充給我 我如果有解法也可以提供你參考 XD

### DRY 困境

這種做法不是最後才想到 而是滿早就想到
而對於每一個都去實現也違背我個人開發習慣的初衷
(我是個比起 Code Generator 更喜歡用 Generic 來解決問題的人)
所以當初也是因為 DRY 這種會寫到手斷掉的問題而放棄這種做法過
不過隨著經驗的累積 與上述各種嘗試的過程
讓我發現問題不是只能從程序上解決
而這件事情可以從架構上去解決
也就是透過像是過去的三曾或多層式的架構的橫切概念
從架構上對每一個橫切面來做裝飾
當然這不是對系統的橫切 而是依照其特性歸類後各自按需橫切
這樣做的目的在就是在削減一些高重複度的裝飾類型
而這樣的做法其實也很符合當初 AOP 的基礎論述
雖然不能完全解決手寫到斷掉的問題
但這也至少是權衡之下我個人比較能接受的作法了

## Dependency Injection

裝飾器跟依賴注入又有什麼關係
從上述其實不難發現 當我們把額外服務與裝飾對象都從建構子帶入的時候
就會知道他會是一個複雜的建構方式
而依賴注入碰巧有可以解決複雜建構與抽換的特質
而像是 [Scrutor](https://github.com/khellang/Scrutor) 就有支援用 DI 來裝飾

### 透過 DI 來裝飾

我們都知道 DI 可以讓實作類別易於抽換
所以透過這樣的特性就可以做到類似這樣的行為

```csharp
/// 注入被裝飾對象
services.AddSingleton<IWriter, Writer>();
/// 注入裝飾器
services.Replace<IWriter, WriterDecorator>(provider => new WriterDecorator(provider.GetService<Writer>()))
services.AddSingleton<Writer>();
```

上面這段程式碼只是用於解釋在註冊時如何做到去追加裝飾類別
至於為什麼要寫的那麼麻煩
原因是 .Net Core 內建的 DI 沒有去實現我上面提到的那種裝飾器模式
而且這件事情也是 [Scrutor](https://github.com/khellang/Scrutor) 出現的理由

### .Net Core DI 的限制

上面提出的解決方案中 裝飾器類別的建構子與他實作的介面相同
這對於 DI 這種以型別為索引的根本是一種創造無限迴圈的惡質行為
而另一個限制就是對於泛型型別的裝飾注入
在上一篇介紹依賴注入的文章中有提過
我認為 .Net Core 內建的 DI 最強大的功能就是對於 Open Generic Type 的注入
然後過沒多久他就成為我最大的障礙
在其他的 DI 套件都可以透過工廠注入的方式來實現的時候
內建的 DI 無法用相同的方式來完成
因為其他 DI 也用工廠來實現泛型注入
而在內建 DI 中我只能用型別來注入
而型別注入就會跟他實現的介面重複而發生問題
所以就連 [Scrutor](https://github.com/khellang/Scrutor) 都沒有提供相關的範例
~~看 README 中 scan 功能就有 所以 我不認為他是忘記寫 XD~~
當然我也還是有去確認他的 source 看能不能獲得一線生機
而他的 Decorator 是全部轉為工廠的作法 那就不可能實現了

### 一個不是很優的做法

這應該是我研究裝飾的最後一次結果
這是專門用於解決上述提到難以解決的問題 [附上連結](https://github.com/dcvsling/Core.Lib.Decorator)
而這也是其中一個我覺得 [Microsoft.Extensions.Options] 這個套件強大的原因
簡單來說就是我透過另一個容器來保存裝飾的型別
並且透過類似 ```IOptions<T>``` 的概念
用 ```IDecorator<T>``` 來提供被裝飾過的類別
程式碼比前一個嘗試簡單一整個檔次 (前一個根本不堪入目 XD)
這個庫有含測試 有相關困境或有興趣的可以參考看看 也歡迎討論

## 結語

其實對裝飾器的研究 是其中一個導致我的程式碼越來越往函數式編程的原因
因為對於函數來說 裝飾器就是 ```Func<T, T>```
而如果你有注意到的話
AspNetCore 的 Middleware 也是由 ```Func<RequestDelegate, RequestDelegate>``` 所組成的 pipeline
而 ```RequestDelegate``` 就是 ```Func<HttpContext, Task>```
如此一來就可以透過一個主要的 ```RequestDelegate``` 簡單完成整個 pipeline
~~所以說 別在爭 OOP 與 FP 了, AspNetCore 骨子裡也是很 FP 的 (誤)~~
從這件事情就可以感覺到 OOP 的極限與 FP 所帶來的一些優點
但最後並不是要說服大家開始學 F# 這種事情
而是看到別人的優點如果可以就轉化成為自己的優點
而不是去放大別人的缺點來掩蓋
OOP 也是有 FP 取代的特質 而我相信 FP 也會去克服相關的缺陷
~~但我不在那個環境 所以 我不太清楚囉~~


# Rx Like

[範例程式碼](https://github.com/dcvsling/fluent-api-example/tree/main/src/Calculator.RxLike)
直接切入幾個重點程式碼吧

```csharp
public static partial class RxLikeCalculator {
  public static IRxLikeCalculator<R> Compute<T, R>(this IRxLikeCalculator<T> calculator, Func<T, R> func)
      where T : struct, IConvertible
      where R : struct, IConvertible
      => new ComputeRxLikeCalculator<T, R>(calculator, func);
}

internal class ComputeRxLikeCalculator<T, R> : IRxLikeCalculator<R>
  where T : struct, IConvertible
  where R : struct, IConvertible
{
  private readonly IRxLikeCalculator<T> _last;
  private readonly Func<T, R> _func;
  public ComputeRxLikeCalculator(IRxLikeCalculator<T> last, Func<T, R> func)
  {
      _last = last;
      _func = func;
  }
  public IDisposable Then(ICalculator<R> calculator)
      => _last.Then(new ComputeCalculator<T, R>(calculator, _func));
}

internal class ComputeCalculator<T, R> : ICalculator<T> 
  where T : struct, IConvertible
  where R : struct, IConvertible
{
  private readonly ICalculator<R> _last;
  private readonly Func<T, R> _func;

  public ComputeCalculator(ICalculator<R> last, Func<T, R> func) {
      _last = last;
      _func = func;
  }

  public void Dispose()
      => _last.Dispose();

  public void Next(T value) {
      _last.Next(_func(value));
  }
}
```
基本上這段與前面說的 Linq Like 的架構非常相近
但即使如此非常相近的程式碼 在使用上的思維也是讓許多人難以理解
所謂的 Rx 的 R 是只 Reactive 也就是反應式的
顧名思義就是他不會有像是 ```Result``` 讓你取出來使用
也不會有任何可以取得回傳結果的地方
但也因為這樣 所以其實可以把它想像成在建立一個資料的通道
從最源頭丟資料進去就會一路被傳遞到最後面
而不是像前面 Linq Like 適從最後面 Result 來跟源頭要資料

## Reactive 的特性與優勢

### 有沒有需要回傳值其實差很多

從我們一開始學習程式碼的時候 就已經習慣了從方法取得結果來接續執行的思維
而這個思維應該是比較容易被大家所接受 
但也是這個思維導致程式邏輯編寫上出現非常多的問題
我簡單以下面程式碼舉例
```csharp
public class ReturnStyle {
  public void ExecuteOnConsole(int a, int b) {
    var result = Add(a, b);
    WriteConsole(result);
  }

  private int Add(int a, int b)
    => a + b;
  private void WriteConsole(int result)
    => Console.WriteLine(result);
}

public class ReactiveStyle {
  public void ExecuteOnConsole(int a, int b) {
    Add(a, b, WriteConsole);
  }

  private int Add(int a, int b, Action<int> next)
    => next(a + b);
  private void WriteConsole(int result)
    => Console.WriteLine(result);
}
```

- 其實不管怎麼做 我們都不會是最終輸出的接收者  
那我們等著方法告訴我結果不如告訴方法下一步該怎麼走反而更單純  
而且如果中途發生了什麼例外也不會因為不符合回傳型別而導致只能循其他管道 (ex: throw)  
我們也不可能統一制定且統一要求回傳一定至少要有什麼格式
- Reactive 的每一個階段可以由當下階段按需自行決定該往哪條路走並且自行決定參數  
而需要返回值的則被迫必須回傳約定的物件  
導致 後來還必須使用 ```out var name``` 這樣的參數寫法來追加
- 在非同步的環境下更是具有顯著的差異  
因為非同步就意味著需要等待與進行同步  
但是在 Reactive 的狀況下永遠只有接下來要往哪去 接下來要執行哪個方法  
更不用說有人繞別的管道早已離開 卻還有人在那痴痴的等  
(這就是造成 web host 資源耗盡的主因之一)
- 當發生例外時, 可以產生更為清楚執行過程的 Trace Stack  
因為當方法結束時 就會從 stack 移除  
而始終執行下一個方法的狀況就會依照執行順序堆疊出來  
或許這樣會有 StackOverflow 的疑慮  
但是 只要不是用遞迴的方式來建構流程  
基本上要發生這個問題是很困難的  
~~我就是那個用遞迴建構而遇上 StackOverflow 的那個人~~

### 單向資料流

這種思維後來我在 Angular 的文件上看到稱之為單向資料流
其實我們可以從一些資料架構或是設計模式看到類似的概念
- Queue: Queue 也是具有在資料同步上有優勢的模型
~~好吧 我只想到 Queue 因為其他都是 Queue 的衍生~~

但即使有那麼多的問題 但這樣的思維到底有多困難呢?
- 困難到 Message Queue 聽起來像是一個低階又困難的技術
- 困難到 Mvc 用 ActionResult 來讓使用者仍然可以用 Return Style 來撰寫程式
  - Http 從 Request 到 Response 出去是單向的阿
- 困難到 為此打造出 async / await 與 Task 然後被大家詬病 還動用黑科技
  - 黑科技是指 他沒告訴你編譯器會重組你的方法或是 塞一堆程式碼進去之類的事情  
  有點像是賣藥不講藥理, 這種藥通常沒人敢買吧
~~看來 Microsoft 才是 Over Design 的最大使用者~~
~~在座的各位都是被他們寵壞的使用者呢 當然我也不例外~~

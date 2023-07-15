[上一篇](./one-class-per-operator.md)

# Linq Like

之所以叫做 Linq 'Like' 顧名思義 只是很像
以下會描述我在做 Linq Like 時遭遇到的問題 
並且從中去了解 Linq 本身的某些功能是如何產生 或是 解決了什麼問題 如何解決的
~~~這是事後諸葛論 而且不一定與當初開發時期相符 但具有其成為解決方法的合理性~~~

## 如何實作

首先先宣告一個可以提供計算結果的介面
這個介面主要是為了提供計算結果 作法如下
```csharp
public interface ICalculator<T> where T : struct, IConvertible {
    T Result { get; }
}
```
這裡實作 ```IConvertible``` 的原因是為了保證型別 ```T``` 是可以做值轉型 接下來就會用到
接著需要一個可以帶入起始值得進入點與建立產生進入點的方法
```csharp
internal class RootCalcuator<T> : ICalculator<T> where T : struct, IConvertible
{
    private readonly T[] _values;

    public RootCalcuator(T values) {
        _values = values;
    }
    public T Result => _values;
}

public static ICalculator<R> Create<T>(T value)
    where T : struct, IConvertible
    => new RootCalculator<R>(value);
```
接著建立擴充方法 Compute 讓使用者提供計算方式
```csharp
public static ICalculator<R> Compute<T, R>(this ICalculator<T> calculator, Func<T, R> func)
    where T : struct, IConvertible
    where R : struct, IConvertible
    => new ComputeCalculator<R>(func(calculator.Result));

internal class ComputeCalculator<T> : ICalculator<T> 
    where T : struct, IConvertible
    where R : struct, IConvertible
{
    public ComputeCalculator(T result) {
        Result = result;
    }

    public R Result { get; }
}
```
到這邊為止看起來似乎都能符合我們這次的需求
除了 Linq 是處理多個對象 目前處理的是單一對象
而這個差別其實是相差非常遠的
如果我們簡單地把 ```T``` 變成 ```T[]```的時候
上述的程式碼將會即時的做完整個數組之後才往下一個步驟去
這意味著在此過程中 程式將會每有一個階段就快取一次結果
最終導致 ```OutOfMemoryException```
因此產生了一個需求 

### 新任務: 是否能將一筆資料整個做完之後在處理下一筆 (Iterator Pattern)

為何會有這種需求 因為一筆資料整個做完之後 或許他就不在了
可能寫入到檔案中 可能透過 web host Response 結束了
甚至有些資料在中途就可以過濾掉了
如果仍然留在程式中那本來就無法避免 OutOfMemory
~~意思就是把 OutOfMemory 的責任推給別人 不關我的事的概念 XD~~

#### 思考解決方法

在上述的作法 我們立即性的去計算結果後傳遞
或許我們可以不要這樣立即計算 而是當之後有需要的時候才計算結果
如果開始計算的時間 總是由最後一個步驟來啟動
那我們就可以在總是在最後一個步驟取到一個結果之後才做下一個

#### 實作解決方法

首先將 ```ICalculator<T>``` 改成能夠適應於上述的被動狀態
並且調整實作 ```RootCalculator<T>``` 使其能接受一整個數組

```csharp
public interface ICalculator<T> where T : struct, IConvertible {
    bool Next();
    T Current { get; }
}

internal class RootCalculator<T> : ICalculator<T> where T : struct, IConvertible
{
    private readonly T[] _values;
    private int _index = -1;
    public RootCalcuator(T[] values) {
        _values = values;
    }
    public T Current => _values[_index];

    public bool Next()
        => ++_index <= _values.Length - 1;
}
```

你可能需要不同的 Calculator 類別來將不同的資料型態轉為 ```ICalculator<T>```
這裡只示範將陣列型別作為主要來源為起點的案例
如此一來他就是當被問是否有下一個值的時候 才會改變其```Current```成員的值
並且在沒有下一個值的時候 會以 ```Next``` 方法回傳 ```false```
接著從上述的結論 建立一個最後步驟 ```ToList``` 方法並且依照其規則來依序取得數組中每一個結果

```csharp
public static List<T> ToList<T>(this ICalculator<T> calculator)
    where T : struct, IConvertible
{
    var result = new List<T>();
    while(calculator.Next()) {
        result.Add(innerCalculator.Current);
    }
    return result;
}
```

接著要把 ```Compute``` 運算元也改成新的做法
前面我們於擴充方法中完成運算的這個做法必須延後到被存取時才做計算
這意味著我們在 ```ComputeCalculator<T>``` 必須要有第二個泛型參數來表達在此之前的值型別

```csharp
public static ICalculator<R> Compute<T, R>(this ICalculator<T> calculator, Func<T, R> func)
    where T : struct, IConvertible
    where R : struct, IConvertible
    => new ComputeCalculator<T, R>(calculator, func);

internal class ComputeCalculator<T, R> : ICalculator<R> 
    where T : struct, IConvertible
    where R : struct, IConvertible
{
    private readonly ICalculator<T> _last;
    private readonly Func<T, R> _func;

    public ComputeCalculator(ICalculator<T> last, Func<T, R> func) {
        _last = last;
        _func = func;
    }

    public R Current => _func(_last.Current);

    public bool Next()
        => _last.Next();
}
```

透過從建構子帶入前一個階段的 ```ICalculator<T>``` 與使用者提供的運算方式 ```Func<T, R>``` 為參數
建立新的 ```ICalculator<R>``` 因為接下來並不需要關注 ```T``` 而是要關注 ```R``` 來接續建立階段

```這就是 Linq 的延遲執行的做法與行為方式```
```return yield 則只是在編譯時幫你產生 ComputeCalculator 這種類別的語法糖```

到這裡為止功能大致上已經完成的差不多了
但人們使用方式是難以想像的
有人將 ```ICalcaulator<T>``` 留存於變數 並且透過該變數存取多次而產生了多個分支
接著悲劇就發生了 因為同時多個分支都在存取 ```Next``` 導致其他分支出現跳號 遺漏元素等等的問題
以下為問題的範例
```
var seed = Enumerable.Range(1, 3).ToArray(); // create [1, 2, 3]
var root = Calculator.Create(seed).Compute(x => x + 1);
var first = root.Compute(x => root.Compute(y => x + y).List()[0]);
```
都完成到這裡了 勢必要想辦法解決這個問題

### 新任務: 能否在被存取第二次或更多次時 總是從頭開始產生新的過程 (Aggregate Lazy Factory)

因為問題出自於最一開始被不同的結尾存取導致其他結尾被跳號的
所以 如果能夠保證每一次都能保證他自己是一條獨立的流程 
被存取第二次時就會是新的一條流程提供給薪的分支接續
這樣問題就能夠排除了

#### 思考解決方法

前一次我們把值型計算留到最後才執行
這次我們也許可以把"建立整個流程"的觸發方式也放在最後一刻才開始
因為這樣做的話就表示 到最後一刻開始之前
沒有任何過程被建立 那在此時做出任何的分支行為都不會影響最後要建立的流程

#### 實踐解決方法

該如何讓最後一個階段才建立整個流程 
我們勢必無法再直接操作 ```ICalculator<T>```來串接 
因為在此時這個物件應該是還沒被建立的
而且 我們需要一個工廠來幫我們再最後建立 ```ICalculator<T>```
因此 把這兩件事情合起來產生一個新的介面 既可以做為工廠 也做為新的用於串接 api 的對象

```csharp
public interface ILinqLikeCalculator<T> where T : struct, IConvertible {
    ICalculator<T> GetCalculator();
}
```

有了這個介面 接著把既有實作 ```ICalculator<T>``` 的類別
都額外實作 ```ILinqLikeCalculator<T>``` 並且讓 ```ICalculator<T>``` 的建立
移動到 ```GetCalculator``` 方法中
而且必須要在工廠方法被執行的時候才可以去跟前一個步驟拿 ```ICalculator<T>```
太早拿就會發生這次的問題

```csharp
internal class RootLinqLikeCalculator<T> : ILinqLikeCalculator<T> where T : struct, IConvertible {
        
    private readonly T[] _values;

    public RootLinqLikeCalculator(T[] values) {
        _values = values;
    }

    public ICalculator<T> GetCalculator()
        => new RootCalculator<T>(_values);
}

internal class ComputeLinqLikeCalculator<T, R> : ILinqLikeCalculator<R>
    where T : struct, IConvertible
    where R : struct, IConvertible
{
    private readonly ILinqLikeCalculator<T> _last;
    private readonly Func<T, R> _func;
    public ComputeLinqLikeCalculator(ILinqLikeCalculator<T> last, Func<T, R> func)
    {
        _last = last;
        _func = func;
    }
    public ICalculator<R> GetCalculator()
        => new ComputeCalculator<T, R>(_last.GetCalculator(), _func);
}
```

並且修改建立階段的擴充方法 使其改為用 ```ILinqLikeCalculator<T>``` 來串連

```csharp
public static ILinqLikeCalculator<T> Create<T>(params T[] values) where T : struct, IConvertible
    => new RootLinqLikeCalculator<T>(values);

public static ILinqLikeCalculator<R> Compute<T, R>(
    this ILinqLikeCalculator<T> calculator, 
    Func<T, R> func)
    where T : struct, IConvertible
    where R : struct, IConvertible
    => new ComputeLinqLikeCalculator<T, R>(calculator, func);
```


最後調整結尾的方法 使其改為透過 ```ILinqLikeCalculator<T>``` 來取得 ```ICalculator<T>```
並且以此 ```ICalculator<T>``` 來產生最後的結果

```csharp
public static List<T> ToList<T>(this ILinqLikeCalculator<T> calculator)
    where T : struct, IConvertible
{
    var result = new List<T>();
    var innerCalculator = calculator.GetCalculator();
    while(innerCalculator.Next()) {
        result.Add(innerCalculator.Current);
    }
    return result;
}
```

### 新任務: 解決記憶體洩漏(Memory Leaking)

其實在程式領域是沒有偶發事件的
有哪些狀況會讓事情看起來像是偶發呢?
- 外部影響內部
  - ex: 參數在預料之外
- 須累積而成 
  - ex: Memory Leak 而造成的記憶體不足
- 無紀錄的 handle 狀況
  - throw empty, 或是 自動 restart 外加沒有人知道紀錄寫在事件檢視器 (IIS就會這樣做)
除了透過事後被這些問題搞到而學會之外
事前其實也還是可以多少推測出這些問題
以這次的案例來說
當我們找方法修正了分支造成的問題後
每一個流程在執行後 並不會真正的被回收
因為在一開始的地方永遠都會跟來源連結
而其他的分支也會跟來源連結而導致來源在全部分支都脫離前被釋放
因此這裡是一個很容易堆積已經用完的流程的地方

#### 思考解決方法

談到釋放那一定就得用 ```IDisposable``` 來解決了
在結尾事情完成之後 透過 ```Dispose``` 來告訴進入點可以脫離了
因此將 ```ICalculator<T>``` 加上 ```IDisposable```
並且在各個實作上追加實作

```csharp
public interface ICalculator<T> : IDisposable where T : struct, IConvertible {
    bool Next();
    T Current { get; }
}

internal sealed class RootCalculator<T> : ICalculator<T> where T : struct, IConvertible
{
    private T[] _values;
    private int _index = -1;
    public RootCalculator(T[] values) {
        _values = values;
    }
    public T Current => _values[_index];

    public void Dispose()
        => _values = null;

    public bool Next()
        => ++_index <= _values.Length - 1;
}

internal sealed class ComputeCalculator<T, R> : ICalculator<R> 
    where T : struct, IConvertible
    where R : struct, IConvertible
{
    private readonly ICalculator<T> _last;
    private readonly Func<T, R> _func;

    public ComputeCalculator(ICalculator<T> last, Func<T, R> func) {
        _last = last;
        _func = func;
    }

    public R Current => _func(_last.Current);

    public void Dispose()
        => _last.Dispose();

    public bool Next()
        => _last.Next();
}
```
這裡加上 sealed 的原因是避免他被繼承並引發在 ```Dispose``` 時
子類別也需要釋放資源而必須實作完整 Dispose Pattern (就是於自動產生時會問你並且產生較多方法的那個)
避免這件事情的原因是因為每一個運算員都應該只會運用自己的 Calculator 
而一個運算元同時也只是代表一個動作 或一件事情
因此他本來就應該要是 sealed 只是之前是否必須要有 sealed 其實沒有差別

接著在結尾加上 ```using``` 關鍵字來使其方法結束時 執行 ```Dispose```
```csharp
public static List<T> ToList<T>(this ILinqLikeCalculator<T> calculator)
    where T : struct, IConvertible
{
    var result = new List<T>();
    using var innerCalculator = calculator.GetCalculator();
    while(innerCalculator.Next()) {
        result.Add(innerCalculator.Current);
    }
    return result;
}
```
```using var```是新的語法
如果語言版本偏舊的請使用 ```using(var .....) { }```

## 總結

做到這裡應該有發現這個實現與 Linq 有非常多相似之處
像是物件結構非常的相似
|Linq|Calculator|
|IEnumerable<T>|ILinqLikeCalculator<T>|
|IEnumerable<T>.GetEnumerator()|ILinqLikeCalculator<T>.GetCalculator()|
|IEnumerator<T>|ICalculator<T>|
|IEnumerator<T>.Next()|ICalculator<T>.Next()|
|IEnumerator<T>.Current|ICalculator<T>.Current|
|IEnumerator<T>.Dispose()|ICalculator<T>.Dispose()|
|IEnumerable<T>.Select(...)|ILinqLikeCalculator<T>.Compute(...)|
|IEnumerable<T>.ToList()|ILinqLikeCalculator<T>.ToList()|

只差 Reset 方法沒有被提及 我想他勢必也有其存在的目的
因此這樣應該可以視為一種 Linq Like 的做法
並且依照這種對照方式 與實作過程
想要在 Linq 上追加新的 Operator 也應該不是難事了

[最終範例程式碼](https://github.com/dcvsling/fluent-api-example/tree/main/src/Calculator.LinqLike)

## 後記

其實我始終都覺得學習 Linq 用一個個 Operator 來解釋與嘗試的方式 
只會讓人覺得東西很多 事情很複雜的感覺
最終讓人感覺有需要在用 最後乾脆以太複雜為由捨棄真的是很可惜
由上述內容可以得知 他並不是一個有限功能的工具
而是完全覆蓋所有可能的語法 因此應該是依照自己所想的流程 
從裡面找符合的 Operator 甚至 Operator 組合來實現
因為沒有任何事情是既有的 Linq Operator 極其組合無法實現的
如果有 你也有辦法自行追加一個你需要的 Operator
之所以說他是一種語法 與一般的語法的差別就是在於 Iterator Pattern
它可以讓我們在思考過程時降低一個維度的複雜度
把需要思考做多次該如何做的事情 
簡化成只需要思考做一次該如何做的事情
我想這也是為什麼會用 Linq 的人會講 No Linq Or Die 的原因了 
~~包括我自己在內~~


[下一篇: Rx Like](./rx-like.md)
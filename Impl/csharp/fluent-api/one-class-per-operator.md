
# One Class Per Operator

其實 Fluent Api 的做法比想像中的還要 OO
現在要講的範例程式碼[連結](https://github.com/dcvsling/fluent-api-example/tree/main/src/Calculator.OneClassPerOperator)

```csharp
public interface ICalculator<T> where T : struct, IConvertible {
  T Value { get; }
}
public static class CalculatorExtensions 
{
    public static ICalculator<T> Add<T>(this ICalculator<T> calculator, T numberForAdd) where T : struct, IConvertible
        => new SimpleComputeCalculator<T>(
            calculator, 
            last =>  (T)Convert.ChangeType(Convert.ToDecimal(last) + Convert.ToDecimal(numberForAdd), typeof(T)));

}
internal class SimpleComputeCalculator<T> : ICalculator<T>
    where T : struct, IConvertible
{
    private readonly ICalculator<T> _last;
    private readonly Func<T, T> _compute;

    public SimpleComputeCalculator(ICalculator<T> last, Func<T, T> compute) {
        _last = last;
        _compute = compute;
    }
    public T Value => _compute(_last.Value);
}

Calculator.Create(1m)
  .Add(1)
  .Devide(3)
  .Value;
```

## 與前一個的差異

首先 ```ICalculator<T>``` 變乾淨了, 只留下產生結果的屬性
接著把前面是寫在 ```Calculator``` 上的運算方法通通都搬到擴展方法上
透過擴展方法被執行時建立新的 ```ICalculator<T>``` 而不再是回傳自己了
最後就是把該怎麼運算的方法作為建立新的 ```ICalculator<T>``` 的參數

## 優點

- 擴展追加功能很容易 因為每一個功能都是獨立的
- 同時具有組合的特性 可以用上下文來產生不同的效果
- 透過這種方式產生的結構可以重用
  - 因為這個邏輯依賴的是程式碼所寫的內容 而程式碼在 runtime 並不會改變  
  所以這個建構好的物件就得以重用
- 除了重用 還內建了可以擴展一些設計模式上的原型
  - 像是這種一層層包裹的方式就有辦法實現像是 undo redo 之類的功能
- 甚至誇張一點可以於 runtime 時從這個結構去理解程式碼行為 甚至修改原本法改變的過程
  - 像適用 Visitor 就可以把這樣的結構轉換成別的樣子
  - Entity Framework 就是這樣產生 SQL

## 缺點

- 開發需要理解比較多的知識 這種做法相關的技術內容還滿多的
- 擴充方法在其他語言上不一定有
  - 但是有別種做法 請參考 rxjs 的 pipe
- 他需要有較為明確的定義範圍
  - 如果沒有定義邊界的話 整體使用時會看起來變得很混亂
- 因此除了有辦法實現外 如果不要求 UX 的話 使用上一樣會體驗很差
  - 我個人覺得 這種作法應該要可以引導使用的人寫出具有很好的可讀性的程式碼
  - 所以前面才會說 相關技術內容真的是太多了
  - ~~原來高手不是要求別人要怎麼寫 而是寫個套件讓他們自然變成這樣寫~~

# Fluent Api 中的型別推斷

先前有人提了相關的問題
確實這也是 Fluent Api 在 C# 很好使用的主因之一
畢竟沒有人喜歡加上那看起來又臭又長的泛型參數
但為什麼 Linq, Rx.Net, rxjs 可以順順的寫
自己在做的時候就會被 intellisense 說看不懂
接下來就用 Linq 與 Rx.Net 的精簡版
來看看 型別到底是如何被推斷的
也順便談談 Rx.Net 與 Linq 之間到底差了多少

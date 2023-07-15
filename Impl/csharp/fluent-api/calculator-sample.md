
# 簡介

先前看到有人在問如何實現類似 Linq 那樣可以連續呼叫的服務  
這種寫法不只具有很好的可讀性, 好用, 又有很好的擴展性 ~~(而且以前很潮)~~
所以在過去也研究過這個做法
這理想介紹幾個我最後仍然會選擇使用的做法
以下就用計算機作為例子來介紹

## 常見案例

這是最常見也是幾乎是這種做法的基礎 
完整程式碼於此[連結](https://github.com/dcvsling/fluent-api-example/tree/main/src/Calculator)

```csharp

public interface ICalculator<T> where T : struct, IConvertible
{
    ICalculator<T> Add(T number);
    ICalculator<T> Subtract(T number);
    ICalculator<T> Multiply(T number);
    ICalculator<T> Divide(T number);
    T Result { get; }
}

var result = Calculator.Create()
  .Add(1)
  .Subtract(2)
  .Multiply(3)
  .Divide(4)
  .Result;

```

## 面臨的困境

上述的作法最簡單的概念就是每一個方法都回傳自己
可以把他理解成為事情還要接著繼續做
一直持續到存取得 Result 時才會知道結果
但這樣寫其實在維護上會有很多問題
像是擴展時被迫要用修改既有程式碼的方式
甚至這樣寫會導致邏輯無法重用 等等
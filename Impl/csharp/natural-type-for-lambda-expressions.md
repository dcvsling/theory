# Lambda 運算式與方法群組的自然型別

## Intro

在 C# 10 中對於方法型別上終於等到了改善
接下來會快速帶過改善內容
以及改動後可以有哪些優勢

### Lambda 的新格式

- 範例方法型別
  - ```Func<int, int>```
- 舊的格式
  - 範例: ```x => x + 1```
  - 模式: ```{ParameterName} => {Expression}```
- 新的格式
  - 範例: ```[ReturnAttr] int ([ParameterAttr] int x) => x + 1```
  - 模式: ```[{Attribute}] {ReturnType} ([{Attribute}] {ParameterType} {ParameterName}) => {Expression}```
- 方法群組
  - 範例: ```[return: ReturnAttr] int Invoke([ParameterAttr] int x) { return x + 1; }```

新的格式詳細定義了參數與回傳型別
且寫法更接近於一般的方法群組
希望這可以讓更多人能夠容易閱讀
個人也推薦改用新的格式比較好

### 自然型別

在過去 lambda, 委派, 方法群組, object 之間是無法互通的
永遠都必須透過強制轉型來轉換
而現在 lambda 與 方法群組都有了相對應的委派型別作為基礎型別
並且加上前面提到的新的格式詳細定義了參數與回傳型別的緣故
讓 lambda 與 方法群組可以用 var 來定義參數
並且預設 lambda 就是委派 除非刻意宣告為 Expression

```csharp
var parse = (string s) => int.Parse(s);
object parse = (string s) => int.Parse(s);   // Func<string, int>
Delegate parse = (string s) => int.Parse(s); // Func<string, int>
LambdaExpression parseExpr = (string s) => int.Parse(s); // Expression<Func<string, int>>
Expression parseExpr = (string s) => int.Parse(s);       // Expression<Func<string, int>>
Func<int> read = Console.Read;
Action<string> write = Console.Write;
var read = Console.Read; // Just one overload; Func<int> inferred
var write = Console.Write; // ERROR: Multiple overloads, can't choose
```

### 文件上沒提到的優勢

上述的那些更新綜合起來使用的話
意味著在過去想要存取方法之類的功能可以正常化了
也就是上述的這個部分

```csharp
Func<int> read = Console.Read;
Action<string> write = Console.Write;
```

這個改動也是讓 C# 又更往 Functional Programming 更近了一些
而之所以這很重要的原因
就是因為相較於 類別與介面
方法在設計模式的實踐是更容易且更有彈性

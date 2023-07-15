
# Context Style

跟上述的比較起來 後來我覺得用 Context 來實現這種可組織邏輯的架構更為容易
[範例程式碼](https://github.com/dcvsling/fluent-api-example/tree/main/src/Calculator.Context)

```csharp
public interface ICalculator 
{
    decimal Compute(decimal value);
}

public interface ICalculatorOperator
{
    void Compute(CalculateContext context);
}

public class CalculateContext
{
    public CalculateContext(decimal value) {
        Value = value;
    }

    public decimal Value { get; set; }
}

public class CalculatorBuilder 
{
    private readonly List<ICalculatorOperator> _operators = new List<ICalculatorOperator>();
    public CalculatorBuilder Append(ICalculatorOperator @operator) {
        _operators.Add(@operator);
        return this;
    }

    public ICalculator Build()
        => new DefaultCalculator(_operators);
}

internal class DefaultCalculator : ICalculator 
{
    private readonly IEnumerable<ICalculatorOperator> _operators;

    public DefaultCalculator(IEnumerable<ICalculatorOperator> operators) {
        _operators = operators;
    }
    public decimal Compute(decimal value)
        => _operators.Aggregate(new CalculateContext(value), Reduce).Value;

    private static CalculateContext Reduce(CalculateContext context, ICalculatorOperator @operator)
    {
        @operator.Compute(context);
        return context;
    }
}
var calculator = new CalculatorBuilder()
    .Append(new Add(1))
    .Append(new Divide(3))
    .Build();
var actual = calculator.Compute(1);

```

context style 的 fluent api 
主要是將每一個階段都用一個上下文(Context)串起
用這樣的方式 可以迴避掉泛型方法參數的轉換
還可以用利用 context 型別的不同來表示不同的業務邏輯
不用特別處理泛型這件事情應該就能簡單很多
而且這個做法還有多種變形
可以適應於 DI, 也可以用靜態陣列來組織
以下羅列一些我個人研究與使用上的優缺點

## 優點

- context 作為階段方法的參數有助於單元測試
- 可以避免處理回傳的 Task 始終使用無須包裹 Task 的物件
  - 這個情境是把 context 作為當下 scoped 的 local variable 存放區的概念  
  所以如果執行了一個具有回傳值的非同步的方法時  
  就可以在 context 上放等待後的結果  
  而這樣可行的原因是因為 ```Func<Context, Task>``` 也是回傳 ```Task```  
  所以最外層一定也會等裡面做完了他才繼續往下進行
  來一起看一下下面這段程式碼
  ```csharp
    public class Context<T> { 
        public List<T> Data { get; set; } = new List<T>();
    }
    public class LoadTFromDbContext<T> {
        private readonly DbContext dbContext;
        public LoadDataFromDbContext(DbContext dbContext) {
            _dbContext = dbContext;
        }
        async public Task ExecuteAsync(Context context) {
            context.Data = await _dbContext.Set<T>().ToListAsync();
        }
    }
  ```  
  ```Context.Data```並不需要宣告為 ```Task<>``` 一樣可以在非同步的方法上運行
  
## 缺點

- 使用時需要遵循的原則或技術比較多
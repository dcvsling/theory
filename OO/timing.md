# Extensions Method Timing

我個人在使用擴充方法時會考量的幾個時機點 :

1. 取代 overload
   1. 功能單一原則
   2. Method Naming 符合其功能
   3. 自由擴充進出口
2. 實現 Fluent Api
   1. 重新串接個物件的行為
   2. 易於實現多種 pattern
   3. 提升可讀性
   4. 提升程式性能
3. 介面導向開發的輔助

撰寫擴充發法的原則  
1. 盡可能以單一lambda 或五行內\(分號算一行\) 完成  
2. 利用Method Name 明確表達其功能性並銜接前後類別名稱  
3. 不包含自己開發的商業邏輯  
4. 所屬的靜態類別以擴充對象命名 並加上 Extensions or Helper  
5. namespace name 會設定與目標取用namespace 相同  
6. 通常只串接interface 而非實體物件本身

### 案例： o : 我會這樣做　x ：我可能不會這樣做

```text
interface IRepository<T> {
    void Add(T t);            // o
    void AddRange(...T[] ts); // x
    void Add(CastToT cbt)     // x
    string Add(T t);          // x
}

public static class RepositoryExtensions {
    public static IRepository<T> Add(
        this IRepository<T> repo,
        params T[] ts)　                                     // 同型同名Method 會以本體為優先
    {
        if(t == null) throw new ArgumentException(ErrMsg);   // Quick Fail
        return ts.Aggregate(repo,(t,rp) => rp.Add(t));       // 累加的方式可以視情況而定 並同時實現Fluent
    }

    public static IRepository<T> AddRange(    // 明確定義這個只能用在Array Like Argument
        this IRepository<T> repo,
        IEnumerable<T> ts,                    // 額外定義IEnumerable 可減少param　而必須ToArray 的情境
        string ErrMsg = default(string))　    // 可由使用者自行定義錯誤訊息 Func<T,string> format = default(Func<,>) 亦可
      => ts.IsNotNull(ErrMsg)
          .Aggregate(
              repo,
              (t,rp) => rp.Add(t)
          );                                  //　實現 one line

    public static IRepository<T> AddToRepo(
        this IEnumerable<T> ts,
        IRepository<T> repo
    )
      => ts.Aggregate(repo,(t,rp) => rp.Add(t)); // 反向串接　有必要的話
      // ts.Select(t => repo.Add(t));            //　if return IEnumerable<T>
}

public static class NullCheckHelper {
    public static T IsNotNull(this T t,string ErrMsg)                   // 額外定義輔助擴充方法
        => t != null ? t : throw new ArgumentException(ErrMsg,t);       // inline throw allow in C# 7
}

public class foo {
    public void foo(IRepository<T> t) {
        t.Add(new T()).Add(Enumerable.Range(0,10).Select(i => new T()));
        Enumerable.Range(0,10).Select(i => new T()).AddToRepo(t);
    }
}
```

## 案例：Pattern 實作

```text
public static class PatternExtensions {
    // Use in branching without if else or switch
    public static TResult WorkEntry<T,TResult>(this T t) 
        => new WorkWithA(t).Result ?? 
    public static TResult WorkWithA<T,TResult>(this T t)
        => new WorkWithA(t).Result ?? this.WorkWithB(t);
    public static TResult WorkWithB<T,TResult>(this T t)
        => new WorkWithB(t).Result ?? this.WorkWithC(t);    
    public static TResult WorkWithC<T,TResult>(this T t)
        => new WorkWithC(t).Result ?? throw new Exception("it's not work");  

    public static UseWorkEntry() => new T().WorkEntry(); 

    // Logger Decorator
    public static FuncT,T> WorkEntryWithLog<T>(this T t,Func<T,T> logger) 
        => t => logger(next(t)));

    public static T DecorateWorkWithA(this Func<T,T> lastWork,Func<T,T> newWork,T realt) 
        => (t => lastWork(newWork(t)))(realt);

    public static T UseWorkWithLogEntry()
        => new T().WorkEntryWithLog(
                 x => Console.Write(x),
                 x => new WorkWithＡ.Result（x) ?? x.WorkWithB();
            }).; 

}
```


# Start From Iterator

## 查詢與 Linq

談到Linq，通常都會連想到資料庫與EntityFramework

據說這也是原出打造Linq的主要目的，可以於應用程式端進行查詢

而Iterator Pattern在此方面上其實與SQL相同 \(Select 也是Iterator, 只是他的做法較為不同\)

不過於效能上要做大量資料的查閱，效能仍然會輸給Db 建立Index的

但Linq在應用程式上的使用方法可不單單只有集合查詢的作用

事實上他也不是集合\(Collection\)查詢，而是序列\(Sequence\)列舉

其中之差異可以參考 [Design Pattern 篇的 Iterator Pattern](https://github.com/dcvsling/Tech-Tree/tree/258e7afd792743917b6702094d23297cdcd800c8/topic/reactive/topic/design-pattern/iterator.md)

而且物件與資料最大的不同在於，物件存在著可以執行的方法\(Method\)

在基於 Linq 本身龐大且完整的 Api 數量，和具有 [Fluent Api](https://github.com/dcvsling/Tech-Tree/tree/258e7afd792743917b6702094d23297cdcd800c8/topic/reactive/topic/design-pattern/fluent-api.md) 的特性

所以將 Linq 用於方法運行上的可行性，與其產生的變化度就很可觀了

## Linq To Anything 與 當 Linq 不再只是查詢時

Linq 的後續發展大概可以分兩個方向，一種仍然是查詢

例如：Linq to Sql，Linq to Xml，Linq to Object，Linq to Chart

以上非本次主題內容，有興趣的可以請教Google大神

而另一方面則是流程迭代，例如將序列資料依序透過某串Linq所組成連鎖方法

```csharp
    DependencyContext.Default.GetDefaultAssemblyNames()
        .Select(AssemblyLoadContext.Default.LoadFromAssemblyName)
        .SelectMany(x => x.ExportedTypes)
        .Aggregate(new ServiceCollection(),(seed,next) => seed.AddSingleton(next))
        .BuildServiceProvider()
        .GetService(typeof(MyClass))
        ... //　基本上要繼續往下寫也是可以，但此純屬作效果
```

這是一個net core 2.0 的範例，過程是將Dependency Model中所有Assembly中的ExportType取出

然後放進 DI 容器中並且 BuildServiceProvider 最後取出型別為 MyClass 的物件

其中 Select，SelectMany，Aggregate 就是 Linq 的 Api

但這終究是基於查詢的 Api 所以仍然有不少不便之處

ex:

如果我想要用 Do 也就是列舉的物件執行某個方法後，沒有回傳值，必且用原本的物件繼續往下執行

那就必須得自行創造方法像是

```csharp
public static IEnumerable<T> Do<T>(this IEnumerable<T> sequence,Action<T> action)
{
    foreach(var i in sequence)
    {
        action(i);
        yield return i;
    }
}
// ------------ or ------------

sequence.Select(x => {
    action(x);
    return x;
})
```

總之在 Linq 裡面是沒有 Do 的或是類似 Do 的功能的

所以 Linq 就另外發展出一套新的擴展

[Interactive & Async](https://github.com/dcvsling/Tech-Tree/tree/258e7afd792743917b6702094d23297cdcd800c8/topic/reactive/topic/reactive/interactive-and-async.md)


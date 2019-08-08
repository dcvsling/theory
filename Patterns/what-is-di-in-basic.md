# what is DI in basic?

## 什麼是DI ?

一般我們再說DI 的時候會分為兩種不同的情境：

一種是Dependency Inversion\(相依性反轉\)，通常在討論設計模型與相依性時使用．

此時關注點在於兩個模型之間的關係

另一種是Dependency Injection\(相依性注入\)，通常會在討論如何實作時使用．

此時關注點在於如何實作Dependency Inversion

不過大致上來說概念與目的皆相同，所以不會再特別用其他名詞來區別

## 什麼是DI Container？

DI Container 是指用來提供Dependency Injection\(相依性注入\)的容器，容器內的結構基本上都是各種Class 的Constructor，並且在需要提供注入的地方建立Object , Ex:

```text
public class DIContainer
{
    private List<Type> objectFactory = new List<Type>();

    public void Register(Type type) 
        => objectFactory.Add(type);

    public object Resolve(Type type) 
        => objectFactory.ContainKey(type) ? CreateObject(type) : throw;

    private object CreateObject(Type type) 
        => Activator.CreateInstance(type);
}

var di = new DIContainer();
di.Register(typeof(Foo));
Foo foo = di.Resolve(typeof(Foo)); 
Bar bar = di.Resolve(typeof(Bar)); // throw error
```

利用這樣的一個基礎概念模型，我們可以得到任何我們註冊在DIContainer中的物件，進而利用DIContainer來將已完成的物件提供到指定的相依性關注點上，更多其他的DI Container衍生功能會在後續介紹

## DI Container 用語

* Register : 註冊指定物件建構方式
* Resolve : 依照所註冊的建構方式建構指定物件
* Inject , Injector : 於關注點輸出建構好的物件
* Lifetime : 物件的生命週期
  * Singleton：僅會在首次執行建構流程，之後都提供已建構完成的物件，並在Container Dispose時Dispose．
  * Scope：具有線生命週期的物件，具有IDisposable 特性，在完成一些任務後可讓其失效．
  * Transient：每一次的要求Container都會執行建構流程來建構全新的物件

## 所以然後哩!?

既然相依性問題已經能獲得妥善的處理，那勢必就會再繼續想這樣的模型還可以變出什麼樣的花樣，或帶給我們什麼樣的優勢

所以才會有個家DI Container爭霸的局勢．這部分就讓看倌們自行體驗吧．

所以現階段DI Container的目的早已不只適用於相依性問題或關注點分離，這是內建的

如同JS的3大SPA framework 解決SPA困境是內建的 最後則是看各家能變出什麼樣的花樣來讓使用者有更好的體驗


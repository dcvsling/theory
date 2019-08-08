## 建構子注入\(Constructor Injection\)

---

Constructor Injection為通常DI Container都會提供的相依性反轉解決方案，Ex: Microsoft DI,Angular2 DI, Autofac

它的運作方式如下：

1. 依照Type找到符合條件的Class 並開始執行建構式
2. 建構式中所需要的參數在一次從以註冊的清單中找到符合條件的class 
3. 如果找到則進入1開始建構該物件
4. 如果找不到則return null or throw exception

以Microsoft DI為例子，如果我們Api的Class結構如下

```
public class A {/*...*/}
public class B {/*...*/}
public class CWithA { public CWithA(A a){/*...*/} /*...*/ }
public class DWithBC { public DWithBC(B b,C c){/*...*/} /*...*/ }
public class EWithCD : IMyService { public EWithCD(C c,D d){/*...*/} /*...*/ }
public interface IMyService { }
```

如果依照他的模型設計，我們則會這樣建立IMyService

```
var a = new A();
var b = new B();
var c = new CWithA(a);
var d = new DWithBC(b,c);
var e = new EWithCD(c,d);
return e as IMyService;
```

如果使用Microsoft DI則寫法如下

```
new ServiceCollection()                    // all lifetime use singleton
    .AddSingleton<A>()
    .AddSingleton<B>()
    .AddSingleton<CWithA>()
    .AddSingleton<DWithBC>()
    .AddSingleton<IMyService,EWithCD>()
    .BuildServiceProvider()                //將 ServiceCollection Build 成IServiceProvider
    .GetService<IMyService>();             //從 IServiceProvider上使用擴充方法 GetService<T>()來取得服務
```

這看似更複雜的Method Chain 其實反而是更簡單的概念

如果我們手動建立會遇到下面幾種狀況：

1. 我們必須親自找出那些物件是如何件出來的，上述案例算是簡單的，一但按照近期對ＯＯＰ的開發原則下去實作那就會很難看懂其建構模式，這勢必在每一次需要時都將被重新撰寫或使用
2. 如果將此建構邏輯模組化則會在跨專案使用上遇到改版問題，以至於最終因版本混亂而難以維護
3. 元件抽換不容易，即便是模組化的建構，要能以最簡單的方式做到抽換或增加項目仍然還有一段距離

而上述的DI寫法簡單的地方在於：

1. 把我們所有需要的元件都往Container 丟，讓Container去幫我們找出建構邏輯
2. 藉由Extensions Method 來達到Service外部元件抽換與加入及改變LifeTime，而無須動到原本程式
3. 無論後續怎麼疊加最後的Service結果是恆不變的\(工程師之關注點分離的概念, 我不關心如何建構, 我只關心是否得到可用的物件\)

並且可以獲得額外好處如：

1. 任何整組的API都可以往裡面丟，然後變出我要的東西\(但通常還是需要加一點工去串皆非一般建構的模型\)
2. 可以從Container中取得任何一個已註冊的中繼元件，只要他有註冊
3. 因為上述2的理由，我們可以從任何一個中繼分支做擴展或測試或更多其他應用

我相信好處還有很多，現階段只想到常常感受到的優勢

但這也並不是完全沒缺點：

1. 這並非過去學習階段所使用的建構方式，我們應該要避免過度通用的型別當成建構子的參數 ex: string,datetime etc.. 
   Solve : 請參考DTO\(Data Transfer Object\) 及 Context \(上下文\)相關的議題 通常會對這些參數統一包裹成獨立型別

2. 需要較長的啟動時間，通常DI Container 都只會在程式啟動時對Container進行註冊，並且讓Container進行Build以建構提供物件的類別，而且DI Container 通常都不是執行續安全的物件．  
   Solve：通常僅會在程式啟動時來做，並且避免於使用中途額外重建或更新   
   ex：.NetCore : Startup class, Angular2 也存在Provider Inject階段

3. 容易寫成ServiceLocator，ServiceLocator 為DI Solution的Anti-Pattern 主要原因為ServiceLocator 只是轉移依賴對象而不是反轉  
   所以以前我自己的習慣是會在單一Service中建立屬於自己的DI Container，而.Net Core則改成提供用於DI設定的Extensions  
   ex：IServiceCollection.AddMvc\(\) 實際上裡面做了一大堆的Add動作

而關於缺點二則衍生出另一個議題，關於DI Magic Box




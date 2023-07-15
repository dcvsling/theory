
# CSharp 語法簡化只是語法糖?

## 開始之前

先前受到大家熱烈討論的 C# api 簡化的功能
剛好最近在翻 source 的時後發現了一些有趣的事情
順便帶大家一起來看 我們使用的 sdk 裡面長什麼樣子
以及一些我個人覺得也很好用的一些設計模式

## 簡化成一頁式有比較好嗎? 目的是什麼?

這件事情要從 MVC 開始說起
這裡指的 MVC 是指 .Net Team 所開發的 MVC Framework
(而不是名為 MVC 的抽象架構)
先講結論 這個簡化不只是單純的語法糖而已
外國人尤其微軟在嘗試解決問題的同時 
他們都會為這個解決問題的做法賦予他一個新的完整功能
這次這個也不例外 
而這個問題就是: 
我們真的需要 Controller 才能完成 web project 嗎?

### 歷代 MVC 中的 Controller 

我們可以從官方的 api 文件找出 MVC5, WebApi2, MVC Core
三個版本的 Controller 看看他們有什麼差別

#### MVC5

MVC5 的 Controller 最上層有一個介面 IController
他具有一個方法 Execute 
讓我們順藤摸瓜一路摸上去
大致上可以看到這些內容 [api 連結](https://docs.microsoft.com/zh-tw/dotnet/api/system.web.mvc.icontroller?view=aspnet-mvc-5.2)
```csharp
public interface IController {
    void Execute(RequestContext context);
}

public class RequestContext {
    public RequestContext(HttpContextBase context, RouteData data);
    // ....etc
}
```
從這裡就可以得知 我們是透過 ```IController``` 來操作 ```RequestContext```
而 ```RequestContext``` 再對 ```HttpContext``` 進行操作以實現返回訊息
MVC5 我們先看到這就好
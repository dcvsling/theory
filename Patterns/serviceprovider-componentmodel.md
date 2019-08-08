# IServiceProvider & ComponentModel

## IServiceProvider

不知道是否有人跟我產生過一樣的疑惑，也就是IServiceProvider到底是用來做什麼用的？

IServiceProvider 裡面宣告了必須實作 object GetService\(Type type\)

但當時我仍然不清楚這樣到底可以做到什麼事情？

尤其是他與System.ComponentModel.ISite 綁在一塊的時候更是冒出了許多的問號

畢竟ComponentModel即便有很多的文章可以閱讀，也很難搞懂Container與Component 跟Site到底是什麼三角關係

而明確搞懂這一切是在我看到Microsoft DI在IServiceProvider上的用法時，才將一切的抽象概念的迷思與運用通通解開

也了解微軟的技術團隊的技術能力比我們前進了多久\(可能在名稱還沒出現之前他們就已經在用了\)

## 先從 DI Container 開始了解吧

如果我們已知DI Container的 Resolve 就是 IServiceProvider的 GetService 那麼 GetService的 參數Type 就只是一個Tag ，GetService 會將符合這個Tag條件的Object Return ，如果用在DI Container就是在我們所Add的Class中，找出Type符合參數的type的建構式，並依照LifeTime設定取得Object

所以我們得知GetService 這個Method 的用途是在要求object利用Type 當作搜尋條件在Object的已知範圍內找到符合條件的Object回傳，以DI Container為例子如下  
線索 : Type

條件：ServiceType == Key

範圍：Container有註冊成ServiceType的Class or Interface

先將此規則保留．

## IContainer & IComponent and ISite?

Container 與 Component的關係按照字面上的意思是很好理解的

容器\(Container\)就是可以裝載多個元件\(Component\)的東西

而站台\(Site\)又是指什麼呢？這裡必須要用一點小技巧就是，不要看中文XD

Microsoft 是如何定義Site這個字眼? 我們可以依循著他的Framework來找到一些線索

ex: System.Runtime.CompilerServices.CallSite 這個Class 的說明上寫用在動態呼叫站台的基底類別

所以Site 可能是指一個調用點，所以才會叫做Call Site \(framework內部有多處method delegate 參數名稱也都較callsite\)

基於這個理由可以推斷ISite 實際的意思是調用元件服務用的Object

而同時也可以符合 ISite必須實作IServiceProvider 以及 IServiceProvider於Microsoft DI上的作用

## Microsoft DI vs ComponentModel

按照前面的分析方式來分析ComponentModel 的 GetService的話，就會得到

Key：Type

條件：ServiceType == Key

範圍：Site中有關連到的Component 和 Container以及其擁有的Component

再把上方的條件拉下來比對

線索 : Type

條件：ServiceType == Key

範圍：Container有註冊成ServiceType的Class or Interface

會發現差別只有一個是用戶端註冊一個是開發端內建，所以ComponentModel 其實是具有DI Container雛型的模型

而WinForm 是由 ComponentModel 這個模型建構起來的

所以早在很久以前我們就已經是在使用微軟提供的DI 模型來進行開發\(這只是結果論~\)

而WinForm的開發模式也正好類似是DI的運作方式，所以WinForm當作DI雛形的成功案例也不為過

ComponentModel的運作請參考其他章節


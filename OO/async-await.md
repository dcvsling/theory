# Task & Async/Await

關鍵字

* Task : 是讓指定的Delegate 以另一個線程來執行.
* async : 指定目標delegate 為存在非同步語法的方法，回傳值改變為Task & Task&lt;T&gt; \(這裡不建議以void 作為回傳值\(註1\)\)
* await : 等待目標排程結束

理解方法

1. 遇到Task 時會另開線程與當下線程一起往下執行
2. 遇到await 時程式會等待目標Task完成
3. await Task.Run 為等待一個新開的排程
4. 等待時不見得指有這個排程在執行　可能還有其他尚未進入等待的排程也在執行
5. async 是 可以讓method 中的　await Task.Run 簡化成為　await 的method 關鍵字
6. 從單一個排程的視角來看程式的運行流程仍然沒有改變 只是多了一個等待的通知繼續進行的狀態\(如果看不懂那張複雜的圖的話\)
7. 需要圖輔助的話,可以以下面這張圖來做為參考

```text
------------------->　主線程時間軸
單線程
------------------->
非同步(非多線程同時進行)
----->|      |----->  中間為等待的時間 (用 await)
      |----->|        另一個排程開始運作並結束 (用Task)
多線程
------------------->  主線程於中途創造了兩個排程(用Task)　但並未等待任何一個排程
   |-------->         排程一執行結束
       |-------->     排程二執行結束
非同步與多線程交錯情境
----->|    |----->|   |-------> M  M建立T1,T3 並都於建立時等待 (用 await Task)
      |--->|                    T1 T1建立T2但沒有進行等待(用 Task)
        |-------|   |---->      T2 T2建立了T4但是並未立即進行等待進行等待(先用task 後面才await)
                  |-->|         T3 T3建立後運行結束
             |----->|           T4 T4建立後與T2一起運行 中途T2進入等待直到T4結束
```

使用方法

```text
public void NonAsyncMethod() 
{
    Task task = Task.Run(() => DemoAction()); 　　　　　　　　　　　// 基本使用方法Task.Run中的Action or Func<T> 
                                                        　　　　 　// 會在另一個線程完成,排程會回傳型別為Task的物件

    Task<object> resultTask = Task.Run<object>(() => DemoFunc()); // 具有回傳值的非同步方法則用泛型指定回傳型別
                                                                  // 具回傳值的排程型別為Task<T>

    //resultTask.Wait();                      // Wait 會鎖死當下線程 所以使用不當容易造成deadlock 故不建議使用 07/23/2017
                                              // 當需要回傳值的時候必須以Wait method 等待排程完成
                                              // 否則通常都會拿到null (因為回傳值還沒回來)

    object result = resultTask.Result;        // 以Task.Result取得回傳值
                                              // Task.Result 即為 等待並取回Result 7/23/2017 by 余小章大大
                                              // 於multi await 時 似乎有些狀況 尚未查證

    // 無回傳值的Task 可以無需執行Wait方法
    // await task; 也同樣可以利用Wait　方法等待排程完成後　在繼續往下執行
}

async public Task InvokeAsync()  //async 標示法可以寫在開頭,個人建議如已知包含非同步內容的method加入Async在name後面
{
    await DemoTask();                              // 於標示為非同步方法的方法中　可以用await 來取代 Task.Wait()

    var result = await DemoFuncTask();             // 亦可以使用await 來直接得到 Task<T>.Wait() 後的　Task.Result

    Task.Run(() => DemoTask())                     // 於非同步標記的方法中亦可以使用Task.Run 來另開排程, 
                                                   // 那此排程就會與當下的線程走不同線程

    await Task.Run(() => DemoFuncTask());          // 當然也可以宣告Task.Run之後用await等待
                                                   // 只是這與 await DemoTask()並無差異

    result = await Task.Run<obejct>(() => {        // 所以通常會要這樣用是因為使用了lambda 來臨時執行多個 method 
        DemoAction();
        return DemoFunc();                         // 這裡一樣可以回傳所需的回傳值
    });

    result = await Task.Run(DemoActionManyMethod); // 如果不想寫lambda 寫成Method 並帶入名稱也是可以的

    // 因為仍然沒有等待中途執行的　Task.Run(() => DemoTask()) 所以該排程仍然在持續進行中
    // 以此可以做出類似MultiThread的效果 
}

// Property Value 為 Delegate時亦可以把Property當Method呼叫
// 當無回傳值的Task,且無標示async的時候,可回傳Task.CompletedTask 以作為完成工作的回傳Task
public Func<Task> DemoActionTask => () => Task.CompletedTask; 

// 無標示async 且具有Task<T>回傳值的方法可用Task.FromResult<T>(T t)來把結果回傳
public Func<Task<object>> DemoFuncTask => () => Task.FromResult<object>(new object())

// ConfigureAwait(bool) 的意思是指設定是否此awaiter 需要留存上下文　預設為false（沒寫亦為false）
// 這裡的上下文是指於排程建立時的那個線程的SynchronizationContext之類 會方法執行前後有關連性的東西
// ex: TraceStack (呼叫堆疊),如果ConfigureAwait(true) 時,可以看到最前面的堆疊,反之只能看到排程開始時的堆疊
// 建議ConfigureAwait(false)主要還是效能首選考量
// 但如因為不留上下文而讓開發時難以追蹤問題的話　不訪設定變數來統一開關這個flag 也不失一方法
public void ConfigAwaitDemo() => Task.Run(DemoAction).ConfigureAwait(false);　　

// 以下demo 用示範用例
public void DemoActionManyMethod() {
DemoAction();
DemoAction();
DemoAction();
}
public Action DemoAction => () => {}
public Func<object> DemoFunc => () => new Object();
```


### Task 與 Linq 之間共用會發生的問題

---

* Linq 本身具有延遲執行的 效果 也就是 yield 所以 在這上面用 Task 會感覺 Task 在最後才執行的理由是這個
  , 
  * 解決方法是 宣告完後直接先ToArray\(\)之類的就可以了 
* Linq Chain 在接到 Task 後的行為難以想像
  * 如果follow Linq 這樣 iterator pattern 的目的 去思考的化 大多都不會有問題 
* Linq 最後return 的 IEnumerable&lt;Task&lt;T&gt;&gt;,Task&lt;IEnumerable&lt;T&gt;&gt; 與 IEnumerableAsync&lt;T&gt;  之間的關係

  * 沒有關係 他們是完全不同的型別

    * IEnumerable&lt;Task&lt;T&gt;&gt; 是Task  的 列舉 但這不是 Task

    * Task&lt;IEnumerable&lt;T&gt;&gt; 是Task 回傳值是 列舉

    * IEnumerableAsync&lt;T&gt; 是 Task 而且是列舉\(類似IQueryable\)




# Options Pattern

```text
public class ClassA<T> : IClassA<T> where T : class {
    public ClassA(/*建構子參數*/) {
        /*將參數私有化exp: this._value = value */
    }

    public void Action(T target) {
        /* 讓私有參數影響target */
    }
}
```


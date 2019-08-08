## Observable vs Observer

---

```
interface IObserver<T>
{
    void OnNext(T t);
    void OnError(Exception e);
    void OnComplete();
}

interface IObserable<T>
{
    IDisposable Subscribe(IObserver observer);
}
```

so if i have a method like...

```
void Invoke(T t) { ... }
//-----then Wrap by Observer-------
class Observer<T> : IObserver<T>
{
    public void OnNext(T t) 
    {
        try{
            invoke(t);
        }
        catch(Exception ex) 
        {
            OnError(ex);
        }
        finally
        {
            OnComplete();
        }
    }
    public void OnError(Exception ex) 
    {
        //...
    }

    public void OnComplete()
    {
        //...
    }
}
//-----finally use Observable-------
class Observable<T> : IObservable<T>
{
    IDisposable Subscribe(IObserver observer)
    {
        
    }
}
```




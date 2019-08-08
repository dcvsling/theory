# Decorator & Convention

```csharp
public class Factory<T> 
{
    private Func<IResolver,T> constructor;
    public Factory(Func<IResolver,T> constructor)
    {
        this.constructor = constructor;
    }
    public T Create(IResolver resolver) 
    {
        return constructor(resolver);
    }
}
```

```csharp
public class Convention<T>
{
    private Action<T> apply;
    public Convention(Action<T> apply)
    {
        this.apply = apply;
    }
    public void Apply(T t)
    {
        apply(t);
    }
}
```

```csharp
public static class ConventionExtensions 
{
    public static IRegistyModel<T> Register<T>(this IRegister register,Func<IResolver,T> constructor)
    {
        return register.CreateModel<T>(constructor);
    }

    public static IRegister UseConvention(this IRegistryModel<T> model,Action<T> convention) 
    {
        return model.register.Add(resolver => {
            var result = model.constructor(resolver);
            convention(result);
            return result;
        };
    }
}
```

```csharp
public class Runner
{
    public void Main()
    {
        var expectstr = "test";
        var expectint = 2;
        var container = (new Register() as IRegister)
            .Register<ModelA<string>>().UseConvention(x => x.Value = expectstr)
            .Register(ModelA<int>)().UseConvention(x => x.Value = expectint)
            .Build();

        Assert.Equal(expectstr,container.Resolve<ModelA<string>>());
        Assert.Equal(expectint,container.Resolve<ModelA<int>>());
    }
}
```

```csharp
public class ModelA<T>
{
    T Value { get; set; }
}
```


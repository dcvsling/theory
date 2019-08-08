# Pattern In DI

Ex: \(DotNet Core DI\)

i have following pattern want other programmer may use this pattern

```text
public interface ICommandHandler<in TCommand> where TCommand : class
{
    void Invoke(TCommand command);
}

public interface ICommandHandlerDecorator<TCommand> : ICommandHandler<TCommand> where TCommand : class
{
}
```

so i design follow script

```text
//-------extensions methods-------
public static IServiceCollection AddHandler<T, TServiceType, TImplementationType>(this IServiceCollection services)
    where TServiceType : class, ICommandHandler<T>
    where TImplementationType : class, TServiceType
    where T : class
{
    services.InGeneric<T>(srv => srv.AddSingleton(
        typeof(TServiceType).GetGenericTypeDefinition(),
        typeof(TImplementationType).GetGenericTypeDefinition()));
    return services;
}

public static IServiceCollection AddHandlerDecorator<T, TServiceType, TImplementationType>(this IServiceCollection services)
    where TServiceType : class, ICommandHandlerDecorator<T>
    where TImplementationType : class, TServiceType
    where T : class
{
    services.InGeneric<T>(srv => srv.AddSingleton(
        typeof(TServiceType).GetGenericTypeDefinition(),
        typeof(TImplementationType).GetGenericTypeDefinition()));
    return services;
}

private static IServiceCollection InGeneric<T>(this IServiceCollection services,Action<IServiceCollection<T>> config)
{
    config(new ServiceCollectionDecorator<T>(services));
    return services;
}
//--------decorate IServiceCollection-----
internal class ServiceCollectionDecorator<T> : IServiceCollection<T>
{
    private readonly IServiceCollection _services;

    public ServiceCollectionDecorator(IServiceCollection services)
    {
        _services = services;
    }

    void ICollection<ServiceDescriptor>.Add(ServiceDescriptor item)
    {
        if (item.ServiceType.IsGenericType
            && (item.ImplementationType?.IsGenericType ?? false))
        {
            item = ServiceDescriptor.Describe(
                item.ServiceType.GetGenericTypeDefinition(),
                item.ImplementationType.GetGenericTypeDefinition(),
                item.Lifetime);
        }
        this.Add(item);
    }
    #region impl code
    //impl etc...
    #endregion
}
```

then other programmer should register like follow script:

```text
new ServiceCollection()
    .AddHandler<object, IWriteTo<object>, OutputWithString<object>>()
    .AddHandler<object, IWriteTo<object>, OutputWithJson<object>>()
    .AddHandlerDecorator<object, IDoManyTimes<object>, Twice<object>>()
    .AddHandlerDecorator<object, IDoManyTimes<object>, ThreeTimes<object>>()
```

這樣寫的不僅可以不讓Design Pattern 的文字充斥於程式中

還可以引導或以較為簡單的方式完成 DI Container 的註冊

ps : 泛型的註冊實際上不需要額外宣告一個無意義的泛行參數 只需要 .AddService\(Type,Type\)

這裡只是個人發懶 讓編譯器幫我去判斷是否符合DI註冊的規則而已 XD

ps2 : 完整範例 [連結](https://github.com/dcvsling/CoWorker.HolisticAbstractions)


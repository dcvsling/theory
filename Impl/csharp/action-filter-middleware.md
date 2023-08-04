---
title: action-filter-middleware
description: action-filter-middleware
---

# Action & Filter & Middleware

```csharp
public delegate Task RequestDelegate(HttpContext context);
public delegate Task OwinMiddleware(RequestDelegate request,HttpContext context);
public delegate RequestDelegate CoreMiddleware(RequestDelegate request);
public delegate Task FilterDelegate<TContextExecuting,TContextExecuted>(TContextExecuting before,Task<TContextExecuted> after);
```

```csharp
public IAppBuilder Use(this IAppBuilder app,RequestDelegate request)
{
    app.Use(async (req,ctx) =>{
        await request(context);
        await req(context);
    });
    return app;
}

public IApplicationBuilder Use(this IApplicationBuilder app,RequestDelegate request)
{
    app.Use(next => async ctx => {
        await request(ctx);
        await next(ctx);
    })
    return app;
}
```

```csharp
public RequestDelegate Build(this IAppBuilder app)
{
    return app.middlewares.Reverse()
        .Aggregate(
            ctx => { },
            (seed,next) => ctx => next(seed,ctx)
        );
}

public IApplicationBuilder Use(this IApplicationBuilder app,RequestDelegate request)
{
    return app.middlewares.Reverse()
        .Aggregate(
            ctx => { }
            async (seed,next) => ctx => next(seed(ctx));
        );
}
```

```csharp
public IAppBuilder UseFilter(this IAppBuilder app,FilterDelegate filter)
{
    app.Use(async (req,ctx) => {
        var executeAsync = () => {
            await req(ctx);
            return ctx;
        } 
        filter(ctx,await executeAsync());
    });
    return app;
}
```

```csharp
async public Task Decorator<TContextExecuting,TContextExecuted>(TContextExecuting before,Task<TContextExecuted> after)
{
    // you can filter again
    await after();
}
```

```csharp
async public Task Intercetor<TContextExecuting,TContextExecuted>(TContextExecuting before,Task<TContextExecuted> after)
{
    // you can filter again
    var ctx = consider(before) ? await after() : before.BedReqeust(); // interceptor can stop pipe
}
```


# 使用 MediatR 实现领域事件

# 领域服务、领域事件

“领域服务”、“领域事件”这些概念源自 DDD 模式（领域驱动设计），若要对 DDD 模式有一个较为全面的理解，建议读一读《Implementing Domain-driven Design》这本书。
在《领域驱动设计：软件核心复杂性应对之道.Eric.Eva》这本书中，作者这样定义领域服务：有些重要的领域操作，不适合归到实体(Entity)和值对象(Value Object)的类别中，这些操作从本质上讲是有些活动或动作，而不是事物，当领域中的某个重要过程或转换操作不属于实体或值对象的自然职责时，应该在模型中添加一个作为独立接口的操作，并将其申明为Service。

简单理解，领域服务是指按照模型职责分解后产生的职能单一的服务类，而领域事件指的是在领域类中发生的事件（不知道这样说有没有问题）。
如果想对 DDD 模式有更深入的理解，推荐阅读 [Equinox](https://github.com/EduardoPires/EquinoxProject) 开源项目。

---


# 在 ASP.NET CORE 中使用 MediatR

使用 NuGet 安装 MediatR 及 MediatR.Extensions.Microsoft.DependencyInjection

![image.png](https://cdn.nlark.com/yuque/0/2019/png/263297/1558336218053-8a13c35d-af24-4af7-a343-5b54c0c096c1.png#align=left&display=inline&height=175&name=image.png&originHeight=175&originWidth=986&size=18579&status=done&width=986)

并且在 Startup.cs 的 ConfigureServices 中注入 MediatR
```csharp
// 注入 MediatR
services.AddMediatR(RuntimeHelper.GetAssembly("WGMes.Services"));
```
由于我在 Services 层中使用领域事件，所以需要 将 Services 层的 Assembly 添加到 MediatR 中。


---


# 使用发布、订阅的 EventBus 模式

## 发布
在 MissionService.cs 中，通过依赖注入获取 IMediatR。
```csharp
private readonly IMediator _mediator;
public MissionService(IMediator mediator)
{
  this._mediator = mediator;
}

```

在需要发布领域事件的地方调用 Publish 方法。
```csharp
// 使用 MediatR 发布订阅消息
await this._mediator.Publish(new MissionGenFlowMessage()
                             {
                               Mission = moPlan,
                               BatchQty = batchQty,
                               InputQty = inputQty,
                               RouteID = routeId
                               });
```

## 领域事件
领域事件 MissionGenFlowMessage 需要继承 INotification 接口。
```csharp
    public class MissionGenFlowMessage : INotification
    {
        public TMesProdMission Mission { get; set; }

        public int InputQty { get; set; }

        public int BatchQty { get; set; }

        public int RouteID { get; set; }

    }
```

## 订阅
我们在 FlowServiceEventHandler.cs 中订阅 MissionGenFlowMessage 领域事件
FlowServiceEventHandler 只需要实现 `INotificationHandler<MissionGenFlowMessage>` 接口
```csharp
public partial class FlowService : INotificationHandler<MissionGenFlowMessage>


public async Task Handle(MissionGenFlowMessage notification, CancellationToken cancellationToken)
{
  // do something
}
```

如此这般，便完成了最简单的发布订阅模式。

---




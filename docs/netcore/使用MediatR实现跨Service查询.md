# 使用 MediatR 实现跨 Service 查询

# 跨职责查询

在多数项目中，我们经常会按照职责对 Service 层进行分解，按照不同职责分解成不同的 Service 类。但单个  Service 类的业务逻辑有时也是错综复杂的。当一个 Service 需要查询与自身职责不相关或弱相关的信息时，我们称之为“跨职责查询”。比如 ERP 系统，在新增销售出库单时，需要对许多基础模块的验证，如验证单位是否可用、验证仓库是否可用、验证库存是否满足等等，这些都属于跨职责查询。

---


# 错误的跨职责查询方式

在以往项目中，经常看到实现跨职责查询的方式主要是2种

- Service 类之间相互引用

如上述例子中，假设销售出库对应的为 SaleOutService，单位对应的为 UnitService，仓库对应的为 StockService。则 SaleOutService 在查询单位状态时直接引用 UnitService，查询仓库状态时直接引用 StockService。这样的设计会导致 Service 类之间的依赖关系变得混乱，且同层的依赖关系往往有可能引发相互依赖/环形依赖/双向依赖等不可控的依赖问题。


- 引用相应的数据访问层

引用相应的数据访问层（DAL 或 IRepository），我们以 DAL 为例，如上述例子中。SaleOutService 在查询单位状态时，引用 UnitDAL；查询仓库状态时，引用 StockDAL。乍看之下，这样的设计没有任何问题。但当跨职责功能日益增多时，一个 Service 类可能会引用二三十个 DAL 类，使得代码结构变得杂乱。

---


# Request/response 消息

[Request/response](https://github.com/jbogard/MediatR/wiki#requestresponse) 消息是 MediatR 提供一种基础的消息指令。上一篇中 Notification 消息可以由多个 Handler 接收处理，且不关心处理结果。而 Request/response 消息则只能由一个 Handler 处理，并且可返回一个 Response。这样我们便可以利用 Request/response 消息来返回查询结果。
官方还特别指出，我们可以使用 AsyncRequestHandler 来处理不需要 Response 的情况，但我们对此暂时不作考虑。

---


# 在项目中实现跨 Service 查询

在 WGMes19 项目中，WGMes.MessageEntities 项目用于存放供 MediatR 使用的消息实体，主要包括实现“IRequest”、“INotification”接口的相关类。
WGMes.MessageEntities 消息实体一般在 WGMes.Services 项目中作为 Services 类的通讯机制使用。

## Request/response 消息相关实体

　　与命令查询方式相关的类主要包括：ITransactionRequest<T>、QueryRequest<T>、IQueryRequestHandler<T>。
ITransactionRequest<T> 继承自 IRequest<T> ，ITransactionRequest<T> 主要是要求实现 Transaction 属性，使查询支持存储过程。
　　QueryRequest<T> 则继承自 ITransactionRequest<T>，是当前框架中主要使用的 Request 实体。泛型实现使我们不需要每次都编写从 IRequest<T> 继承的消息实体。
　　建议一般情况下直接使用 QueryRequest<T> 作为消息实体；当查询条件较为复杂时，再考虑从 ITransactionRequest<T> 或 IRequest<T> 实现继承类。
　　IQueryRequestHandler<T> 则是对应处理 QueryRequest<T> 的 Handler 接口。

## 使用 Send 方法查询

下面，我们以 ChangeRouteService 的 ChangeRoute 方法中的部分代码为例。

```csharp
// 获取流程单的生产记录信息
List<VMesProdRecord> records = await this._mediator.Send(new FlowRecordRequest { FlowID = flowV.FInterID });
// 查看新工艺路线
VMesTechRoute newRoute = await this._mediator.Send(new QueryRequest<VMesTechRoute>(model.FRouteID));
```

该段代码中，有 2 种查询方式，查询工艺路线的语句直接使用 QueryRequest<T> 的实现，而查询生产记录的使用了从 ITransactionRequest<T> 继承的自定义类。
从上面 2 个语句来看，2 种方式没有太大差别。只是 FlowRecordRequest 需要再手动定义一个类，而 QueryRequest<T> 则不需要，实际上 FlowRecordRequest 也可以直接使用 QueryRequest<List<VMesProdRecord>> 代替。

## Handler 处理查询

上述查询工艺路线 QueryRequest<VMesTechRoute> 对应的 Handler 为继承自 IRequestHandler<QueryRequest<VMesTechRoute>, VMesTechRoute> 接口的类，在项目中 RouteRequestHanlder 实现了该接口。

```csharp
public class RouteRequestHanlder : IRequestHandler<QueryRequest<VMesTechRoute>, VMesTechRoute>
{
}
```

而查询生产记录 FlowRecordRequest 对应的Handler 为继承自 IRequestHandler<FlowRecordRequest, List<VMesProdRecord>> 接口的类，在项目中 FlowRecordRequestHanlder 实现了该接口。

```csharp
public class FlowRecordRequestHanlder : IRequestHandler<FlowRecordRequest, List<VMesProdRecord>>
{
}
```

二者在 Handler 层面上的实现基本没有区别，只是传递的参数不同。

这样，我们便将跨职能的查询分解到不同的类中，使得 Service 类的职能更加清晰，代码更加简明。

---



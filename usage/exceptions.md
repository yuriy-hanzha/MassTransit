# Handling exceptions

Let's face it, bad things happen. Networks partition, servers crash, remote endpoints become non-responsive. And when bad things happen, exceptions get thrown. And when exceptions get thrown, people die. Okay, maybe that's a bit dramatic, but the point is, exceptions are a fact of software development.

Fortunately, MassTransit provides a number of features to help your application recover from and deal with exceptions. But before getting into that, an understanding of what happens when a message is consumed is needed.

Take, for example, a consumer that simply throws an exception.

```csharp
public class SubmitOrderConsumer :
    IConsumer<SubmitOrder>
{
    public Task Consume(ConsumeContext<SubmitOrder> context)
    {
        throw new Exception("Very bad things happened");
    }
}
```

When a message is delivered to the consumer, the consumer throws an exception. With a default bus configuration, the exception is caught by middleware in the transport (the `ErrorTransportFilter` to be exact), and the message is moved to an *_error* queue (prefixed by the receive endpoint queue name). The exception details are stored as headers with the message for analysis and to assist in troubleshooting the exception.

> In addition to moving the message to an error queue, MassTransit also produces a `Fault<T>` event. See below for more details on _faults_.

## Retrying messages

In some cases, the exception may be a transient condition, such as a database deadlock, a busy web service, or some similar type of situation which usually clears up on a second attempt. With these exception types, it is often desirable to retry the message delivery to the consumer, allowing the consumer to try the operation again.

```csharp
public class SubmitOrderConsumer :
    IConsumer<SubmitOrder>
{
    ISessionFactory _sessionFactory;

    public async Task Consume(ConsumeContext<SubmitOrder> context)
    {
        using(var session = _sessionFactory.OpenSession())
        using(var transaction = session.BeginTransaction())
        {
            var customer = session.Get<Customer>(context.Message.CustomerId);

            // continue with order processing

            transaction.Commit();
        }
    }
}
```

With this consumer, an `ADOException` can be thrown, say there is a deadlock or the SQL server is unavailable. In this case, the operation should be retried before moving the message to the error queue. This can be configured on the receive endpoint or the consumer. Shown below is a retry policy which attempts to deliver the message to a consumer five times before throwing the exception back up the pipeline.

```csharp
var sessionFactory = CreateSessionFactory();

var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
{
    var host = cfg.Host(new Uri("rabbitmq://localhost/"), h =>
    {
        h.Username("guest");
        h.Password("guest");
    });

    cfg.ReceiveEndpoint(host, "submit-order", e =>
    {
        e.UseMessageRetry(r => r.Immediate(5));
        e.Consumer(() => new SubmitOrderConsumer(sessionFactory));
    });
});
```

The `UseMessageRetry` method is an extension method that configures a middleware filter, in this case the `RetryFilter`. There are a variety of retry policies available, which are detailed in the [reference section](retries.md).

<div class="alert alert-info">
<b>Note:</b>
    In this example, the <i>UseMessageRetry</i> is at the receive endpoint level. Additional retry filters can be added at the bus and consumer level, providing flexibility in how different consumers, messages, etc. are retried.
</div>

MassTransit retry filters execute in memory and maintain a _lock_ on the message. As such, they should only be used to handle short, transient error conditions. Setting a retry interval of an hour would fall into the category of _bad things_. To retry messages after longer waits, look at the next section on redelivering messages.

## Redelivering messages

Some errors take a while to resolve, say a remote service is down or a SQL server has crashed. In these situations, it's best to dust off and nuke the site from orbit - at a much later time obviously. Redelivery is a form of retry (some refer to it as _second-level retry_) where the message is removed from the queue and then redelivered to the queue at a future time.

To add message redelivery, first, the bus must be configured with a message scheduler (see the [scheduling](usage/scheduling/README.md) section for more details). With a scheduler configured, the above example that only used retry can be modified to add scheduled redelivery as shown below.

```csharp
var sessionFactory = CreateSessionFactory();

var busControl = Bus.Factory.CreateUsingRabbitMq(cfg =>
{
    var host = cfg.Host(new Uri("rabbitmq://localhost/"), h =>
    {
        h.Username("guest");
        h.Password("guest");
    });

    cfg.UseMessageScheduler(new Uri("rabbitmq://localhost/quartz"));

    cfg.ReceiveEndpoint(host, "submit-order", e =>
    {
        e.UseScheduledRedelivery(r => r.Intervals(TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(15), TimeSpan.FromMinutes(30)));
        e.UseMessageRetry(r => r.Immediate(5));
        e.Consumer(() => new SubmitOrderConsumer(sessionFactory));
    });
});
```

Now, if the initial 5 immediate retries fail (the database is really, really down), the message will retry an additional three times after 5, 15, and 30 minutes. This could mean a total of 15 retry attempts (on top of the initial 4 attempts prior to the retry/redelivery filters taking control).

## Adding an outbox

If the consumer publishes events or sends messages (using `ConsumeContext`, which is provided via the `Consume` method on the consumer) and subsequently throws an exception, it isn't likely that those messages should still be published or sent. MassTransit provides an outbox to buffer those messages until the consumer completes successfully. If an exception is thrown, the buffered messages are discarded.

To configure the outbox with redelivery and retry:

```csharp
    cfg.ReceiveEndpoint(host, "submit-order", e =>
    {
        e.UseScheduledRedelivery(r => r.Intervals(TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(15), TimeSpan.FromMinutes(30)));
        e.UseMessageRetry(r => r.Immediate(5));
        e.UseInMemoryOutbox();

        e.Consumer(() => new SubmitOrderConsumer(sessionFactory));
    });
});
```

## Configuring for a consumer or saga

If there are multiple consumers (or saga) on the same endpoint (which could potentially get you on the _naughty list_), and the retry/redelivery behavior should only apply to a specific consumer or saga, the same configuration can be applied specifically to the consumer or saga.

To configure a specific consumer.

```csharp
    cfg.ReceiveEndpoint(host, "submit-order", e =>
    {
        e.Consumer(() => new SubmitOrderConsumer(sessionFactory), c =>
        {
            c.UseScheduledRedelivery(r => r.Intervals(TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(15), TimeSpan.FromMinutes(30)));
            c.UseMessageRetry(r => r.Immediate(5));
            c.UseInMemoryOutbox();
        });
    });
});
```

Sagas are configured in the same way, using the saga configurator.

## Handling faults

As shown above, MassTransit delivers messages to consumers by calling the _Consume_ method. When a message consumer throws an exception instead of returning normally, a `Fault<T>` is produced, which may be published or sent depending upon the context.

A `Fault<T>` is a generic message contract including the original message that caused the consumer to fail, as well as the `ExceptionInfo`, `HostInfo`, and the time of the exception.

```csharp
public interface Fault<T>
    where T : class
{
    Guid FaultId { get; }
    Guid? FaultedMessageId { get; }
    DateTime Timestamp { get; }
    ExceptionInfo[] Exceptions { get; }
    HostInfo Host { get; }
    T Message { get; }
}
```

If the message headers specify a `FaultAddress`, the fault is sent directly to that address. If the _FaultAddress_ is not present, but a `ResponseAddress` is specified, the fault is sent to the response address. Otherwise, the fault is published, allowing any subscribed consumers to receive it.

## Consuming faults

Developers may want to do something with faults, such as updating an operational dashboard. To observe faults separate of the consumer that caused the fault to be produced, a consumer can consume fault messages the same as any other message.

```csharp
public class DashboardFaultConsumer :
    IConsumer<Fault<SubmitOrder>>
{
    public async Task Consume(ConsumeContext<Fault<SubmitOrder>> context)
    {
        // update the dashboard
    }
}
```

Faults can also be observed by state machines when specified as an event:

```csharp
Event(() => SubmitOrderFaulted, 
    x => x.CorrelateById(m => m.Message.Message.OrderId)
    .SelectId(m => m.Message.Message.OrderId));

public Event<Fault<SubmitOrder>> SubmitOrderFaulted { get; private set; }
```

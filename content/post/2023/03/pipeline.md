---
title: "Under the hood of NServiceBus: Pipelines"
date: 2023-03-30
---

A pipeline is a way of executing a sequence of code without knowing what that code is. You can think of it a bit like a function. You call it, pass in some parameters, and something happens.

Lets look at an example. When you send a message using NServiceBus it looks like this:

```cs
await messageSession.Send(new MyMessage());
```

It looks simple but under the covers, there's a whole of work to be done.

<!--more-->

Some examples include:

- Log the operation (if enabled)
- Figure out which logical endpoint to route the message to
- Serialize the message into a wire format
- Add headers for things like message type and reply address
- Hand off the message to a message mutator (if present)
- Intercept outgoing messages to persist in the outbox
- Pass the message to the underlying client library to handle with the transport

All of these are things that need to be done when a message is sent. We call these behaviors. A collection of behaviors to be executed is called a pipeline.

Building a pipeline looks a little like this:

```cs
// NOTE: Not real code
var pipelineBuilder = new PipelineBuilder();
if(enableLogging)
{
    pipelineBuilder.AddBehavior(new LoggingBehavior());
}
pipelineBuilder.AddBehavior(new RoutingBehavior());
pipelineBuilder.AddBehavior(new SerializeBehavior());
pipelineBuilder.AddBehavior(new AddHeadersBehavior());
if(messageMutator != null)
{
    pipelineBuilder.AddBehavior(new MutateBehavior(messageMutator));
}
pipelineBuilder.AddBehavior(new OutboxBehavior());
pipelineBuilder.AddBehavior(new TransportBehavior());

var sendPipeline = pipelineBuilder.Build();
```

And then the call to `messageSession.Send` looks something like this:

```cs
public Task Send<TMessage>(TMessage message)
{
    var outgoingMessageContext = new OutgoingMessageContext(message);
    return sendPipeline.Invoke(outgoingMessageContext);
}
```

What's that `OutgoingMessageContext` thing? 

All of these actions are things that need to be done when a message is sent, and each of these is provided by a different feature. 


Not only that, but there is a rough order to these that must be followed. It doesn't make sense to serialize the message after we have passed it to a message mutator (which is expecting an already serialized message).


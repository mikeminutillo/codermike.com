---
title: "Under the hood of NServiceBus: EndpointConfiguration"
date: 2019-06-18
---

I've always liked reading through code. There's something about exploring a codebase and discovering all of the little tricks and architectural patterns that have been used. I always learn something new.

I've been working at Particular Software for just over 4 years now, so it should come as no surprise that I have spent some time digging in the NServiceBus code[^1]. I thought I would write a blog post to cover some of the things that I see along the way.

<!--more-->

NOTE: This isn't a tutorial. We have a ton of great documentation[^2]. This is a look "under the hood".

## It all starts with endpoint configuration

Here is what setting up a basic NServiceBus endpoint looks like in code:

```cs
var endpointConfiguration = new EndpointConfiguration("MyEndpoint");

endpointConfiguration.UseTransport<LearningTransport>();

var endpoint = await Endpoint.Start(endpointConfiguration);
```

Now your endpoint is running and you can send and recieve messages. There's really only two pieces of required information. A name for your endpoint, and a transport.[^3] Both of those get added to an `EndpointConfiguration` object so that's where everything starts. Let's take a look at it's public API[^4]:

```cs
public class EndpointConfiguration : NServiceBus.Configuration.AdvancedExtensibility.ExposeSettings
{
    public EndpointConfiguration(string endpointName) { }
    public NServiceBus.Notifications Notifications { get; }
    public NServiceBus.Pipeline.PipelineSettings Pipeline { get; }
    public NServiceBus.ConventionsBuilder Conventions() { }
    public void RegisterComponents(System.Action<NServiceBus.ObjectBuilder.IConfigureComponents> registration) { }
    public void SendOnly() { }
    public void UseContainer<T>(System.Action<NServiceBus.Container.ContainerCustomizations> customizations = null)
        where T : NServiceBus.Container.ContainerDefinition, new () { }
    public void UseContainer(System.Type definitionType) { }
    public void UseContainer(NServiceBus.ObjectBuilder.Common.IContainer builder) { }
}
```

We can see the name gets passed in to the constructor but `UseTransport<T>` isn't actually a method on the class. It's an extension method. This is a pretty common pattern. There's lots of subsystems that need to extend `EndpointConfiguration` and many of them are in different assemblies/nuget packages. While a few of them use the public API above to add their data, the most common technique is not immediately obvious.

## Settings exposed

The first piece of the puzzle lies in the base class `ExposeSettings`. Let's look at it's public API:

```cs
public abstract class ExposeSettings
{
    protected ExposeSettings(NServiceBus.Settings.SettingsHolder settings) { }
}
```

It's an abstract class with a protected constructor, and that's it. Or is it? If you look at it's implementation[^5], you will find an internal property:

```cs
internal SettingsHolder Settings { get; }
```

This is where all of the settings for our endpoint actually live. When you call `UseTransport<T>` it is putting something in this object. Many of the configuration methods that extend `EndpointConfiguration` will add data to these Settings.

I'll go into the details of the `SettingsHolder` class in a later post. It is public, so you could use it. The important question here is if Settings is an internal property, how do extension methods defined in other assemblies get access to it?

## Advanced extensibility

The last piece of the puzzle is an extension method added in the `NServiceBus.Configuration.AdvancedExtensibility` namespace. Here's it's API:

```cs
public class static AdvancedExtensibilityExtensions
{
    public static NServiceBus.Settings.SettingsHolder GetSettings(this NServiceBus.Configuration.AdvancedExtensibility.ExposeSettings config) { }
}
```

So if you have an `EndpointConfiguration` instance, you can't see it's `Settings` property because it's internal. If you bring in the AdvancedExtensibility namespace, you can get the settings by calling `GetSettings()`. Pretty clever.

## A template for extending NServiceBus configuration

All of this leads to a very common pattern for extending NServiceBus configuration that gets seen over and over. It looks like this:

```cs
namespace NServiceBus
{
  using NServiceBus.Configuration.AdvancedExtensibility;
  
  public static class MyFeatureConfigurationExtensions
  {
    public static void MyFeature(this EndpointConfiguration configuration)
    {
      var settings = configuration.GetSettings();
      // Do stuff with settings
    }
  }
}
```

The important bits are:

- We put our extension method class in the `NServiceBus` namespace. This gives users access to it as soon as they reference our assembly.
- We import the `NServiceBus.Configuration.AdvancedExtensibility` namespace with a using statement. This gives us access to the internal settings on the endpoint configuration.
- We add an extension method to `EndpointConfiguration`. This is the starting point for most configuration extensions.
- We call the `GetSettings()` extension method to get access to the internal settings property. 

Once we have the settings we need to do something with it. That'll be the subject of the next post. 

[^1]: https://github.com/Particular/NServiceBus

[^2]: [Start here](https://docs.particular.net/get-started/)

[^3]: A [transport](https://docs.particular.net/transports/) is the underlying technology that sends and receives messages. Typically this is a queuing technology like [Azure Service Bus](https://docs.particular.net/transports/azure-service-bus/) or [RabbitMQ](https://docs.particular.net/transports/rabbitmq/). Most examples use the [`LearningTransport`](https://docs.particular.net/transports/learning/) which uses the local file system. It's great for quickly trying things out. 

[^4]: Most (all?) of our code that gets turned into NuGet packages is covered by an Approval Test that checks it's public API. That means the full public API for NServiceBus is available [in github](https://github.com/Particular/NServiceBus/blob/develop/src/NServiceBus.Core.Tests/ApprovalFiles/APIApprovals.ApproveNServiceBus.netframework.approved.txt). It also means we can't accidentally break the public API :)

[^5]: https://github.com/Particular/NServiceBus/blob/develop/src/NServiceBus.Core/Features/ExposeSettings.cs

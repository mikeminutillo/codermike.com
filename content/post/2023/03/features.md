---
title: "Under the hood of NServiceBus: Features"
date: 2023-03-24
---

In the previous posts we looked at [EndpointConfiguration](../../2019/06/under-the-hood-of-nsb.md) and [SettingsHolder](../../2023/03/hold-my-settings.md) as the basic building blocks of extending NServiceBus. These classes give us a way to extend configuration but how do we use that configuration to do something?

That's where Features come in.

<!--more-->

A `Feature` is a unit of functionality that can be applied to an endpoint. Here is a simple example:

```cs
class MyFeature : Feature
{
    protected override void Setup(FeatureConfigurationContext context)
    {
        Console.WriteLine("MyFeature got activated");
    }
}
```

To add this to an endpoint you would use this:

```cs
endpointConfiguration.EnableFeature<MyFeature>();
```

There are a lot of `Features` in NServiceBus that all contribute to an endpoint at runtime. [Audit](https://github.com/Particular/NServiceBus/blob/0e1358ff896a8d694f23b73f998c8ba08850efc7/src/NServiceBus.Core/Audit/Audit.cs#L9) is a Feature. [Saga](https://github.com/Particular/NServiceBus/blob/0e1358ff896a8d694f23b73f998c8ba08850efc7/src/NServiceBus.Core/Sagas/Sagas.cs#L12) support is implemented as a feature.

You can even explicitly turn a feature off:

```cs
endpointConfiguration.DisableFeature<MyFeature>();
```

Why would you want to do that if features need to be explicitly turned on in the first place? Because you can mark a feature to be enabled by default:

```cs
class MyFeature : Feature
{
    public MyFeature()
    {
        EnableByDefault();
    }

    protected override void Setup(FeatureConfigurationContext context)
    {
        Console.WriteLine("MyFeature got activated");
    }
}
```

If you do this, then the feature is enabled (and subsequently activated) just by being present in the AppDomain (more on assembly scanning in a later post).

The NServiceBus core library gathers all of the available features at runtime and decides which ones to activate. If you look in the [startup diagnostics file](https://docs.particular.net/nservicebus/hosting/startup-diagnostics), you will find the list of features that were available and whether or not they were activated.

## What can a feature do?

While features can do a lot of different things we have two primary use cases:

1. Extending the message processing and sending pipelines
2. Running something when the endpoint is created and when it is stopped

I'll go more into message pipelines in the next post. For this one I will focus on the second use case.

### FeatureStartupTask

A `FeatureStartupTask` is an abstraction that will run code when the endpoint is created and when it is stopped. Here's a simple example:

```cs
class MyFeatureStartupTask : FeatureStartupTask
{
    public virtual Task OnStart(IMessageSession session, CancellationToken cancellationToken = default)
    {
        Console.WriteLine("MyFeature is started");
        return Task.CompletedTask;
    }

    public virtual Task OnStop(IMessageSession session, CancellationToken cancellationToken = default)
    {
        Console.WriteLine("MyFeature is stopped");
        return Task.CompletedTask;
    }
}
```

Note that the `Start` and `Stop` methods both get handed an `IMessageSession` so they could send messages or publish events if you need them to. You could use this to have endpoints send messages to a monitoring service when they start and stop.

And to add this startup task to our feature:

```cs
class MyFeature : Feature
{
    protected override void Setup(FeatureConfigurationContext context)
    {
        context.RegisterStartupTask(new MyFeatureStartupTask());
    }
}
```

If you need something from the service provider you can pass in a factory method:

```cs
context.RegisterStartupTask(sp => new MyFeatureStartupTask(sp.GetRequiredService<IDoSomethingCool>()));
```

## Feature lifecycle

Features can be in one of 4 states:

- `Disabled`: The feature has not been enabled. This is the default if you do nothing or if you call `DisableFeature'.
- `Enabled`: The feature is enabled and the endpoint will attempt to activate the feature. This is the default if you explicity enable a feature or if you mark it as `EnableByDefault` and don't explicitly deactivate it.
- `Active`: The feature has been activated (it's `Setup` method has been called).
- `Deactivated`: The feature was not disabled but could not be activated (usually because a prerequisite was not met or it was dependent on another feature which was not activated).

## Need input

Some features have specific needs in order to function. Perhaps our feature needs a configuration value. The `FeatureConfigurationContext` provides access to the endpoint settings collection that we looked at in the previous post:

```cs
class MyFeature : Feature
{
    protected override void Setup(FeatureConfigurationContext context)
    {
        context.RegisterStartupTask(new MyFeatureStartupTask(context.Settings.Get<string>("MyConfigValue")));
    }
}
```

What if the configuration value isn't there? Well, you have a couple of options. You could just check for it:

```cs
class MyFeature : Feature
{
    public MyFeature()
    {
        EnableByDefault();
    }

    protected override void Setup(FeatureConfigurationContext context)
    {
        if(context.Settings.TryGet("MyConfigValue", out var configValue))
        {
            context.RegisterStartupTask(new MyFeatureStartupTask(configValue));
        }
    }
}
```

That still means your feature gets activated but it doesn't do anything. To be a nicer citizen you can declare this as an actual prerequisite:

```cs
class MyFeature : Feature
{
    public MyFeature()
    {
        EnableByDefault();
        Prerequisite(context => context.HasSetting("MyConfigValue"), "MyFeature is not configured.");
    }

    protected override void Setup(FeatureConfigurationContext context)
    {
        context.RegisterStartupTask(new MyFeatureStartupTask(context.Settings.Get<string>("MyConfigValue")));
    }
}
```

You can create any number of prerequisite conditions for a feature. If they are all met, then the feature can be activated. If any are not met then the feature is not activated and the provided reason is documented in the endpoint diagnostics.

If you declare a prerequisite, then you can assume it has passed when `Setup` is called.

### Defaults

Sometimes a feature has a required configuration value but there's a reasonable default. You can declare that explicitly:

```cs
public class MyFeature : Feature
{
    public MyFeature()
    {
        Defaults(settings => settings.SetDefault<string>("MyConfigValue", "DefaultValue"));
    }

    // ...
}
```

Remember that setting a default does not override any value that the user has explicitly supplied. You can call `Set` or one of it's overrides here but you probably shouldn't as it means that any value explicitly supplied will be overwritten.

## Feature dependencies

Some features rely on others to perform their task. Maybe your feature adds information to audited messages and there's no point activating it if the audit feature is also not activated. You can declare this dependency in code:

```cs
public class MyFeature : Feature
{
    public MyFeature()
    {
        EnableByDefault();
        DependsOn<Audit>();
    }
}
```

The combination of `EnableByDefault` with `DependsOn` is very powerful. You can build sets of features that turn on and off based on other configuration without getting in to the specifics and without cluttering up the configuration API.

### Activation order

There's another advantage to declaring dependencies. NServiceBus activates features in order so that your `Setup` will be called only after all of your declared dependencies have run their `Setup`. The `Audit` feature sets up the audit pipeline so you can assume that it is there when your feature runs if you have a dependency on `Audit`.

`FeatureStartupTask` instances are all started in the same order as well, so if you have a dependency on a `Feature` then you can assume that all of it's startup tasks have started before your startup task is called. They are stopped in reverse order.

### Depending on things you don't have access to

Note that not all NServiceBus features are public, so you can use an overload of `DependsOn` that accepts the name of the feature as a string:

```cs
public class MyFeature : Feature
{
    public MyFeature()
    {
        EnableByDefault();
        DependsOn("Audit");
    }
}
```

This can also be useful if you need to take a dependency on `Feature` defined in an assembly that you don't have a reference to.

You should be cautious using this because the name of the feature could change and then your feature would not be activated.

### Optional dependencies

Sometimes a feature needs to be activated after another _if it is present_. That is, if the dependency is there it needs to be activated first. If it's not there, then it doesn't prevent your feature from activating.

```cs
public class MyFeature : Feature
{
    public MyFeature()
    {
        DependsOnOptionally<SomeFeatureThatMayNotBePresent>();
        // OR
        DependsOnOptionally("SomeFeatureThatMayNotBePresent");
    }
}
```

### Multiple dependencies

If a feature depends on multiple things, you can simply declare them:

```cs
public class MyFeature : Feature
{
    public MyFeature()
    {
        DependsOn<Audit>();
        DependsOn<Heartbeats>();
    }
}
```

Now the feature will only run if _both_ dependencies are met.

There is another use case. What if you want the feature to be activated if either `Audit` or `Heartbeats` is activated?

```cs
public class MyFeature : Feature
{
    public MyFeature()
    {
        DependsOnAtLeastOne(typeof(Heartbeats), typeof(Audit));
        // OR
        DependsOnAtLeastOne("Heartbeats", "Audit");
    }
}
```

## Under the covers

The work to control features is handled by the `FeatureActivator`. This class is non-public but if it was available you'd be able to use it like this:

```cs
var activator = new FeatureActivator(settings); // settings is a SettingsHolder
activator.Add(new MyFeature());
activator.Add(new MyOtherFeature());

var featureConfigurationContext = new FeatureConfigurationContext(
    settings,
    serviceCollection,
    // other stuff you don't have access to
    pipelineSettings,
    routing,
    receiveConfiguration
);

var diagnosticData = activator.SetupFeatures(featureConfigurationContext);

await activator.StartFeatures(serviceProvider, messageSession, cancellationToken);

// Endpoint runs

await activator.StopFeatures(cancellationToken);
```

Note that unless the feature is `EnableByDefault` it is `settings` that decides whether the feature is enabled or disabled. There's an extension method to set that, but it is non-public:

```cs
settings.EnableFeature(typeof(MyFeature));
```

You can interrogate whether a feature is enabled:

```cs
settings.IsFeatureEnabled(typeof(MyFeature));
```

It is probably better to declare it as a dependency instead. Sometimes we use this to check for specific combinations of features to detect incompatible configurations.

## Putting it all together

With everything we have learned we can write a feature with a startup task:

```cs
class AnnounceLifecycleStartupTask : FeatureStartupTask
{
    readonly string identifier;

    public AnnounceLifecycleStartupTask(string identifier)
    {
        this.identifier = identifier;
    }

    public override Task OnStart(IMessageSession session, CancellationToken cancellationToken)
    {
        return session.Publish(new EndpointStarted { Identifier = identifier });
    }

    public override Task OnStop(IMessageSession session, CancellationToken cancellationToken)
    {
        return session.Publish(new EndpointStopped { Identifier = identifier });
    }
}

class AnnounceLifecycle : Feature
{
    public AnnounceLifecycle()
    {
        Defaults(settings => settings.SetDefault(IdentifierKey, settings.GetEndpointName()));
    }

    public override void Setup(FeatureConfigurationContext context)
    {
        var identifier = context.Settings.Get<string>(IdentifierKey);
        context.RegisterStartupTask(new AnnounceLifecycleStartupTask(identifier));
    }

    public const string IdentifierKey = "AnnounceLifecycleIdentifier";
}
```

And a configuration method to turn it on:

```cs
using NServiceBus.AdvancedExtensibility;

public static class AnnounceLifecycleConfigurationExtensions
{
    public static void AnnounceLifecycle(this EndpointConfiguration endpointConfiguration, string identifier = null)
    {
        endpointConfiguration.EnableFeature<AnnounceLifecycle>();
        if(identifier != null)
        {
            endpointConfiguration.Getsettings().Set(AnnounceLifecycle.IdentifierKey, identifier);
        }
    }
}
```

Hopefully you are beginning to see the power of putting functionality inside of features.

## Next time

In the next post I'll talk about a powerful NServiceBus capability and the other frequent use of features: message pipelines.
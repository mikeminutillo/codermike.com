---
title: "Under the hood of NServiceBus: SettingsHolder"
date: 2023-03-08
---

In the [last post](/post/2019/06/under-the-hood-of-nsb.md) I explained that `EndpointConfiguration` is really just a wrapper around a `SettingsHolder` and how to use the `GetSettings()` extension method in the `AdvancedExtensibility` namespace to get access to it. 

So once you have access to a `SettingsHolder`, what can you do with it?

<!--more-->

In simplest terms, a `SettingsHolder` is a key-value store or dictionary. You can put a value in with a key and then pull that value back out again by providing the same key. Like this:

```cs
var settings = new SettingsHolder();

settings.Set("My Name", "Mike");

var myName = (string)settings.Get("My Name");
```

There are many other methods that help to do this in different ways:

```cs
// Get a strongly typed value
var myName2 = settings.Get<string>("My Name");

// Check if a setting exists before retrieving it
if(settings.HasSetting("My Name"))
{
  // The key isn't case sensitive
  return settings.Get<string>("MY NAME");
}

// Or do it in one step
if(settings.TryGet<string>("my name", out var myName3))
{
  return myName3;
}

// Using a type name as the key
settings.Set<SqlPersistenceSettings>(new SqlPersistenceSettings());
var sqlSettings = settings.Get<SqlPersistenceSettings>();

// Creating it if it's not in there
var sqlSettings2 = settings.GetOrCreate<SqlPersistenceSettings>();
```

## Defaults

The thing that makes `SettingsHolder` special is the concept of defaults. A default is a value that is returned for a given key if no explicit value has been set.

```cs
settings.SetDefault("Greeting", "Hi there");
settings.Set("Greeting", "Howdy");

// Prints "Howdy"
var greeting = settings.Get<string>("Greeting");
Console.WriteLine(greeting);
```

The interesting thing about defaults is that they don't overwrite a set value. If you reverse the first two lines above and set the default after setting the value, you will still get the same greeting printed out at the end.

```cs
settings.Set("Greeting", "Howdy");
settings.SetDefault("Greeting", "Hi there");

// Still prints "Howdy"
var greeting = settings.Get<string>("Greeting");
Console.WriteLine(greeting);
```

This is useful for letting different features configure defaults, even if the user has already specified a value.

You can check if a given key has an explicit value by using `HasExplicitValue(key)` or `HasExplicitValue<T>()`.

## Cleaning up

You can clear a settings collection[^1].

```cs
settings.Clear();
```

This will dispose any of the defaults and settings that have been added to the collection. In NServiceBus we only do this during a shutdown.

## Nobody move

Although it's not exposed publicly, a `SettingsHolder` can be locked to prevent further changes. In NServiceBus this happens during endpoint initialization, before the endpoint is started.

NServiceBus should never be hand you a `SettingsHolder` if you can't make changes to it. Instead, you get get given an instance of the `ReadOnlySettings` interface. This has all of the methods for getting data out of a `SettingsHolder` but none of the ones for setting values or defaults. Even if you cast back to a `SettingsHolder`, any calls that modify the collection will throw an `Exception`.

If you had access to the API, it would look like this:

```cs
settings.PreventChanges();
```

## With great power

`SettingsHolder` is not overly complex but it's important to understand if you are going to get under the hood with NServiceBus. Nearly every feature touches this class in some capacity.

There are a couple of rules of engagement if you are going to use it.

1. Do your best to make sure your key is unique. There is one `SettingsHolder` per endpoint. If you set a value or a default with the same key as another feature, something unexpected could happen. If you're lucky, you'll get an error quickly. If you are unlucky, you'll introduce a subtle bug that's hard to pin down.
2. Be wary of secret knowledge. I know that the endpoint name is stored in a key called `NServiceBus.Routing.EndpointName`. I can use code to go and grab that value (or even change it). It's a dangerous thing to do. If the core NServiceBus library decides to store the endpoint name in a different place (or as a different type using the same key) then my code will break at runtime. In most cases, if you're supposed to have access to it, NServiceBus will provide a way to get it. In the case of the endpoint name, there's an extension method on `ReadOnlySettings` called `EndpointName()`. 

## In use

This is what the simplest NServiceBus endpoint startup code looks like:

```cs
var config = new EndpointConfiguration("EndpointName");
config.UseTransport<LearningTransport>();
config.UsePersistence<LearningPersistence>();

var endpoint = await Endpoint.Start(config);
```

The call to `UseTransport` is a configuration extension method (so is the call to `UsePersistence`) and under the covers it's putting an instance of `LearningTransport` into the settings as a `TransportDefinition`. You can even ask for it back:

```cs
var transportDefinition = config.GetSettings().Get<TransportDefinition>();
```

## Next time

We've seen how to set settings and defaults but how do we use them? That's what features are for.

[^1]: You shouldn't clear up the one NServiceBus gives you
---
title: "Optimal Logging API for C#"
date: 2022-05-18T20:48:33+02:00
draft: true
hidden: true
tags: ["programming", "csharp"]
---

There are often situations where logging can affect performance. It is probably rare in the server world but it comes quite often when we try to log stuff in our Unity game. There its effect can be quite visible.

That is why, so far, we disabled all logger calls in our production builds by using a `[Conditional("USE_LOGGING")]` attribute on logger methods and not defining the `USE_LOGGER` symbol in the build. You can read about that attribute [here](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.conditionalattribute?view=net-6.0).

But our project grew and grew and at one point became complex enough that we had to reconsider. We needed an option to leave the logging lines in the code but have them disabled by default unless the backend server tells the game to enable some log levels for some tags. So the logging line would look something like this:

```cs
// First the logger is defined somewhere in the class with the name of the class as the tag:
private Logger logger = new Logger(nameof(MyClass));

// The call then looks like this:
logger.Info($"User {userId} is starting a match {matchId} with {injuredNum} injured players.");
```

The code inside that method would check if `Info` level is enabled for tag `MyClass` and wouldn't log
the message if it isn't. Here we were still concerned that the string interpolations would create enough garbage for the Garbage Collector that its cleanups would again slow down our game. Now, if you did any C# programming you will now be thinking: "Why are you using string interpolations? That is not how you should use the logger API. You should pass in a format string in arguments separately.". You would be right, but it also doesn't really stop you from using it that way. Also there is already a lot of code in our 7 years old project that uses code like this. Even if we did it the "correct" way it would still need to execute code that generates the values for arguments and to box those arguments only to then conclude it doesn't need any of that inside the method because logs are disabled.

The best solution we found online is a great [Zero Allocation Logger](https://github.com/Cysharp/ZLogger/). It also avoids boxing of parameters by having many overloads for logging methods with different number of generic parameters. It would still require that we refactor our huge code base never to use string interpolation and the api still expects string as a first parameter so it can't force the developer not to use string interpolation.

In the end we did some measurements and concluded that impact isn't as bad as we thought so we will try to just roll with what we have. But then I had a...

## Brilliant Idea

So, from performance standpoint it would be ideal if our developers wouldn't hate to always type:

```
if (logger.IsLevelEnabled(LogLevel.Info)) {
    logger.Info($"User {userId} is starting a match {matchId} with {injuredNum} injured players.");
}
```

But of course they do. Not just to type but also to read. But what if we make it much less painful? Something like this might be acceptable:


```cs
logger.Info?.Log($"User {userId} is starting a match {matchId} with {injuredNum} injured players.");
```

`logger.Info` is now a property that returns an optional struct. If the logging is disabled it will return null and the part after `?.` will not be executed at all. Unity still doesn't support all types to be nullable like .net6 does. That is why it must return an optional struct so that the compiler in Unity won't allow you to just call the `.Log()` method directly. It will only compile if you call it using`?.Log()`.

If we wanted to convert all our log calls to this we could just execute Regex search and replace a few times.

We haven't yet tried to put this idea into practice but it felt too cool not to share immediately.
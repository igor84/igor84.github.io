---
title: "Optimal Logging API for C#"
date: 2022-05-19T20:40:33+02:00
draft: false
hidden: true
tags: ["programming", "csharp"]
---

There are often situations where logging can affect performance. It is probably rare in the server world but it comes quite often when we try to log stuff in our Unity game. There its effect can be quite visible.

That is why, so far, we disabled all logger calls in our production builds by using a `[Conditional("USE_LOGGING")]` attribute on logger methods and not defining the `USE_LOGGER` symbol in the build. You can read about that attribute [here](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.conditionalattribute?view=net-6.0).

But our project grew and grew and at one point became complex enough that we had to reconsider. We needed an option to leave the logging lines in the code but have them disabled by default unless the backend server tells the game to enable some log levels for some tags. So the logging line would look something like this:

```cs
// logger instance is created somewhere with some
// specific tag that it will add for each log line

// The call then looks like this:
logger.Info($"User {userId} from {cityName} logged in on {logInTime}.");
```

The code inside that method would check if `Info` level is enabled for that logger's tag and
wouldn't log the message if it isn't. Here we were still concerned that the string interpolations
would create enough garbage for the Garbage Collector that its cleanups would again slow down our
game. Now, if you did any C# programming you will now be thinking: "Why are you using string
interpolations? That is not how you should use the logger API. You should pass in a format string in
arguments separately.". You would be right, but it also doesn't really stop you from using it that
way. Also there is already a lot of code in our 7 years old project that uses code like this. Even
if we did it the "correct" way it would still need to execute code that generates the values for
arguments and to possibly box those arguments only to then conclude it doesn't need any of that
inside the method because logs are disabled for that tag and level.

The best solution we found online is a great [Zero Allocation Logger](https://github.com/Cysharp/ZLogger/).
It also avoids boxing of parameters by having many overloads for logging methods with different number
of generic parameters. It would still require that we refactor our huge code base never to use string
interpolation and the api still expects string as a first parameter so it can't force the developer not
to use string interpolation.

In the end we did some measurements and concluded that impact isn't as bad as we thought so we will try to
just roll with what we have. But then I had a...

## Brilliant Idea

So, from performance standpoint it would be ideal if our developers wouldn't hate to always type:

```cs
if (logger.IsEnabledFor(LogLevel.Info)) {
    logger.Info($"User {userId} from {cityName} logged in on {logInTime}.");
}
```

But of course they do. Not just to type but also to read. There comes my brilliant idea. What if
`Info` is a property instead of a method and that property returns an optional struct that has the
`Log()` method. Then the usage would look like this:

```cs
logger.Info?.Log($"User {userId} is starting a match {matchId} with {injuredNum} injured players.");
```

This is much less painful to type and read. If the logging is disabled the property will return null
and the part after `?.` will not be executed at all. On the other hand if it is enabled it will
return the struct which doesn't allocate any memory on the heap. If you accidentally try to call the
`.Log()` method without `?.` you will get a compile error.

If we wanted to convert all our log calls to this we could just execute Regex search and replace a
few times.

We haven't yet tried to put this idea into practice but it felt too cool not to share immediately.

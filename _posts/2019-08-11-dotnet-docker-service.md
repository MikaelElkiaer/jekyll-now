---
layout: post
title: Using .NET Core for Docker Services
---

I recently implemented my first Docker image intended to be used as a service: [Beehive](https://github.com/MikaelElkiaer/Beehive).
I have previously used Topshelf to create .NET Windows services, but this time around I would like to create a .NET Core console app running as a Docker service in a Linux-based image.
This brought with it some interesting findings.

## Creating a stoppable service
In the case of Beehive, I wanted to do an isolated loop - in order to not having to concern myself with managing state.
I also wanted a way to stop the service, while having a chance to finish ongoing scheduling and perhaps do some cleaning up.
In order to achieve this, I stumbled upon a neat little trick of combining a while loop and a `CancellationToken`.

Simply, a `CancellationToken` is an object you can provide for any `Task` that you start, so that you can check for `.IsCancellationRequested` and either exit prematurely, but gracefully, or throw an `OperationCanceledException` via `ThrowIfCancellationRequested()`.
Combining this with an async main function, we can create a cancellable endless loop:
```csharp
class Program
{
    static CancellationTokenSource tokenSource = new CancellationTokenSource();

    async Task main(string[] args)
    {
        while (!tokenSource.IsCancellationRequested)
        {
            DoLogic();
            if (ShouldExit())
                tokenSource.Cancel();
        }
    }
}
```
This is fine for handling logic-related reasons to exit.
But for this to be a real system-controllable service, we also need for it to react to signals.

First up is the `Console.CancelKeyPress` event, which is fired when hitting Ctrl+C - corresponding to a SIGINT on Linux.
This is useful for gracefully stopping the program - while running it directly or while running it as a non-detached container.
In the event handler, it is neccesary to set `ConsoleCancelEventArgs.Cancel` to true.
If it is kept at its default (false) it will kill the process once the event handler is finished.
If set to true, we get to do a graceful exit via the CancellationToken.

Next up is the `AppDomain.ProcessExit` event - corresponding to a SIGTERM on Linux.
This is fired after the cancel, but more importantly it is also fired by itself on a SIGTERM - e.g. whenever a container is stopped or restarted by the Docker daemon.

Extending on the program above, we now have a stoppable service:
```csharp
class Program
{
    static CancellationTokenSource tokenSource = new CancellationTokenSource();

    async Task main(string[] args)
    {
        AppDomain.CurrentDomain.ProcessExit += CurrentDomain_ProcessExit;
        Console.CancelKeyPress += Console_CancelKeyPress;
        while (!tokenSource.IsCancellationRequested)
        {
            DoLogic();
            if (ShouldExit())
                tokenSource.Cancel();
            await Task.Delay(TimeSpan.FromSeconds(10), tokenSource.Token);
        }
    }

    static void CurrentDomain_ProcessExit(object sender, EventArgs e)
    {
        tokenSource.Cancel();
    }
   
    static void Console_CancelKeyPress(object sender, ConsoleCancelEventArgs e)
    {
        e.Cancel = true;
        tokenSource.Cancel();
    }
}
```

### Creating a non-loop service
In other cases, it might not be neccesary with a loop to have the service go forever.
Perhaps the service starts some background threads for subscribing to a message queue or another asynchronous form of communication.

For this it is possible to instead set up the background thread and then utilize `Task.Delay` again, but without an actual delay and instead only relying on the `CancellationToken` to stop the service.
```csharp
// ...
async Task main(string[] args)
{
    AppDomain.CurrentDomain.ProcessExit += CurrentDomain_ProcessExit;
    Console.CancelKeyPress += Console_CancelKeyPress;

    SubscribeToQueue();

    await Task.Delay(-1, tokenSource.Token);
}
// ...
```

## Logging
This section is about Serilog and how to take advantage of it for a console application running inside a container.

By using the simple Serilog configuration below, we get the following advantages right out-of-the-box:
1. Structured (template-based) log events
2. Multiple log levels (Verbose, Debug, Information, Warning, Error, Fatal)
3. Logging to stderr and stdout
4. Nicely formatted log entries with timestamp and error level
5. Globally available or DI-driven ILogger
6. Colored output

A single package is needed to be installed: `Serilog.Sinks.Console`.
In order to set it up I prefer the programmatic approach over using a config file:
```csharp
public static ILogger CreateLogger(LogEventLevel visibleLogLevel)
{
    return Log.Logger = new LoggerConfiguration()
        .MinimumLevel.Is(visibleLogLevel) # Allow users to set log level
        .WriteTo.ColoredConsole(standardErrorFromLevel: LogEventLevel.Error) # Send errors and fatals to stderr
        .CreateLogger();
}
```
Note the use of `standardErrorFromLevel` which needs to be set for log events of level Error and Fatal to be logged to stderr.
It might also be useful to let the minimum log level be controlled via an environment variable - allowing the user of the service to determine the visible log level.
Once that it is set up, all that needs to be done is use it - either via the globally available `Log.Logger` or by injecting the `ILogger` via a DI framework.

### Example
A log entry is created as such:
```csharp
logger.Information("Log event at {StartUtc:o}", DateTime.UtcNow)
```
and will be outputted as:
```
2019-08-11 20:14:49 INF Log event at 2019-08-11T18:14:49.8269445Z
```

## Configuration
Docker services are easily controlled via environment variables.
In .NET (Core) environment variables are read via `System.Environment` - typically via `Environment.GetEnvironmentVariable(KEYNAME)`.
By using environment varibles for configuration, it is easy to create a single build and configure it for uses in different environments.
Such as swapping out log levels, database connectionstrings, etc.

## Building
So far I've managed to keep it simple - my go-to configuration is:
```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.2 AS build-env
COPY . ./
RUN dotnet test
RUN dotnet publish -c Release -o /out

FROM mcr.microsoft.com/dotnet/core/runtime:2.2-alpine3.9
COPY --from=build-env /out .
ENTRYPOINT ["dotnet", "Program.dll"]
```
This configuration uses the larger SDK image for testing and publishing the service.
Then it creates another image based on a lighter runtime image, by copying in the published program from the previous image.

I won't go into detail about how to deploy this.
Personally, I am using Drone CI to publish my images to both a local registry and to Docker Hub.

# Conclusion
There we have it - a runnable, stoppable, logging, configurable, linux-based .NET Core Docker service.
What is next up for me? Probably to create some kind of framework to contain this - probably something similar to Topshelf.

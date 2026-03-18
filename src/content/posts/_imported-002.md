+++
date = '2021-03-17T09:24:57Z'
title = '[Imported] .NET 6.0 console app - Configuration, tricks and tips'
description = 'Imported post about how to use the new .NET 6.0 console app template and its configuration system'
url = 'dotnet-console-app'
tags = [
    "imported-devto",
    "dotnet",
    "dev",
    "console-app"
]
+++

**This post was originally published on [Dev.to](https://dev.to/krusty93/net-core-5-0-console-app-configuration-trick-and-tips-c1j).**

One of the biggest .NET benefit is the flexibility and the portability of its features.
As ASP.NET Core relies on the `IHostBuilder`, we can do the same with a classic console application.
This approach will help us to have a basic infrastructure in our application, including the support for logging, dependency injection, app settings and so on.

To start with this, add the `Microsoft.Extensions.Hosting` NuGet package to your project and replace the "Hello World template" with this code:

```csharp
class Program
{
    static async Task<int> Main(string[] args)
    {
        var host = CreateHostBuilder();
        await host.RunConsoleAsync();
        return Environment.ExitCode;
    }

    private static IHostBuilder CreateHostBuilder()
    {
        return Host.CreateDefaultBuilder();
    }
}
```

Doing this, we are asking for an `HostBuilder` instance with the default infrastructure implementation.
The `RunConsoleAsync` starts the host builder with the "classic console" capabilities, such as the listener for the CTRL+C shortcut. Finally, it returns the application exit code (default is 0).
This is a starting point; now we can extend our capabilities.

## IHostedService

The `Main` method is the application entry point, but we now need to continue the execution of our app. To achieve this, add a class that implement the `IHostedService` interface and register it to the IoC container using the extension method `AddHostedService`:

```csharp
private static IHostBuilder CreateHostBuilder()
{
    return Host.CreateDefaultBuilder()
        .ConfigureServices(services =>
        {
            services.AddHostedService<Worker>();
        });
}

public class Worker : IHostedService
{
    public Task StartAsync(CancellationToken cancellationToken)
    {
        throw new NotImplementedException();
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        throw new NotImplementedException();
    }
}
```

Now at the startup, the application runs the `StartAsync` method;\
Instead `StopAsync` is executed when the is going to be terminated.

## Use the Dependency Injection

We can now use the dependency injection system provided by .NET.
In the `ConfigureServices` method, add the services you need to run your application:

```csharp
public class MyService : IMyService
{
    public async Task PerformLongTaskAsync()
    {
        await Task.Delay(5000);
    }
}

public interface IMyService
{
    Task PerformLongTaskAsync();
}

public class Worker : IHostedService
{
    private readonly IMyService _myService;

    public Worker(IMyService service)
    {
        _myService = service ?? throw new ArgumentNullException(nameof(service));
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        await _myService.PerformLongTaskAsync();
    }
}

private static IHostBuilder CreateHostBuilder()
{
    return Host.CreateDefaultBuilder()
        .ConfigureServices(services =>
        {
            services.AddHostedService<Worker>();

            services.AddTransient<IMyService, MyService>();
        });
}
```

## Logging and configuration

Using the `HostBuilder`'s `ConfigureLogging` extension method we have a full access to the logging configuration. In this case, we want to replace the default .NET implementation with one of the most used logging library, Serilog.\
First of all, install Serilog NuGet packages:

- `Serilog.Extensions.Hosting`
- `Serilog.Settings.Configuration`
In this example, we are going to use the Console and the File sinks:
- `Serilog.Sinks.Console`
- `Serilog.Sinks.File`

Our goal is to log events in a log file when running in a `Production` environment; we stick instead to the console when debugging the app.\
To start with this, edit the `CreateHostBuilder` method as follow:

```csharp
return Host.CreateDefaultBuilder()
    .ConfigureLogging(logging =>
    {
        logging.ClearProviders();
    })
    .UseSerilog((hostContext, loggerConfiguration) =>
    {
        loggerConfiguration.ReadFrom.Configuration(hostContext.Configuration);
    })
    .ConfigureServices(services =>
    {
        services.AddHostedService<Worker>();

        services.AddTransient<IMyService, MyService>();
    });
```

The `logging.ClearProviders()` is used to clear all the default `HostBuilder` logging providers. These are:

- Console
- Debug
- Event Source
- EventLog (Windows only)

Then we read from the `appsettings.json` (added in the next paragraph) the Serilog configuration using `loggerConfiguration.ReadFrom.Configuration(hostContext.Configuration);`.

Now it's time to give a configuration to our app. Add the `appsettings.json` file to your project (ensure to set the `BuildAction` to `Content` and the `Copy to Output Directory` to `Copy if newer`). Then, paste the Serilog's configuration:

```json
"Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "File",
        "Args": {
          "path": "log.txt",
          "rollingInterval": "Day",
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}",
          "shared": true
        }
      }
    ],
    "Enrich": [ "FromLogContext" ]
  }
```

The snippet above is setting the Serilog default level to `Information` (with some override for the `Microsoft` and `System` namespaces) and configure the `File` sink. In particular, it rolls out a different log file each day.\
But as stated above, we want to redirect all the application output to the console when debugging the app. Then we need to setup the `appsetting.Development.json` file to the project and override the Serilog configuration, changing the default minimum level and setting the console as output provider:

```json
"Serilog": {
    "MinimumLevel": {
      "Default": "Debug",
      "Override": {
        "Microsoft": "Information",
        "System": "Information"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "theme": "Serilog.Sinks.SystemConsole.Themes.AnsiConsoleTheme::Code, Serilog.Sinks.Console",
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} <s:{SourceContext}>{NewLine}{Exception}"
        }
      }
    ],
    "Enrich": [ "FromLogContext" ]
  }
```

We don't need to do anything else to support different environemnts than defining them in the `launchSettings` file. Add a folder called `Properties` and a file named `launchSettings.json`:

```json
{
    "profiles": {
        "Development": {
            "commandName": "Project",
            "environmentVariables": {
                "DOTNET_ENVIRONMENT": "Development"
            }
        },
        "Production": {
            "commandName": "Project",
            "environmentVariables": {
                "DOTNET_ENVIRONMENT": "Production"
            }
        }
    }
}
```

The `DOTNET_ENVIRONMENT` variable is automatically read from the `HostBuilder`. It is possible to customize them, more info are available in the [official doc](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-6.0).\

We are now ready to use the logger and eventually the values coming from the app settings files. Inject the `ILogger` and the `IConfiguration` instances in your classes:

```csharp
public class Worker : IHostedService
{
    private readonly IMyService _myService;
    private readonly string _configKey;
    private readonly ILogger<Worker> _logger;

    public Worker(IMyService service, IConfiguration configuration, ILogger<Worker> logger)
    {
        _myService = service ?? throw new ArgumentNullException(nameof(service));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _configKey = configuration["ConfigKey"];
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Read {key} from settings", _configKey);

        await _myService.PerformLongTaskAsync();
    }

...
}
```

*NOTE: In order to make this code working, you need to add a `ConfigKey` property in your app settings. If you do the same in the `appsettings.Development.json` but with a differenva lue, you could notice that value changes depending on the current environment*

We could also add the user secrets and environment variables support using:

```csharp
.ConfigureLogging(logging =>
{
    ...
})
.UseSerilog((hostContext, loggerConfiguration) =>
{
    ...
})
.ConfigureAppConfiguration((hostContext, builder) =>
{
    builder.AddEnvironmentVariables();

    if (hostContext.HostingEnvironment.IsDevelopment())
    {
        builder.AddUserSecrets<Program>();
    }
})
```

## Terminates the app properly using Exit Code

Injecting the `IHostApplicationLifetime` instance in the `Worker` class, we can terminate properly the console process when the job is done. Calling `StopApplication` is enough and it will redirect the running path to the `StopAsync` method. Moreover, we can use Exit Code to tell the user why the app stopped:

```csharp
private readonly IHostApplicationLifetime _hostLifetime;
private int? _exitCode;
...

public Worker(IMyService service, IConfiguration configuration, IHostApplicationLifetime hostLifetime, ILogger<Worker> logger)
{
    ...
    _hostLifetime = hostLifetime ?? throw new ArgumentNullException(nameof(hostLifetime));
}

public async Task StartAsync(CancellationToken cancellationToken)
{
    _logger.LogInformation("Read {key} from settings", _configKey);

    try
    {
        await _myService.PerformLongTaskAsync();

        _exitCode = 0;
    }
    catch (OperationCanceledException)
    {
        _logger?.LogInformation("The job has been killed with CTRL+C");
        _exitCode = -1;
    }
    catch (Exception ex)
    {
        _logger?.LogError(ex, "An error occurred");
        _exitCode = 1;
    }
    finally
    {
        _hostLifetime.StopApplication();
    }
}

public Task StopAsync(CancellationToken cancellationToken)
{
    Environment.ExitCode = _exitCode.GetValueOrDefault(-1);
    _logger?.LogInformation("Shutting down the service with code {exitCode}", Environment.ExitCode);
    return Task.CompletedTask;
}
```

## Use command line arguments efficiently

One of the better way to handle the command line arguments in your app, is through the `McMaster.Extensions.CommandLineUtils` NuGet package.
It is a fork of the deprecated Microsoft library and allow you to easily manage the input commands. I show you a brief example of use. You need to modify the `Program.cs` as follow:

```csharp
[Command(
    Name = "MyApp"
    , Description = "My app is very cool 😎")]
[HelpOption(
    "-h"
    , LongName = "help"
    , Description = "Get info")]
[VersionOptionFromMember(
    "-v"
    , MemberName = nameof(GetVersion))]
class Program
{
    [Option(
        "-o"
        , "Some option"
        , CommandOptionType.SingleValue
        , LongName = "option")]
    [Range(1, 5)]
    public int Option { get; } = 1;

    public static Task<int> Main(string[] args) =>
        CommandLineApplication.ExecuteAsync<Program>(args);

    public async Task<int> OnExecuteAsync(CancellationToken cancellationToken)
    {
        var host = CreateHostBuilder();
        await host.RunConsoleAsync(cancellationToken);
        return Environment.ExitCode;
    }

private static IHostBuilder CreateHostBuilder() {...}

private static string GetVersion()
    => typeof(Program).Assembly.GetCustomAttribute<AssemblyInformationalVersionAttribute>()?.InformationalVersion;
```

Here's a description:

- `Command`: gives a name and a description to you app
- `HelpOption`: enables the `-h` (or `--help`) option
- `VersionOption`: returns the app version number
- `Option`: represents the value passed as input argument to your app

Then, the Main method needs some refactoring. Instead, the `OnExecuteAsync` is called automatically by the framework (it becomes the old `Main` method basically). Not that `OnExecuteAsync` is **not** static, so it can access to the class properties such has `Option` - that you can register in the IoC container using the `IOptions` pattern:

```csharp
public class MyOptions
{
    public int Value { get; set; }
}

class Program
{

    [Option(
        "-o"
        , "Some option"
        , CommandOptionType.SingleValue
        , LongName = "option")]
    [Range(1, 5)]
    public int Option { get; } = 1;

...

.ConfigureServices(services =>
    {
        services.AddHostedService<Worker>();

        services.Configure<MyOptions>(options =>
        {
            options.Value = Option;
        });

        services.AddTransient<IMyService, MyService>();
    });

...

public class Worker : IHostedService

    private readonly MyOptions _options;

    public Worker(IMyService service, IConfiguration configuration, IHostApplicationLifetime hostLifetime, ILogger<Worker> logger, IOptions<MyOptions> options)
    {
        ...
        _options = options?.Value ?? throw new ArgumentNullException(nameof(options));
        ...
    }
```

Here the `CommandLineUtils` in action:

{{< cloudinary src="v1773852690/https___dev-to-uploads.s3.amazonaws.com_uploads_articles_gqt98q7ddrdk45zzisey_ghpumo.webp" alt="Localized app" >}}

Unfortunately my console doesn't support emoji yet 😒

## Conclusions

I hope this tutorial gave you the right hint to start your next project based on a .NET console application! These are just an introduction of all the features you can add, basically with no limits! Yeah, it is true: you can use everything you are already using in your ASP.NET Core projects through the `IHostBuilder` interface. Sounds good, isn't it? 🐱‍👤

As usual the source code is available in my [GitHub profile](https://github.com/Krusty93/NetCoreConsoleAppTips).

+++
date = '2022-05-30T21:58:57Z'
title = "[Imported] Implementing health checks PT.1 - Asp.Net Core 6 configuration"
description = 'Imported post about how to implement health checks in an Asp.Net Core 6 application and how to configure them to monitor the application availability and connectivity with the database'
url = 'aspnet-healthcheck-pt1'
tags = [
    "imported-devto",
    "dotnet",
    "csharp",
    "tutorial",
    "devops"
]
+++

**This post was originally published on [Dev.to](https://dev.to/krusty93/implementing-health-checks-pt1-aspnet-core-6-configuration-6gp).**

1. Implementing health checks PT.1 - Asp.Net Core 6 configuration (this post)
2. [Implementing health checks PT.2 - Azure Application Insights configuration](/aspnet-healthcheck-pt2/)

---

When a production running application is not available to the customers due to technical issues, we are not only losing money but also losing trust and faith in our customers. In fact, they may spread a bad word about our product on social media, friends, store reviews or - even worst - move to competitors.

_Basically, when an application is not available to the stakeholders, it is as if it didn't exist._

Of course it may happen. It might because of a human mistake, an erroneous configuration or a bug. Or maybe because our cloud provider has some connectivity issues.

Monitoring the application availability it also allows you to elaborate some metrics to define SLA of your service.

For these reasons, we must know immediately when application goes offline.

One of the most used and simplest techniques are health checks endpoint.
An health check endpoint, is an API exposed by an application that has no logic except for replying with a positive status code; if the caller doesn't receive any response or the status code is not the one expected, it means there is an error and an alert must be sent using the technology of your choice (email, push notification.. SMS and so on).

Of course, Asp.Net Core provides a built-in mechanism to implement an health check endpoint.

## Implementing a simple health check endpoint

Create an Asp.Net Core project. For this demo (you can find the source code at the end of this post) I am going to use dotnet 6 with the new minimal host startup template.

Install the NuGet package `Microsoft.Extensions.Diagnostics.HealthChecks` and configurate it:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks();

var app = builder.Build();

app.MapHealthChecks("/alive");

app.Run();
```

Run the application and navigate to `localhost:<port>/alive` and you should see something like this:

{{< cloudinary src="v1773854719/ffacj3u3fq62xrjek6dq_oneaed.png" alt="Healthy endpoint" caption="Healthy endpoint" >}}

You if you try to call the same endpoint with Postman or a similar tool, you see the status code is `200 OK`.

And that's it - the health check endpoint is ready to be regularly consumed by a watch dog application 🎉

### Advanced scenario

It might be a scenario where the application is online but it can't communicate with the database for some reason. Potentially it is a big disservice to our customers and it is likely that users cannot perform almost all of the actions.

#### Entity Framework Health Checks

Also Entity Framework provides a built-in mechanism to monitor the connectivity between an application and the SQL instance database.

_NB: to go ahead, of course you must have Entity Framework configured and pointing to a running SQL database_

Install the package `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore` from NuGet and configure it:

```csharp
services
  .AddHealthChecks()
  .AddDbContextCheck<MyDbContext>("/dbcontext");
```

Running the application and pointing again to `localhost:<port>/alive` you should have the same result:

{{< cloudinary src="v1773854790/uyovhhnz510jw575l133_bfrvj8.png" alt="Healthy dbcontext endpoint" caption="Healthy dbcontext endpoint" >}}

By the default, under the hood it checks for:
> The DbContextHealthCheck calls EF Core's CanConnectAsync method. You can customize what operation is run when checking health using AddDbContextCheck method overloads.
> The name of the health check is the name of the TContext type.

It now works but there is still a problem: no matter what health check endpoint we hit (`/alive` or `/dbcontext`), both checks are performed so exposing two different APIs is useless.

How is it possible to set different endpoints for different checks? The answer is, through health checks names.

Change the endpoints registration as follow:

```csharp
const string ALIVE = "alive";
const string ALIVE_DBCONTEXT = "dbcontext";

...
builder.Services.AddHealthChecks().AddDbContextCheck<MyDbContext>(name: ALIVE_DBCONTEXT);

app.MapHealthChecks(
    "/alive",
    new HealthCheckOptions { Predicate = (c) => c.Name == ALIVE });

app.MapHealthChecks(
    "/dbcontext",
    new HealthCheckOptions { Predicate = (c) => c.Name == ALIVE_DBCONTEXT });
```

In this case, we are giving a name to a specific health check endpoint. When the name is matched, only the specific health check is executed.

To have a confirmation, we can do a test:

- run the application and the database in a local Docker container
- navigate to `localhost:<port>/alive`: you should get `Healthy` as result
- stop the database container and refresh the page: you should still get `Healthy` as result
- navigate to `localhost:<port>/dbcontext`: you should get `Unhealthy` as result
- restart the database container and refresh the page: you should get `Healthy` as result

#### Custom checks: Azure Cosmos DB

Asp.Net allows to define some custom code while performing an health check. This is the case where we can test our connection to an Azure Cosmos DB instance, because the .NET SDK doesn't provide any mechanism to check the connection between the application and the database service.

The following example tests the connection using `container.ReadContainerAsync()`, a method exposed by the Cosmos SDK that needs database access to be executed.

```csharp
// Cosmos initialization

builder.Services
    .AddHealthChecks()
    .AddDbContextCheck<MyDbContext>(name: ALIVE_DBCONTEXT)
    .AddCheck<CosmosHealthChecker>(name: ALIVE_COSMOS);

...

app.MapHealthChecks(
    "/cosmos",
    new HealthCheckOptions { Predicate = (c) => c.Name == ALIVE_COSMOS });
```

```csharp
public class CosmosHealthChecker : IHealthCheck
{
    private readonly IServiceScopeFactory _serviceScopeFactory;
    private readonly IConfiguration _configuration;
    private readonly ILogger<CosmosHealthChecker> _logger;

    public CosmosHealthChecker(
        IServiceScopeFactory scopeFactory,
        IConfiguration configuration,
        ILogger<CosmosHealthChecker> logger)
    {
        _serviceScopeFactory = scopeFactory;
        _configuration = configuration;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            string collectionName = _configuration["Cosmos:Collection"];

            using IServiceScope scope = _serviceScopeFactory.CreateScope();
            Database db = scope.ServiceProvider.GetRequiredService<Database>();
            Container container = db.GetContainer(collectionName);

            await container.ReadContainerAsync(cancellationToken: cancellationToken);
            return new HealthCheckResult(HealthStatus.Healthy);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error pinging Cosmos db");

            return new HealthCheckResult(
                context.Registration.FailureStatus, "An unhealthy result.");
        }
    }
}
```

## Conclusions

This post explains how we can provide some health check endpoints in an Asp.Net project with just few lines of code 😎

As usual the source code is available in my [GitHub profile](https://github.com/Krusty93/HealthChecks).

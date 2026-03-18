+++
date = '2021-12-24T08:36:57Z'
title = '[Imported] Hide sensitive data in Azure Application Insights logs of our ASP.NET (Core) APIs'
description = 'Imported post about how to sensitive information from Application Insights logs'
url = 'application-insights-sensitive-data'
tags = [
    "imported-devto",
    "dotnet",
    "dev",
    "application-insights",
    "azure",
    "aspnet"
]
+++

**This post was originally published on [Dev.to](https://dev.to/krusty93/hide-sensitive-data-in-azure-application-insights-logs-of-our-aspnet-core-apis-4ji7).**

Azure Application Insights is a wonderful tool to log data and errors in your (not only) .NET applications.

But the Microsoft SDKs don't solve an issue I faced in one of the project I worked on: would it be possible to hide sensitive data from logs? Nobody should be able to read user sensitive data, especially from logs.

## How we used to log APIs payloads

We have various log sources in our distributed system and one of them is the APIM (Azure Api Management): with a simple checkbox you can log APIs requests and responses bodies. But since it is a _simple checkbox_, you can't do anything more than that.
So, we moved our logging system out of it.

{{< cloudinary src="v1773853088/4eukiiei5yk9vof93g1n_sisots.png" alt="APIM log checkbox" caption="APIM log checkbox" >}}

## Where we log APIs payloads now

The easiest way to do log in front of all the APIs in an ASP.NET Core project, is to add a `FilterAttribute` class to the ASP.NET action invocation pipeline:

``` csharp
// Startup.cs
protected override void ConfigureMvc(MvcOptions options)
{    
    options.Filters.Add<ApplicationInsightsActionFilterAttribute>();    
}

// ApplicationInsightsActionFilterAttribute.cs
public class ApplicationInsightsActionFilterAttribute : ActionFilterAttribute
{
    internal const string REQUEST_TELEMETRY_KEY = "Request-Body";
    internal const string RESPONSE_TELEMETRY_KEY = "Response-Body";

    private readonly ILogger<ApplicationInsightsActionFilterAttribute> _logger;

    public ApplicationInsightsActionFilterAttribute(
        ILogger<ApplicationInsightsActionFilterAttribute> logger)
    {
        _logger = logger;
    }

    public override async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        if (context.HttpContext is null)
        {
            await next();
            return;
        }

        try
        {
            string fromBodyParameter = context.ActionDescriptor.Parameters
                .Where(item => item.BindingInfo.BindingSource == BindingSource.Body)
                .FirstOrDefault()
                ?.Name.ToUpperInvariant();

            if (string.IsNullOrWhiteSpace(fromBodyParameter))
                return;

            KeyValuePair<string, object> arguments = context.ActionArguments
                .First(x => x.Key.ToUpperInvariant() == fromBodyParameter);

            string argument = SerializeDataUsingObfuscator(arguments.Value);

            RequestTelemetry request = context.HttpContext.Features.Get<RequestTelemetry>();
            request.Properties.Add(REQUEST_TELEMETRY_KEY, argument);
        }
        catch (Exception ex)
        {
            _logger.LogCritical(ex, "Error executing the response logger task");
        }
        finally
        {
            await next();
        }
    }

    public override async Task OnResultExecutionAsync(ResultExecutingContext context, ResultExecutionDelegate next)
    {
        if (context?.HttpContext is null)
        {
            await next();
            return;
        }

        try
        {
            if (context.Result is ObjectResult result &&
                result.Value != null)
            {
                string serializedObj = SerializeDataUsingObfuscator(result.Value);
                RequestTelemetry request = context.HttpContext.Features.Get<RequestTelemetry>();
                request.Properties.Add(RESPONSE_TELEMETRY_KEY, serializedObj);
            };
        }
        catch (Exception ex)
        {
            _logger.LogCritical(ex, "Error executing the response logger task");
        }
        finally
        {
            await next();
        }
    }

    private static string SerializeDataUsingObfuscator(object value) =>
        JsonConvert.SerializeObject(value,
            new JsonSerializerSettings
            {
                ContractResolver = new ObfuscatorContractResolver()
            });
}

```

The code above is defining two methods: `OnActionExecutionAsync` and `OnResultExecutionAsync`. The first one is fired before the incoming request triggers the controller action; the latter is fired when the controller action returns. Let's focus on `OnActionExecutionAsync` for a moment: firstly it gets the controller action parameters that have a body, if any - you can change that code to get the query strings, paths, or whatever you'd like - and then calls the method `SerializeDataUsingObfuscator` passing the CLR object that represents the request body. To better explain this concept, consider the following API:

```csharp
public async Task<ActionResult> MyWonderfulApi([FromBody] RequestDto dto)
{
    // code
    return Ok();
}
```

The object consumed by the `SerializeDataUsingObfuscator` method is of type `RequestDto` and it contains the incoming data.

As you can see, we are using a custom JSON.NET `ContractResolver` implementation, called `ObfuscatorContractResolver`. This class serializes the request body following our obfuscation rules - that we are going to define in a minute.

### The Sensitive attribute

In order to define which properties must be obfuscated in ApplicationInsights logs, we defined a custom attribute named `Sensitive`. Moreover, this attribute allows some customization to obfuscate properties using different patterns - ie. truncate the string, show the first/last N chars, etc. Here's the definition and an usage example:

```csharp
// WonderfulDto.cs
public class WonderfulDto
{
    [Sensitive(ObfuscationType.MaskChar)]
    [JsonProperty("pin")]
    public string Pin { get; set; }
}

// SensitiveAttribute.cs using the Factory pattern to separate responsibilities
[AttributeUsage(
    AttributeTargets.Property
    , AllowMultiple = false)]
public class SensitiveAttribute : Attribute
{
    private SensitiveBase Sensitive { get; }
    public int? TruncateAfter { get; }

    public SensitiveAttribute(
        ObfuscationType obfuscationType,
        int? truncateAfter)
    {
        TruncateAfter = truncateAfter;
        Sensitive = GetSensitive(obfuscationType);
    }

    public SensitiveAttribute(
        ObfuscationType obfuscationType)
    {
        TruncateAfter = null;
        Sensitive = GetSensitive(obfuscationType);
    }

    private SensitiveBase GetSensitive(ObfuscationType obfuscationType)
    {
        return obfuscationType switch
        {
            ObfuscationType.HeadVisible => new SensitiveHeadVisible(TruncateAfter),
            ObfuscationType.TailVisible => new SensitiveTailVisible(),
            ObfuscationType.MaskChar => new SensitiveMaskChar(TruncateAfter),
            _ => throw new NotImplementedException($"None type {nameof(obfuscationType)}: {obfuscationType}")
        };
    }

    internal string Obfuscate(string strValue)
    {
        return Sensitive.Obfuscate(strValue);
    }

    internal string Obfuscate(Guid guid) => Sensitive.Obfuscate(guid);

    internal string Obfuscate(DateTime dateTime) => Sensitive.Obfuscate(dateTime);

    internal string Obfuscate(Enum @enum) => Sensitive.Obfuscate(@enum);

    internal virtual string ObfuscateDefault(object value) => Sensitive.ObfuscateDefault(value);
}

```

Now that obfuscation rules are defined, I can show you how to use them in `ObfuscatorContractResolver`:

```csharp
// ObfuscatorContractResolver.cs

public class ObfuscatorContractResolver : DefaultContractResolver
{
    protected override IList<JsonProperty> CreateProperties(Type type, MemberSerialization memberSerialization)
    {
        IList<JsonProperty> baseProperties = base.CreateProperties(type, memberSerialization);

        var sensitiveData = new Dictionary<string, SensitiveAttribute>();

        foreach (PropertyInfo p in type.GetProperties())
        {
            var customAttributes = p.GetCustomAttributes(false);

            var jsonPropertyAttribute = customAttributes
                .OfType<JsonPropertyAttribute>()
                .FirstOrDefault();

            if (jsonPropertyAttribute is null)
                continue;

            var sensitiveAttribute = customAttributes
                .OfType<SensitiveAttribute>()
                .FirstOrDefault();

            if (sensitiveAttribute is null)
                continue;

            var propertyName = jsonPropertyAttribute.PropertyName.ToUpperInvariant();

            sensitiveData.Add(propertyName, sensitiveAttribute);
        }

        if (!sensitiveData.Any())
            return baseProperties;

        var processedProperties = new List<JsonProperty>();

        foreach (JsonProperty baseProperty in baseProperties)
        {
            if (sensitiveData.TryGetValue(baseProperty.PropertyName.ToUpperInvariant(), out SensitiveAttribute sensitiveAttribute))
            {
                baseProperty.PropertyType = typeof(string);
                baseProperty.ValueProvider = new ObfuscatorValueProvider(baseProperty.ValueProvider, sensitiveAttribute);
            }

            processedProperties.Add(baseProperty);
        }

        return processedProperties;
    }
}

// ObfuscatorValueProvider.cs
    internal class ObfuscatorValueProvider : IValueProvider
{
    private readonly IValueProvider _valueProvider;
    private readonly SensitiveAttribute _sensitiveAttribute;

    public ObfuscatorValueProvider(
        IValueProvider valueProvider,
        SensitiveAttribute sensitiveAttribute)
    {
        _valueProvider = valueProvider;
        _sensitiveAttribute = sensitiveAttribute;
    }

    public object GetValue(object target)
    {
        var originalValue = _valueProvider.GetValue(target);

        var result = originalValue switch
        {
            null => null,
            string strValue => _sensitiveAttribute.Obfuscate(strValue),
            Guid guid => _sensitiveAttribute.Obfuscate(guid),
            Enum @enum => _sensitiveAttribute.Obfuscate(@enum),
            DateTime dateTime => _sensitiveAttribute.Obfuscate(dateTime),
            _ => _sensitiveAttribute.ObfuscateDefault(originalValue),
        };

        return result;
    }

    public void SetValue(object target, object value)
    {
        // we don't care
    }
}
```

And that's it: based on the `Sensitive` attribute definition over DTOs properties, JSON.NET understands if and how a given property should be serialized. The result will be sent to ApplicationInsights.

### And what about my API response body?

The solution is pretty straightforward. The code above shows the `OnResultExecutionAsync` implementation: if code runs smooth and no exception is thrown, the response body serialized using the obfuscation rules is attached to the `RequestTelemetry` object.

### Exception handling

If an exception occurs, you will realize that the `OnResultExecutionAsync` will not be fired. This happens because the flow runs out the normal ASP.NET action invocation pipeline.
But adding the exception handling middleware is enough to handle this:

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseExceptionHandler(async (context) =>
    {
        RequestTelemetry request = context.Features.Get<RequestTelemetry>();

        string serializedObj = JsonConvert.SerializeObject(error,
            new JsonSerializerSettings
            {
                ContractResolver = new ObfuscatorContractResolver(),
            });

        request.Properties.Add(ApplicationInsightsActionFilterAttribute.RESPONSE_TELEMETRY_KEY, serializedObj);
    }));
}
```

Here you can see references to `SensitiveHeadVisible`, `SensitiveMaskChar` and `SensitiveTailVisible` classes that implement some code to apply different obfuscation rules - such as hiding the value with a specific char, showing the string tail, and so on. You are free to create your preferred ones.

## Performance

After the implementation described in this post, we spent some time doing some benchmark to understand the performance impact over our APIs. So we searched the APIs with the biggest incoming and outgoing DTOs and then we stressed these APIs running 1000 requests each through Postman. We suddenly realized that we were going to lose 60-90ms for each request on each APIs! This performance drop was caused by the string serialization performed by JSON.NET, because the rest of the code needed a couple of milliseconds to run. Our code is fine then, how can we cut down the serialization time then?
After some research, we found a simple way to minimize the performance impact, keeping our APIs safe. Basically, we understood that we didn't need to wait that the serialization process to be finished before invoking the invocation pipeline!

Coming back at `ApplicationInsightsActionFilterAttribute` class, we did the following changes:

```csharp
// OnActionExecutionAsync
var loggerTask = Task.Run(() =>
{
    string fromBodyParameter = context.ActionDescriptor.Parameters
        
    // same code as before
});

await next();

try
{
    if (loggerTask != null)
    {
        await loggerTask;
    }
}
catch (Exception ex)
{
    _logger.LogCritical(ex, "Error executing the response logger task");
}

// OnResultExecutionAsync
var loggerTask = Task.Run(() =>
{
    if (context.Result is ObjectResult result &&
    
    // same code as before
});

await next();

try
{
    if (loggerTask != null)
    {
        await loggerTask;
    }
}
catch (Exception ex)
{
    _logger.LogCritical(ex, "Error executing the response logger task");
}
```

This implementation doesn't wait for the serialization to be completed, but instead it runs a task that will be awaited AFTER the invocation pipeline is completed. Most of the times, the `loggerTask` is already completed when awaited, so the impact is almost nil.

## Conclusion

We saw how to hide sensitive information from Application Insights logs, keeping our data (and users) safe.
Since all the logic is wrapped in a custom `ContractResolver`, you can obfuscate properties everywhere in your code! For example, we used the same approach for the log generated by our `HttpClient` instances.

I hope this tutorial was helpful! 🎉

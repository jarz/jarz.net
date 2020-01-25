+++ 
date = 2020-01-21T20:54:18-05:00
title = "Azure Application Insights payload logging in ASP.NET Core 3.1"
description = "Changes in the ASP.NET Core pipeline mean middleware is needed"
slug = "app-insights-logging-middleware" 
tags = ["Application Insights", "ASP.NET Core"]
categories = []
externalLink = ""
series = []
+++

At the beginning of January I presented on [Azure Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) at CodeMash. While building my [demo](https://github.com/jarz/AlbumViewerVNext/tree/test/add-ai) against an ASP.NET Core 3.1 project, I couldn't reuse an existing _TelemetryInitializer_ to log request bodies for troubleshooting. A _TelemetryInitializer_ allows developers to add or modify properties on telemetry sent to Application Insights; [Microsoft's example](https://docs.microsoft.com/en-us/azure/azure-monitor/app/api-filtering-sampling#addmodify-properties-itelemetryinitializer) overrides the default success logic such that 400-level errors aren't considered failures (the default in Application Insights). Changes made in ASP.NET Core 3.0 no longer allowed me to use this:

```cs {linenos=table}
public class RequestBodyInitializer : ITelemetryInitializer
{
    readonly IHttpContextAccessor httpContextAccessor;

    public RequestBodyInitializer(IHttpContextAccessor httpContextAccessor)
    {
        this.httpContextAccessor = httpContextAccessor;
    }

    public void Initialize(ITelemetry telemetry)
    {
        if (telemetry is RequestTelemetry requestTelemetry)
        {
            if ((httpContextAccessor.HttpContext.Request.Method == HttpMethods.Post ||
                 httpContextAccessor.HttpContext.Request.Method == HttpMethods.Put) &&
                 httpContextAccessor.HttpContext.Request.Body.CanRead)
            {
                const string jsonBody = "JsonBody";

                if (requestTelemetry.Properties.ContainsKey(jsonBody))
                {
                    return;
                }

                // Allows re-usage of the stream
                httpContextAccessor.HttpContext.Request.EnableRewind(); // .EnableBuffering() in ASP.NET Core 3

                var stream = new StreamReader(httpContextAccessor.HttpContext.Request.Body);
                var body = stream.ReadToEnd();

                // Reset the stream so data is not lost
                httpContextAccessor.HttpContext.Request.Body.Position = 0;
                requestTelemetry.Properties.Add(jsonBody, body);
            }
        }
    }
}
```
[(Source Stack Overflow answer)](https://stackoverflow.com/a/50024016)

In ASP.NET Core 1.x and 2.x, this would add a new property named _JsonBody_ to the telemetry, populated with the associated request's body content. The telemetry would be uploaded to Application Insights, visible on the request and associated with additional server-side processing, such as database requests.

After upgrading to ASP.NET Core 3 and switching to `EnableBuffering()`, the next exception is a bit concerning: _**System.InvalidOperationException**: 'Synchronous operations are disallowed. Call ReadAsync or set AllowSynchronousIO to true instead.'_ One of the bigger breaking changes in ASP.NET Core 3.0 was [disabling synchronous server IO by default](https://docs.microsoft.com/en-us/dotnet/core/compatibility/2.2-3.0#http-synchronous-io-disabled-in-all-servers
). While Microsoft provides a mitigation, I've still run into issues with disposed objects afterward. It's time for a different approach.

## Writing custom ASP.NET Core custom middleware

Instead of continuing to fight exceptions, I decided to write custom middleware to snag the request body. [_Middleware_](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-3.1) is all of the code that runs between the server receiving a request and returning a response. We mainly just add a few lines to our `Startup` class (like `app.UseAuthentication()`) and utilize built-in middleware from ASP.NET Core. However, we can easily [write our own middleware code](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/write?view=aspnetcore-3.1) to be run for every request and response.

When writing a class for custom middleware, there's no _required\*_ interface or abstract class to implement. Instead, the class must:
- Have a public constructor that accepts a [`RequestDelegate`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.requestdelegate).
  - The `RequestDelegate` is the next piece of middleware to be executed after yours.
- Implement a public method named either `Invoke` or `InvokeAsync`.
  - It must return a `Task`, no matter the name.
  - It's first parameter must be `HttpContext`.
  - Any additional parameters will be populated by your dependency injection container.

_*Note: ASP.NET Core 2.1 added an [`IMiddleware` interface](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/extensibility?view=aspnetcore-3.1) that can be implemented._

The pattern to implement is straightforward:
```cs
public async Task InvokeAsync(HttpContext context, RequestDelegate next)
{
    // Incoming request

    await next(context);    // Call the next middleware

    // Outgoing response
}
```

The method begins with a client HTTP request. If the middleware sets up anything necessary for the rest of the pipeline (authentication, authorization), it would be implemented at the beginning. Afterward, the method calls `await next(context)` to continue processing the request, potentially making it's way to a controller. Control returns as the stack is rewinding and a response is on the way back to the client. 

Knowing this, we can write our own middleware to capture the request body before it's read downstream:
```cs {linenos=table}
public class ApplicationInsightsLoggingMiddleware : IMiddleware
{
    private TelemetryClient TelemetryClient { get; }

    public ApplicationInsightsLoggingMiddleware(TelemetryClient telemetryClient)
    {
        TelemetryClient = telemetryClient;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        // Inbound (before the controller)
        var request = context?.Request;
        if (request == null)
        {
            await next(context);
            return;
        }

        request.EnableBuffering();  // Allows us to reuse the existing Request.Body

        // Swap the original Response.Body stream with one we can read / seek
        var originalResponseBody = context.Response.Body;
        using var replacementResponseBody = new MemoryStream();
        context.Response.Body = replacementResponseBody;

        await next(context); // Continue processing (additional middleware, controller, etc.)

        // Outbound (after the controller)
        replacementResponseBody.Position = 0;

        // Copy the response body to the original stream
        await replacementResponseBody.CopyToAsync(originalResponseBody).ConfigureAwait(false);
        context.Response.Body = originalResponseBody;

        var response = context.Response;
        if (response.StatusCode < 500)
        {
            return;
        }

        var requestTelemetry = context.Features.Get<RequestTelemetry>();
        if (requestTelemetry == null)
        {
            return;
        }

        if (request.Body.CanRead)
        {
            var requestBodyString = await ReadBodyStream(request.Body).ConfigureAwait(false);
            requestTelemetry.Properties.Add("RequestBody", requestBodyString);  // limit: 8192 characters
            TelemetryClient.TrackTrace(requestBodyString);
        }

        if (replacementResponseBody.CanRead)
        {
            var responseBodyString = await ReadBodyStream(replacementResponseBody).ConfigureAwait(false);
            requestTelemetry.Properties.Add("ResponseBody", responseBodyString);
            TelemetryClient.TrackTrace(responseBodyString);
        }
    }

    private async Task<string> ReadBodyStream(Stream body)
    {
        if (body.Length == 0)
        {
            return null;
        }

        body.Position = 0;

        using var reader = new StreamReader(body, leaveOpen: true);
        var bodyString = await reader.ReadToEndAsync().ConfigureAwait(false);
        body.Position = 0;

        return bodyString;
    }
}
```

This version goes beyond the original _ITelemetryInitializer_ implementation, logging both the request and response bodies if a server error occurred. It also uses the `TrackTrace` method, allowing for four times the data to be sent to Application Insights. **An important note**: this doesn't discrimate based on endpoint or request/response body content; be careful of logging sensitive data in a production environment.

Finally, two changes are made in the _Startup_ class in order to run the middleware.

First, register the middleware with the dependency injection container in the `ConfigureServices` method:
```cs
public void ConfigureServices(IServiceCollection services)
{
    ...
    services.AddTransient<ApplicationInsightsLoggingMiddleware>(); 
    ...
}
```

Second, register the middleware in the request processing pipeline:
```cs
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseExceptionHandler("/Error");
    }

    app.UseHttpsRedirection();
    app.UseStaticFiles();

    app.UseRouting();
    app.UseCors();

    app.UseAuthentication();
    app.UseAuthorization();

    // Placement matters!
    // Place custom middleware after security...
    app.UseMiddleware<ApplicationInsightsLoggingMiddleware>();

    // ...but before specifying endpoints  
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapRazorPages();
    });
}
```

Middleware will execute in the [order of registration](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-3.1#middleware-order). It's important to register security middleware correctly. The `ApplicationInsightsLoggingMiddleware` will only execute after successful processing of the built-in CORS, authentication, and authorization middleware.
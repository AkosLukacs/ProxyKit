# ProxyKit [![Build Status][travis build]][project] [![NuGet][nuget badge]][nuget package]

A toolkit to create **HTTP proxies** hosted in ASP.NET Core as middleware. This
allows focused code-first proxies that can be embedded in an existing ASP.NET
Core applications or deployed as a standalone server. Deployable anywhere
ASP.NET Core is deployable such as Windows, Linux, Containers and Serverless
(with caveats).

Having built proxies many times before, I felt it is time make a package. Forked
from [ASP.NET labs][aspnet labs], it has been heavily modified with a different
API, to facilitate a wider variety of proxying scenarios (i.e. routing based on
a JWT claim) and interception of the proxy requests / responses for
customization of headers and (optionally) request / response bodies. It also
uses [`HttpClientFactory`] internally that will mitigate against dns caching
issues and handler life cycles making it suitable for microservice / container
environments.

<!-- TOC depthFrom:2 -->autoauto- [1. Quick Start](#1-quick-start)auto- [2. Customising the upstream request](#2-customising-the-upstream-request)auto- [3. Customising the upstream response](#3-customising-the-upstream-response)auto- [4. X-Forwarded Headers](#4-x-forwarded-headers)auto- [5. Making upstream servers reverse proxy friendly](#5-making-upstream-servers-reverse-proxy-friendly)auto- [6. Configuring ProxyOptions](#6-configuring-proxyoptions)auto- [7. Error handling](#7-error-handling)auto- [8. Testing](#8-testing)auto- [9. Load Balancing](#9-load-balancing)auto    - [9.1. Weighted Round Robin](#91-weighted-round-robin)auto- [10. Further examples](#10-further-examples)auto- [11. Performance considerations](#11-performance-considerations)auto- [12. Note about Serverless](#12-note-about-serverless)auto- [13. Comparison with Ocelot](#13-comparison-with-ocelot)auto- [14. Contributing / Feedback / Questions](#14-contributing--feedback--questions)autoauto<!-- /TOC -->

## 1. Quick Start

ProxyKit is a `NetStandard2.0` package. Install into your ASP.NET Core project:

```bash
dotnet add package ProxyKit
```

In your `Startup`, add the proxy service:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    ...
    services.AddProxy();
    ...
}
```

Forward requests to `localhost:5001`:

```csharp
public void Configure(IApplicationBuilder app)
{
    app.RunProxy(context => context
        .ForwardTo("http://upstream-server:5001")
        .Execute());
}
```

What is happening here?

 1. `context.ForwardTo(upstreamHost)` is an extension method over the
    `HttpContext` that creates and initializes an `HttpRequestMessage` with
    the original request headers copied over and returns a `ForwardContext`.
 2. `Execute` forwards the request to the upstream server and returns an
    `HttpResponseMessage`.
 3. The proxy middleware then takes the response and applies it to
    `HttpContext.Response`.

Note: `RunProxy` is [terminal] - anything added to the pipeline after `RunProxy`
will never be executed.

## 2. Customising the upstream request

One can modify the upstream request headers prior to sending them to suit
customisation needs. ProxyKit doesn't add, remove nor modify any headers by
default; one must opt in any behaviours explicitly.

In this example we will add a `X-Correlation-Id` header if it does not exist:

```csharp
public const string XCorrelationId = "X-Correlation-ID";

public void Configure(IApplicationBuilder app)
{
    app.RunProxy(context =>
    {
        var forwardContext = context.ForwardTo("http://localhost:5001");
        if (!forwardContext.UpstreamRequest.Headers.Contains(XCorrelationId))
        {
            forwardContext.UpstreamRequest.Headers.Add(XCorrelationId, Guid.NewGuid().ToString());
        }
        return forwardContext.Execute();
    });
}
```

This can be encapsulated as an extension method:

```csharp
public static class CorrelationIdExtensions
{
    public static ForwardContext ApplyCorrelationId(this ForwardContext forwardContext)
    {
        if (forwardContext.UpstreamRequest.Headers.Contains("X-Correlation-ID"))
        {
            forwardContext.UpstreamRequest.Headers.Add("X-Correlation-ID", Guid.NewGuid().ToString());
        }
        return forwardContext;
    }
}
```

... making the proxy code a little nicer to read:

```csharp
public void Configure(IApplicationBuilder app)
{
    app.RunProxy(context => context
        .ForwardTo("http://localhost:5001")
        .ApplyCorrelationId()
        .Execute());
}
```

## 3. Customising the upstream response

Response from an upstream server can be modified before it is sent to the
client. In this example we are removing a header:

```csharp
 public void Configure(IApplicationBuilder app)
{
    app.RunProxy(async context =>
    {
        var response = await context
            .ForwardTo("http://localhost:5001")
            .Execute();

        response.Headers.Remove("MachineID");

        return response;
    });
}
```

## 4. X-Forwarded Headers

Many applications will need to know what their "outside" host / url is in order
to generate correct values. This is achieved using `X-Forwarded-*` and
`Forwarded` headers. ProxyKit supports applying `X-Forward-*` headers out of the
box (applying `Forwarded` headers support is on backlog). At time of writing,
`Forwarded` is [not supported](https://github.com/aspnet/AspNetCore/issues/5978)
in ASP.NET Core.

To apply `X-Forwarded-Headers` to the request to the upstream server:

```csharp
public void Configure(IApplicationBuilder app)
{
    app.RunProxy(context => context
        .ForwardTo("http://upstream-server:5001")
        .ApplyXForwardedHeaders()
        .Execute());
}
```

This will add `X-Forward-For`, `X-Forwarded-Host` and `X-Forwarded-Proto`
headers to the upstream request using values from `HttpContext`. If the proxy
middleware is hosted on a path and a `PathBase` exists on the request, then an
`X-Forwarded-PathBase` is also added.

## 5. Making upstream servers reverse proxy friendly

Applications that are deployed behind a reverse proxy typically need to be
somewhat aware of that so they can generate correct URLs and paths when
responding to a browser. That is, they look at `X-Forward-*` \ `Forwarded`
headers and use their values .

In ASP.NET Core, this means using the Forwarded Headers middleware in your
application. Please refer to the [documentation][forwarded headers middleware]
for correct usage (and note the security advisory!).

**Note:** the Forwarded Headers middleware does not support
`X-Forwarded-PathBase`. This means if you proxy `http://example.com/foo/` to
`http://upstream-host/` the `/foo/` part is lost and absolute URLs cannot be
generated unless you configure your applications PathBase directly.

Related issues and discussions:

- https://github.com/aspnet/AspNetCore/issues/5978
- https://github.com/aspnet/AspNetCore/issues/5898


To support PathBase dynamically in your application with `X-Forwarded-PathBase`,
examine the header early in your pipeline and set the `PathBase` accordingly:

```csharp
var options = new ForwardedHeadersOptions
{
   ...
};
app.UseForwardedHeaders(options);
app.Use((context, next) => 
{
    if (context.Request.Headers.TryGetValue("X-Forwarded-PathBase", out var pathBases))
    {
        context.Request.PathBase = pathBases.First();
    }
    return next();
});
```

Alternatively you can use ProxKit's `UseXForwardedHeaders` extension that
perform the same as the above (including calling `UseForwardedHeaders`):

```csharp
var options = new ForwardedHeadersOptions
{
   ...
};
app.UseXForwardedHeaders(options);
```

## 6. Configuring ProxyOptions

When adding the Proxy to the service collection there are a couple of options 
available to configure the proxy behavior:

```csharp
services.AddProxy(options => /* configure options here */);
```

There are two options :

1. `options.ConfigureHttpClient`: an `Action<HttpClient>` to configure the HTTP
   Client on each request e.g. configuring a timeout:
    ```csharp
    services.AddProxy(options 
        => options.ConfigureHttpClient =
            (serviceProvider, client) => client.Timeout = TimeSpan.FromSeconds(5));
    ```

2. Configure the primary `HttpMessageHandler` factory. This is typically used in
   testing to inject a test handler (see Testing below). This is a factory
   method as, internally, `HttpClientFactory` recreates the HttpHandler pipeline 
   disposing previously created handlers.

    ```csharp
    services.AddProxy(options =>
        options.GetMessageHandler = () => _httpMessageHandler);
    ```

## 7. Error handling

When `HttpClient` throws the following logic applies:

1. When upstream server is not reachable then `ServiceUnavailable` is returned.
2. When upstream server is slow and client timeouts then `GatewayTimeout` is
   returned.

Not all exception scenarios and variations are caught which may result in a
`InternalServerError` being returned to your clients. Please create an issue if
a scenario is missing.

## 8. Testing

As ProxyKit is standard ASP.NET Core middleware, it can be tested using the
standard in-memory `TestServer` mechanism.

Often you will want to test ProxyKit with your application and perhaps test the
behaviour of your application when load balanced with two or more instances as
indicated below.

```
                               +----------+
                               |"Outside" |
                               |HttpClient|
                               +-----+----+
                                     |
                                     |
                                     |
                         +-----------+---------+
    +-------------------->RoutingMessageHandler|
    |                    +-----------+---------+
    |                                |
    |                                |
    |           +--------------------+-------------------------+
    |           |                    |                         |
+---+-----------v----+      +--------v---------+     +---------v--------+
|Proxy TestServer    |      |Host1 TestServer  |     |Host2 TestServer  |
|with Routing Handler|      |HttpMessageHandler|     |HttpMessageHandler|
+--------------------+      +------------------+     +------------------+
```

`RoutingMessageHandler` is an `HttpMessageHandler` that will route requests to
to specific host based on on the origin it is configured with. For ProyKit
to forward requests (in memory) to the upstream hosts, it needs to be configured
to use the `RoutingMessageHandler` as it's primary `HttpMessageHandler`.

Full example can been viewed [here](src/Examples/06_Testing.cs).

## 9. Load Balancing

Load balancing is mechanism to decide which upstream server to forward the
request to. Out of the box, ProxyKit currently supports one type of
load balancing - Weighted Round Robin. Other types are planned.

### 9.1. Weighted Round Robin

Round Robin simply distributes requests as they arrive to the next host in a
distribution list. With optional weighting, more reqeusts are send to host with
greater weights.

```csharp
public void Configure(IApplicationBuilder app)
{
    var roundRobin = new RoundRobin
    {
        new UpstreamHost("http://localhost:5001", weight: 1),
        new UpstreamHost("http://localhost:5002", weight: 2)
    };

    app.RunProxy(
        async context =>
        {
            var host = roundRobin.Next();

            return await context
                .ForwardTo(host)
                .Execute();
        });
}
```

## 10. Further examples

Browse [Examples](src/Examples) for further inspiration.

## 11. Performance considerations

According to TechEmpower's Web Framework Benchmarks, ASP.NET Core [is up there
with the fastest for plain text](https://www.techempower.com/benchmarks/#section=data-r17&hw=ph&test=plaintext).

Stress testing shows that ProxyKit is approximately 8% slower than nginx for
simple forwarding. If absolute raw throughput is a concern for you then consider
nginx or alternatives. For me being able to create flexible proxies using C# is a
reasonable tradeoff for the (small) performance cost. Note that depending on what
your proxy does may impact performance so you should measure yourself.

Memory wise, ProxyKit maintained a steady ~20MB of RAM after processing millions
of requests for simple forwarding. Again, it depends on what your proxy does so
you should analyse and measure yourself.

## 12. Note about Serverless

Whilst is it is possible to run full ASP.NET Core web application in [AWS
Lambda] and [Azure Functions] it should be noted that Serverless systems are
message based and not stream based. Incoming and outgoing HTTP request messages
will be buffered and potentially encoded as Base64 if binary (so larger). This
means ProxyKit should only be used for API (json) proxying in production on
Serverless. (Though proxing other payloads is fine for dev / exploration /
quick'n'dirty purposes.)

## 13. Comparison with Ocelot

[Ocelot] is an API Gateway also on ASP.NET Core, and  A key difference between API
Gateways and general Reverse Proxies is that the former tend to be **message**
based whereas a reverse proxy is **stream** based. That is, an API gateway will
typically buffer the every request and response message to be able to perform
transformations. This is fine for an API gateway but not suitable for a reverse
proxy performance wise nor for responses that are chunked-encoded. See [Not
Supported Ocelot docs][ocelot not supported].

Combining ProxyKit with Ocelot would give some nice options for a variety of
scenarios.

## 14. Contributing / Feedback / Questions

Any ideas for features, bugs or questions, please create an issue. Pull requests 
gratefully accepted but please create an issue first.

[travis build]: https://travis-ci.org/damianh/ProxyKit.svg?branch=master
[project]: https://travis-ci.org/damianh/ProxyKit
[nuget badge]: https://img.shields.io/nuget/v/ProxyKit.svg
[nuget package]: https://www.nuget.org/packages/ProxyKit
[aspnet labs]: https://github.com/aspnet/AspLabs
[`httpclientfactory`]:  https://github.com/aspnet/Extensions/tree/master/src/HttpClientFactory
[terminal]: https://docs.microsoft.com/en-ie/dotnet/api/microsoft.aspnetcore.builder.runextensions.run?view=aspnetcore-2.1
[forwarded headers middleware]: https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/proxy-load-balancer?view=aspnetcore-2.2
[aws lambda]: https://aws.amazon.com/blogs/developer/running-serverless-asp-net-core-web-apis-with-amazon-lambda/
[azure functions]: https://blog.wille-zone.de/post/serverless-webapi-hosting-aspnetcore-webapi-in-azure-functions/
[ocelot]: https://github.com/ThreeMammals/Ocelot
[ocelot not supported]: https://ocelot.readthedocs.io/en/latest/introduction/notsupported.html

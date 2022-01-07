---
title:  "Creating a multi-tenant IdentityServer federation gateway"
date: 2020-09-24 15:00
categories: development
redirect_from:
  - /2020/09/24/creating-a-multitenant.html
---

[IdentityServer](https://github.com/IdentityServer) is quite an awesome framework for creating your own OAUTH-based authentication server in .NET Core. At its core, its purpose is to provide OAUTH endpoints to allow clients request access tokens to then call APIs. IdentityServer also [supports being used as a federation gateway](https://docs.identityserver.io/en/dev/topics/federation_gateway.html), by utilizing the built-in authentication system in .NET Core. 

This is great if you're creating an authentication server that has a fixed number of authentication providers that serves applications that should use the same authentication providers, or a subset of these. However, in some scenarios, like the one I have in my own organization, where we allow customers to bring their own authentication, using OpenID Connect or WS Federation. We have a lot of customers on our cloud solution, so setting up each of the authentication providers statically in the authentication server is not going to work.

There's some resources for this online, but it was hard to find concrete examples for how to achieve this. Here's a few tips that might be helpful if you're trying to achieve this in your application. I'll show the steps here for implementing federated OpenID Connect here, but the steps for adding support for WS Federation is more or less exactly the same (just swap out "OpenIdConnect" with "WsFederation" in most of the class names).

## The setup
We will define our authentication services in our ConfigureServices method:
```csharp
var authBuilder = services.AddAuthentication();
authBuilder.Services.TryAddEnumerable(ServiceDescriptor.Singleton<IPostConfigureOptions<OpenIdConnectOptions>, OpenIdConnectPostConfigureOptions>());
authBuilder.AddRemoteScheme<OpenIdConnectOptions, MultitenantOpenIdConnectHandler>("openid-connect", "OpenID Connect", options =>
{
    options.CallbackPath = "/signin-oidc";
});

services.AddSingleton<IOptionsMonitor<OpenIdConnectOptions>, OpenIdConnectOptionsProvider>();
services.AddSingleton<IConfigureOptions<OpenIdConnectOptions>, OpenIdConnectOptionsInitializer>();
```

Notice that we don't just do `AddOpenIdConnect()` - we want to register a custom OpenIdConnectHandler that is multi-tenant aware. We'll get back to those details soon. Also notice that we register a custom `IOptionsmonitor<OpenIdConnectOptions>` and `IConfigureOptions<OOpenIdConnectOptions>` class. These will be used to fill in the client ID, authority and other parameters needed to authenticate at runtime.

Then we'll configure the login URL for IdentityServer:
```csharp
services.AddIdentityServer(options =>
{
  options.UserInteraction.LoginUrl = "/api/challenge/redirect";
  options.UserInteraction.LoginReturnUrlParameter = "returnUrl";
})
```
The /api/challenge/redirect endpoint will resolve which authentication service to be used for a tenant, based on acr_values where we will add the tenant ID. 

Given all this, the authentication flow will be as follows:

0. Initial page load
1. Redirect to /connect/authorize?...
2. Redirect to /api/challenge/redirect?returnUrl=... - this looks up the authentication scheme to be used (e.g. openid-connect or ws-federation based on tenant ID, which we send through acr_values).
3. Redirect to /api/challenge?scheme=...&returnUrl=...&tenantId=....
4. Redirect to /api/challenge calls Challenge() with the specified scheme and returnUrl, which invokes the ASP.NET Core authentication mechanisms.
5. Redirect to external auth provider. This is where the user logs in with username+password.
6. Redirect to callback /signin-oidc/<tenant-id>
7. Redirect to the page you wanted to load.

## The challenge endpoint
First we need to create an endpoint to route the user to an endpoint which authenticates the user. This involves a lookup in a DB or something similar to get the ASP.NET Core authentication scheme for that tenant. We pass the tenant ID as in acr_values. More on that later.
```csharp
[HttpGet]
[Route("redirect")]
public async Task<IActionResult> Redirect(string scheme, string returnUrl)
{
    var authContext = await _interaction.GetAuthorizationContextAsync(returnUrl);
    var tenantId = authContext.Tenant;
    // Look up the tenant's authentication provider here. This can be a database lookup. This should resolve to either: openid-connect or ws-federation
    var scheme = GetAuthenticationSchemeForTenant(tenantId);
    return Redirect(string.Format("/api/challenge?scheme={0}&tenantId={1}&returnUrl={2}", theScheme, tenantId, Uri.EscapeDataString(returnUrl)));
}
```

Then for the actual challenge endpoint, it's quite bare bone. Note that you should validate the input here (return URL, etc.), but for brevity, I've excluded this here. This is very standard, almost taken straight out of the quickstart UI for IdentityServer4.

```csharp
[HttpGet]
[Route("")]
public async Task<IActionResult> Challenge(string scheme, string returnUrl, string tenantId)
{
    var props = new AuthenticationProperties
    {
        RedirectUri = "/api/challenge/callback",
        Items =
        {
            { "returnUrl", returnUrl },
            { "scheme", scheme },
        }
    };
    return Challenge(props, scheme);
}
```

Finally you have the challenge callback. I just use the same code as in the quickstart's Callback function, just adapted for using it in the API controller: [https://github.com/IdentityServer/IdentityServer4.Quickstart.UI/blob/main/Quickstart/Account/ExternalController.cs](https://github.com/IdentityServer/IdentityServer4.Quickstart.UI/blob/main/Quickstart/Account/ExternalController.cs)

## Serving configuration to the OpenIdConnectHandler at runtime
As mentioned earlier, we have added a few singletons in our startup code:
```csharp
services.AddSingleton<IOptionsMonitor<OpenIdConnectOptions>, OpenIdConnectOptionsProvider>();
services.AddSingleton<IConfigureOptions<OpenIdConnectOptions>, OpenIdConnectOptionsInitializer>();
```
The whole purpose of these are to serve configuration to the OpenIdConnectHandler at runtime. I have used the implementation of these as described here: [https://stackoverflow.com/questions/52955238/how-can-i-set-the-authority-on-openidconnect-middleware-options-dynamically](https://stackoverflow.com/questions/52955238/how-can-i-set-the-authority-on-openidconnect-middleware-options-dynamically). This works great. One detail that is important to mention here is:

1. Set options.CallbackPath to "/signin-oidc/" + tenantId; That way, after authentication, tenants are redirected to a tenant-specific endpoint.
2. When there's no tenant ID, just set it to /signin-oidc. This happens on the first configuration call.

As for the tenant provider described in this StackOverflow post, for my usecase this doesn't work because not all URLs are prefixed with the tenant ID. Perhaps that could be done in IdentityServer - but I opted for a different solution. Here's the solution I went with for TenantProvider:

```csharp
public TenantAuthOptions GetCurrentTenant()
{
    var request = _httpContextAccessor.HttpContext.Request;
    string tenant = null;
    PathString remainingPath;
    if (request.Query.ContainsKey("tenantId"))
        tenant = request.Query["tenantId"];
    //OpenID Connect
    else if (request.Path.StartsWithSegments(new PathString("/signin-oidc"), StringComparison.InvariantCultureIgnoreCase, out remainingPath))
        tenant = remainingPath.Value.Trim('/');
    //Do the DB lookup for the tenant authentication options (client ID, etc.)
    return GetTenant(tenant);
}
```

Here we look for the tenant ID in various places:

1. If it's set on the query string, we get it from there. That's needed when you redirect from /api/challenge/redirect to /api/challenge. You COULD probably add the tenant ID to the path on this endpoint instead. That's up to you, and then you wouldn't need to parse the query string.
2. If we're at the signin-oidc endpoint, we get the tenant ID from the path.
  
## The multi-tenant OpenIdConnectHandler
As mentioned previously, we can't use the built-in OpenIdConnectHandler in our usecase. The reason is (and I don't know WHY), is that some of the configuration options for the OpenIdConnectHandler is fetched per request, other values are cached seemingly forever after retrieving the configuration once. Most notably, the callback URL: If you use standard OpenIdConnectHandler, it will redirect correctly to /signin-oidc/(tenant-id), however, ShouldHandleRequestAsync() in the OpenIdConnectHandler() will return false, because apparently the CallbackPath is cached. Because of this, the handler won't handle the /signin-oidc/(tenant-id) requests, and you'll get a HTTP 404. I haven't looked into the details on WHY this is, but this is thankfully easily solvable:
  
```csharp
public class MultitenantOpenIdConnectHandler : OpenIdConnectHandler
{
  public MultitenantOpenIdConnectHandler(IOptionsMonitor<OpenIdConnectOptions> options, ILoggerFactory logger, HtmlEncoder htmlEncoder, UrlEncoder encoder, ISystemClock clock)
      : base(options, logger, htmlEncoder, encoder, clock)
  { 
  }
  public override async Task<bool> ShouldHandleRequestAsync()
  {
      if (await base.ShouldHandleRequestAsync())
          return true;
      // We expect a path on the format: <callbackpath>/<tenant-id>
      PathString remaining;
      if (!Request.Path.StartsWithSegments(Options.CallbackPath, StringComparison.InvariantCultureIgnoreCase, out remaining))
          return false;
      // The remaining segment should only have one path segment (== the tenant)
      return remaining.Value.Trim('/').Split('/').Length == 1;
  }
}
```
In other words, we just parse the request path, treating the CallbackPath as the base. The original code in the framework uses a simple equality check, which fails in this case.

## In conclusion
And this is pretty much it! I have based my implementation on the quickstart UI, so if you're new to IdentityServer I certainly advise you to start exploring that FIRST before attempting this crazyness right here.
  
I hope this is of use to anyone. It certainly has been a learning experience for me.
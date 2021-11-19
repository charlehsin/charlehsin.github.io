---
title:  "Use external OAuth 2.0 provider in ASP.NET (ASP.NET Core) 5 Web API."
date:   2021-11-19 06:20:00 -0800
categories: coding net5
permalinks: /:categories/:year/:month/:day/:title.html
---

ASP.NET 5 provides the [documentation](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/social/?view=aspnetcore-5.0&tabs=visual-studio) for how to use external OAuth 2.0 provider. However, the documentation is for the web app, not for the web API. This post talks about how to use Facebook as the external OAuth 2.0 provider for the web API. This post is based on this [commit](https://github.com/charlehsin/net5-webapi-tutorial/commit/cd5d100390aa3ffe8244208ccb06f918b6ed980d) in my [GitHub repository](https://github.com/charlehsin/net5-webapi-tutorial). The assumption is that you already have a working ASP.NET 5 Web API with JWT bearer authentication, and you only want to add the external OAuth 2.0 provider to your Web API.

First, follow the ASP.NET 5 documentation above to get the Facebook App ID and the Facebook App secret, and to store the App ID and the App secret using Secret Manager. (For production usage, you need to use other approaches to store the secretes like App ID and App secret. This is beyond the scope of this post.)

Then add Microsoft.AspNetCore.Authentication.Facebook package to your project. You need to specify the correct version since the latest version is only for ASP.NET 6.

At the CreateHostBuilder method in [Program.cs](https://github.com/charlehsin/net5-webapi-tutorial/blob/main/TodoApi/Program.cs), since CreateDefaultBuilder is used, the user secrets configuration source is automatically added in Development mode.

At Startup.ConfigureServices, use services.AddAuthentication().AddFacebook(), as shown in the following code block.

{% highlight csharp linenos %}
services.AddAuthentication()
    .AddCookie(options =>
        {
            // Your original codes.
        })
    .AddJwtBearer(x =>
        {
            // Your original codes.
        })
    .AddFacebook(facebookOptions =>
        {
            // This is assuming that we use the secrete storage to store the secretes.
            facebookOptions.AppId = Configuration["Authentication:Facebook:AppId"];
            facebookOptions.AppSecret = Configuration["Authentication:Facebook:AppSecret"];
            facebookOptions.SaveTokens = true;
        });
{% endhighlight %}

Then, we need to create the APIs to challenge Facebook OAuth 2.0 provider and then authenticate via Facebook OAuth 2.0 provider. In my GitHub repository codes, they are the UsersController.AuthenticateAsFacebookAsync method and UsersController.AuthenticateAsFacebookCallbackAsync method. The ordered action flow is the following.
1. The front-end application, e.g., the web browser, uses a POST request to challenge Facebook. Check the "POST api/Users/authenticate/facebook API" code block below to find how to handle this request. This is from [UsersController class](https://github.com/charlehsin/net5-webapi-tutorial/blob/main/TodoApi/Controllers/UsersController.cs). In short, we will challenge the Facebook to get the redirection URL and then return the Redirection (302).
2. The front-end application receives the Redirection (302). The value in the "location" response header indicates the redirection URL. The front-end application redirects to that URL to log into Facebook.
3. A corresponding "GET" request (with the same API path as the "POST" request above) will be automatically sent by the front-end. Check the "GET api/Users/authenticate/facebook API" code block below to find how to handle this request. This is from [UsersController class](https://github.com/charlehsin/net5-webapi-tutorial/blob/main/TodoApi/Controllers/UsersController.cs). In short, we will authenticate via Facebook. After the authentication is done, we choose to generate and return a JWT since our other APIs use JWT authentication.  At the end of this post, we will discuss another approach, using the Facebook authentication scheme directly.
4. The front-end application will get the JWT in the response of the "GET" request. Then it can use the JWT for other APIs.

{% highlight csharp linenos %}
// This is the "POST api/Users/authenticate/facebook API"
[AllowAnonymous]
[HttpPost("authenticate/facebook")]
[Produces("application/json")]
[ProducesResponseType(StatusCodes.Status302Found)]
[ProducesResponseType(StatusCodes.Status401Unauthorized)]
public async Task<IActionResult> AuthenticateAsFacebookAsync()
{
    var authScheme = FacebookDefaults.AuthenticationScheme;

    // Challenge the Facebook OAuth 2.0 provider.
    await Request.HttpContext.ChallengeAsync(authScheme);

    // Get the location response header.
    if (Response.Headers.TryGetValue("location", out var locationResponseHeader))
    {
        return Redirect(locationResponseHeader);
    }

    return Unauthorized();
}
{% endhighlight %}

{% highlight csharp linenos %}
// This is the "GET api/Users/authenticate/facebook API"
[AllowAnonymous]
[HttpGet("authenticate/facebook")]
[ProducesResponseType(StatusCodes.Status200OK, Type = typeof(string))]
[ProducesResponseType(StatusCodes.Status401Unauthorized)]
public async Task<IActionResult> AuthenticateAsFacebookCallbackAsync()
{
    var authScheme = FacebookDefaults.AuthenticationScheme;

    // Try to authenticate.
    var authResult = await Request.HttpContext.AuthenticateAsync(authScheme);
    if (!authResult.Succeeded
        || authResult?.Principal == null
        || !authResult.Principal.Identities.Any(id => id.IsAuthenticated)
        || string.IsNullOrEmpty(authResult.Properties.GetTokenValue("access_token")))
    {
        return Unauthorized();
    }

    // Then you need to get the role and user ID based on the user name.
    // In real world scenario, you may have other flows to add the target user
    // from external OAuth 2.0 provider into ASP Identity.
    // This is beyond the scope of this sample codes.
    // For tutorial purpose here, we will use fixed Admin role.

    // In the API design of this sample codes, the APIs will only accept JWT Bearer authentication
    // or Cookies authentication.
    // Therefore, we need to get JWT or create the auth cookie.
    // For tutorial purpose, we use JWT.

    // In real world scenario, you may want to use auth.Properties.ExpiresUtc to set your JWT expiration
    // or auth cookie expiration accordingly.

    var token = _jwtAuth.GetToken(claimsIdentity.Name, new List<string> { UserRole.RoleAdmin });
    return Ok(token);
}
{% endhighlight %}

In our sample codes, we choose to use JWT authentication for our APIs. Different design can be used here. For example, you can configure API to use Facebook authentication scheme directly (instead of JWT or cookie) with [Authorize(AuthenticationSchemes = FacebookDefaults.AuthenticationScheme)]. In this case, once the authentication via Facebook is done, the API is authenticated. If you choose to do so, consider the following.
1. If other claims, e.g., the role claim, need to be added, check this ASP.NET [post](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/claims?view=aspnetcore-6.0#extend-or-add-custom-claims-using-iclaimstransformation).
2. When you send the request to an API using Facebook authentication scheme, if the authentication is not done yet, a Redirection (302) will be returned, instead of Unauthorized (401). You may want to change this behavior to return Unauthorized (401) instead. You probably don't want to let non-authentication-type of APIs to handle the authentication redirection request.

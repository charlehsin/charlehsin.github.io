---
title                    : "Facebook OAuth 2.0 provider in ASP.NET Core 5 Web API."
date                     : 2021-11-19 06:20:00 -0800
last_modified_at         : 2021-11-21 16:00:00 -0800
categories               : Coding DotNet5
permalinks               : /:categories/:year/:month/:day/:title.html
header:
  teaser                 : /assets/images/teaser-oauth-provider.jpg
---

ASP.NET 5's [documentation](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/social/?view=aspnetcore-5.0&tabs=visual-studio) is not clear about how to use external OAuth 2.0 provider for the web API. 

This post talks about how to use Facebook as the external OAuth 2.0 provider for the web API. This is based on this [commit](https://github.com/charlehsin/net5-webapi-tutorial/commit/ef42ffa0f3633106fcb805001d38efca75595df6) in my [GitHub repository](https://github.com/charlehsin/net5-webapi-tutorial). The assumption is that you already have a working ASP.NET 5 Web API with JWT bearer authentication, and you only want to add the external OAuth 2.0 provider to your Web API.

## Configuring Facebook OAuth 2.0 provider and storing the secrets 

First, follow the ASP.NET 5 documentation above to get the Facebook App ID and the Facebook App secret, and to store the App ID and the App secret using Secret Manager. (For production usage, you need to use other approaches to store the secrets like App ID and App secret. This is beyond the scope of this post.) When you set up at the Facebook side, remember to configure the "https://our_hostname/signin-facebook" as the redirection URI. This will be used by ASP.NET Core.

## Enabling Facebook at the ASP.NET Core Web API

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
            // This is assuming that we use the secrete storage to store the secrets.
            facebookOptions.AppId = Configuration["Authentication:Facebook:AppId"];
            facebookOptions.AppSecret = Configuration["Authentication:Facebook:AppSecret"];
            facebookOptions.SaveTokens = true;
        });
{% endhighlight %}

## Implementing the authentication API

Then, we need to create the API to sign in via the Facebook OAuth 2.0 provider. In my GitHub repository codes, it is the SignInFacebookAsync method from the [UsersController class](https://github.com/charlehsin/net5-webapi-tutorial/blob/main/TodoApi/Controllers/UsersController.cs). To explain this method better, the ordered authentication flow is described below.
1. The front-end application sends the target API request. In our codes, it is "GET /api/Users/authenticate/facebook" (handled by SignInFacebookAsync method).
2. Upon receiving the request, SignInFacebookAsync method goes to the "challenge" flow. ASP.NET Core's internal FacebookHandler does the real "challenge" flow to prepare for the redirection information. The redirection information includes the target provider's AuthorizationEndpoint path, the client-id, and the redirection_uri, etc. ASP.NET Core internally uses "/signin-facebook" as the redirection_uri. We do not need to change this. We only need to configure this path at Facebook OAuth 2.0 provider side. Then Redirection (302) is returned to the front-end application.
3. The front-end application gets Redirection (302) and redirects to the target Facebook URL to perform the authentication at Facebook.
4. When Facebook finishes the authentication, it responds with another Redirection (302) to the front-end application. This redirection location is "https://our_hostname/signin-facebook".
5. The front-end application gets Redirection (302) and redirects to our ASP.NET Core Web API. ASP.NET Web API internally will process this "https://our_hostname/signin-facebook" request, by working with Facebook OAuth 2.0 provider.
6. After the above is done, ASP.NET Core internally responds with yet another Redirection (302) to the front-end. This time, the redirection location is the original API URL. In our codes, it is "GET /api/Users/authenticate/facebook" (handled by SignInFacebookAsync method).
7. The front-end application gets Redirection (302) and redirects to the API URL.
8. Upon receiving the request, SignInFacebookAsync method goes to the "authenticate" flow. And eventually returns with a JWT.
9. Other APIs can use the JWT for authentication. At the end of this post, we will discuss another approach, using the Facebook authentication scheme directly.

The code block of SignInFacebookAsync method is the following.

{% highlight csharp linenos %}
// This is the "GET api/Users/authenticate/facebook API"
[AllowAnonymous]
[HttpGet("authenticate/facebook")]
[ProducesResponseType(StatusCodes.Status200OK, Type = typeof(string))]
[ProducesResponseType(StatusCodes.Status302Found)]
[ProducesResponseType(StatusCodes.Status401Unauthorized)]
public async Task<IActionResult> SignInFacebookAsync()
{
    var authScheme = FacebookDefaults.AuthenticationScheme;

    // Try to authenticate.
    var authResult = await Request.HttpContext.AuthenticateAsync(authScheme);
    if (!authResult.Succeeded
        || authResult?.Principal == null
        || !authResult.Principal.Identities.Any(id => id.IsAuthenticated)
        || string.IsNullOrEmpty(authResult.Properties.GetTokenValue("access_token")))
    {
        // Challenge the Facebook OAuth 2.0 provider.
        await Request.HttpContext.ChallengeAsync(authScheme, new AuthenticationProperties
        {
            AllowRefresh = true,
            ExpiresUtc = DateTimeOffset.UtcNow.AddHours(1),
            IssuedUtc = DateTimeOffset.UtcNow,
            // We provide this API's own path here so that the final redirection can go
            // to this method.
            RedirectUri = Url.Action("SignInFacebookAsync")
        });

        // Get the location response header.
        if (Response.Headers.TryGetValue("location", out var locationResponseHeader))
        {
            return Redirect(locationResponseHeader);
        }
        return Unauthorized();
    }

    // Then you need to get the role and user ID based on the user name.
    // In real world scenario, you may have other flows to add the target user from external
    // OAuth 2.0 provider into ASP Identity.
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

## Discussion about using Facebook authentication scheme directly

In our sample codes, we choose to use JWT authentication for our APIs. Different design can be used here. For example, you can configure API to use Facebook authentication scheme directly (instead of JWT or cookie) with [Authorize(AuthenticationSchemes = FacebookDefaults.AuthenticationScheme)]. In this case, once the authentication via Facebook is done, the API is authenticated. If you choose to do so, consider the following.
1. If other claims, e.g., the role claim, need to be added, check this ASP.NET [post](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/claims?view=aspnetcore-6.0#extend-or-add-custom-claims-using-iclaimstransformation).
2. When you send an API request using Facebook authentication scheme, if the authentication is not done yet, Redirection (302) will be returned, instead of Unauthorized (401). You may want to change this behavior to return Unauthorized (401) instead. You probably don't want to let non-authentication-type of APIs to handle the authentication redirection request.

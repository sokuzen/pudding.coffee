+++
title = "Apply ASP.NET Core 3.1 - Identity on Existing Web API Project (No UI)"
description = "Play around with ASP.NET Core - Identity on Web APIs built on .NET Core 3.1"
toc = true
authors = "pudding"
tags = []
categories = ["tech", "guide"]
series = []
date =  "2021-04-11T15:48:55+08:00"
lastmod = "2021-04-11T15:48:55+08:00"
featuredImage = "featured.jpg"
draft = false
+++

Photo by [Jason Dent](https://unsplash.com/@jdent?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/password?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
  
## Introduction

.NET Core developers are #blessed :yellow_heart: to be provided an easy-to-use out-of-the-box feature to integrate user authentication and authorization on their Web APIs.

The said feature is called *Identity*. Identity is .NET Core's framework to manage user authentication: registration, login, etc. This feature is even able to generate a token to confirm the user's email.

While these features are possible on other language frameworks, .NET Core Identity goes a step further by providing a well-designed default, yet flexible, implementations to apply on APIs.

There are numerous guides, posts on how to apply this feature online. However, many of those say that the API needs to be scaffolded from scratch with Identity in mind for the developer to apply the feature. While this makes things easier, this isn't a requirement, especially when the API we're working on isn't designed to have a UI on the same project.

Without further ado, let's talk about how to apply Identity on existing .Net Core 3.1 Web APIs *without the UI*.

## Set-up

If this is your first time visiting this blog, I recommend checking out [how to set-up EF Core and Migrations first here](/posts/set-up-ef-core-on-asp-net-core-web-api-3-1/) first as we'll use both features for this guide.

Also, we'll continue that project from where we left off. Like the other guide, we'll be applying the feature on top of our existing API, with minimal changes to the classes. Specifically, *we would only be allowing authenticated users to access the API*.

## 1. Install Nuget Packages
Add the following packages as additional dependencies:

    Microsoft.AspNetCore.Identity.EntityFrameworkCore Version=3.1.10
    Microsoft.AspNetCore.Identity.UI Version=3.1.10
    Microsoft.AspNetCore.Authentication.JwtBearer Version=3.1.10

## 2. Switch the defined DB Context to IdentityDbContext
Simply switch the parent class of our DbContext to IdentityDbContext. You may also create a new class that extends IdentityDbContext, if we are to be pedantic about SOLID, but I think that's already overkill. :sweat_smile:

So from:

    public class WeatherForecastDbContext : DbContext

To:

    public class WeatherForecastDbContext : IdentityDbContext

Importing `Microsoft.AspNetCore.Identity.EntityFrameworkCore` namespace along the way.

    using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
    using Microsoft.EntityFrameworkCore;
    using Weather.Models;

    namespace Weather.Data
    {
        public class WeatherForecastDbContext : IdentityDbContext
        {
            public WeatherForecastDbContext(DbContextOptions options) : base(options)
            {
            }

            public DbSet<WeatherForecast> Forecasts { get; set; }
        }
    }

At this point you may re-run EF Migrations (from previous post) and apply the changes to the DB, or as soon as you start testing.

## 3. Add API models
We'll also add two additional API models: `LoginApiModel`, and `RegisterApiModel`

    public class LoginApiModel
    {
        [Required]
        [DataType(DataType.EmailAddress)]
        public string Email { get; set; }

        [Required]
        [StringLength(100, ErrorMessage = "The {0} must be at least {2} and at max {1} characters long.", MinimumLength = 6)]
        [DataType(DataType.Password)]
        public string Password { get; set; }
    }


    public class RegisterApiModel
    {
        [Required]
        [DataType(DataType.EmailAddress)]
        public string Email { get; set; }

        [Required]
        [StringLength(100, ErrorMessage = "The {0} must be at least {2} and at max {1} characters long.", MinimumLength = 6)]
        [DataType(DataType.Password)]
        public string Password { get; set; }
    }

These API models are only to define our API requests, so we won't be adding these on the DB Context.

Finally, let's add another that holds our JWT security key, `JwtOptions`:

    public class JwtOptions
    {
        public SecurityKey SecurityKey { get; set; }
    }

As we'll be utilizing JSON Web Tokens (JWT) for our login.

## 4. Update project startup
Here we'll be updating our project dependencies to utilize the interfaces provided by Identity. Simply define Identity to the `ConfigureServices` method and connect it to the DBContext. We'll also retrieve the defined JWT `SecurityKey` at our config:

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        ...     
        services.AddDbContext...

        services.AddDefaultIdentity<IdentityUser>(options => options.SignIn.RequireConfirmedAccount = true)
            .AddEntityFrameworkStores<WeatherForecastDbContext>()
            .AddDefaultTokenProviders();      

        SecurityKey securityKey = new SymmetricSecurityKey(Encoding.ASCII.GetBytes(_configuration["Authentication:Jwt:SecurityKey"]));
        services.Configure<JwtOptions>(options =>
            options.SecurityKey = securityKey);      
        ...
    }   

To define the `SecurityKey`, we'll then add the following entry to our `appsettings.json`:

    {
        "ConnectionStrings": {
        ...
        "Authentication": {
            "JWT": {
                "SecurityKey": "6mtve((g*owenqv%i_naeta+9ui)ozdgw!vd)u-+tj7rw5^zv3"
            }
        },
        "Logging": {
        ...
    }

Up to you what to put as `SecurityKey`. There is a minimum length for this though, just make sure it's long enough.

## 5. Then, let's make a controller for users
And we'll call it, UsersController. 

...let's set up three methods, two POSTs, and one GET. We'll tackle each method in their own sections.

    namespace Weather.Controllers
    {

        public UsersController(SignInManager<IdentityUser> signinManager,
            UserManager<IdentityUser> userManager,
            IOptions<JwtOptions> jwtOptions)
        {
            _signinManager = signinManager;
            _userManager = userManager;
            _jwtOptions = jwtOptions.Value;
        }

        [ApiController]
        [Route("[controller]")]
        public class UsersController : ControllerBase
        {
            [HttpPost]
            public async Task<IActionResult> Register([FromBody] RegisterApiModel form)
            {
                /*codes here*/
            }

            [HttpPost("login")]
            public async Task<ActionResult> Login([FromBody] LoginApiModel model)
            {
                /*codes here*/
            }

            [HttpGet("{userId}/email/confirm")]
            public async Task<ActionResult> ConfirmEmail(string userId, [FromQuery] string code)
            {
                /*codes here*/
            }
        }
    }

As you probably noticed, three dependencies were injected via the constructor. The first two, `_signinManager` and `_userManager`, were defined when we updated the project startup. These are interfaces provided by Identity.

Let's go over each method we'll be creating in the following sections.

### 5a. Register (POST users)

This method will allow a user to register their email address with a password.

We'll let Identity handle the validation: existing email, password characters, etc. And if the values are succesfully validated, Identity will provide us a confirmation code, which the API can send an email to the registered address. 

    [HttpPost]
    public async Task<IActionResult> Register([FromBody] RegisterApiModel form)
    {
        var identityUser_withNameAsEmail = new IdentityUser
        {
            UserName = form.Email,
            Email = form.Email
        };

        var result = await _userManager.CreateAsync(identityUser_withNameAsEmail, form.Password);

        if (result.Succeeded)
        {
            var code = await _userManager.GenerateEmailConfirmationTokenAsync(identityUser_withNameAsEmail);
            var callbackUrl = Url.Action("ConfirmEmail", 
                "Users", 
                new { userId = identityUser_withNameAsEmail.Id, code }, protocol: HttpContext.Request.Scheme);

            var message = $"Please confirm your account by <a href='{HtmlEncoder.Default.Encode(callbackUrl)}'>clicking here</a>.";

            //you can send the email here

            return Accepted("", new { message });
        }

        return BadRequest();
    }

We utilized `UserManager`'s two abstracted methods: `CreateAsync` and `GenerateEmailConfirmationTokenAsync`. Isn't it cool that default implementations of these are already provided by the framework? We'll utilize more methods in the following sections.

Sending an email, however, is out of this post's scope (I suggest look up articles about SendGrid or SmtpClient). We'll simply print out the link to confirm.

### 5b. Confirm Email (GET users/email/confirm)
The API has to prove that the user owns the address they registered. They have to open the link sent to their email (which in our case is only printed out) with the token string generated upon their registration. This method handles the checking of this email code sent earlier.

    [HttpGet("{userId}/email/confirm")]
    public async Task<ActionResult> ConfirmEmail(string userId, [FromQuery] string code)
    {
        if (userId == null || code == null)
            return BadRequest();
        
        var user = await _userManager.FindByIdAsync(userId);
        
        if (user == null)
            return BadRequest();
        
        var result = await _userManager.ConfirmEmailAsync(user, code);
        
        if (result.Succeeded)
            return Ok(new { message = "Your email is confirmed; you may now login your registered credentials" });
        else
            return BadRequest();
    }


Once the validation is successful, the user is flagged as registered and can finally use the API.

### 5c. Login (POST users/login)
The last method will allow the user to login utilizing JWT.

    [HttpPost("login")]
    public async Task<ActionResult> Login([FromBody] LoginApiModel model)
    {
        var result = await _signinManager.PasswordSignInAsync(model.Email,
                model.Password,
                isPersistent: false,
                lockoutOnFailure: false);

        if (result.Succeeded)
        {
            var identityUserToLogin = await _userManager.FindByEmailAsync(model.Email);
            var token = GenerateTokenAsync(identityUserToLogin, _jwtOptions.SecurityKey);
            return Ok(new { message = "You have successfully logged-in!", token });
        }

        return BadRequest();
    }

    private string GenerateTokenAsync(IdentityUser user, SecurityKey securityKey)
    {
        IList<Claim> userClaims = new List<Claim>
        {
            new Claim("Id", user.Id),
            new Claim("Email", user.Email)
        };

        return new JwtSecurityTokenHandler().WriteToken(new JwtSecurityToken(
            claims: userClaims,
            expires: DateTime.UtcNow.AddMonths(1),
            signingCredentials: new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256)
        ));
    }

As mentioned, we're utilizing JSON Web Tokens (JWT). If the user provides the correct login details, the app will generate a token according to the claims the user has. 

Explaining JWT in detail is out of the post's scope (check jwt.io for more info), but for starters, the application simply has to create a string that the user passes every time they make a request.

The login can be done manually by passing the token as one of the request headers. Postman can do this automatically though, just define the token as *Bearer Token* at the *Authentication* tab.

## 6. Allow only authenticated users to access the Weather API
To authorize only authenticated users from the Weather API, first annotate the `WeatherForecastController` with `[Authorize]`. This restricts unauthenticated users from finding themselves to the controller:

    [ApiController]
    [Route("[controller]")]
    [Authorize]
    public class WeatherForecastController : ControllerBase

Then we define the authentication scheme (JWT) and enable authentication at the Startup:

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();

        ...
            options.SecurityKey = securityKey);

        services.AddAuthentication(options => {
            options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
            options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
        }).AddJwtBearer(options => {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateAudience = false,
                ValidateIssuer = false,
                IssuerSigningKey = securityKey
            };
        });

        ...
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        ...

        app.UseAuthentication();
        app.UseAuthorization();

        ...
    }

## Final Thoughts
So that's it, we were able to make a new controller utilizing Identity's built-in features. As mentioned, this can be tested on Postman register, confirm the generated code, then try to login. The generated token upon login can be used to perform the other API methods, i.e. GET/POST weather forecast.

Identity has a lot more features out of the box. You don't need to scaffold your project from scratch with a UI. You may even customize the methods discussed at this post to your liking (e.g. lockout after unsuccessful login attempts, challenge?).

You may find the sample codes on [the official aspnetcore repo](https://github.com/dotnet/aspnetcore/tree/main/src/Identity/UI/src/Areas/Identity/Pages/V4/Account). Check the class for each feature available. You may also find easy to read samples [here](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/samples/IdentitySample.Mvc/Controllers/AccountController.cs). 

Source code can be found at the [apply-identity](https://github.com/sokuzen/SimpleWeatherForecast/tree/apply-identity) branch. Feel free to send me an email or a tweet and let me know your thoughts!

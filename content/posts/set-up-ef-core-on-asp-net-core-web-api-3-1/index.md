+++
title = "Set-up Entity Framework Core on .NET Core 3.1 Web APIs"
description = "Utilize EF Core and Migrations to persist your data on an SQL database"
date = "2020-12-12T19:59:00+08:00"
lastmod = "2020-01-09T18:40:29+08:00"
featuredImage = "featured.jpg"
toc = true
authors = "pudding"
tags = ["csharp", "netcore", "rest", "web", "api", "entityframework", "migrations", "database", "persistence", "sql", "orm", "thanksOOP"]
categories = ["tech", "guide"]
series = []
+++

Photo by [Chester Alvarez](https://unsplash.com/@chesteralvarez?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/gears?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## Introduction

I have recently been dabbling with .NET Core after years of being a Java specialist. And so far I'm liking it.

That's another story, but in the meantime let's talk about how to set up EF Core, code-first, on .NET Core 3.1.

It took my time-- a whole weekend-- to find references and make it work. I don't know if it's my inexperience in the framework kicking in, or there isn't much on the web.

Either way here's a post for people who will experience (or are experiencing) the same thing, so they won't need to spend the same time I did.

## Code-First

Let me first describe what made me write this up. 

I have an existing Web API that serves non-persistent data. The API is designed in a way that one can switch and persist to another data context easily, the latter of which I wanted to do. And I don't want to make a whole database schema by hand; I want the DB to be based on the models I already defined. I know that's possible in Spring, so I'm sure I can do that too in .NET Core.

And there, I found EF Core and Migrations.

## Set-up

For this tutorial, I'll use SQL Server Development Edition. I also customized the pre-loaded Weather Forecast Controller-- that guy Visual Studio generates when one makes a .NET Core Web API from a template. I'll make this simple so you can easily grasp the concept. Feel free to modify the steps as you go for your case.

This API has a single model class, `WeatherForecast`, which we plan to persist in the installed database. There are two REST methods:
- a GET that lists all weather forecasts;
- and POST that creates one. 

The data currently is saved in a non-persistent object injected as a singleton dependency. See the [project's master branch on Github](https://github.com/sokuzen/SimpleWeatherForecast) for the codes.

Our goal here is to minimize the impact on the already existing classes, simply adding a feature on top of them.

You can test it by running POST and GET curls. Or Postman, whichever you prefer.

## 1. Install Nuget Packages

FIrst off is to install the necessary Nuget packages. Install the following on the solution, whether via command line or via Visual Studio:
    
    Microsoft.EntityFrameworkCore version 3.1.10
    Microsoft.EntityFrameworkCore.Design version 3.1.10
    Microsoft.EntityFrameworkCore.SqlServer version 3.1.10
    
EF Core is the main package we'll be applying. It has to _talk_ to our database, and therefore importing its SQL Server driver. Lastly, Design is required by Migrations. Here's `ItemGroup` node under the project's csproj file:

    <ItemGroup>
        <PackageReference Include="Microsoft.EntityFrameworkCore" Version="3.1.10" />
        <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="3.1.10">
            <PrivateAssets>all</PrivateAssets>
            <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
        </PackageReference>
        <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="3.1.10" />
    </ItemGroup>

## 2. Create a class that implements DbContext

The simplest way to create a DB Context is to create a class that implements `Microsoft.EntityFrameworkCore.DbContext`. Define a constructor to inject a `DbContextOptions` object. Then define `DbSet` fields for each model.
    
    using Microsoft.EntityFrameworkCore;
    using Weather.Models;

    namespace Weather.Data
    {
        public class WeatherForecastDbContext : DbContext
        {
            public WeatherForecastDbContext(DbContextOptions options) : base(options)
            {
            }

            public DbSet<WeatherForecast> Forecasts { get; set; }
        }
    }

Also, I had to add an annotated ID property to the WeatherForecast class. *EF is throwing up when a model lacks a primary key*. [There are probably other ways to do this](https://stackoverflow.com/questions/15381233/can-we-have-table-without-primary-key-in-entity-framework/15381324), I just opted for the easier one.
    
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }


## 3. Install Entity Framework Migrations v3.1.10

This feature allows us to create our _codes-first_, then create the DB schema based on our defined models and DB Context.

*You can skip this step and the next*, but do you want to write the DB schema manually? And tinker on the tables every time you have changed your models?

Up to you, but here to install EF Migrations, run the following command on your command line:
    
    dotnet tool install --global dotnet-ef --version 3.1.10

This simply installs dotnet-ef on the global scope. Also using version 3.1.10.

## 4. Apply Migrations

To apply Migrations to our DbContext, first, we have to create a class that implements `IDesignTimeDbContextFactory`. EF Core looks into the class that implements the said interface, referencing the DbContext class we made earlier.

    using Microsoft.EntityFrameworkCore;
    using Microsoft.EntityFrameworkCore.Design;
    using Microsoft.Extensions.Configuration;
    using System.IO;
    using Weather.Data;

    namespace Weather
    {
        public class WeatherForecastDesignTimeDbContextFactory : IDesignTimeDbContextFactory<WeatherForecastDbContext>
        {
            public WeatherForecastDbContext CreateDbContext(string[] args)
            {
                var configuration = new ConfigurationBuilder()
                    .SetBasePath(Directory.GetCurrentDirectory())
                    .AddJsonFile("appsettings.json")
                    .Build();

                var dbContextBuilder = new DbContextOptionsBuilder();

                var connectionString = configuration
                            .GetConnectionString("SqlConnectionString");

                dbContextBuilder.UseSqlServer(connectionString);

                return new WeatherForecastDbContext(dbContextBuilder.Options);
            }
        }
    }

As you can see there's a reference to `appsettings.json` on this class. Update `appsettings.json` with the DB's connection string.

    {
        "ConnectionStrings": {
            "SqlConnectionString": "[CONNECTION STRING HERE]"
        },
        "Logging": {
            *ommited for brevity*
    }

Note: If your `IDesignTimeDbContextFactory` class is on another project where appsettings is located (i.e. you have a Repositories layer/project that handles all the DB-related calls, separate from the API layer), you can create a separate `appsettings.json` file in the same directory as the `IDesignTimeDbContextFactory` class. Or navigate the already defined file at the definition part (`AddJsonFile`, I haven't done the latter advice though so try at your own risk).

Finally, we need to run the following commands: 

    dotnet ef migrations add Initial
    dotnet ef database update

The first command generates Migrations classes that create/update the tables based on the defined DbContext and `IDesignTimeDbContextFactory`. The next line runs the generated classes and updates the database defined as your connection string. You may also replace `Initial` with your preferred name.

At this point, you should now see a new table-- `Forecast`-- on your defined DB.

Note that in case you have changed the models, i.e. you end up adding another model or field, run the above commands again with a different name.

    dotnet ef migrations add SomeColumnAdded
    dotnet ef database update

## 5. Create a new service implementation

By SOLID principle, we would rather add classes rather than change those that already exist. So we'll make a new implementation of `IWeatherForecastRepository` and inject the DbContext we made earlier.

    using System.Collections.Generic;
    using System.Linq;
    using Weather.Models;

    namespace Weather.Data
    {
        public class DbWeatherForecastRepository : IWeatherForecastRepository
        {
            private readonly WeatherForecastDbContext _dbContext;
            public DbWeatherForecastRepository(WeatherForecastDbContext dbContext)
            {
                _dbContext = dbContext;
            }

            public IEnumerable<WeatherForecast> GetAll()
            {
                return _dbContext.Forecasts.AsEnumerable();
            }

            public void Save(WeatherForecast weatherForecast)
            {
                _dbContext.Add(weatherForecast);
                _dbContext.SaveChanges();
            }
        }
    }


## 6. Update the API to utilize the new service implementation

Simply configure the startup class to use the DbContext and the repository we just made. Replace:

    services.AddSingleton<DefaultWeatherForecastDataContext>();
    services.AddScoped<IWeatherForecastRepository, DefaultWeatherForecastRepository>();

To:

    string connection = _configuration.GetConnectionString("SqlConnectionString");
    services.AddDbContext<WeatherForecastDbContext>(options =>
        options.UseSqlServer(connection,
            b => b.MigrationsAssembly("Weather")));
    services.Configure<WeatherForecastDbContext>(options => {
        options.Database.Migrate();
    });

    services.AddScoped<IWeatherForecastRepository, DbWeatherForecastRepository>();

And that's it! You should now be able to see the data persisted on the database. Turning off the server will not reset the data.

## Final Notes

As seen, we didn't even change anything on the controller class. I had to add an ID to the model though, I hope that's within the rules :wink:. 

But ultimately, we simply added an implementation of the existing interface, then switched-out the existing one by injecting the new one to the already existing classes.

Also, let me reiterate here that when you have changes to the models, you have to re-apply Migrations by running the two commands mentioned in step 4. EF wants to prevent data loss when you update your schema.

You can see the changes we applied on the [apply-ef-migrations branch](https://github.com/sokuzen/SimpleWeatherForecast/tree/apply-ef-migrations) of the same project.
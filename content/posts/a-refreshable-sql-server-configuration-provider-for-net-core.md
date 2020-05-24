---
title: "A Refreshable Sql Server Configuration Provider for .NET Core"
date: 2020-05-22T00:04:09+04:30
draft: true

images: ["/images/alexander-andrews-MrCetm-g5n4-unsplash.jpg"]
---

As you know you there are already [configuration providers](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-2.2#custom-configuration-provider) for a variety of sources. ‌But if you didn't find your source configuration provider, you can simply implement it. It just take a few minutes to implement your own configuration provider. In this blog post I want to show you how you can implement custom SQL Server configuration provider and more importantly how you can refresh the configuration whenever your data in the database changes.

This article has two parts. First you learn how to create your provider. Then you learn how to reload the configuration. The source code is available in [GitHub](https://github.com/Mousavi310/dotnet-training/tree/master/ConfigurationProviders).

## Implement the SQL Server Configuration Provider

To implement a configuration provider you need to have at least 2 classes: a `Configuration Source` and  a `Configuration Provider`. The configuration source has the information about the source (such as the location of JSON file, or the connection string of settings table and etc). The configuration provider uses the configuration source to read key/value pair settings from the source and store them in its `Data` property. `Data` is a `Dictionary<string, string>` type and your job is to fill it by reading database data. Let's begin to create a simple Sql Server Configuration provider:

Create a solution:

``` dotnet
mkdir ConfigurationProviders
cd ConfigurationProviders
dotnet new sln
```

Then the provider class library:

``` dotnet
dotnet new classlib -o ConfigurationProviders.SqlServer
dotnet sln add ConfigurationProviders.SqlServer
```

To implement provider base classes you must add `Microsoft.Extensions.Configuration` package to your project. You also need to add `Microsoft.Data.SqlClient` package to read data from SQL Server. 

``` dotnet
dotnet add .\ConfigurationProviders.SqlServer\ConfigurationProviders.SqlServer.csproj package Microsoft.Data.SqlClient --version 1.1.3
dotnet add .\ConfigurationProviders.SqlServer\ConfigurationProviders.SqlServer.csproj package Microsoft.Extensions.Configuration --version 3.1.4
```

To test our provider I create a `Web API` sample project:

``` dotnet
dotnet new webapi -o ConfigurationProviders.Samples
dotnet sln add ConfigurationProviders.Samples
```

And add a reference to the provider project:

``` dotnet
dotnet add .\ConfigurationProviders.Samples\ConfigurationProviders.Samples.csproj reference .\ConfigurationProviders.SqlServer\ConfigurationProviders.SqlServer.csproj
```

In provider project add `SqlServerConfigurationSource` implement `IConfigurationSource` interface. This is our configuration source which keep connection string to the database. Notice `IConfigurationSource` has a `Build` parameter that is used to create an instance of the provider.

``` csharp
public class SqlServerConfigurationSource : IConfigurationSource
{
    public string ConnectionString { get; set; }
    public IConfigurationProvider Build(IConfigurationBuilder builder)
    {
        return new SqlServerConfigurationProvider(this);
    }
}
```

Add `SqlServerConfigurationProvider` class that is responsible for reading data from the database:

``` csharp
public class SqlServerConfigurationProvider : ConfigurationProvider
{
    private readonly SqlServerConfigurationSource _source;
    private const string Query = "select [Key], [Value] from Settings";

    public SqlServerConfigurationProvider(SqlServerConfigurationSource source)
    {
        _source = source;
    }

    public override void Load()
    {
        var dic = new Dictionary<string, string>();
        using (var connection = new SqlConnection(_source.ConnectionString))
        {
            var query = new SqlCommand(Query, connection);

            query.Connection.Open();

            using (var reader = query.ExecuteReader())
            {
                while (reader.Read())
                {
                    dic.Add(reader[0].ToString(), reader[1].ToString());
                }
            }
        }

        Data = dic;
    }
}
```

This assumes that we have stored our configuration in `Settings` table. For convinience in using the provider we add an extension method `AddSqlServer` which receives a connection string and an `IConfigurationBuilder`. This way we can easily add our provider to the builder:

``` csharp
public static class SqlServerConfigurationBuilderExtensions
{
    public static IConfigurationBuilder AddSqlServer(this IConfigurationBuilder builder, string connectionString)
    {
        return builder.Add(new SqlServerConfigurationSource{ConnectionString = connectionString});
    }
}
```

Now let's verify our provider in the `Web API` project. I want to inject the configuration using `Options` pattern. Assume that we use an email service and to send emails we need an API key. So we can create following class:

``` csharp
public class EmailServiceOptions
{
    public string ApiKey { get; set; }
}
```

And configure it in the `Startup`:

``` csharp
public void ConfigureServices(IServiceCollection services)
{
    services.Configure<EmailServiceOptions>(Configuration.GetSection("EmailService"));
    services.AddControllers();
}
```

And finally add the provider to configuration builder `Program.cs` file:

``` csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureAppConfiguration(builder =>
                    {
                        //Here we added our configuration provider.
                        builder.AddSqlServer(
                            "Server=localhost;Database=Example;User Id=sa;Password=your(#SecurePassword!123)");
                    })
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                });
```


To test our configuration I inject `IOptions<EmailServiceOptions>` in a controller:

``` csharp
public class HomeController : ControllerBase
{
    private readonly IOptions<EmailServiceOptions> _options;

    public HomeController(IOptions<EmailServiceOptions> options)
    {
        _options = options;
    }

    [Route("api/email-service/key")] 
    public IActionResult Index()
    {
        return Ok(_options.Value.ApiKey);
    }
}
```

Finally insert API key configuration in the `Settings` table:

``` sql
Insert into Settings([Key], Value) Values('EmailService:ApiKey', 'f08e37a7-af75-49a7-80c3-9ecd7df9ba74')
```

``` dotnet
dotnet run --project .\ConfigurationProviders.Samples\ConfigurationProviders.Samples.csproj
```

In another terminal if we curl the API:

``` bash
curl https://localhost:5001/api/email-service/key
f08e37a7-af75-49a7-80c3-9ecd7df9ba74
```

We see that we could successfully read configuration from SQL Server!

Notice that if we change a value in database, the new value is not available in our configuration provider, lets test it:

``` sql
Update Settings Set Value = '0aaf9ffc-d637-4c40-8a1d-2ff7d73b3b6a' Where [Key] = 'EmailService:ApiKey'
```

``` bash
curl https://localhost:5001/api/email-service/key
f08e37a7-af75-49a7-80c3-9ecd7df9ba74
```

It does not work. Lets return back the value:

``` sql
Update Settings Set Value = 'f08e37a7-af75-49a7-80c3-9ecd7df9ba74' Where [Key] = 'EmailService:ApiKey'
```


The above code works and we can test it by creating an ASP.NET application.


But what if a config change in the database? Shouldn't we refresh or options?

## Reload configuration

As you know, IOptions<T> does not reload configurations. It just read once from `Data` and cache it for entire lifetime of your application. You can use IOptionsSnapshot<T> which read configuration from `Data` in each HTTP Request. Even if we use IOptionsSnapshot<T> we know that `Data` is loaded once. To reload `Data` we can use something called `IChangeToken`. Using `IChangeToken` we can **Watch** something is changed and in change event we can call a callback (here we want to reload our `Data` from database by just calling `Load` method again).

So lets do the change. First change IOptions<EmailServiceOptions> to IOptionsSnapshot<EmailServiceOptions>:

``` csharp
public HomeController(IOptionsSnapshot<EmailServiceOptions> options)
{
    _options = options;
}
```

Add change token in your provider constructor:

``` csharp
 public SqlServerConfigurationProvider(SqlServerConfigurationSource source)
{
    _source = source;

    if (_source.SqlServerWatcher != null)
    {
        _changeTokenRegistration = ChangeToken.OnChange(
            () => _source.SqlServerWatcher.Watch(),
            Load
        );
    }
}

```

``` csharp
public class SqlServerConfigurationSource : IConfigurationSource
{
    //...
    public ISqlServerWatcher SqlServerWatcher { get; set; }
    //...
}
```

``` csharp
public interface ISqlServerWatcher : IDisposable
{
    IChangeToken Watch();
}
```

``` csharp
internal class SqlServerPeriodicalWatcher : ISqlServerWatcher
{
    private readonly TimeSpan _refreshInterval;
    private IChangeToken _changeToken;
    private readonly Timer _timer;
    private CancellationTokenSource _cancellationTokenSource;

    public SqlServerPeriodicalWatcher(TimeSpan refreshInterval)
    {
        _refreshInterval = refreshInterval;
        _timer = new Timer(Change, null, TimeSpan.Zero, _refreshInterval);
    }

    private void Change(object state)
    {
        _cancellationTokenSource?.Cancel();
    }

    public IChangeToken Watch()
    {
        _cancellationTokenSource = new CancellationTokenSource();
        _changeToken =  new CancellationChangeToken(_cancellationTokenSource.Token);

        return _changeToken;
    }

    public void Dispose()
    {
        _timer?.Dispose();
        _cancellationTokenSource?.Dispose();
    }
}
```

Finally

``` csharp
public static IConfigurationBuilder AddSqlServer(this IConfigurationBuilder builder, 
            string connectionString,
            TimeSpan? refreshInterval = null)
{
    return builder.Add(new SqlServerConfigurationSource
    {
        ConnectionString = connectionString,
        SqlServerWatcher = refreshInterval.HasValue ? 
            new SqlServerPeriodicalWatcher(refreshInterval.Value) : null 
    });
}
```

And in your Program.cs

``` csharp
builder.AddSqlServer(
                "Server=localhost;Database=Example;User Id=sa;Password=your(#SecurePassword!123)",
                TimeSpan.FromSeconds(5));
```

``` dotnet
dotnet run --project .\ConfigurationProviders.Samples\ConfigurationProviders.Samples.csproj
```

In another terminal

``` dotnet
curl https://localhost:5001/api/email-service/key
f08e37a7-af75-49a7-80c3-9ecd7df9ba74
```
Lets change the value:

``` sql
Update Settings Set Value = '0aaf9ffc-d637-4c40-8a1d-2ff7d73b3b6a' Where [Key] = 'EmailService:ApiKey'
```

Waits for 5 seconds and then retry:

``` dotnet
curl https://localhost:5001/api/email-service/key
0aaf9ffc-d637-4c40-8a1d-2ff7d73b3b6a
```

CancellationChangeToken is very easy to work. This change token receive an cancellationToken as input in the constructor.

Whenever you cancel the token, it will call consumer action, which in our case is reloading configuration from the SQL Server.
we can refresh our data whenever we cancel the token. If you want to refresh all of your configuration every 10s, just use a timer.


[This is todo: If the number of config is two large then you can check for a sentinal value. You just check the value of this key and whenever this value changes in SQL server, you can reload all configurations from SQL Server.]


- 




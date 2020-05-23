---
title: "A Refreshable Sql Server Configuration Provider for .NET Core"
date: 2020-05-22T00:04:09+04:30
draft: true
---

As you know you the ...

This article has two part. First you learn how to create your provider. Then you learn how to reload configuration. 

But if you want to read from SQL Server you can implement that by inheriting...

## Implement the SQL Server Configuration Provider

To implement a configuration provider you need 2 class: `Configufation source` and `Confiugration Provider`. A configuration source has information about the source (such as location of json file, or connection string of settings table and etc). Configuration provider uses Configurtion source to read key/value pair settings from the source and store them in its `Data` property.

Lets begin to create a simple Sql Server Configuration provider:

- Create Solution

``` dotnet
mkdir ConfigurationProviders
cd ConfigurationProviders
dotnet new sln
```

- Create Provider project

``` dotnet
dotnet new classlib -o ConfigurationProviders.SqlServer
dotnet sln add ConfigurationProviders.SqlServer
```

You also need `Microsoft.Data.SqlClient` to read data from SQL Server. 

``` dotnet
dotnet add .\ConfigurationProviders.SqlServer\ConfigurationProviders.SqlServer.csproj package Microsoft.Data.SqlClient --version 1.1.3
dotnet add .\ConfigurationProviders.SqlServer\ConfigurationProviders.SqlServer.csproj package Microsoft.Extensions.Configuration --version 3.1.4
```

- Create Asp.net Core sample project

``` dotnet
dotnet new webapi -o ConfigurationProviders.Samples
dotnet sln add ConfigurationProviders.Samples
```

``` dotnet
dotnet add .\ConfigurationProviders.Samples\ConfigurationProviders.Samples.csproj reference .\ConfigurationProviders.SqlServer\ConfigurationProviders.SqlServer.csproj
```

- Implement source

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

- Implement provider

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

- Implement extensions

``` csharp
public static class SqlServerConfigurationBuilderExtensions
{
    public static IConfigurationBuilder AddSqlServer(this IConfigurationBuilder builder, string connectionString)
    {
        return builder.Add(new SqlServerConfigurationSource{ConnectionString = connectionString});
    }
}
```

- Add EmailServiceOptions Model

``` csharp
public class EmailServiceOptions
{
    public string ApiKey { get; set; }
}
```

- Configure Options in startup

``` csharp
public void ConfigureServices(IServiceCollection services)
{
    services.Configure<EmailServiceOptions>(Configuration.GetSection("EmailService"));
    services.AddControllers();
}
```

And set it in Program.cs file:

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

- Set EmailServiceOptions:ApiKey it in database

Now assuming you have a `Settings` table in your datbase, add thise records:

``` sql
Insert into Settings([Key], Value) Values('EmailService:ApiKey', 'f08e37a7-af75-49a7-80c3-9ecd7df9ba74')
```

- To test our configuration can be read correctly from database, I inject IOptions<EmailServiceOptions> in a simple controller

``` csharp
public class HomeController : ControllerBase
{
    private readonly IOptions<EmailServiceOptions> _options;

    public HomeController(IOptions<EmailServiceOptions> options)
    {
        _options = options;
    }

    [Route("api")] 
    public IActionResult Index()
    {
        return Ok(_options.Value.ApiKey);
    }
}
```

``` dotnet
dotnet run --project .\ConfigurationProviders.Samples\ConfigurationProviders.Samples.csproj
```

In another console if we curl the API:

``` bash
curl https://localhost:5001/api
f08e37a7-af75-49a7-80c3-9ecd7df9ba74
```

The above code works and we can test it by craeting an ASP.NET application.

- Test with IOptionsSnapshot<ConfigModel>
- Describe why snapshot didn't work.

But what if a config change in the database? Shouldn't we refresh or options?

## Reload configuration

To refresh configuration Data, you need to use a change token. 

CancellationChangeToken is very easy to work. This change token receive an cancellationToken as input in the constructor.

Whenever you cancel the token, it will call consumer action, which in our case is reloading configuration from the SQL Server.
we can refresh our data whenever we cancel the token. If you want to refresh all of your configuration every 10s, just use a timer.

- Implement periodical watcher
- Use watcher
- Test againg

[This is todo: If the number of config is two large then you can check for a sentinal value. You just check the value of this key and whenever this value changes in SQL server, you can reload all configurations from SQL Server.]


- 




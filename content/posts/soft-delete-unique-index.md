---
title: "Unique Index Supporting Soft Delete in ASP.NET Core"
date: 2020-05-08T12:24:37+04:30
draft: false
images: ["/images/jessica-ruscello-DoSDQvzjeH0-unsplash.jpg"]
---

A unique index on a field help developers to ensure that, their table will not contains two records with the same value for the field. A soft delete helps to 

The problems arises when

So lets start with an exmaple using `ASP.NET Core 3.1 Web API`. Create new app:

``` dotnet
dotnet new webapi -o SoftDeleteSample
cd SoftDeleteSample
dotnet add package Microsoft.EntityFrameworkCore --version 3.1.3
```

Open the project in your IDE (Visual Studio in my case):

``` bash
start .\SoftDeleteSample.csproj
```

Set Sql Server connection string in your `appsettings.json` as follow:

``` json
{
    "ConnectionStrings" : {
        "DefaultConnection": "Server=.;Database=Store-Dev;User Id=sa;Password=yourStrong(!)Password"
    }
}
```

Add `Product` class:

``` csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }

    public bool IsDeleted { get; set; }
}
```

And `StoreContex`:

``` csharp
public class StoreContext
{
    public DbSet<Product> Products { get; set; }
}
```

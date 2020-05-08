---
title: "Unique Index Supporting Soft Delete in ASP.NET Core"
date: 2020-05-08T12:24:37+04:30
draft: false
images: ["/images/jessica-ruscello-DoSDQvzjeH0-unsplash.jpg"]
---


### TLDR;


### Real Example


A unique index on a field help developers to ensure that, their table will not contains two records with the same value for the field. A soft delete helps to 

The problems arises when

So lets start with an exmaple using `ASP.NET Core 3.1 Web API`. Create new app:

``` dotnet
dotnet new webapi -o SoftDeleteSample
cd SoftDeleteSample
dotnet add package Microsoft.EntityFrameworkCore --version 3.1.3
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 3.1.3
```
and install tools for enabling database migration:

``` dotnet
dotnet tool install --global dotnet-ef
dotnet add package Microsoft.EntityFrameworkCore.Design
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
public class StoreContext : DbContext
{
    public DbSet<Product> Products { get; set; }
}

```

In `Startup` class, add StoreContext to service collection:

``` csharp
```

Now we configure Product

First we want a unique index on `Name` field:


Secondly, we want to fitler 


Now migrate database:

Create `ProductsController` and inject `StoreContext`:

Add API for creating a product

Add API for getting list of products

Add API for deleting a product


So lets have a scenario:

If we delete a record `Red Hat` and Want to add new `Red Had` product we receive following error:


### Use Filter Index to Rescue

migrate database:

### Conclusion
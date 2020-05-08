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
        "DefaultConnection": "Server=.;Database=Store-Dev;User Id=sa;Password=your(#SecurePassword!123)"
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

``` dotnet
dotnet ef migrations add "Initial"
dotnet ef database update
```

Create `ProductsController` and inject `StoreContext`:

Add API for creating a product

Add API for getting list of products

Add API for deleting a product


So lets have a scenario:

If we delete a record `White Shirt` and Want to add new `White Shirt` product we receive following error:

``` curl
curl --location --request POST 'https://localhost:5001/api/products' --insecure --header 'Content-Type: application/json' --data-raw '{
"name": "White Shirt"
}'
```

``` curl
 curl --location --request GET 'https://localhost:5001/api/products' --insecure

 [{"id":6,"name":"White Shirt","isDeleted":false}]
```

Try to delete:
 

``` curl
curl --location --request DELETE 'https://localhost:5001/api/products/6' --insecure
```

``` curl
 curl --location --request GET 'https://localhost:5001/api/products' --insecure

 []
```

If we retry to add `White Shirt`, we get following 

``` curl
curl --location --request POST 'https://localhost:5001/api/products' --insecure --header 'Content-Type: application/json' --data-raw '{
"name": "White Shirt"
}'

Microsoft.EntityFrameworkCore.DbUpdateException: An error occurred while updating the entries. See the inner exception for details.
 ---> Microsoft.Data.SqlClient.SqlException (0x80131904): Cannot insert duplicate key row in object 'dbo.Products' with unique index 'IX_Products_Name'. The duplicate key value is (White Shirt).
The statement has been terminated.
   at Microsoft.Data.SqlClient.SqlCommand.<>c.<ExecuteDbDataReaderAsync>b__164_0(Task`1 result)
   at System.Threading.Tasks.ContinuationResultTaskFromResultTask`2.InnerInvoke()
   at System.Threading.Tasks.Task.<>c.<.cctor>b__274_0(Object obj)
   at System.Threading.ExecutionContext.RunInternal(ExecutionContext executionContext, ContextCallback callback, Object state)
--- End of stack trace from previous location where exception was thrown ---
   at System.Threading.Tasks.Task.ExecuteWithThreadLocal(Task& currentTaskSlot, Thread threadPoolThread)
--- End of stack trace from previous location where exception was thrown ---
   at Microsoft.EntityFrameworkCore.Storage.RelationalCommand.ExecuteReaderAsync(RelationalCommandParameterObject parameterObject, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Storage.RelationalCommand.ExecuteReaderAsync(RelationalCommandParameterObject parameterObject, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Storage.RelationalCommand.ExecuteReaderAsync(RelationalCommandParameterObject parameterObject, CancellationToken cancellationToken)
   at Microsoft.EntityFrameworkCore.Update.ReaderModificationCommandBatch.ExecuteAsync(IRelationalConnection connection, CancellationToken cancellationToken)
```
Indicating that `White Shirt` is duplicated.
### Use Filter Index to Rescue

migrate database:

### Conclusion
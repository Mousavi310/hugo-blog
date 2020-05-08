---
title: "Unique Index With Soft Delete Support in Entity Framework Core"
publishdate: 2020-05-08T12:24:37+04:30
date: 2020-05-08T12:24:37+04:30
draft: false
images: ["/images/jessica-ruscello-DoSDQvzjeH0-unsplash.jpg"]
---

A `unique index` on a table field, helps developers to ensure that, their table will not have any duplicated values for that field. On the other hand, a `soft delete` helps us to logically delete a record while keeping the record itself, using a flag field (for example `IsDeleted` boolean field) in the table. The problems arise when you want to add a record that its unique indexed field value, has been already used in a deleted record. Although the record is logically deleted, the unique index is not aware that the record has been deleted (softly) and raise duplication error in response. In this blog post, I want to show how we can solve this issue.

### Real Example in .NET Core 3.1
So let's consider an example using `ASP.NET Core 3.1 Web API` (you can see the full source code in [GitHub](https://github.com/Mousavi310/dotnet-training/tree/master/SoftDeleteSample)). Assuming `StoreContext` is our `DbContext` and  `Product` is our entity class:

``` csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public bool IsDeleted { get; set; }
}
```

To achieve soft deletion and unique index, we must add following configuration to `StoreContext` class:

``` csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    //This will make sure we have unique product names
    modelBuilder
        .Entity<Product>()
        .HasIndex(p => p.Name)
        .IsUnique();

    //This will exclude products with IsDeleted=1 when querying Products using EF
    modelBuilder
        .Entity<Product>()
        .HasQueryFilter(p => !p.IsDeleted);
}
```

I already created an `ProductsController` to expose our REST API. So lets create a product named `White Shirt`:

``` curl
curl --location --request POST 'https://localhost:5001/api/products' --insecure --header 'Content-Type: application/json' --data-raw '{
"name": "White Shirt"
}'

{"id":6,"name":"White Shirt","isDeleted":false}
```

The product created successfully with `Id` 6. We can view all products:

``` curl
 curl --location --request GET 'https://localhost:5001/api/products' --insecure

 [{"id":6,"name":"White Shirt","isDeleted":false}]
```

Let's delete the product:

``` curl
curl --location --request DELETE 'https://localhost:5001/api/products/6' --insecure
```
The product successfully deleted. To make sure the product is not shown in our list API, let’s fetch the list of products again:

``` curl
 curl --location --request GET 'https://localhost:5001/api/products' --insecure

 []
```

If we retry to add `White Shirt` again, we will get a    `SqlException` error:

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
Indicating that `White Shirt` is already inserted and `IX_Products_Name` unique index prevents that.

### Use Filter Index to Rescue
SQL Server 2008 introduced [filtered index](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes) feature which allows us to have an index with conditions. We have only one condition: `IsDeleted = 0`; we want to index all not deleted records and make sure only not deleted records are unique. To achieve this in Entity Framework Core, add `HasFilter("IsDeleted = 0")` to `StoreContext` configuration:

``` csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    //This will make sure we have unique product names
    modelBuilder
        .Entity<Product>()
        .HasIndex(p => p.Name)
        //This make index filtered
        .HasFilter("IsDeleted = 0")
        .IsUnique();

    //This will exclude products with IsDeleted=1 when querying Products using EF
    modelBuilder
        .Entity<Product>()
        .HasQueryFilter(p => !p.IsDeleted);
}
```

Migrate your database and test it again by adding `White Shirt` product:
``` curl
curl --location --request POST 'https://localhost:5001/api/products' --insecure --header 'Content-Type: application/json' --data-raw '{
"name": "White Shirt"
}'

{"id":8,"name":"White Shirt","isDeleted":false}
```
As you see above the product created with `Id` 8. We can see it in the list API:

``` curl
 curl --location --request GET 'https://localhost:5001/api/products' --insecure

 [{"id":8,"name":"White Shirt","isDeleted":false}]
```

If you want to add `White Shirt` product while it is not (soft) deleted, the filter index prevents it:

``` curl
curl --location --request POST 'https://localhost:5001/api/products' --insecure --header 'Content-Type: application/json' --data-raw '{
"name": "White Shirt"
}'

Microsoft.EntityFrameworkCore.DbUpdateException: An error occurred while updating the entries. See the inner exception for details.
 ---> Microsoft.Data.SqlClient.SqlException (0x80131904): Cannot insert duplicate key row in object 'dbo.Products' with unique index 'IX_Products_Name'. The duplicate key value is (White Shirt).
```

### Conclusion
In this article, first, I tried to show you how we can add soft deletion to our data models. Then I showed you if we have a unique index, this could cause a problem if we want to add a record that its unique indexed field value, has been already used in a deleted record. Then using `Filtered Index` I only unique indexed the records that are not soft deleted. 
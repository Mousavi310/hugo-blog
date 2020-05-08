---
title: "Unique Index Supporting Soft Delete in ASP.NET Core"
date: 2020-05-08T12:24:37+04:30
draft: false
images: ["/images/jessica-ruscello-DoSDQvzjeH0-unsplash.jpg"]
---

A unique index on a field help developers to ensure that, their table will not contains two records with the same value for the field. A soft delete helps to logically delete a record while keeping its data, using a flag field (for example `IsDeleted`) in the table. The problems arises when you want to add a record that has the same value in unique indexed field that is used already in a deleted record. Although the record is logically deleted, but unique index is not aware that record is soft-deleted and raise duplication error.

### TLDR;
You can add `HasFilter("IsDeleted = 0")` in your EF Core configuration as follow:

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

            //....
        }
```

that make your index filtered. This way, unique index works only on not deleted records.

### Real Example
So lets start with an exmaple using `ASP.NET Core 3.1 Web API`. Assuming `StoreContext` is our `DbContext` and  `Product` entity class:

``` csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }

    public bool IsDeleted { get; set; }
}
```

To achieve soft delete and unique index we must add following configuration to `StoreContext` class:

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

So lets create a product named `White Shirt`:

``` curl
curl --location --request POST 'https://localhost:5001/api/products' --insecure --header 'Content-Type: application/json' --data-raw '{
"name": "White Shirt"
}'

{"id":6,"name":"White Shirt","isDeleted":false}
```

The product created successfully with `Id` 6. Now get all products:

``` curl
 curl --location --request GET 'https://localhost:5001/api/products' --insecure

 [{"id":6,"name":"White Shirt","isDeleted":false}]
```

Let's delete the product:

``` curl
curl --location --request DELETE 'https://localhost:5001/api/products/6' --insecure
```
The product successfully deleted. To make sure the product is not shown in our list API, let's fetch list of products again:

``` curl
 curl --location --request GET 'https://localhost:5001/api/products' --insecure

 []
```

If we retry to add a product `White Shirt`, we get an `SqlException` error:

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
Sql server in 2008 introduced [filtered index](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes) concept
which allow us to only check index based on some conditions. The conditions in our case is `IsDeleted = 0`; we want only index not deleted records and make sure only not deleted records are unique. To do this add `HasFilter("IsDeleted = 0")` to `StoreContext` configuration:

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

And the filter index prevent duplication of not deleted products:

``` curl
curl --location --request POST 'https://localhost:5001/api/products' --insecure --header 'Content-Type: application/json' --data-raw '{
"name": "White Shirt"
}'

Microsoft.EntityFrameworkCore.DbUpdateException: An error occurred while updating the entries. See the inner exception for details.
 ---> Microsoft.Data.SqlClient.SqlException (0x80131904): Cannot insert duplicate key row in object 'dbo.Products' with unique index 'IX_Products_Name'. The duplicate key value is (White Shirt).
```

### Conclusion
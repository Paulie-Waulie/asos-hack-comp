# CosmosDB Credentials

* CosmosDB Uri: `https://hack-comp-brum.documents.azure.com:443/`
* CosmosDB read-only key: `IFiZ8KOf7optUqyn82HyZkOmnYCSsbHBG9OqhAaBwykBNnfJZo78PL2KygLObU7WoAhm5YWFQvQMPApGdyhxQQ==`
* CosmosDB Database name: `hackathon`
* CosmosDB Connection string: `AccountEndpoint=https://hack-comp-brum.documents.azure.com:443/;AccountKey=IFiZ8KOf7optUqyn82HyZkOmnYCSsbHBG9OqhAaBwykBNnfJZo78PL2KygLObU7WoAhm5YWFQvQMPApGdyhxQQ==;`

Keep in mind that the key is a read-only key. You will be unable to modify the data in this CosmosDB account. If you need to store data you need to create your own database. It doesn't have to be a CosmosDB.

**Also note, that if you are copying connection strings from a PDF version of this file then there may be line breaks which will break the connection in Cosmos Data Explorer.**

# Package installation

In order to connect to CosmosDB you will need to install a package in your project.

The package is called [Cosmonaut](https://github.com/Elfocrash/Cosmonaut) and can be installed using one of the following three ways.

1. By browsing Nuget and searching for `Cosmonaut`
![](https://ibin.co/w800/4YTut5IRAwtE.png)

2. By running the Package Manager command : `Install-Package Cosmonaut`

3. By running the .NET CLI command: `dotnet add package Cosmonaut`


# Connecting to the Product store

First you need a connection policy. It's recommended to use Direct TCP for better performance. Here's how you can do that.
```csharp
var connectionPolicy = new ConnectionPolicy
{
    ConnectionProtocol = Protocol.Tcp,
    ConnectionMode = ConnectionMode.Direct
};
```

Then you need a `CosmosStoreSettings` object that points to the database where the products are stored. You need the database name, cosmosdb url and key. The key you will be using is a Read-Only key which means that you will only be able to do querying.

```csharp
var cosmosSettings = new CosmosStoreSettings("hackathon",
    "https://hack-comp-brum.documents.azure.com:443/",
    "IFiZ8KOf7optUqyn82HyZkOmnYCSsbHBG9OqhAaBwykBNnfJZo78PL2KygLObU7WoAhm5YWFQvQMPApGdyhxQQ==", connectionPolicy);
```

Finally you can initialise your CosmosStore by simply specifying the settings object created earlier.

```csharp
var productStore = new CosmosStore<Product>(cosmosSettings);
```

**Using a web app?**

Cosmonaut offers a package that adds DI support.

You can find it in nuget using the ways described above but changing the package name to: `Cosmonaut.Extensions.Microsoft.DependencyInjection`.

It adds the `.AddCosmosStore<TEntity>()` extension for the `IServiceCollection` interface which is used in the `Startup.cs` `ConfigureServices` method.

You can registed the `Product` `CosmosStore` in the DI container by doing:
```csharp
serviceCollection.AddCosmosStore<Product>(cosmosSettings);
```

You can then add `ICosmosStore<Product>` as a parameter in your constructors to inject the `CosmosStore` into your class.

Example:
```csharp
private readonly ICosmosStore<Product> _productStore;

public ProductController(ICosmosStore<Product> productStore)
{
    _productStore = productStore;
}
```

**Product class**

The `Product` object is a POCO/DTO that describes the products stored in the CosmosDB collection.

**You can download the Product.cs file [here](https://gist.github.com/Elfocrash/90f475e5df70a51f33dad5c85a30b71a)**

# Querying products

Cosmonaut offers multiple ways to query and read documents.

The main querying way is to use the `.Query()` method. It is an entry point method that allows you to create complex queries using LINQ.

Once you are done with your expression simply use `.ToListAsync()` (make sure it's coming from Cosmonaut not EntityFramework) to get back the data that matches the expression.

Let's see how we can get all the Sweatshirts.

```csharp
var sweatShirts = await productStore.Query().Where(x => x.Category == "Sweatshirts").ToListAsync();
```
or the ones that also have the word "Jacket" in the name
```csharp
var jacketSweatShirts = await productStore.Query().Where(x => x.Category == "Sweatshirts" && x.Name.Contains("Jacket")).ToListAsync();
```

You can also read a product directly using it's `id`:

Example:
```csharp
var aProduct = await productStore.FindAsync("10001230");
```

You can create more complicated queries but there are some things that the CosmosDB SDK doesn't support such as:

* `Equals()`
* `Skip()`
* More, which I don't remember but you will get errors when you try them.

**Querying scope**

Remember that you are working with pretty big documents and a really big collection.
It is highly recommended that you are not doing queries with really wide scopes because there is a chance that you won't get a fast response back.

This means that queries such as the ones below are not recommended.
```csharp
var allProducts = await productStore.Query().ToListAsync();
```

Cosmonaut supports pagination. Using wide scoped queries with pagination is fast.

Here's an example of how you can get the first page of products
```csharp
var firstPageOfProducts = await productStore.Query().WithPagination(1, 100).ToListAsync();
```

`ToListAsync()` on a paged query will just return the results. `ToPagedListAsync()` on the other hand will return a `CosmosPagedResults` object. This object contains the results but also a boolean indicating whether there are more pages after the one you just got but also the continuation token you need to use to get the next page.

**Pagination recommendations**

Because page number + page size pagination goes though all the documents until it gets to the requested page, it's potentially slow and expensive.
The recommended approach would be to use the page number + page size approach once for the first page and get the results using the `.ToPagedListAsync()` method. This method will return the next continuation token and it will also tell you if there are more pages for this query. Then use the continuation token alternative of `WithPagination` to continue from your last query.

Let's see how getting 3 pages of results would look like with `ToPagedListAsync`.

```csharp
var firstPageOfProducts = await productStore.Query().WithPagination(1, 100).ToPagedListAsync();
var secondPageOfProducts = await productStore.Query().WithPagination(firstPageOfProducts.NextPageToken, 100).ToPagedListAsync();
var thirdPageOfProducts = await productStore.Query().WithPagination(secondPageOfProducts.NextPageToken, 100).ToPagedListAsync();
```

Realistically, you would have to get the `NextPageToken`, store it on the client side of your app and then send it with the next page request.

(If you add ordering to the mix your query will be really slow again, so narrowing the scope is highly recommended)


> **If you have any further questions about Cosmonaut, such as adding objects, updating or removing, you can use the official cheat sheet which can be found here: [https://github.com/Elfocrash/Cosmonaut/blob/develop/README.md](https://github.com/Elfocrash/Cosmonaut/blob/develop/README.md)**

# Querying data using SQL

You can query the Product data before you integrate it in any application by using the CosmosDB Data Explorer.

CosmosDB Data Explorer link: https://cosmos.azure.com/
You will need to use the CosmosDB connection string: `AccountEndpoint=https://hack-comp-brum.documents.azure.com:443/;AccountKey=IFiZ8KOf7optUqyn82HyZkOmnYCSsbHBG9OqhAaBwykBNnfJZo78PL2KygLObU7WoAhm5YWFQvQMPApGdyhxQQ==;`

**Note, that if you are copying connection strings from a PDF version of this file then there may be line breaks which will break the connection in Cosmos Data Explorer.**

Here is an example of a CosmosDB SQL query:

![](https://ibin.co/w800/4YZW6WDWhPqf.png)

CosmosDB uses a modified version of TSQL. You can read more about it here: https://docs.microsoft.com/en-us/azure/cosmos-db/sql-api-sql-query-reference

# Other programming languages

You can also query CosmosDB data using a different programming language. Please find links to SDKs for different languages below.

* Java SDK: https://docs.microsoft.com/en-us/azure/cosmos-db/sql-api-sdk-java
* Node.js SDK: https://docs.microsoft.com/en-us/azure/cosmos-db/sql-api-sdk-node
* Python SDK: https://docs.microsoft.com/en-us/azure/cosmos-db/sql-api-sdk-python
* Using the REST API: https://docs.microsoft.com/en-us/rest/api/cosmos-db/

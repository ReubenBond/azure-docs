---
title: Hierarchical partition keys
titleSuffix: Azure Cosmos DB
description: Learn about subpartitioning in Azure Cosmos DB, how to use the feature, and how to manage logical partitions.
author: seesharprun
ms.author: sidandrews
ms.reviewer: dech
ms.service: cosmos-db
ms.topic: conceptual
ms.date: 05/05/2023
ms.custom: event-tier1-build-2022, ignite-2022, devx-track-arm-template, build-2023
---

# Hierarchical partition keys in Azure Cosmos DB

[!INCLUDE[NoSQL](includes/appliesto-nosql.md)]

Azure Cosmos DB distributes your data across logical and physical partitions based on your partition key to enable horizontal scaling. With hierarchical partition keys, or subpartitoning, you can now configure up to a three level hierarchy for your partition keys to further optimize data distribution and enable higher scale.

If you use synthetic keys today or have scenarios where partition keys can exceed 20 GB of data, subpartitioning can help. With this feature, logical partition key prefixes can exceed 20 GB and 10,000 RU/s, and queries by prefix are efficiently routed to the subset of partitions with the data.

## Example use case

Suppose you have a multi-tenant scenario where you store event information for users in each tenant. This event information could include event occurrences including, but not limited to, as sign-in, clickstream, or payment events.

In a real world scenario, some tenants can grow large with thousands of users, while the many other tenants are smaller with a few users. Partitioning by **/TenantId** may lead to exceeding Azure Cosmos DB's 20-GB storage limit on a single logical partition, while partitioning by **/UserId** makes all queries on a tenant cross-partition. Both approaches have significant downsides.

Using a synthetic partition key that combines **TenantId** and **UserId** adds complexity to the application. Additionally, the synthetic partition key queries for a tenant are still cross-partition, unless all users are known and specified in advance.

With hierarchical partition keys, we can partition first on **TenantId**, and then **UserId**. If you expect the **TenantId** and **UserId** combination to produce partitions that exceed 20 GB, you can even partition further down to another level, such as **SessionId**. The overall depth can't exceed three levels. When a physical partition exceeds 50 GB of storage, Azure Cosmos DB automatically splits the physical partition so that roughly half of the data is on one physical partition, and half on the other. Effectively, subpartitioning means that a single TenantId can exceed 20 GB of data, and it's possible for a TenantId's data to span multiple physical partitions.

Queries that specify either the **TenantId**, or both **TenantId** and **UserId** is efficiently routed to only the subset of physical partitions that contain the relevant data. Specifying the full or prefix subpartitioned partition key path effectively avoids a full fan-out query. For example, if the container had 1000 physical partitions, but a particular **TenantId** was only on five of them, the query would only be routed to the smaller number of relevant physical partitions.

## Guidance on choosing hierarchical partition keys

Hierarchical partition keys are highly recommended to users that may have multi-tenant applications. This is because hierarchical partitions allows users to scale beyond the logical partition key limit of 20 GB. If your current partition key or a single partition key is frequently hitting 20 GB, hierarchical partitions are a great choice for your workload. When choosing your hierarchical partition keys, it is important to keep the following general partitioning concepts in mind depending on your workload:
For all containers, the full path of your hierarchical partition key should:

-	Have a high cardinality. In other words, the full path of the hierarchical partition should have a wide range of possible values.
-	Spread request unit (RU) consumption and data storage evenly across all logical partitions. This spread ensures even RU consumption and storage distribution across your physical partitions.

For large read-heavy workloads, we recommend choosing hierarchical partition keys that appear frequently in your queries. For example, a workload that frequently runs queries to filter out specific user sessions in a multi-tenant application can benefit from hierarchical partition keys of TenantId, UserId, and SessionId in that order. Queries can be efficiently routed to only the relevant physical partitions by including the partition key in the filter predicate. For more guidance on partition keys for read-heavy workloads, visit the [relevant section in our partition documentation.](partitioning-overview.md).

## Using item ID in hierarchy 

If your container has a property that has a large range of possible values, it is likely a great partition key choice for the last level of your hierarchy. One possible example of this is the **item ID**. The system property item ID exists in every item in your container. If you are unsure whether you need to add a second or third hierarchical partition key, adding item ID as another level will guarantee that you can scale beyond the logical partition key limit of 20 GB for the first or first and second level of keys. 

For example, suppose we have a container for a multi-tenant workload partitioned by TenantId and UserId. If it is possible for a single combination of TenantId and UserId to exceed 20 GB, then partitioning with three levels of keys, where the 3rd level key has high cardinality - e.g. a GUID is recommended. It is unlikely that the combination of TenantId, UserId, and a GUID will exceed 20 GB, so the combination of TenantId and UserId can effectively scale beyond 20 GB.

For more information on choosing item ID as a partition key, visit our [partitioning documentation.](partitioning-overview.md).

## Getting started

> [!IMPORTANT]
> Working with containers that use hierarchical partition keys is supported only in following SDK versions. You must use a supported SDK to create new containers with hierarchical partition keys and to perform CRUD/query operations on the data.
> If you would like to use an SDK or connector that isn't currently supported, please file a request on our [community forum](https://feedback.azure.com/d365community/forum/3002b3be-0d25-ec11-b6e6-000d3a4f0858).

Find the latest preview version of each supported SDK:

| SDK | Supported versions | Package manager link |
| --- | --- | --- |
| **.NET SDK v3** | *>= 3.33.0* | <https://www.nuget.org/packages/Microsoft.Azure.Cosmos/3.33.0/> |
| **Java SDK v4** | *>= 4.42.0* | <https://github.com/Azure/azure-sdk-for-java/blob/main/sdk/cosmos/azure-cosmos/CHANGELOG.md#4420-2023-03-17/> |
| **JavaScript SDK v3** | *3.17.4-beta.1* | <https://www.npmjs.com/package/@azure/cosmos/v/3.17.4-beta.1/> |

## Create a container with hierarchical partition keys

To get started, create a new container using a predefined list of subpartitioning key paths up to three levels of depth.

### Using the portal

The simplest way to create a container and specify hierarchical partition keys is with the Azure portal.

1. Sign in to the [Azure portal](https://portal.azure.com).

1. Navigate to the existing Azure Cosmos DB for NoSQL account page.

1. From the Azure Cosmos DB for NoSQL account page, select the **Data Explorer** navigation menu option.

    :::image type="content" source="media/hierarchical-partition-keys/data-explorer-menu-option.png" lightbox="media/hierarchical-partition-keys/data-explorer-menu-option.png" alt-text="Screenshot of the page for a new Azure Cosmos DB for NoSQL account with the Data Explorer option highlighted.":::

1. In the **Data Explorer**, select the **New Container** option.

    :::image type="content" source="media/hierarchical-partition-keys/new-container-option.png" lightbox="media/hierarchical-partition-keys/new-container-option.png" alt-text="Screenshot of the New Container option within the Data Explorer.":::

1. In the **New Container** dialog, specify `/TenantId` for the partition key field. Enter any value for the remaining fields (database, container, throughput, etc.).

    > [!NOTE]
    > We are using `/TenantId` as an example here. You can specify any key for the first level when implementing hierarchical partition keys on your own containers.

1. Select **Add Hierarchical Partition Key** twice.

    :::image type="content" source="media/hierarchical-partition-keys/add-hierarchical-partition-key.png" lightbox="media/hierarchical-partition-keys/add-hierarchical-partition-key.png" alt-text="Screenshot of the button to add a new hierarchical partition key.":::

1. For the second and third tiers of subpartitioning, specify `/UserId` and `/SessionId` respectively.

    :::image type="content" source="media/hierarchical-partition-keys/hierarchical-partition-key-list.png" lightbox="media/hierarchical-partition-keys/hierarchical-partition-key-list.png" alt-text="Screenshot of a list of three hierarchical partition keys.":::

1. Select **OK** to create the container.

### Using an SDK

When creating a new container using the SDK, define a list of subpartitioning key paths up to three levels of depth. Use the list of subpartition keys when configuring the properties of the new container.

#### [.NET SDK v3](#tab/net-v3)

```csharp
// List of partition keys, in hierarchical order. You can have up to three levels of keys.
List<string> subpartitionKeyPaths = new List<string> { 
    "/TenantId",
    "/UserId",
    "/SessionId"
};

// Create container properties object
ContainerProperties containerProperties = new ContainerProperties(
    id: "<container-name>",
    partitionKeyPaths: subpartitionKeyPaths
);

// Create container - subpartitioned by TenantId -> UserId -> SessionId
Container container = await database.CreateContainerIfNotExistsAsync(containerProperties, throughput: 400);
```

#### [Java SDK v4](#tab/java-v4)

```java
// List of partition keys, in hierarchical order. You can have up to three levels of keys.
List<String> subpartitionKeyPaths = new ArrayList<String>();
subpartitionKeyPaths.add("/TenantId");
subpartitionKeyPaths.add("/UserId");
subpartitionKeyPaths.add("/SessionId");

//Create a partition key definition object with Kind("MultiHash") and Version V2
PartitionKeyDefinition subpartitionKeyDefinition = new PartitionKeyDefinition();
subpartitionKeyDefinition.setPaths(subpartitionKeyPaths);
subpartitionKeyDefinition.setKind(PartitionKind.MULTI_HASH);
subpartitionKeyDefinition.setVersion(PartitionKeyDefinitionVersion.V2);

// Create container properties object
CosmosContainerProperties containerProperties = new CosmosContainerProperties("<container-name>", subpartitionKeyDefinition);

// Create throughput properties object
ThroughputProperties throughputProperties = ThroughputProperties.createManualThroughput(400);

// Create container - subpartitioned by TenantId -> UserId -> SessionId
Mono<CosmosContainerResponse> container = database.createContainerIfNotExists(containerProperties, throughputProperties);
```

---

## Using Azure Resource Manager templates

The Azure Resource Manager template for a subpartitioned container is mostly identical to a standard container with the only key difference being the value of the ``properties/partitionKey`` path. For more information about creating an Azure Resource Manager template for an Azure Cosmos DB resource, see [the Azure Resource Manager template reference for Azure Cosmos DB](/azure/templates/microsoft.documentdb/databaseaccounts).

Configure the ``partitionKey`` object with the following values to create a subpartitioned container.

| Path | Value |
| --- | --- |
| **paths** | List of hierarchical partition keys (max three levels of depth) |
| **kind** | ``MultiHash`` |
| **version** | ``2`` |

### Example partition key definition

For example, assume we have a hierarchical partition key composed of **TenantId -> UserId -> SessionId**. The ``partitionKey`` object would be configured to include all three values in the **paths** property, a **kind** value of ``MultiHash``, and a **version** value of ``2``.

#### [Bicep](#tab/bicep)

```bicep
partitionKey: {
  paths: [
    '/TenantId',
    '/UserId',
    '/SessionId'
  ]
  kind: 'MultiHash'
  version: 2
}
```

#### [JSON](#tab/arm-json)

```json
"partitionKey": {
    "paths": [
        "/TenantId",
        "/UserId",
        "/SessionId"
    ],
    "kind": "MultiHash",
    "version": 2
}
```

---

For more information about the ``partitionKey`` object, see [ContainerPartitionKey specification](/azure/templates/microsoft.documentdb/databaseaccounts/sqldatabases/containers#containerpartitionkey).

## Using the Azure Cosmos DB emulator

You can test the subpartitioning feature using the latest version of the local emulator for Azure Cosmos DB. To enable subparitioning on the emulator, start the emulator from the installation directory with the ``/EnablePreview`` flag.

```powershell
.\CosmosDB.Emulator.exe /EnablePreview
```

For more information, see [Azure Cosmos DB emulator](./local-emulator.md).

## Use the SDKs to work with containers with hierarchical partition keys

Once you have a container with hierarchical partition keys, use the previously specified versions of the .NET or Java SDKs to perform operations and execute queries on that container.

### Add an item to a container

There are two options to add a new item to a container with hierarchical partition keys enabled.

#### Automatic extraction

If you pass in an object with the partition key value set, the SDK can automatically extract the full partition key path.

##### [.NET SDK v3](#tab/net-v3)

```csharp
// Create new item
UserSession item = new UserSession()
{
    id = "f7da01b0-090b-41d2-8416-dacae09fbb4a",
    TenantId = "Microsoft",
    UserId = "8411f20f-be3e-416a-a3e7-dcd5a3c1f28b",
    SessionId = "0000-11-0000-1111"
};

// Pass in the object and the SDK will automatically extract the full partition key path
ItemResponse<UserSession> createResponse = await container.CreateItemAsync(item);
```

##### [Java SDK v4](#tab/java-v4)

```java
// Create new item
UserSession item = new UserSession();
item.setId("f7da01b0-090b-41d2-8416-dacae09fbb4a");
item.setTenantId("Microsoft");
item.setUserId("8411f20f-be3e-416a-a3e7-dcd5a3c1f28b");
item.setSessionId("0000-11-0000-1111");
   
// Pass in the object and the SDK will automatically extract the full partition key path
Mono<CosmosItemResponse<UserSession>> createResponse = container.createItem(item);
```

---

#### Manually specify path

The ``PartitionKeyBuilder`` class in the SDK can construct a value for a previously defined hierarchical partition key path. Use this class when adding a new item to a container that has subpartitioning enabled.

> [!TIP]
> At scale, it is often more performant to specify the full partition key path even if the SDK can extract the path from the object.

##### [.NET SDK v3](#tab/net-v3)

```csharp
// Create new item object
PaymentEvent item = new PaymentEvent()
{
    id = Guid.NewGuid().ToString(),
    TenantId = "Microsoft",
    UserId = "8411f20f-be3e-416a-a3e7-dcd5a3c1f28b",
    SessionId = "0000-11-0000-1111"
};

// Specify the full partition key path when creating the item
PartitionKey partitionKey = new PartitionKeyBuilder()
            .Add(item.TenantId)
            .Add(item.UserId)
            .Build();

// Create the item in the container
ItemResponse<PaymentEvent> createResponse = await container.CreateItemAsync(item, partitionKey);
```

##### [Java SDK v4](#tab/java-v4)

```java
// Create new item object
UserSession item = new UserSession();
item.setTenantId("Microsoft");
item.setUserId("8411f20f-be3e-416a-a3e7-dcd5a3c1f28b");
item.setSessionId("0000-11-0000-1111");
item.setId(UUID.randomUUID().toString());

// Specify the full partition key path when creating the item
PartitionKey partitionKey = new PartitionKeyBuilder()
            .add(item.getTenantId())
            .add(item.getUserId())
            .add(item.getSessionId())
            .build();
       
// Create the item in the container     
Mono<CosmosItemResponse<UserSession>> createResponse = container.createItem(item, partitionKey);
```

---

### Perform a key/value lookup (point read) of an item

Key/value lookups (point reads) are performed in a manner similar to a non-subpartitioned container. For example, assume we have a hierarchical partition key composed of **TenantId -> UserId -> SessionId**. The unique identifier for the item is a Guid, represented as a string that serves as a unique document transaction identifier. To perform a point read on a single item, pass in the ``id`` property of the item and the full value for the partition key including all three components of the path.

#### [.NET SDK v3](#tab/net-v3)

```csharp
// Store the unique identifier
string id = "f7da01b0-090b-41d2-8416-dacae09fbb4a";

// Build the full partition key path
PartitionKey partitionKey = new PartitionKeyBuilder()
    .Add("Microsoft") //TenantId
    .Add("8411f20f-be3e-416a-a3e7-dcd5a3c1f28b") //UserId
    .Add("0000-11-0000-1111") //SessionId
    .Build();

// Perform a point read
ItemResponse<UserSession> readResponse = await container.ReadItemAsync<UserSession>(
    id,
    partitionKey
);
```

#### [Java SDK v4](#tab/java-v4)

```java
// Store the unique identifier
String id = "f7da01b0-090b-41d2-8416-dacae09fbb4a"; 

// Build the full partition key path
PartitionKey partitionKey = new PartitionKeyBuilder()
    .add("Microsoft") //TenantId
    .add("8411f20f-be3e-416a-a3e7-dcd5a3c1f28b") //UserId
    .add("0000-11-0000-1111") //SessionId
    .build();
    
// Perform a point read
Mono<CosmosItemResponse<UserSession>> readResponse = container.readItem(id, partitionKey, UserSession.class);
```

---

### Run a query

The SDK code to run a query on a subpartitioned container is identical to running a query on a non-subpartitioned container.

When the query specifies all values of the partition keys in the ``WHERE`` filter or a prefix of the key hierarchy, the SDK automatically routes the query to the corresponding physical partitions. Queries that provide only the "middle" of the hierarchy are cross partition queries.

For example, assume we have a hierarchical partition key composed of **TenantId -> UserId -> SessionId**. The components of the query's filter determines if the query is a single-partition, targeted cross-partition, or fan out query.

| Query | Routing |
| --- | --- |
| ``SELECT * FROM c WHERE c.TenantId = 'Microsoft' AND c.UserId = '8411f20f-be3e-416a-a3e7-dcd5a3c1f28b' AND c.SessionId = '0000-11-0000-1111'`` | Routed to the **single logical and physical partition** that contains the data for the specified values of ``TenantId``, ``UserId`` and ``SessionId``. |
| ``SELECT * FROM c WHERE c.TenantId = 'Microsoft' AND c.UserId = '8411f20f-be3e-416a-a3e7-dcd5a3c1f28b'`` | Routed to only the **targeted subset of logical and physical partition(s)** that contain data for the specified values of ``TenantId`` and ``UserId``. This query is a targeted cross-partition query that returns data for a specific user in the tenant. |
| ``SELECT * FROM c WHERE c.TenantId = 'Microsoft'`` | Routed to only the **targeted subset of logical and physical partition(s)** that contain data for the specified value of ``TenantId``. This query is a targeted cross-partition query that returns data for all users in a tenant. |
| ``SELECT * FROM c WHERE c.UserId = '8411f20f-be3e-416a-a3e7-dcd5a3c1f28b'`` | Routed to **all physical partitions**, resulting in a fan-out cross-partition query. |
| ``SELECT * FROM c WHERE c.SessionId = '0000-11-0000-1111'`` | Routed to **all physical partitions**, resulting in a fan-out cross-partition query. |

#### Single-partition query on a subpartitioned container

Here's an example of running a query that includes all of the levels of subpartitioning effectively making the query a single-partition query.

##### [.NET SDK v3](#tab/net-v3)

```csharp
// Define a single-partition query that specifies the full partition key path
QueryDefinition query = new QueryDefinition(
    "SELECT * FROM c WHERE c.TenantId = @tenant-id AND c.UserId = @user-id AND c.SessionId = @session-id")
    .WithParameter("@tenant-id", "Microsoft")
    .WithParameter("@user-id", "8411f20f-be3e-416a-a3e7-dcd5a3c1f28b")
    .WithParameter("@session-id", "0000-11-0000-1111");

// Retrieve an iterator for the result set
using FeedIterator<PaymentEvent> results = container.GetItemQueryIterator<PaymentEvent>(query);

while (results.HasMoreResults)
{
    FeedResponse<UserSession> resultsPage = await resultSet.ReadNextAsync();
    foreach(UserSession result in resultsPage)
    {
        // Process result
    }
}
```

##### [Java SDK v4](#tab/java-v4)

```java
// Define a single-partition query that specifies the full partition key path
String query = String.format(
    "SELECT * FROM c WHERE c.TenantId = '%s' AND c.UserId = '%s' AND c.SessionId = '%s'",
    "Microsoft",
    "8411f20f-be3e-416a-a3e7-dcd5a3c1f28b",
    "0000-11-0000-1111"
);

// Retrieve an iterator for the result set
CosmosPagedFlux<UserSession> pagedResponse = container.queryItems(
    query, options, UserSession.class);

pagedResponse.byPage().flatMap(fluxResponse -> {
    for (UserSession result : page.getResults()) {
        // Process result
    }
    return Flux.empty();
}).blockLast();
```

---

#### Targeted multi-partition query on a subpartitioned container

Here's an example of a query including a subset of the levels of subpartitioning effectively making this query a targeted multi-partition query.

##### [.NET SDK v3](#tab/net-v3)

```csharp
// Define a targeted cross-partition query specifying prefix path[s]
QueryDefinition query = new QueryDefinition(
    "SELECT * FROM c WHERE c.TenantId = @tenant-id")
    .WithParameter("@tenant-id", "Microsoft")

// Retrieve an iterator for the result set
using FeedIterator<PaymentEvent> results = container.GetItemQueryIterator<PaymentEvent>(query);

while (results.HasMoreResults)
{
    FeedResponse<UserSession> resultsPage = await resultSet.ReadNextAsync();
    foreach(UserSession result in resultsPage)
    {
        // Process result
    }
}
```

##### [Java SDK v4](#tab/java-v4)

```java
// Define a targeted cross-partition query specifying prefix path[s]
String query = String.format(
    "SELECT * FROM c WHERE c.TenantId = '%s'",
    "Microsoft"
);

// Retrieve an iterator for the result set
CosmosPagedFlux<UserSession> pagedResponse = container.queryItems(
    query, options, UserSession.class);

pagedResponse.byPage().flatMap(fluxResponse -> {
    for (UserSession result : page.getResults()) {
        // Process result
    }
    return Flux.empty();
}).blockLast();
```

---

## Limitations and known issues

* Working with containers that use hierarchical partition keys is supported only in .NET v3, Java v4 SDKs, and the preview version of the JavaScript SDK. You must use a supported SDK to create new containers with hierarchical partition keys and to perform CRUD/query operations on the data. Support for other SDKs, like Python, are not yet available.
* There are limitations with various Azure Cosmos DB connectors (ex. Azure Data Factory).
* You can only specify hierarchical partition keys up to three layers in depth.
* Hierarchical partition keys can currently only be enabled on new containers. The desired partition key paths must be specified at the time of container creation and can't be changed later. To use hierarchical partitions on existing containers, you should create a new container with the hierarchical partition keys set and move the data using [container copy jobs](intra-account-container-copy.md).
* Hierarchical partition keys are currently supported only for API for NoSQL accounts (API for MongoDB and Cassandra aren't currently supported).

## Next steps

* See the FAQ on [hierarchical partition keys.](hierarchical-partition-keys-faq.yml)
* Learn more about [partitioning in Azure Cosmos DB.](partitioning-overview.md)
* Learn more about [using Azure Resource Manager templates with Azure Cosmos DB.](/azure/templates/microsoft.documentdb/databaseaccounts)

# Proximity Service

A proximity service enables you to discover nearby places such as restaurants, hotels, theatres, etc.

# Step 1 - Understand the problem and establish design scope
Sample questions to understand the problem better:
 * C: Can a user specify a search radius? What if there are not enough businesses within the search area?
 * I: We only care about businesses within a certain area. If time permits, we can discuss enhancing the functionality.
 * C: What's the max radius allowed? Can I assume it's 20km?
 * I: Yes, that is a reasonable assumption
 * C: Can a user change the search radius via the UI?
 * I: Yes, let's say we have the options - 0.5km, 1km, 2km, 5km, 20km
 * C: How is business information modified? Do we need to reflect changes in real-time?
 * I: Business owners can add/delete/update a business. Assume changes are going to be propagated on the next day.
 * C: How do we handle search results while the user is moving?
 * I: Let's assume we don't need to constantly update the page since users are moving slowly.

## Functional requirements
 * Return all businesses based on user's location
 * Business owners can add/delete/update a business. Information is not reflected in real-time.
 * Customers can view detailed information about a business

## Non-functional requirements
 * Low latency - users should be able to see nearby businesses quickly
 * Data privacy - Location info is sensitive data and we should take this into consideration in order to comply with regulations
 * High availability and scalability requirements - We should ensure system can handle spike in traffic during peak hours in densely populated areas

## Back-of-the-envelope calculation
 * Assuming 100mil daily active users and 200mil businesses
 * Search QPS == 100mil * 5 (average searches per day) / 10^5 (seconds in day) == 5000

# Step 2 - Propose High-Level Design and get Buy-In
## API Design
We'll use a RESTful API convention to design a simplified version of the APIs.
```
GET /v1/search/nearby
```

This endpoint returns businesses based on search criteria, paginated.

Request parameters - latitude, longitude, radius

Example response:
```
{
  "total": 10,
  "businesses":[{business object}]
}
```

The endpoint returns everything required to render a search results page, but a user might require additional details about a particular business, fetched via other endpoints.

Here's some other business APIs we'll need:
 * `GET /v1/businesses/{:id}` - return business detailed info
 * `POST /v1/businesses` - create a new business
 * `PUT /v1/businesses/{:id}` - update business details
 * `DELETE /v1/businesses/{:id}` - delete a business

## Data model
In this problem, the read volume is high because these features are commonly used:
 * Search for nearby businesses
 * View the detailed information of a business

On the other hand, write volume is low because we rarely change business information. Hence for a read-heavy workflow, a relational database such as MySQL is ideal.

In terms of schema, we'll need one main `business` table which holds information about a business:
![business-table](images/business-table.png)

We'll also need a geo-index table so that we efficiently process spatial operations. This table will be discussed later when we introduce the concept of geohashes.

## High-level design
Here's a high-level overview of the system:
![high-level-design](images/high-level-deisgn.png)
 * The load balancer automatically distributes incoming traffic across multiple services. A company typically provides a single DNS entry point and internally routes API calls to appropriate services based on URL paths.
 * Location-based service (LBS) - read-heavy, stateless service so its easy to scale horizontally, responsible for serving read requests for nearby businesses, QPS is high especially during peak hours in dense areas.
 * Business service - supports CRUD operations on businesses.
   - write operations QPS low (create update delete businesses), read operations QPS high during peak hours (view detailed info about businesses).
 * Database cluster - stores business information and replicates it in order to scale reads. This leads to some inconsistency for LBS to read business information, which is not an issue for our use-case.Uses primary secondary setup where primary handles all the writes and replicates to the multiple replicas which are used for reading. Data is first saved to primary and then replicated to replicas - has a small delay which is ok because business info does not need to be updated real time.
 * Scalability of business service and LBS - since both services are stateless, we can easily scale them horizontally for peak hour traffic (meal time) and remove for off peak hours (sleep time). If system is on cloud, we can setup different regions and availability zones to further improve availability.

## Algorithms to fetch nearby businesses
In real life, one might use a geospatial database, such as Geohash in Redis or Postgres with PostGIS extension.

Let's explore how these databases work and what other alternative algorithms there are for this type of problem.

### Option1: Two-dimensional search
The most intuitive and naive approach to solving this problem is to draw a circle around the person and fetch all businesses within the circle's radius:
![2d-search](images/2d-search.png)

This can easily be translated to a SQL query:
```
SELECT business_id, latitude, longitude,
FROM business
WHERE (latitude BETWEEN {:my_lat} - radius AND {:my_lat} + radius) AND
      (longitude BETWEEN {:my_long} - radius AND {:my_long} + radius)
```

This query is not efficient because we need to query the whole table. An alternative is to build an index on the longitude and latitude columns but that won't improve performance by much.

This is because we still need to subsequently filter a lot of data regardless of whether we index by long or lat:
![2d-query-problem](images/2d-query-problem.png)

We can, however, build 2D indexes and there are different approaches to that:
* Hash: even grid, geohash, cartesian tiers, etc
* Tree: quadtree, Google S2, RTree, etc
Even though implementations are different, they all divide the map into smaller areas and build indexes for fast search. Geohash, quadtree and Google S2 are most widely used in real world applications.
![2d-index-options](images/2d-index-options.png)

We'll discuss the ones highlighted in purple - geohash, quadtree and google S2 are the most popular approaches.

### Option 2: Evenly divided grid
Another option is to divide the world in small grids (one grid can have one or more businesses and every business is in one grid only).
![evenly-divided-grid](images/evenly-divided-grid.png)

The major flaw with this approach is that business distribution is uneven as there are a lot of businesses concentrated in new york and close to zero in the sahara desert. Ideally we want to use more granular grids for dense areas and large grids in sparse areas. Another potential challenge is to find neighboring grids of a fixed grid.

### Option 3: Geohash
Better than option2, it works by reducing the two dimensional longitude and latitude data into a one dimensional string of letters and digits. Geohash works similarly to the previous approach, but it recursively divides the world into smaller and smaller grids with each additional bit. Start by splitting the world into four quadrants where each two bits correspond to a single quadrant. Now, divide each grid into four smaller grids. Each grid can be represented by alternating between longitude bit and latitude bit. Repeat till the grid size is within the precision desired.
![geohash-example](images/geohash-example.png)

Geohashes are typically represented in base32. Here's the example geohash of google headquarters:
```
1001 10110 01001 10000 11011 11010 (base32 in binary) → 9q9hvu (base32)
```

It supports 12 levels of precision and precision factor determines the size of the grid. We only need up to 6 levels for our use-case since geohashes longer than 6 have grids of size too small and less than 4 has grids of size too large. 4 to 6 is ideal.
![geohash-precision](images/geohash-precision.png)

To choose the right precision level, we want to find the minimal geohash length that covers the whole circle drawn by the user defined radius. Relationship between radius and length of geohash is shown:
Radius(Kilometers) Geohash length
0.5 km(0.31 miles)       6
1 km(0.62 miles)         5
2 km(1.24 miles)         5
5 km(3.1 miles)          4
20 km(12.42 miles)       4

This approach works great mostly but has some edge cases that need to be discussed.

Geohashes enable us to quickly locate neighboring regions based on a substring of the geohash (the longer a shared prefix is between two geohashes, the closer they are):
![geohash-substring](images/geohash-substrint.png)

However, one issue \w geohashes is that there can be places which are very close to each other which don't share any prefix, because they're on different sides of the equator or meridian:
![boundary-issue-geohash](images/boundary-issue-geohash.png)
So a simple prefix SQL query will fail to fetch all nearby businesses:
```
SELECT * FROM geohash_index WHERE geohash LIKE '9q8zn%'
```

Another issue is that two businesses can be very close but not share a common prefix because they're in different quadrants:
![geohash-boundary-issue-2](images/geohash-boundary-issue-2.png)

This can be mitigated by fetching neighboring geohashes as well, not just the geohash of the user.

A benefit of using geohashes is that we can use them to easily implement the bonus problem of increasing search radius in case insufficient businesses are fetched via query:
![geohash-expansion](images/geohash-expansion.png)

This can be done by removing the last digit of the target geohash to increase radius. If there are not enoug businesses, we continue to expand the scope by removing another digit. This way, the grid size is gradually expanded until the result is greater than the desired number of results. 

### Option 4:Quadtree
A quadtree is a data structure, which partitions a 2D space by recursively subdividing into four quadrants until the contents of the grids meet certain criteria. For example, the criterion can be to keep subdividing until the number of busiensses in the gride is not more than 100. This number is arbitrary as the actual number can be determined by business needs. With a quadtree we build an in-memory tree structure to answer queries:
![quadtree-example](images/quadtree-example.png)

This is an in-memory solution which can't easily be implemented in a database. It runs on each LBS server, and the data structure is biult at server startup time.

Here's how it might look conceptually:
![quadtree-concept](images/quadtree-concept.png)

Example pseudocode to build a quadtree:
```
public void buildQuadtree(TreeNode node) {
    if (countNumberOfBusinessesInCurrentGrid(node) > 100) {
        node.subdivide();
        for (TreeNode child : node.getChildren()) {
            buildQuadtree(child);
        }
    }
}
```

In a leaf node, we store:
 * Top-left, bottom-right coordinates to identify the quadrant dimensions - 32 bytes (8 bytes x 4)
 * List of business IDs in the grid - 8 bytes per ID x 100 (maximal number of businesses allowed in one grid)
   Total - 832 bytes

In an internal node we store:
 * Top-left, bottom-right coordinates of quadrant dimensions - 32 bytes (8 bytes x 4)
 * pointers to 4 children - 32 bytes (8 bytes x 4)
   Total - 64 bytes

The total memory to represent the quadtree is calculated as ~1.7GB in the book if we assume that we operate with 200mil businesses (number of leaf nodes = 200 mil/100 = about 2 mil, number of internal nodes = 2mil x 1/3 = about 0.67 mil, total memory requirement = 2mil x 832 bytes + 0.67 mil x 64 bytes = about 1.71 GB). Time taken to build it = (n/100) log (n/100).

Hence, a quadtree can be stored in a single server, in-memory, although we can of course replicate it for redundancy and load balancing purposes.

One consideration to take into consideration if this approach is adopted - startup time of server can be a couple of minutes while the quadtree is being built.Time taken to build it = (n/100) log (n/100) = a few minutes for 200 million businesses. This is not ideal for server startup since server cannot serve traffic when the quadtree is being built. So we should rollout new releases to small subset of servers at a time. This avoids taking a large swath of the server cluster offline and causes service brownout. Blue/green deployment can also be used, but an entire cluster of new servers fetching 200 million businesses at the same time from teh database service can put a lot of strain on the system. This can be done, but it may complicate the design and you should mention that in the interview. 

* build quadtree in memory
* after it is built, start searching from teh root and traverse the tree, until we find the leaf node where the search origin is. If that leaf node has 100 businesses, return the node. Otherwise, add businesses from its neighbors until enough businesses are returned. 

Hence, this should be taken into account during the deployment process. Eg a healthcheck endpoint can be exposed and queried to signal when the quadtree build is finished.

Another consideration is how to update the quadtree. Given our requirements, a good option would be to update it every night using a nightly job due to our commitment of reflecting changes at start of next day.

It is nevertheless possible to update the quadtree on the fly, but that would complicate the implementation significantly.

Example quadtree of Denver:
![denver-quadtree](images/denver-quadtree.png)

### Option 5:Google S2
Google S2 is a geometry library, which supports mapping 2D points on a 1D plane using Hilbert curves (Objects close to each other on the 2D plane are close on the hilbert curve as well and are close in the 1D space - search is more efficient in 1D space):
![hilbert-curve](images/hilbert-curbe.png)

This library is great for geofencing, which supports covering arbitrary areas vs. confining yourself to specific quadrants. A geofence is a virtual perimeter for a real world geographic area. A geo fence could be dyanmically generated - as in a radius around a point location, or a geo fence can be predefined set of boundaries (such as school zones or neighborhood boundaries). Geofencing allows u sot define perimeters that surround the areas of interest and to send notifications to users who are out of the areas. This can provide richer functionalities than just returning nearby businesses. 
![geofence-example](images/geofence-example.png)

This functionality can be used to support more advanced use-cases than nearby businesses.

Another benefit of Google S2 is its Region Cover algorithm, which enables us to define more granular precision levels (min level, max level and max cells - cell sizes are flexible), than those provided by geohashes.

### Recommendation
There is no perfect solution, different companies adopt different solutions:
![company-adoptions](images/company-adoptions.png)

Author suggest choosing geohashes or quadtree in an interview as those are easier to explain than Google S2.

Here's a quick summary of geohashes:
 * Easy to use and implement, no need to build a tree
 * supports returning businesses within a specified radius
 * Geohash precision is fixed. More complex logic is required if a more granular precision is needed
 * Updating the index is easy

Here's a quick summary of quadtrees:
 * Slightly harder to implement as it requires us to build a tree
 * Supports fetching k-nearest neighbors instead of businesses within radius, which can be a good use-case for certain features
 * Grid size can be dynamically adjusted based on population density
 * Updating the index is more complicated than updating the geohash variant. All problems with updating and balancing trees are present when working with quad trees. Rebalancing is necessary if, for eg, a leaf node has no room for a new additon. A possible fix is to over allocate the ranges.

# Step 3 - Design Deep Dive
Let's dive deeper into some areas of the design.

## Scale the database
The business table can be scaled by sharding it by business ID (ensures load is evenly distributed among all the shards and operationally it is easy to maintain) in case it doesn't fit in a single server instance.

Geospatial index table can use either geohash or quadtree. We use geohash because of its simplicity.
The geohash table can be represented by two columns:
![geohash-table-example](images/geohash-table-example.png)
We can either keep the business IDs as JSON or keep each business ID as a separate row. First option has a lot of edge cases and updating is a pain and needs locks, so second option is preferred and it doesn't need any locks when updating.

We don't need to shard the geohash table as we don't have that much data. We calculated that it takes ~1.7gb to build a quad tree and geohash space usage is similar.

We can, however, replicate the table to scale the read load.

Two general approaches for spreading the load of a RDB server - add read replicas or shard the database. Usually sharding is good if read and write are similar or if the size is huge but adding read replicas is better for geohash table since size is small and reads >>> writes.

## Caching
Before using caching, we should ask ourselves if it is really necessary. In our case, the workflow is read-heavy and data can fit into a single server, so this kind of data is ripe for caching.

We should be careful when choosing the cache key. Location coordinates are not a good cache key as they often change and can be inaccurate.

Using the geohash is a more suitable key candidate.

Here's how we could query all businesses in a geohash:
```
SELECT business_id FROM geohash_index WHERE geohash LIKE `{:geohash}%`
```

Here's example code to cache the data in redis:
```
public List<String> getNearbyBusinessIds(String geohash) {
    String cacheKey = hash(geohash);
    List<string> listOfBusinessIds = Redis.get(cacheKey);
    if (listOfBusinessIDs  == null) {
        listOfBusinessIds = Run the select SQL query above;
        Cache.set(cacheKey, listOfBusinessIds, "1d");
    }
    return listOfBusinessIds;
}
```

We can cache the data on all precisions we support, which are not a lot, ie `geohash_4, geohash_5, geohash_6`.

As we already discussed, the storage requirements are not high and can fit into a single redis server, but we could replicate it for redundancy purposes as well as to scale reads.

We can even deploy multiple redis replicas across different data centers.

We could also cache `business_id -> business_data` as users could often query the details of the same popular restaurant.

## Region and availability zones
We can deploy multiple LBS service instances across the globe so that users query the instance closest to them. This leads to reduced latency;
![cross-dc-deployment](images/cross-dc-deployment.png)

It also enables us to spread traffic evenly across the globe. This could also be required in order to comply with certain data privacy laws.

## Follow-up question - filter businesses by type or time
Once businesses are filtered, the result set is going to be small, hence, it is acceptable to filter the data in-memory - return all the business ids in the geohash, hydrate the business object and filter them based on opening time or business type. This assumes the business object contains all this info in the business table.

## Final design diagram
![final-design](images/final-design.png)
 * Client tries to locate restaurants within 500meters of their location
 * Load balancer forwards the request to the LBS (location based service or proximity service)
 * LBS maps the radius to geohash with length 6
 * LBS calculates neighboring geohashes and adds them to the list
 * For each geohash, LBS calls the redis server to fetch corresponding business IDs. This can be done in parallel.
 * Finally, LBS hydrates the business ids, filters the result and returns it to the user
 * Business-related APIs are separated from the LBS into the business service, which checks the cache first for any read requests before consulting the database
 * Business updates are handled via a nightly job, which updates the geohash store

# Step 4 - Wrap Up
Summary of some of the more interesting topics we covered:
 * Discussed several indexing options - 2d search, evenly divided grid, geohash, quadtree, google S2
 * Discussed caching, replication, sharding, cross-DC deployments in the deep dive section

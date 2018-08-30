# Orders API - technical exercise

Design a RESTful API using RAML that contains a single resource, orders, and allows the following:

* List orders **GET /orders**
* Create a new order **POST /orders**
* Update an order **PUT /orders/{orderId} (asynchronous)**
* Deletes an order **DELETE /orders/{orderId}**

You may constrain the order object to customer details, purchase order and line items, and the format to JSON.

The API must be designed to support the following consumer use cases at a minimum:

**A consumer may periodically (every 5 minutes) consume the API to enable it (the consumer) to maintain a copy of the provider API's orders (the API represents the system of record)**

**Challenge**

The API should allow consumers to keep a copy of the orders, this is a synchronisation/replication case on a large set of orders.

**Approach**

Use of headers ```Last-Modified``` and ```If-Modified-Since``` to avoid unnecessary transferring of resources if they have been changed since the last request. Pagination was also designed, allowing the consumers to retrieve orders filtering by offset, limit and page. Both approaches save bandwidth and improve performance.

**How it works?**

* Consumers should retrieve all orders the first time to have a baseline
* The API response will include the header ```Last-Modified``` indicating the time when the collection was last modified on the system of record

```
Last-Modified: Sat, 26 Oct 1985 01:20:00 AEST
```
* The consumer caches the collection of orders along with the header
* For subsequent requests the consumer should include the header  ```If-Modified-Since``` using the date time returned with the latest responses

```
If-Modified-Since: Sat, 26 Oct 1985 01:20:00 AEST
```
* If ```If-Modified-Since``` is same as currently modified date then the API returns status code 304 (Not modified) so the consumer cached copy has not changed. If the ```If-Modified-Since``` value is less than the currently modified date then the API returns status code 200 with the modified/new resources along with the new ```Last-Modified``` header so the consumer should update its cache with the new resources.

**Other alternatives**

* Use an event-driven approach instead of polling by defining events that can be consumed and reacted to. This approach eliminates the need of polling and waiting for synchronous delivery freeing resources and reducing overhead. This approach was not use because the use case was to allow polling every 5 min so the proposed solution made more sense.

* Instead of exposing an API for replication, trigger an asynchronous process on update/create to sync multiple backends directly.

**A mobile application used by customer service representatives that uses the API to retrieve and update the order details**

**Challenge**
The order update process has the potential to take long time and a mobile application should provide good user experience and avoid waiting for a response of a blocking call (synchronous).

**Approach**

Included the header ```Etag``` in the GET order response to identify the version of the order resource. It allows caches to be more efficient and improves performance since the API does not return the resource if it has not changed.

Designed the API so the mobile application can trigger asynchronous work for the order update and allow tracking the status. Once the result is ready the application is able to retrieve it or if the update is no longer needed the application is also able to cancel the process. This will also allows the API to be implemented with retries and alerts in case of failure.

**How it works?**

1. Caching of unchanged resources

* The application retrieves an order including the header ```If-None-Match``` if it has a previously returned Etag.
* If the resource has not been modified (it still has the same version), then **HTTP status code 304 Not Modified** (with empty body)
will be returned instead of the actual resource. If the resource has changed or it is a new resource, then **HTTP status code 200** will be returned with body along with the ```Etag``` header that should be use for subsequent calls.

2. Asynchronous API

* The application invokes the API using the method PUT
* The API returns HTTP status code 202 (Accepted) informing the application that its request has been accepted for processing. The response includes the header ```Location``` providing a URI to poll the status of the process
* The status URI returns **HTTP status code 200** if the process is still going and **HTTP status 303** along with the ```Location``` header providing the actual result of the update if the process has finished
* The status URI was also design to allow the cancellation of the process by invoking a DELETE on the status of the order.

**Other alternatives**

* PATCH vs PUT - Although PATCH allows sending a partial representation of the resource to be updated and this improves performance, both have the issue of blocking if they are implemented synchronously. I used PUT because the request is not really the issue here, the API can also be designed to allow both asynchronously.

**Simple extension of the API to support future resources such as customers and products**

The API was designed using the collection/collection-item patterns, and a library for resourceTypes was created. The schemas for the new resources can be easily added and the resourceTypes can be reused because they are parametrised. It would be as simple as inheriting from a resourceType.

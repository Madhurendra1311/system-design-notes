An API or application programmable interface is a software contract that defines the expectations and interactions of a piece of code exposed to external users. This includes the parameters, response, errors, and API name.

We discuss how to design an API and what it takes to make the design scalable, extensible, and easy to use. HTTP APIs are widely used in software systems. HTTP is a stateless protocol, and systems often expose APIs using it.

API Design — Interview Notes and Best Practices
An API (Application Programming Interface) is a contract between two software systems that defines how they communicate, including the endpoints, parameters, response formats, and error handling.
For example, in a homestay booking application, the frontend uses APIs to search properties, view property details, create bookings, and cancel reservations.
1. What are the best practices for designing an API?
1. Use clear, consistent naming conventions.
2. Use HTTP methods correctly (GET, POST, PUT, PATCH, DELETE).
3. Validate request parameters and request bodies.
4. Return consistent response objects and meaningful HTTP status codes.
5. Implement authentication and authorization.
6. Support pagination, filtering, and sorting for large datasets.
7. Version APIs to support backward compatibility.
8. Implement rate limiting, logging, and monitoring.
9. Document APIs using OpenAPI/Swagger.
10. Avoid exposing sensitive information in responses or error messages.
2. API naming conventions
Use resource-based URLs with nouns, rather than action-based URLs.
Bad design	Better design
GET /getUsers	GET /users
GET /getUserById/10	GET /users/10
POST /createUser	POST /users
POST /deleteUser/10	DELETE /users/10
GET /getBookingDetails	GET /bookings/10
Use plural nouns and consistent naming throughout the API.
3. How do you define API parameters?
There are three common ways to pass parameters.
- Path parameters: Identify a particular resource.
- Query parameters: Filter, sort, or paginate results.
- Request body: Send data required to create or update a resource.
Example:
GET /properties/101
GET /properties?city=Patna&page=1&limit=10
POST /bookings


Example request body:
{
  "propertyId": 101,
  "checkIn": "2026-11-01",
  "checkOut": "2026-11-05",
  "guests": 2
}


Always validate inputs on the backend, even if the frontend already validates them.
4. How should you design API response objects?
Keep response structures predictable and consistent.
Successful response:
{
  "success": true,
  "data": {
    "id": 101,
    "name": "Mountain View Homestay"
  }
}


Paginated response:
{
  "success": true,
  "data": [
    {
      "id": 101,
      "name": "Mountain View Homestay"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "totalPages": 10
  }
}


A consistent response structure makes frontend integration and error handling easier.
5. How do you handle API errors?
Use meaningful HTTP status codes and a consistent error format.
Status code	Meaning
200 OK	Request successful
201 Created	Resource created
204 No Content	Successful request with no response body
400 Bad Request	Invalid request
401 Unauthorized	Authentication required or invalid
403 Forbidden	User lacks permission
404 Not Found	Resource not found
409 Conflict	Conflict with the current resource state
429 Too Many Requests	Rate limit exceeded
500 Internal Server Error	Unexpected server error
Example:
{
  "success": false,
  "error": {
    "code": "PROPERTY_NOT_FOUND",
    "message": "The requested property was not found."
  }
}


Never expose database credentials, stack traces, or internal implementation details in production error responses.
6. How do you choose HTTP endpoints and methods?
Method	Purpose	Example
GET	Retrieve data	GET /properties/101
POST	Create a resource or perform an operation	POST /bookings
PUT	Replace a resource	PUT /properties/101
PATCH	Partially update a resource	PATCH /properties/101
DELETE	Delete a resource	DELETE /bookings/501
GET vs POST
GET
- Retrieves data.
- Parameters commonly appear in the URL.
- Should not change application state.
- Is safe and idempotent.
- Responses may be cached when appropriate.
POST
- Commonly creates resources or triggers operations.
- Sends data in the request body.
- May change application state.
- Is not inherently idempotent.
For example, searching properties is normally a GET request, while creating a booking is normally a POST request.
7. What are side effects in API design?
A side effect occurs when an operation changes application state or causes an external action.
Examples include:
- Creating a booking.
- Updating a patient's referral authorization.
- Sending an email notification.
- Charging a payment method.
A GET endpoint should not trigger these changes. Use appropriate write methods and protect sensitive operations with authorization and validation.
Interview tip: Idempotency is especially important for payments and bookings. If a client retries a request because of a timeout, an idempotency key can help prevent duplicate operations.
8. How do you implement pagination?
Pagination prevents the server from returning thousands of records in a single response.
Example:
GET /bookings?page=2&limit=20


Offset-based SQL example:
SELECT *
FROM bookings
ORDER BY id
LIMIT 20 OFFSET 20;


Here, the server returns 20 records, skipping the first 20.
For very large datasets, cursor-based pagination is often more efficient and stable than large offsets, provided the cursor uses an appropriate indexed and consistently ordered field.
Also validate the page and limit, set a maximum page size, and use a deterministic sort order.
9. How do you maintain data consistency?
Data consistency means ensuring clients receive data that follows the system's correctness rules.
Common techniques include:
- Database transactions for related operations.
- Appropriate database constraints.
- Optimistic locking or version numbers to prevent conflicting updates.
- Idempotency keys for retryable write requests.
- Defined consistency guarantees for distributed databases.
- Retry mechanisms with appropriate backoff for transient failures.
For example, a homestay booking API should prevent two users from successfully booking the same room for overlapping dates. This requires reliable database-level concurrency control, not just frontend validation.
10. How do you make APIs scalable and extensible?
For a production system, I would focus on these areas:
- Stateless services: Keep request-specific state out of application memory where possible, so requests can be handled by different instances.
- Load balancing: Distribute traffic across multiple backend instances.
- Database optimization: Use indexes, connection pooling, and efficient queries.
- Caching: Cache frequently requested data when appropriate.
- Rate limiting: Protect services against excessive traffic.
- Asynchronous processing: Move slow tasks, such as email delivery, to queues.
- Backward compatibility: Version APIs and avoid unexpectedly changing existing response fields.
- Observability: Use structured logs, metrics, tracing, and alerts.
Example versioned endpoint:
GET /api/v1/properties


When introducing breaking changes, a new API version can allow existing clients to continue working while they migrate.
11. Example: Designing a booking API in Node.js and Express
const express = require("express");

const app = express();
app.use(express.json());

// Create a booking
app.post("/api/v1/bookings", async (req, res) => {
  try {
    const { propertyId, checkIn, checkOut, guests } = req.body;

    // Validate required fields
    if (!propertyId || !checkIn || !checkOut || !guests) {
      return res.status(400).json({
        success: false,
        error: {
          code: "VALIDATION_ERROR",
          message: "All booking fields are required."
        }
      });
    }

    // In production, validate dates and guest limits,
    // authenticate the user, and check availability.
    // Persist the booking through a service/database layer.

    return res.status(201).json({
      success: true,
      message: "Booking request received."
    });
  } catch (error) {
    // Log the internal error securely
    console.error("Booking API failed:", error);

    return res.status(500).json({
      success: false,
      error: {
        code: "INTERNAL_SERVER_ERROR",
        message: "Something went wrong."
      }
    });
  }
});

app.listen(3000, () => {
  console.log("Server running on port 3000");
});
# API Response Standard

This document defines the standard API response format used throughout the CRMSystem.

The goal is to make every API endpoint return a consistent response structure for both successful requests and errors.

---

## 1. Response Format

All API responses should follow this general structure:

### Success Response

```json
{
  "success": true,
  "message": "Resource retrieved successfully.",
  "data": {}
}
```

### Error Response

```json
{
  "success": false,
  "message": "Resource not found.",
  "data": null
}
```

The response contains three properties:

| Property  | Type     | Description                                       |
| --------- | -------- | ------------------------------------------------- |
| `success` | `bool`   | Indicates whether the request was successful      |
| `message` | `string` | Human-readable result message                     |
| `data`    | `object` | Response data, or `null` when no data is returned |

---

# 2. Important Files

The API response standard is mainly implemented using:

* `ApiResponse.cs`
* `ErrorResponse.cs`
* `BaseController.cs`
* `ExceptionMiddleware.cs`
* `Program.cs`

The responsibilities are:

```text
ApiResponse
    ↓
Standard success response

ErrorResponse
    ↓
Standard error response

BaseController
    ↓
Provides reusable Success() methods

ExceptionMiddleware
    ↓
Handles unhandled exceptions globally

Program.cs
    ↓
Registers the middleware
```

---

# 3. ApiResponse.cs

Create:

```text
Models/Responses/ApiResponse.cs
```

```csharp
namespace CRMSystem.Models.Responses
{
    public class ApiResponse<T>
    {
        public bool Success { get; set; }
        public string Message { get; set; } = null!;
        public T? Data { get; set; }
    }
}
```

`ApiResponse<T>` is the standard wrapper for successful responses.

The generic `T` allows the API to return different types of data.

For example:

```text
ApiResponse<CustomerResponseDto>
ApiResponse<OpportunityResponseDto>
ApiResponse<List<CustomerResponseDto>>
ApiResponse<List<OpportunityResponseDto>>
```

---

# 4. ErrorResponse.cs

Create:

```text
Models/Responses/ErrorResponse.cs
```

```csharp
namespace CRMSystem.Models.Responses
{
    public class ErrorResponse
    {
        public bool Success { get; set; } = false;
        public string Message { get; set; } = null!;
        public object? Data { get; set; }
    }
}
```

This is used when an API request fails.

Example:

```json
{
  "success": false,
  "message": "Opportunity not found.",
  "data": null
}
```

For validation errors, `data` can contain field-specific errors:

```json
{
  "success": false,
  "message": "Validation failed.",
  "data": {
    "name": [
      "The Name field is required."
    ],
    "value": [
      "Value must be greater than 0."
    ]
  }
}
```

---

# 5. BaseController.cs

Create:

```text
Controllers/BaseController.cs
```

```csharp
using CRMSystem.Models.Responses;
using Microsoft.AspNetCore.Mvc;

namespace CRMSystem.Controllers
{
    public abstract class BaseController : ControllerBase
    {
        protected IActionResult Success<T>(string message, T data)
        {
            return Ok(new ApiResponse<T>
            {
                Success = true,
                Message = message,
                Data = data
            });
        }

        protected IActionResult Success(string message)
        {
            return Ok(new ApiResponse<object>
            {
                Success = true,
                Message = message,
                Data = null
            });
        }
    }
}
```

The `BaseController` provides reusable methods for successful API responses.

---

# 6. Inherit BaseController

All API controllers should inherit from `BaseController` instead of directly inheriting from `ControllerBase`.

### Before

```csharp
public class OpportunityController : ControllerBase
```

### After

```csharp
public class OpportunityController : BaseController
```

Example:

```csharp
[Route("api/opportunities")]
[ApiController]
public class OpportunityController : BaseController
{
    private readonly IOpportunityServices _opportunityServices;

    public OpportunityController(IOpportunityServices opportunityServices)
    {
        _opportunityServices = opportunityServices;
    }
}
```

---

# 7. Using Success()

## Response with data

For GET requests:

```csharp
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetOpportunityById(Guid id)
{
    var opportunity = await _opportunityServices.GetOpportunityById(id);

    return Success(
        "Opportunity retrieved successfully.",
        opportunity
    );
}
```

Response:

```json
{
  "success": true,
  "message": "Opportunity retrieved successfully.",
  "data": {
    "id": "8a4d0000-0000-0000-0000-000000000000",
    "name": "Hospital Queue System",
    "value": 150000,
    "status": "OPEN"
  }
}
```

---

## Response without data

For operations such as DELETE:

```csharp
[HttpDelete("{id:guid}")]
public async Task<IActionResult> DeleteOpportunity(Guid id)
{
    await _opportunityServices.DeleteOpportunity(id);

    return Success("Opportunity deleted successfully.");
}
```

Response:

```json
{
  "success": true,
  "message": "Opportunity deleted successfully.",
  "data": null
}
```

---

# 8. HTTP Status Codes

The API should use HTTP status codes to communicate the result of the request.

| Status Code                 | Meaning                                                | Typical Usage                             |
| --------------------------- | ------------------------------------------------------ | ----------------------------------------- |
| `200 OK`                    | Request succeeded                                      | GET, PUT, PATCH, successful DELETE        |
| `201 Created`               | Resource created                                       | POST                                      |
| `204 No Content`            | Request succeeded without response body                | Optional for DELETE                       |
| `400 Bad Request`           | Invalid request                                        | Invalid input, business validation        |
| `401 Unauthorized`          | Authentication required/failed                         | Missing or invalid JWT                    |
| `403 Forbidden`             | Authenticated but not allowed                          | Insufficient role/permission              |
| `404 Not Found`             | Resource does not exist                                | Customer, opportunity, pipeline not found |
| `409 Conflict`              | Resource conflicts with existing data                  | Duplicate record                          |
| `422 Unprocessable Entity`  | Request is syntactically valid but cannot be processed | Complex validation/business rules         |
| `500 Internal Server Error` | Unexpected server error                                | Unhandled application/server exception    |

---

# 9. GET - 200 OK

A successful GET request should normally return `200 OK`.

Example:

```http
GET /api/opportunities/8a4d0000-0000-0000-0000-000000000000
```

Response:

```json
{
  "success": true,
  "message": "Opportunity retrieved successfully.",
  "data": {
    "id": "8a4d0000-0000-0000-0000-000000000000",
    "name": "Hospital Queue System",
    "value": 150000,
    "status": "OPEN"
  }
}
```

---

# 10. POST - 201 Created

POST requests that successfully create a resource should normally return `201 Created`.

Example:

```http
POST /api/opportunities
```

Request:

```json
{
  "customerId": "11111111-1111-1111-1111-111111111111",
  "pipelineId": "22222222-2222-2222-2222-222222222222",
  "stageId": "33333333-3333-3333-3333-333333333333",
  "name": "Hospital Queue System",
  "value": 150000
}
```

Response:

```json
{
  "success": true,
  "message": "Opportunity created successfully.",
  "data": null
}
```

For a production API, the response can optionally return the newly created resource.

---

# 11. PUT - 200 OK

A successful update normally returns `200 OK`.

```json
{
  "success": true,
  "message": "Opportunity updated successfully.",
  "data": null
}
```

---

# 12. PATCH - 200 OK

A successful partial update normally returns `200 OK`.

Example:

```http
PATCH /api/opportunities/{id}/status
```

Response:

```json
{
  "success": true,
  "message": "Opportunity status updated successfully.",
  "data": null
}
```

---

# 13. DELETE - 200 OK

A successful DELETE can return `200 OK` with the standard response:

```json
{
  "success": true,
  "message": "Opportunity deleted successfully.",
  "data": null
}
```

Alternatively, the API may use `204 No Content` when no response body is needed.

For this CRM API, `200 OK` can be used consistently when returning the standard response body.

---

# 14. 400 Bad Request

Use `400 Bad Request` when the client sends an invalid request or violates a business rule.

Example:

```json
{
  "success": false,
  "message": "Pipeline stage does not belong to the opportunity pipeline.",
  "data": null
}
```

Possible causes:

* Invalid request data
* Invalid combination of IDs
* Invalid business operation
* Invalid value
* Invalid state transition

---

# 15. 401 Unauthorized

Use `401 Unauthorized` when the request does not contain valid authentication credentials.

Example:

```json
{
  "success": false,
  "message": "Authentication is required.",
  "data": null
}
```

Typical causes:

* JWT is missing
* JWT is expired
* JWT is invalid
* Authentication failed

---

# 16. 403 Forbidden

Use `403 Forbidden` when the user is authenticated but does not have permission to perform the operation.

Example:

```json
{
  "success": false,
  "message": "You do not have permission to perform this action.",
  "data": null
}
```

Example:

```text
User
    ↓
Authenticated
    ↓
Role = Sales
    ↓
Attempt Admin-only operation
    ↓
403 Forbidden
```

---

# 17. 404 Not Found

Use `404 Not Found` when the requested resource does not exist within the user's organization scope.

Example:

```json
{
  "success": false,
  "message": "Opportunity not found.",
  "data": null
}
```

For a multi-tenant CRM, resource queries should remain organization-scoped.

For example:

```csharp
var opportunity = await _opportunityRepository
    .GetOpportunityById(id, _currentUserServices.OrganizationId);
```

This prevents users from accessing resources belonging to another organization.

---

# 18. 409 Conflict

Use `409 Conflict` when the request conflicts with existing data.

Example:

```json
{
  "success": false,
  "message": "Customer with this email already exists.",
  "data": null
}
```

Possible uses:

* Duplicate customer
* Duplicate email
* Duplicate pipeline name
* Duplicate lead source
* Duplicate organization data

---

# 19. 422 Unprocessable Entity

`422` can be used when the request is structurally valid but cannot be processed because of domain/business rules.

Example:

```json
{
  "success": false,
  "message": "The opportunity cannot be marked as won because the required stage is incomplete.",
  "data": null
}
```

Whether your project uses `400` or `422` for these cases should be standardized rather than mixed randomly.

---

# 20. 500 Internal Server Error

Unexpected exceptions should result in `500 Internal Server Error`.

Example:

```json
{
  "success": false,
  "message": "An unexpected error occurred.",
  "data": null
}
```

Internal exception details should not normally be exposed to the client in production.

Avoid returning:

```json
{
  "success": false,
  "message": "Npgsql.PostgresException: 23505...",
  "data": null
}
```

Instead, log the actual exception on the server and return a safe message to the client.

---

# 21. ExceptionMiddleware.cs

Create:

```text
Middleware/ExceptionMiddleware.cs
```

A basic implementation:

```csharp
using CRMSystem.Models.Responses;

namespace CRMSystem.Middleware
{
    public class ExceptionMiddleware
    {
        private readonly RequestDelegate _next;

        public ExceptionMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            try
            {
                await _next(context);
            }
            catch (Exception ex)
            {
                await HandleExceptionAsync(context, ex);
            }
        }

        private static async Task HandleExceptionAsync(HttpContext context, Exception ex)
        {
            context.Response.ContentType = "application/json";
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;

            var response = new ErrorResponse
            {
                Success = false,
                Message = "An unexpected error occurred.",
                Data = null
            };

            await context.Response.WriteAsJsonAsync(response);
        }
    }
}
```

The middleware catches unexpected exceptions that were not handled elsewhere.

---

# 22. Register ExceptionMiddleware

In `Program.cs`:

```csharp
var app = builder.Build();

app.UseMiddleware<ExceptionMiddleware>();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

The important part is:

```csharp
app.UseMiddleware<ExceptionMiddleware>();
```

This allows the middleware to catch exceptions thrown by downstream application components.

---

# 23. Controller Responsibility

Controllers should remain focused on HTTP concerns.

Example:

```csharp
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetOpportunityById(Guid id)
{
    var opportunity = await _opportunityServices.GetOpportunityById(id);

    return Success(
        "Opportunity retrieved successfully.",
        opportunity
    );
}
```

The controller should not contain large amounts of business logic.

The general flow should be:

```text
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

The controller is responsible for receiving the HTTP request and returning the HTTP response.

The service is responsible for business rules.

The repository is responsible for database operations.

---

# 24. Recommended Response Standard

For CRMSystem, use the following standard:

### Successful request

```json
{
  "success": true,
  "message": "Operation completed successfully.",
  "data": {}
}
```

### Failed request

```json
{
  "success": false,
  "message": "Operation failed.",
  "data": null
}
```

### Validation failure

```json
{
  "success": false,
  "message": "Validation failed.",
  "data": {
    "field": [
      "Validation error."
    ]
  }
}
```

---

# 25. Recommended HTTP Status Mapping

| Operation      | Success | Common Errors                     |
| -------------- | ------: | --------------------------------- |
| GET collection |   `200` | `401`, `403`                      |
| GET by ID      |   `200` | `401`, `403`, `404`               |
| POST           |   `201` | `400`, `401`, `403`, `409`        |
| PUT            |   `200` | `400`, `401`, `403`, `404`, `409` |
| PATCH          |   `200` | `400`, `401`, `403`, `404`, `409` |
| DELETE         |   `200` | `401`, `403`, `404`               |

Unexpected server failures should return:

```text
500 Internal Server Error
```

---

# 26. Final Architecture

The response system should work like this:

```text
                    HTTP Request
                         │
                         ▼
                   Controller
                         │
                         ▼
                     Service
                         │
                         ▼
                   Repository
                         │
                         ▼
                     Database
                         │
                         ▼
                   Service Result
                         │
                         ▼
                   BaseController
                         │
                         ▼
                  ApiResponse<T>
                         │
                         ▼
                    HTTP Response
```

For unexpected errors:

```text
Controller
    │
    ▼
Service
    │
    ├── Exception
    │
    ▼
ExceptionMiddleware
    │
    ▼
ErrorResponse
    │
    ▼
HTTP 500
```

This approach keeps the API response format consistent across Customers, Leads, Opportunities, Pipelines, Tasks, Activities, Notes, Users, and other CRM resources.

---

## Summary

The minimum components required are:

```text
ApiResponse.cs
    → Standard successful response

ErrorResponse.cs
    → Standard error response

BaseController.cs
    → Reusable Success() methods

ExceptionMiddleware.cs
    → Global unexpected exception handling

Program.cs
    → Middleware registration
```

The overall API contract is:

```text
SUCCESS
HTTP 2xx
{
    success: true,
    message: "...",
    data: ...
}

ERROR
HTTP 4xx / 5xx
{
    success: false,
    message: "...",
    data: ...
}
```

This gives the frontend a predictable response structure regardless of which CRM endpoint it calls.

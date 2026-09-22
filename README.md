# API Response Standard

This document defines the standard API response format used throughout **CRMSystem**.

The goal is to make every API endpoint return a consistent response structure for both successful requests and errors.

The API follows these principles:

* All successful responses use a consistent `ApiResponse<T>` structure.
* All application errors use a consistent `ErrorResponse` structure.
* Controllers should not contain repetitive `try/catch` blocks.
* Exceptions are handled globally through `ExceptionHandlingMiddleware`.
* Business logic belongs in services, not controllers.
* HTTP status codes must accurately represent the result of the request.
* Organization-scoped resources must remain isolated in the multi-tenant CRM.

---

# 1. Response Format

All API responses should follow this general structure.

## Success Response

```json
{
  "success": true,
  "message": "Resource retrieved successfully.",
  "data": {}
}
```

## Error Response

```json
{
  "success": false,
  "message": "Resource not found.",
  "data": null
}
```

The response contains three properties:

| Property  | Type     | Description                                  |
| --------- | -------- | -------------------------------------------- |
| `success` | `bool`   | Indicates whether the request was successful |
| `message` | `string` | Human-readable result message                |
| `data`    | `object` | Response data, validation errors, or `null`  |

---

# 2. Important Files

The API response standard is implemented using the following components:

```text
Models/Responses/
├── ApiResponse.cs
└── ErrorResponse.cs

Controllers/
└── BaseController.cs

Exceptions/
├── BadRequestException.cs
├── UnauthorizedException.cs
├── ForbiddenException.cs
├── NotFoundException.cs
├── ConflictException.cs
└── UnprocessableEntityException.cs

Middleware/
└── ExceptionHandlingMiddleware.cs

Program.cs
```

Responsibilities:

```text
ApiResponse<T>
    ↓
Standard successful response

ErrorResponse
    ↓
Standard error response

BaseController
    ↓
Reusable success response methods

Custom Exceptions
    ↓
Represent application/business errors

ExceptionHandlingMiddleware
    ↓
Converts exceptions into HTTP error responses

Program.cs
    ↓
Registers the global exception middleware
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

Examples:

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
        protected IActionResult Created<T>(string message, T data)
        {
            return StatusCode(StatusCodes.Status201Created, new ApiResponse<T> { Success = true, Message = message, Data = data });
        }

        protected IActionResult Created(string message)
        {
            return StatusCode(StatusCodes.Status201Created, new ApiResponse<object> { Success = true, Message = message, Data = null });
        }
    }
}
```

The `BaseController` provides reusable methods for successful API responses.

This prevents controllers from repeatedly creating:

```csharp
return Ok(new ApiResponse<T>
{
    Success = true,
    Message = "...",
    Data = ...
});
```

---

# 6. Inherit BaseController

All API controllers should inherit from `BaseController` instead of directly inheriting from `ControllerBase`.

## Before

```csharp
public class OpportunityController : ControllerBase
```

## After

```csharp
public class OpportunityController : BaseController
```

Example:

```csharp
[ApiController]
[Route("api/opportunities")]
public class OpportunityController : BaseController
{
    private readonly IOpportunityServices _opportunityServices;

    public OpportunityController(
        IOpportunityServices opportunityServices)
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
    var opportunity =
        await _opportunityServices.GetOpportunityById(id);

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

    return Success(
        "Opportunity deleted successfully."
    );
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

| Status Code                 | Meaning                                  | Typical Usage                              |
| --------------------------- | ---------------------------------------- | ------------------------------------------ |
| `200 OK`                    | Request succeeded                        | GET, PUT, PATCH, DELETE with response body |
| `201 Created`               | Resource created                         | POST                                       |
| `204 No Content`            | Request succeeded without response body  | Optional DELETE                            |
| `400 Bad Request`           | Request is invalid                       | Invalid input or malformed request         |
| `401 Unauthorized`          | Authentication is required or failed     | Missing/invalid JWT                        |
| `403 Forbidden`             | Authenticated but not permitted          | Insufficient role/permission               |
| `404 Not Found`             | Resource does not exist                  | Customer, opportunity, pipeline not found  |
| `409 Conflict`              | Resource conflicts with existing data    | Duplicate record                           |
| `422 Unprocessable Entity`  | Request is valid but cannot be processed | Complex business/domain rules              |
| `500 Internal Server Error` | Unexpected server error                  | Unhandled application/server exception     |

---

# 9. Exception Handling Architecture

Controllers should **not** contain repetitive `try/catch` blocks.

Avoid:

```csharp
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetCustomer(Guid id)
{
    try
    {
        var customer =
            await _customerServices.GetCustomerById(id);

        return Success(
            "Customer retrieved successfully.",
            customer
        );
    }
    catch (KeyNotFoundException ex)
    {
        return NotFound(new ErrorResponse
        {
            Message = ex.Message
        });
    }
}
```

Instead:

```csharp
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetCustomer(Guid id)
{
    var customer =
        await _customerServices.GetCustomerById(id);

    return Success(
        "Customer retrieved successfully.",
        customer
    );
}
```

The service throws an application exception:

```csharp
if (customer == null)
{
    throw new NotFoundException(
        "Customer not found."
    );
}
```

The global middleware catches it and converts it into the appropriate HTTP response.

---

# 10. Custom Exceptions

Create:

```text
Exceptions/
├── BadRequestException.cs
├── UnauthorizedException.cs
├── ForbiddenException.cs
├── NotFoundException.cs
├── ConflictException.cs
└── UnprocessableEntityException.cs
```

---

## 10.1 BadRequestException

File:

```text
Exceptions/BadRequestException.cs
```

```csharp
namespace CRMSystem.Exceptions
{
    public class BadRequestException : Exception
    {
        public BadRequestException(string message)
            : base(message)
        {
        }
    }
}
```

Maps to:

```text
400 Bad Request
```

Use when the request itself is invalid.

Example:

```csharp
throw new BadRequestException(
    "Customer name is required."
);
```

---

# 11. UnauthorizedException

File:

```text
Exceptions/UnauthorizedException.cs
```

```csharp
namespace CRMSystem.Exceptions
{
    public class UnauthorizedException : Exception
    {
        public UnauthorizedException(string message)
            : base(message)
        {
        }
    }
}
```

Maps to:

```text
401 Unauthorized
```

Example:

```csharp
throw new UnauthorizedException(
    "Authentication is required."
);
```

### Important

Normal JWT authentication should be handled by ASP.NET Core:

```csharp
[Authorize]
```

Do not manually throw `UnauthorizedException` for every missing or invalid JWT.

ASP.NET Core Authentication should normally handle:

* Missing JWT
* Invalid JWT
* Expired JWT
* Invalid authentication credentials

---

# 12. ForbiddenException

File:

```text
Exceptions/ForbiddenException.cs
```

```csharp
namespace CRMSystem.Exceptions
{
    public class ForbiddenException : Exception
    {
        public ForbiddenException(string message)
            : base(message)
        {
        }
    }
}
```

Maps to:

```text
403 Forbidden
```

Use when the user is authenticated but does not have permission to perform a particular business operation.

Example:

```csharp
throw new ForbiddenException(
    "You do not have permission to modify this customer."
);
```

Normal role-based authorization should still use:

```csharp
[Authorize(Roles = "Admin, SalesManager")]
```

---

# 13. NotFoundException

File:

```text
Exceptions/NotFoundException.cs
```

```csharp
namespace CRMSystem.Exceptions
{
    public class NotFoundException : Exception
    {
        public NotFoundException(string message)
            : base(message)
        {
        }
    }
}
```

Maps to:

```text
404 Not Found
```

This will be one of the most frequently used exceptions in the CRM.

Example:

```csharp
var customer =
    await _customerRepository.GetByIdAsync(customerId);

if (customer == null)
{
    throw new NotFoundException(
        "Customer not found."
    );
}
```

Other examples:

```csharp
throw new NotFoundException(
    "Customer address not found."
);
```

```csharp
throw new NotFoundException(
    "Pipeline stage not found."
);
```

```csharp
throw new NotFoundException(
    "Opportunity not found."
);
```

---

# 14. ConflictException

File:

```text
Exceptions/ConflictException.cs
```

```csharp
namespace CRMSystem.Exceptions
{
    public class ConflictException : Exception
    {
        public ConflictException(string message)
            : base(message)
        {
        }
    }
}
```

Maps to:

```text
409 Conflict
```

Use when the requested operation conflicts with existing data.

Example:

```csharp
throw new ConflictException(
    "A customer with this email already exists."
);
```

Other examples:

```csharp
throw new ConflictException(
    "A pipeline with this name already exists."
);
```

```csharp
throw new ConflictException(
    "Customer contact already exists."
);
```

---

# 15. UnprocessableEntityException

File:

```text
Exceptions/UnprocessableEntityException.cs
```

```csharp
namespace CRMSystem.Exceptions
{
    public class UnprocessableEntityException : Exception
    {
        public UnprocessableEntityException(string message)
            : base(message)
        {
        }
    }
}
```

Maps to:

```text
422 Unprocessable Entity
```

Use this when the request is syntactically valid but a domain/business rule prevents the operation.

Example:

```csharp
throw new UnprocessableEntityException(
    "A closed opportunity cannot be moved back to an open stage."
);
```

Another example:

```csharp
throw new UnprocessableEntityException(
    "A lead cannot be converted without an associated customer."
);
```

---

# 16. Exception-to-HTTP Mapping

The custom exceptions should map to HTTP status codes as follows:

| Exception                      | HTTP Status |
| ------------------------------ | ----------: |
| `BadRequestException`          |       `400` |
| `UnauthorizedException`        |       `401` |
| `ForbiddenException`           |       `403` |
| `NotFoundException`            |       `404` |
| `ConflictException`            |       `409` |
| `UnprocessableEntityException` |       `422` |
| Unknown `Exception`            |       `500` |

---

# 17. ExceptionHandlingMiddleware.cs

Create:

```text
Middleware/ExceptionHandlingMiddleware.cs
```

```csharp
using System.Net;
using CRMSystem.Exceptions;
using CRMSystem.Models.Responses;

namespace CRMSystem.Middleware
{
    public class ExceptionHandlingMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<ExceptionHandlingMiddleware> _logger;

        public ExceptionHandlingMiddleware(
            RequestDelegate next,
            ILogger<ExceptionHandlingMiddleware> logger)
        {
            _next = next;
            _logger = logger;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            try
            {
                await _next(context);
            }
            catch (Exception ex)
            {
                _logger.LogError(
                    ex,
                    "An unhandled exception occurred while processing the request."
                );

                await HandleExceptionAsync(context, ex);
            }
        }

        private static async Task HandleExceptionAsync(
            HttpContext context,
            Exception exception)
        {
            var statusCode = exception switch
            {
                BadRequestException =>
                    (int)HttpStatusCode.BadRequest,

                UnauthorizedException =>
                    (int)HttpStatusCode.Unauthorized,

                ForbiddenException =>
                    (int)HttpStatusCode.Forbidden,

                NotFoundException =>
                    (int)HttpStatusCode.NotFound,

                ConflictException =>
                    (int)HttpStatusCode.Conflict,

                UnprocessableEntityException =>
                    StatusCodes.Status422UnprocessableEntity,

                _ =>
                    (int)HttpStatusCode.InternalServerError
            };

            var message = exception switch
            {
                BadRequestException =>
                    exception.Message,

                UnauthorizedException =>
                    exception.Message,

                ForbiddenException =>
                    exception.Message,

                NotFoundException =>
                    exception.Message,

                ConflictException =>
                    exception.Message,

                UnprocessableEntityException =>
                    exception.Message,

                _ =>
                    "An unexpected error occurred."
            };

            var response = new ErrorResponse
            {
                Success = false,
                Message = message,
                Data = null
            };

            context.Response.StatusCode = statusCode;
            context.Response.ContentType = "application/json";

            await context.Response.WriteAsJsonAsync(response);
        }
    }
}
```

---

# 18. Register ExceptionHandlingMiddleware

In `Program.cs`:

```csharp
using CRMSystem.Middleware;
```

Then:

```csharp
var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

The exception middleware should be registered before the application components whose exceptions it needs to catch.

---

# 19. Service Layer Usage

Business exceptions should normally be thrown from the service layer.

Example:

```csharp
public async Task<CustomerResponseDto> GetCustomerById(
    Guid customerId)
{
    var customer =
        await _customerRepository.GetByIdAsync(
            customerId
        );

    if (customer == null)
    {
        throw new NotFoundException(
            "Customer not found."
        );
    }

    return MapToResponse(customer);
}
```

For duplicate data:

```csharp
var existingCustomer =
    await _customerRepository.GetByEmailAsync(
        dto.Email
    );

if (existingCustomer != null)
{
    throw new ConflictException(
        "A customer with this email already exists."
    );
}
```

For business rules:

```csharp
if (opportunity.Status == "CLOSED")
{
    throw new UnprocessableEntityException(
        "A closed opportunity cannot be modified."
    );
}
```

---

# 20. Controller Responsibility

Controllers should primarily handle HTTP/API concerns.

Example:

```csharp
[HttpGet("{customerId:guid}")]
public async Task<IActionResult> GetCustomerById(
    Guid customerId)
{
    var customer =
        await _customerServices.GetCustomerById(
            customerId
        );

    return Success(
        "Customer retrieved successfully.",
        customer
    );
}
```

There should normally be no repetitive exception handling:

```csharp
try
{
    // ...
}
catch
{
    // ...
}
```

inside every controller action.

The general responsibility is:

```text
Controller
    ↓
HTTP/API concerns

Service
    ↓
Business logic and rules

Repository
    ↓
Database access

ExceptionHandlingMiddleware
    ↓
Global exception → HTTP response
```

---

# 21. GET - 200 OK

A successful GET request normally returns:

```text
200 OK
```

Example:

```http
GET /api/opportunities/{id}
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

# 22. POST - 201 Created

POST requests that successfully create a resource should normally return:

```text
201 Created
```

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

The API may optionally return the newly created resource:

```json
{
  "success": true,
  "message": "Opportunity created successfully.",
  "data": {
    "id": "8a4d0000-0000-0000-0000-000000000000",
    "name": "Hospital Queue System",
    "value": 150000
  }
}
```

---

# 23. PUT - 200 OK

A successful full update normally returns:

```text
200 OK
```

Example:

```json
{
  "success": true,
  "message": "Opportunity updated successfully.",
  "data": null
}
```

If the API returns the updated resource:

```json
{
  "success": true,
  "message": "Opportunity updated successfully.",
  "data": {
    "id": "8a4d0000-0000-0000-0000-000000000000",
    "name": "Updated Opportunity"
  }
}
```

---

# 24. PATCH - 200 OK

A successful partial update normally returns:

```text
200 OK
```

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

# 25. DELETE - 200 OK

A DELETE operation may return:

```text
200 OK
```

when the API returns the standard response body.

Example:

```json
{
  "success": true,
  "message": "Opportunity deleted successfully.",
  "data": null
}
```

For endpoints that do not need a response body, `204 No Content` may also be used.

For consistency, CRMSystem may use `200 OK` when returning `ApiResponse`.

---

# 26. 400 Bad Request

Use `400 Bad Request` when the request itself is invalid.

Example:

```json
{
  "success": false,
  "message": "Customer name is required.",
  "data": null
}
```

Possible causes:

* Invalid request data
* Malformed request
* Invalid input
* Invalid parameter
* Invalid combination of request values

Application-level examples:

```csharp
throw new BadRequestException(
    "Customer name is required."
);
```

---

# 27. 401 Unauthorized

Use `401 Unauthorized` when authentication is required or authentication fails.

Typical causes:

* JWT is missing
* JWT is invalid
* JWT is expired
* Authentication failed

Normal authentication should be handled by ASP.NET Core authentication middleware and `[Authorize]`.

Example endpoint:

```csharp
[Authorize]
[HttpGet]
public async Task<IActionResult> GetCustomers()
{
    // ...
}
```

An unauthenticated request should not reach the protected endpoint.

---

# 28. 403 Forbidden

Use `403 Forbidden` when the user is authenticated but does not have permission to perform the operation.

Example:

```csharp
[Authorize(Roles = "Admin, SalesManager")]
[HttpDelete("{id:guid}")]
public async Task<IActionResult> DeleteCustomer(Guid id)
{
    // ...
}
```

A user who is authenticated but does not have one of the required roles should receive:

```text
403 Forbidden
```

For additional business-level permission checks:

```csharp
throw new ForbiddenException(
    "You do not have permission to modify this customer."
);
```

---

# 29. 404 Not Found

Use `404 Not Found` when the requested resource does not exist within the user's accessible organization scope.

Example:

```json
{
  "success": false,
  "message": "Opportunity not found.",
  "data": null
}
```

Service:

```csharp
if (opportunity == null)
{
    throw new NotFoundException(
        "Opportunity not found."
    );
}
```

For the multi-tenant CRM, resource queries must remain organization-scoped.

Example:

```csharp
var opportunity =
    await _opportunityRepository.GetByIdAsync(
        opportunityId,
        organizationId
    );
```

This prevents users from accessing resources belonging to another organization.

---

# 30. 409 Conflict

Use `409 Conflict` when the requested operation conflicts with existing data.

Example:

```json
{
  "success": false,
  "message": "Customer with this email already exists.",
  "data": null
}
```

Service:

```csharp
if (existingCustomer != null)
{
    throw new ConflictException(
        "Customer with this email already exists."
    );
}
```

Common uses:

* Duplicate customer
* Duplicate email
* Duplicate pipeline name
* Duplicate lead source
* Duplicate organization data
* Duplicate contact

---

# 31. 422 Unprocessable Entity

Use `422 Unprocessable Entity` when the request is syntactically valid but cannot be processed because of a domain or business rule.

Example:

```json
{
  "success": false,
  "message": "A closed opportunity cannot be moved back to an open stage.",
  "data": null
}
```

Service:

```csharp
if (opportunity.Status == "CLOSED")
{
    throw new UnprocessableEntityException(
        "A closed opportunity cannot be moved back to an open stage."
    );
}
```

Typical uses:

* Invalid state transition
* Business rule violation
* Invalid domain operation
* Resource cannot transition to the requested state

`400` and `422` should not be used randomly. The project should follow this standard consistently.

---

# 32. 500 Internal Server Error

Unexpected exceptions should result in:

```text
500 Internal Server Error
```

Example:

```json
{
  "success": false,
  "message": "An unexpected error occurred.",
  "data": null
}
```

Internal exception details should not normally be exposed to clients in production.

Avoid returning:

```json
{
  "success": false,
  "message": "Npgsql.PostgresException: 23505...",
  "data": null
}
```

Instead:

```text
Application
    ↓
Unexpected exception
    ↓
ExceptionHandlingMiddleware
    ↓
Log detailed exception
    ↓
Return safe 500 response
```

The middleware logs the actual exception:

```csharp
_logger.LogError(
    ex,
    "An unhandled exception occurred while processing the request."
);
```

The client receives only:

```json
{
  "success": false,
  "message": "An unexpected error occurred.",
  "data": null
}
```

---

# 33. Validation Errors

Request validation should return a consistent error structure.

Example:

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

For example, a DTO may use:

```csharp
public class CreateCustomerDto
{
    [Required]
    public string Name { get; set; } = null!;

    [EmailAddress]
    public string? Email { get; set; }
}
```

ASP.NET Core's `[ApiController]` automatically performs model validation.

The project's global response handling should preserve the standard:

```text
success
message
data
```

structure for validation errors.

---

# 34. Recommended HTTP Status Mapping

| Operation      | Success | Common Errors                            |
| -------------- | ------: | ---------------------------------------- |
| GET collection |   `200` | `401`, `403`                             |
| GET by ID      |   `200` | `401`, `403`, `404`                      |
| POST           |   `201` | `400`, `401`, `403`, `409`, `422`        |
| PUT            |   `200` | `400`, `401`, `403`, `404`, `409`, `422` |
| PATCH          |   `200` | `400`, `401`, `403`, `404`, `409`, `422` |
| DELETE         |   `200` | `401`, `403`, `404`, `409`               |

Unexpected failures:

```text
500 Internal Server Error
```

---

# 35. Example Complete API Flow

Example:

```http
GET /api/customers/8a4d0000-0000-0000-0000-000000000000
```

### Step 1 — Controller

```csharp
[HttpGet("{customerId:guid}")]
public async Task<IActionResult> GetCustomerById(Guid customerId)
{
    var customer =
        await _customerServices.GetCustomerById(
            customerId
        );

    return Success(
        "Customer retrieved successfully.",
        customer
    );
}
```

### Step 2 — Service

```csharp
public async Task<CustomerResponseDto> GetCustomerById(
    Guid customerId)
{
    var customer =
        await _customerRepository.GetByIdAsync(
            customerId
        );

    if (customer == null)
    {
        throw new NotFoundException(
            "Customer not found."
        );
    }

    return MapToResponse(customer);
}
```

### Step 3 — Resource exists

The service returns the customer.

```text
Service
    ↓
CustomerResponseDto
    ↓
Controller
    ↓
200 OK
```

Response:

```json
{
  "success": true,
  "message": "Customer retrieved successfully.",
  "data": {
    "id": "8a4d0000-0000-0000-0000-000000000000",
    "name": "ABC Corporation"
  }
}
```

### Step 4 — Resource does not exist

The service throws:

```csharp
throw new NotFoundException(
    "Customer not found."
);
```

The middleware catches it:

```text
Service
    ↓
NotFoundException
    ↓
ExceptionHandlingMiddleware
    ↓
404 Not Found
```

Response:

```json
{
  "success": false,
  "message": "Customer not found.",
  "data": null
}
```

---

# 36. Final Architecture

The response architecture should follow this pattern:

```text
                         HTTP Request
                              │
                              ▼
                    ┌──────────────────┐
                    │    Controller    │
                    │  HTTP concerns   │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │     Service      │
                    │  Business logic  │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │    Repository    │
                    │  Data access     │
                    └────────┬─────────┘
                             │
                             ▼
                         Database
                             │
                             ▼
                    ┌──────────────────┐
                    │     Service      │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ BaseController   │
                    │ ApiResponse<T>   │
                    └────────┬─────────┘
                             │
                             ▼
                        HTTP 2xx
```

For errors:

```text
Controller
    │
    ▼
Service
    │
    │ throws
    ▼
Custom Exception
    │
    ▼
ExceptionHandlingMiddleware
    │
    ├── 400 Bad Request
    ├── 401 Unauthorized
    ├── 403 Forbidden
    ├── 404 Not Found
    ├── 409 Conflict
    ├── 422 Unprocessable Entity
    └── 500 Internal Server Error
             │
             ▼
       ErrorResponse
```

---

# 37. Final API Contract

Every successful response should follow:

```json
{
  "success": true,
  "message": "Operation completed successfully.",
  "data": {}
}
```

Every application error should follow:

```json
{
  "success": false,
  "message": "Operation failed.",
  "data": null
}
```

Validation errors may use:

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

The API should consistently use HTTP status codes together with these response structures.

---

# 38. Implementation Checklist

When adding a new CRM feature, follow this checklist:

### Response

* [ ] Use `ApiResponse<T>` for successful responses.
* [ ] Use `ErrorResponse` for application errors.
* [ ] Use `BaseController` for reusable success responses.

### Exceptions

* [ ] Use `BadRequestException` for invalid requests.
* [ ] Use `UnauthorizedException` only for application-level authentication failures.
* [ ] Use `ForbiddenException` for application-level permission failures.
* [ ] Use `NotFoundException` when a resource does not exist.
* [ ] Use `ConflictException` for duplicate/conflicting data.
* [ ] Use `UnprocessableEntityException` for domain/business-rule failures.
* [ ] Allow unexpected exceptions to become `500 Internal Server Error`.

### Controllers

* [ ] Do not add repetitive `try/catch` blocks.
* [ ] Keep controllers focused on HTTP concerns.
* [ ] Delegate business logic to services.
* [ ] Return the appropriate HTTP status code.

### Services

* [ ] Perform business validation.
* [ ] Check organization/tenant scope.
* [ ] Throw the appropriate custom exception.
* [ ] Do not return HTTP responses from services.

### Repositories

* [ ] Handle database access only.
* [ ] Keep organization-scoping requirements in repository/service queries as appropriate.
* [ ] Avoid exposing database-specific exceptions directly to API clients.

### Security

* [ ] Never expose stack traces in production responses.
* [ ] Never expose database exception details to clients.
* [ ] Keep resources organization-scoped.
* [ ] Use `[Authorize]` and role/policy authorization for access control.
* [ ] Log unexpected exceptions server-side.

---

# 39. Summary

The CRMSystem API response architecture consists of:

```text
ApiResponse<T>
    ↓
Standard successful responses

ErrorResponse
    ↓
Standard error responses

BaseController
    ↓
Reusable controller response methods

Custom Exceptions
    ↓
Application/business errors

ExceptionHandlingMiddleware
    ↓
Centralized exception → HTTP status conversion

Program.cs
    ↓
Global middleware registration
```

The final API contract is:

```text
SUCCESS
HTTP 2xx

{
    "success": true,
    "message": "...",
    "data": ...
}
```

```text
ERROR
HTTP 4xx / 5xx

{
    "success": false,
    "message": "...",
    "data": ...
}
```

This standard should be followed consistently across all CRMSystem modules, including:

```text
Organizations
Users
Customers
Customer Contacts
Customer Addresses
Leads
Lead Sources
Lead Statuses
Pipelines
Pipeline Stages
Opportunities
Tasks
Activities
Notes
Audit Logs
Refresh Tokens
```

The objective is to maintain a predictable API contract while keeping controllers clean, services responsible for business logic, repositories responsible for data access, and exception handling centralized.

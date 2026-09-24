# API Response Standard

This document defines the standard API response format used throughout **CRMSystem**.

The goal is to provide a consistent response structure for successful requests and application errors.

---

## 1. Response Structure

CRMSystem uses the following response structure:

### Success Response

```json
{
  "success": true,
  "message": "Customer retrieved successfully.",
  "data": {}
}
```

### Error Response

```json
{
  "success": false,
  "message": "Customer not found.",
  "data": null
}
```

Every standard response contains three properties:

| Property  | Type     | Description                                            |
| --------- | -------- | ------------------------------------------------------ |
| `success` | `bool`   | Indicates whether the request was successful.          |
| `message` | `string` | Human-readable result or error message.                |
| `data`    | `object` | Response data. Can be `null` when no data is returned. |

---

# 2. Response Components

The response standard is implemented using the following components:

```text
Models/Responses/
├── ApiResponse.cs
└── ErrorResponse.cs

Controllers/
└── BaseController.cs

Middleware/
└── ExceptionHandlingMiddleware.cs
```

Their responsibilities are:

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

ExceptionHandlingMiddleware
    ↓
Global exception handling
```

---

# 3. ApiResponse<T>

File:

```text
Models/Responses/ApiResponse.cs
```

Implementation:

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

`ApiResponse<T>` is used for successful API responses.

The generic type `T` allows different endpoints to return different types of data.

Examples:

```csharp
ApiResponse<CustomerResponseDto>
ApiResponse<LeadResponseDto>
ApiResponse<OpportunityResponseDto>
ApiResponse<List<CustomerResponseDto>>
```

Example:

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

---

# 4. ErrorResponse

File:

```text
Models/Responses/ErrorResponse.cs
```

Implementation:

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

`ErrorResponse` is used when an exception is handled by the global exception middleware.

Example:

```json
{
  "success": false,
  "message": "Customer not found.",
  "data": null
}
```

The `Data` property is currently typed as `object?`, which allows future error information such as validation errors to be returned if needed.

---

# 5. BaseController

File:

```text
Controllers/BaseController.cs
```

The `BaseController` provides reusable methods for creating standardized successful responses.

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
            return StatusCode(
                StatusCodes.Status201Created,
                new ApiResponse<T>
                {
                    Success = true,
                    Message = message,
                    Data = data
                });
        }

        protected IActionResult Created(string message)
        {
            return StatusCode(
                StatusCodes.Status201Created,
                new ApiResponse<object>
                {
                    Success = true,
                    Message = message,
                    Data = null
                });
        }
    }
}
```

---

# 6. Controller Inheritance

API controllers should inherit from `BaseController`.

Instead of:

```csharp
public class CustomerController : ControllerBase
```

Use:

```csharp
public class CustomerController : BaseController
```

Example:

```csharp
[ApiController]
[Route("api/customers")]
public class CustomerController : BaseController
{
    private readonly ICustomerService _customerService;

    public CustomerController(ICustomerService customerService)
    {
        _customerService = customerService;
    }
}
```

This allows the controller to use:

```csharp
Success(...)
```

and:

```csharp
Created(...)
```

without manually constructing `ApiResponse<T>` every time.

---

# 7. Success() With Data

Use:

```csharp
Success(string message, T data)
```

when the endpoint needs to return data.

Example:

```csharp
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetCustomer(Guid id)
{
    var customer = await _customerService.GetCustomerById(id);

    return Success(
        "Customer retrieved successfully.",
        customer
    );
}
```

Response:

```http
200 OK
```

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

---

# 8. Success() Without Data

Use:

```csharp
Success(string message)
```

when the operation succeeds but does not need to return data.

Example:

```csharp
[HttpDelete("{id:guid}")]
public async Task<IActionResult> DeleteCustomer(Guid id)
{
    await _customerService.DeleteCustomer(id);

    return Success("Customer deleted successfully.");
}
```

Response:

```http
200 OK
```

```json
{
  "success": true,
  "message": "Customer deleted successfully.",
  "data": null
}
```

---

# 9. Created() With Data

Use:

```csharp
Created(string message, T data)
```

when a resource is successfully created and the API needs to return the created resource.

Example:

```csharp
[HttpPost]
public async Task<IActionResult> CreateCustomer(CreateCustomerDto dto)
{
    var customer = await _customerService.CreateCustomer(dto);

    return Created(
        "Customer created successfully.",
        customer
    );
}
```

Response:

```http
201 Created
```

```json
{
  "success": true,
  "message": "Customer created successfully.",
  "data": {
    "id": "8a4d0000-0000-0000-0000-000000000000",
    "name": "ABC Corporation"
  }
}
```

---

# 10. Created() Without Data

Use:

```csharp
Created(string message)
```

when a resource is created successfully but there is no response data to return.

Example:

```csharp
return Created("Customer created successfully.");
```

Response:

```http
201 Created
```

```json
{
  "success": true,
  "message": "Customer created successfully.",
  "data": null
}
```

---

# 11. HTTP Status Codes

The response body and HTTP status code serve different purposes.

The HTTP status code communicates the general result of the HTTP request, while the response body provides a consistent application-level structure.

Current CRMSystem response methods use:

| Status Code                 | Meaning                                              | Current Usage                 |
| --------------------------- | ---------------------------------------------------- | ----------------------------- |
| `200 OK`                    | Request succeeded                                    | `Success()`                   |
| `201 Created`               | Resource was created                                 | `Created()`                   |
| `403 Forbidden`             | Request is authenticated but not permitted           | `UnauthorizedAccessException` |
| `404 Not Found`             | Requested resource was not found                     | `KeyNotFoundException`        |
| `409 Conflict`              | Request conflicts with the current application state | `InvalidOperationException`   |
| `500 Internal Server Error` | Unexpected server error                              | Unhandled exceptions          |

Authentication middleware and validation handling may produce additional HTTP status codes such as `400`, `401`, and `422`.

---

# 12. ExceptionHandlingMiddleware

File:

```text
Middleware/ExceptionHandlingMiddleware.cs
```

The `ExceptionHandlingMiddleware` provides centralized exception handling for the API.

Its main responsibilities are:

1. Execute the next middleware/request pipeline.
2. Catch unhandled exceptions.
3. Log the exception.
4. Determine the appropriate HTTP status code.
5. Return a standardized `ErrorResponse`.

The request flow is:

```text
HTTP Request
     ↓
ExceptionHandlingMiddleware
     ↓
Controller
     ↓
Service
     ↓
Repository
     ↓
Database
```

If an exception occurs:

```text
Database / Repository / Service / Controller
                    ↓
               Exception
                    ↓
       ExceptionHandlingMiddleware
                    ↓
              ErrorResponse
                    ↓
              HTTP Response
```

---

# 13. Exception Status Mapping

The current middleware uses the following mapping:

```csharp
var statusCode = exception switch
{
    KeyNotFoundException =>
        (int)HttpStatusCode.NotFound,

    InvalidOperationException =>
        (int)HttpStatusCode.Conflict,

    UnauthorizedAccessException =>
        (int)HttpStatusCode.Forbidden,

    _ =>
        (int)HttpStatusCode.InternalServerError
};
```

Therefore:

| Exception                     |                 HTTP Status |
| ----------------------------- | --------------------------: |
| `KeyNotFoundException`        |             `404 Not Found` |
| `InvalidOperationException`   |              `409 Conflict` |
| `UnauthorizedAccessException` |             `403 Forbidden` |
| Any other `Exception`         | `500 Internal Server Error` |

---

# 14. KeyNotFoundException

A `KeyNotFoundException` is returned as:

```http
404 Not Found
```

Example:

```csharp
throw new KeyNotFoundException("Customer not found.");
```

Response:

```json
{
  "success": false,
  "message": "Customer not found.",
  "data": null
}
```

This is useful when a requested CRM resource does not exist.

Examples:

```text
Customer not found
Lead not found
Opportunity not found
Pipeline not found
Task not found
```

---

# 15. InvalidOperationException

An `InvalidOperationException` is currently mapped to:

```http
409 Conflict
```

Example:

```csharp
throw new InvalidOperationException(
    "Customer with this email already exists."
);
```

Response:

```json
{
  "success": false,
  "message": "Customer with this email already exists.",
  "data": null
}
```

This can be used when an operation conflicts with the current state of the application.

Examples:

```text
Duplicate customer
Duplicate email
Invalid state transition
Operation conflicts with existing data
```

---

# 16. UnauthorizedAccessException

An `UnauthorizedAccessException` is currently mapped to:

```http
403 Forbidden
```

Example:

```csharp
throw new UnauthorizedAccessException(
    "You do not have permission to perform this action."
);
```

Response:

```json
{
  "success": false,
  "message": "You do not have permission to perform this action.",
  "data": null
}
```

This represents a user who is not permitted to perform the requested operation.

---

# 17. Unexpected Exceptions

Any exception that does not match the explicitly handled exception types is mapped to:

```http
500 Internal Server Error
```

The middleware returns:

```json
{
  "success": false,
  "message": "An unexpected error occurred.",
  "data": null
}
```

For example, if an unexpected database or application exception occurs:

```text
Exception
    ↓
ExceptionHandlingMiddleware
    ↓
Log exception
    ↓
500 Internal Server Error
```

The detailed exception is logged on the server:

```csharp
_logger.LogError(
    ex,
    "An unhandled exception occurred while processing the request."
);
```

The detailed exception is not returned to the client for unexpected errors.

---

# 18. Exception Logging

The middleware logs unhandled exceptions using `ILogger`.

```csharp
_logger.LogError(
    ex,
    "An unhandled exception occurred while processing the request."
);
```

This allows developers to investigate the actual exception through the application's logging system while returning a safe generic message to the client.

The client receives:

```json
{
  "success": false,
  "message": "An unexpected error occurred.",
  "data": null
}
```

while the server log contains the exception details.

---

# 19. Registering the Middleware

The middleware must be registered in `Program.cs`.

Example:

```csharp
var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

The important registration is:

```csharp
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

It should be placed early enough in the pipeline to catch exceptions thrown by downstream middleware, controllers, services, and repositories.

---

# 20. Controller Responsibility

Controllers should primarily handle HTTP-related responsibilities.

A typical flow is:

```text
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

For example:

```csharp
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetCustomer(Guid id)
{
    var customer = await _customerService.GetCustomerById(id);

    return Success(
        "Customer retrieved successfully.",
        customer
    );
}
```

The controller does not need to manually create:

```csharp
new ApiResponse<CustomerResponseDto>
```

because `BaseController` handles that responsibility.

---

# 21. Service Exception Example

The service can throw an appropriate exception when a resource does not exist.

Example:

```csharp
var customer = await _customerRepository.GetByIdAsync(id);

if (customer == null)
{
    throw new KeyNotFoundException("Customer not found.");
}
```

The exception then travels through the request pipeline:

```text
CustomerService
      ↓
KeyNotFoundException
      ↓
ExceptionHandlingMiddleware
      ↓
404 Not Found
```

The controller does not need to manually handle this exception.

---

# 22. Standard Response Examples

## GET

```http
GET /api/customers/{id}
```

```http
200 OK
```

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

---

## POST

```http
POST /api/customers
```

```http
201 Created
```

```json
{
  "success": true,
  "message": "Customer created successfully.",
  "data": {
    "id": "8a4d0000-0000-0000-0000-000000000000",
    "name": "ABC Corporation"
  }
}
```

---

## DELETE

```http
DELETE /api/customers/{id}
```

```http
200 OK
```

```json
{
  "success": true,
  "message": "Customer deleted successfully.",
  "data": null
}
```

---

## Not Found

```http
404 Not Found
```

```json
{
  "success": false,
  "message": "Customer not found.",
  "data": null
}
```

---

## Conflict

```http
409 Conflict
```

```json
{
  "success": false,
  "message": "Customer with this email already exists.",
  "data": null
}
```

---

## Forbidden

```http
403 Forbidden
```

```json
{
  "success": false,
  "message": "You do not have permission to perform this action.",
  "data": null
}
```

---

## Unexpected Error

```http
500 Internal Server Error
```

```json
{
  "success": false,
  "message": "An unexpected error occurred.",
  "data": null
}
```

---

# 23. API Response Flow

The complete response flow is:

```text
                         HTTP Request
                              │
                              ▼
              ExceptionHandlingMiddleware
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
                         │         │
                         │         │
                    Success()   Created()
                         │         │
                         ▼         ▼
                    ApiResponse<T>
                         │
                         ▼
                    HTTP Response
```

When an exception occurs:

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
                         Exception
                              │
                              ▼
              ExceptionHandlingMiddleware
                              │
                     ┌────────┼────────┐
                     │        │        │
                   404       403      409
                     │        │        │
                     └────────┼────────┘
                              │
                     ErrorResponse
                              │
                              ▼
                       HTTP Response
```

Unexpected exceptions follow:

```text
Exception
    ↓
ExceptionHandlingMiddleware
    ↓
Log exception
    ↓
500 Internal Server Error
    ↓
{
    "success": false,
    "message": "An unexpected error occurred.",
    "data": null
}
```

---

# 24. Response Contract

The standard API contract is:

### Successful Response

```text
HTTP 2xx
```

```json
{
  "success": true,
  "message": "...",
  "data": {}
}
```

### Error Response

```text
HTTP 4xx / 5xx
```

```json
{
  "success": false,
  "message": "...",
  "data": null
}
```

---

# 25. Current Exception Mapping

The current CRMSystem implementation follows this mapping:

```text
KeyNotFoundException
        ↓
404 Not Found

InvalidOperationException
        ↓
409 Conflict

UnauthorizedAccessException
        ↓
403 Forbidden

Any other Exception
        ↓
500 Internal Server Error
```

This mapping provides a centralized way to translate application exceptions into HTTP responses.

---

# 26. Design Principles

The API response system follows these principles:

### Consistent response structure

All standard responses use:

```json
{
  "success": true/false,
  "message": "...",
  "data": {}
}
```

### Reusable controller responses

Controllers use:

```csharp
Success(...)
```

and:

```csharp
Created(...)
```

instead of manually constructing response objects.

### Centralized exception handling

Controllers and services do not need to repeat the same `try/catch` logic for every request.

### Server-side exception logging

Unexpected exceptions are logged by the middleware.

### Safe client responses

Unexpected exceptions return a generic message instead of exposing internal exception details.

---

# 27. File Structure

The relevant project structure is:

```text
CRMSystem/
│
├── Controllers/
│   ├── BaseController.cs
│   ├── CustomerController.cs
│   ├── LeadController.cs
│   ├── OpportunityController.cs
│   └── ...
│
├── Middleware/
│   └── ExceptionHandlingMiddleware.cs
│
└── Models/
    └── Responses/
        ├── ApiResponse.cs
        └── ErrorResponse.cs
```

---

# 28. Summary

CRMSystem uses a centralized API response standard consisting of:

```text
ApiResponse<T>
    ↓
Successful responses

ErrorResponse
    ↓
Exception responses

BaseController
    ↓
Success() / Created()

ExceptionHandlingMiddleware
    ↓
Global exception handling
```

The standard response format is:

```json
{
  "success": true,
  "message": "Operation completed successfully.",
  "data": {}
}
```

or:

```json
{
  "success": false,
  "message": "An error occurred.",
  "data": null
}
```

The current global exception mapping is:

| Exception                     | HTTP Status |
| ----------------------------- | ----------: |
| `KeyNotFoundException`        |       `404` |
| `InvalidOperationException`   |       `409` |
| `UnauthorizedAccessException` |       `403` |
| Other exceptions              |       `500` |

This keeps the CRM API predictable for frontend clients while centralizing response construction and unexpected exception handling.

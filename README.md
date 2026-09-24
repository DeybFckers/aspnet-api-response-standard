# API Response Standard

This document defines the standard API response and error-handling architecture used throughout **CRMSystem**.

The goal is to provide a predictable API contract for the frontend while keeping controllers, services, repositories, validation, and exception handling properly separated.

The API follows these principles:

* All successful responses use a consistent `ApiResponse<T>` structure.
* All application errors use a consistent `ErrorResponse` structure.
* Request validation is handled by **FluentValidation**.
* Business and domain validation is handled by the **Service layer**.
* Controllers should not contain repetitive `try/catch` blocks.
* Exceptions are handled globally through `ExceptionHandlingMiddleware`.
* Controllers are responsible for HTTP/API concerns.
* Services are responsible for business logic and business rules.
* Repositories are responsible for database access.
* HTTP status codes must accurately represent the result of the request.
* Organization-scoped resources must remain isolated in the multi-tenant CRM.
* Database-specific exceptions should not normally be exposed directly to API clients.
* Unexpected exceptions should be logged server-side and returned as a safe `500 Internal Server Error`.

---

# 1. Standard Response Format

All API responses should follow the same general structure.

## Success Response

```json
{
  "success": true,
  "message": "Customer retrieved successfully.",
  "data": {}
}
```

## Error Response

```json
{
  "success": false,
  "message": "Customer not found.",
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

# 2. API Response Architecture

The API separates responsibilities into the following layers:

```text
HTTP Request
     │
     ▼
Controller
     │
     ├── Request/HTTP concerns
     │
     ▼
FluentValidation
     │
     ├── Input validation
     │
     ▼
Service
     │
     ├── Business validation
     ├── Authorization/business rules
     ├── Organization/tenant validation
     ├── Entity existence checks
     └── State transition validation
     │
     ▼
Repository
     │
     └── Database access
```

Errors are handled globally:

```text
Controller / Service
        │
        │ throws exception
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

# 3. Important Files

The API response and error-handling architecture uses the following components:

```text
Models/Responses/
├── ApiResponse.cs
└── ErrorResponse.cs

Controllers/
└── BaseController.cs

Exceptions/
├── BadRequestException.cs
├── ForbiddenException.cs
├── NotFoundException.cs
├── ConflictException.cs
└── UnprocessableEntityException.cs

Middleware/
└── ExceptionHandlingMiddleware.cs

Validators/
└── ... FluentValidation validators

Program.cs
```

Responsibilities:

```text
ApiResponse<T>
    ↓
Standard successful response

ErrorResponse
    ↓
Standard application error response

BaseController
    ↓
Reusable success response methods

FluentValidation
    ↓
Request/input validation

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

# 4. ApiResponse.cs

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

Examples:

```text
ApiResponse<CustomerResponseDto>

ApiResponse<OpportunityResponseDto>

ApiResponse<List<CustomerResponseDto>>

ApiResponse<List<OpportunityResponseDto>>
```

---

# 5. ErrorResponse.cs

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

Example:

```json
{
  "success": false,
  "message": "Customer not found.",
  "data": null
}
```

For validation errors, `data` contains field-specific errors:

```json
{
  "success": false,
  "message": "Validation failed.",
  "data": {
    "email": [
      "Email is required.",
      "Email must be a valid email address."
    ],
    "firstName": [
      "First name is required."
    ]
  }
}
```

---

# 6. BaseController.cs

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
        protected IActionResult Success<T>(
            string message,
            T data)
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

        protected IActionResult Created<T>(
            string message,
            T data)
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

# 7. Controllers Should Inherit BaseController

Controllers should inherit from `BaseController`.

## Before

```csharp
public class CustomerController : ControllerBase
```

## After

```csharp
public class CustomerController : BaseController
```

Example:

```csharp
[ApiController]
[Route("api/customers")]
public class CustomerController : BaseController
{
    private readonly ICustomerServices _customerServices;

    public CustomerController(
        ICustomerServices customerServices)
    {
        _customerServices = customerServices;
    }
}
```

---

# 8. Successful Responses

## GET

```csharp
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetCustomerById(Guid id)
{
    var customer =
        await _customerServices.GetCustomerById(id);

    return Success(
        "Customer retrieved successfully.",
        customer
    );
}
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

---

## DELETE

```csharp
[HttpDelete("{id:guid}")]
public async Task<IActionResult> DeleteCustomer(Guid id)
{
    await _customerServices.DeleteCustomer(id);

    return Success(
        "Customer deleted successfully."
    );
}
```

Response:

```json
{
  "success": true,
  "message": "Customer deleted successfully.",
  "data": null
}
```

---

# 9. HTTP Status Codes

The API uses HTTP status codes to communicate the result of the request.

| Status Code                 | Meaning                                              | Typical Usage                              |
| --------------------------- | ---------------------------------------------------- | ------------------------------------------ |
| `200 OK`                    | Request succeeded                                    | GET, PUT, PATCH, DELETE with response body |
| `201 Created`               | Resource created                                     | POST                                       |
| `204 No Content`            | Request succeeded without response body              | Optional DELETE                            |
| `400 Bad Request`           | Request/input is invalid                             | Invalid input                              |
| `401 Unauthorized`          | Authentication is required or failed                 | Missing/invalid JWT                        |
| `403 Forbidden`             | User is authenticated but not permitted              | Insufficient role/permission               |
| `404 Not Found`             | Resource does not exist                              | Customer, Lead, Opportunity not found      |
| `409 Conflict`              | Request conflicts with existing data                 | Duplicate resource                         |
| `422 Unprocessable Entity`  | Request is valid but violates a business/domain rule | Invalid state transition                   |
| `500 Internal Server Error` | Unexpected server error                              | Unhandled exception                        |

---

# 10. Validation vs Business Logic

Validation is divided into two responsibilities.

## Input Validation

Handled by **FluentValidation**.

Examples:

```text
Required fields
String length
Email format
Phone format
Numeric ranges
Date format
Basic request structure
```

Example:

```csharp
RuleFor(x => x.Email)
    .NotEmpty()
    .EmailAddress()
    .MaximumLength(255);
```

---

## Business Validation

Handled by the **Service layer**.

Examples:

```text
Does the customer exist?
Does the customer belong to the current organization?
Does the assigned user exist?
Is the assigned user active?
Does the pipeline stage belong to the pipeline?
Is the opportunity allowed to change state?
Does the customer already exist?
Is the current user allowed to perform the operation?
```

These checks require database access or knowledge of the current application state and should not be forced into simple DTO validation.

---

# 11. Validation Flow

The expected flow is:

```text
Request
   │
   ▼
FluentValidation
   │
   ├── Invalid
   │      │
   │      ▼
   │    400
   │
   ▼
Controller
   │
   ▼
Service
   │
   ├── Business rule violation
   │      │
   │      ▼
   │    Custom Exception
   │
   ▼
Repository
```

This prevents the frontend from receiving confusing database exceptions.

---

# 12. FluentValidation

Validators should be created for DTOs that require input validation.

Example:

```csharp
public class CreateCustomerDtoValidator
    : AbstractValidator<CreateCustomerDto>
{
    public CreateCustomerDtoValidator()
    {
        RuleFor(x => x.FirstName)
            .NotEmpty()
            .MaximumLength(100);

        RuleFor(x => x.LastName)
            .NotEmpty()
            .MaximumLength(100);

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MaximumLength(255);
    }
}
```

Register validators:

```csharp
builder.Services.AddValidatorsFromAssemblyContaining<CreateCustomerDtoValidator>();
```

FluentValidation should focus on **input validation**.

Do not use it to duplicate database-dependent business logic that belongs in the Service layer.

---

# 13. Validation Error Response

When validation fails, the API should return:

```text
400 Bad Request
```

Example:

```json
{
  "success": false,
  "message": "Validation failed.",
  "data": {
    "firstName": [
      "First name is required."
    ],
    "email": [
      "Email must be a valid email address."
    ]
  }
}
```

This allows the frontend to associate errors directly with form fields.

For example:

```text
data.email
    ↓
Email input

data.firstName
    ↓
First Name input
```

The frontend should not have to parse a general exception message to determine which field failed.

---

# 14. Custom Exceptions

Create:

```text
Exceptions/
├── BadRequestException.cs
├── ForbiddenException.cs
├── NotFoundException.cs
├── ConflictException.cs
└── UnprocessableEntityException.cs
```

These exceptions represent application-level errors.

---

# 15. BadRequestException

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

Use when an application-level request is invalid.

Example:

```csharp
throw new BadRequestException(
    "The requested operation is invalid."
);
```

Simple DTO validation should normally be handled by FluentValidation instead.

---

# 16. ForbiddenException

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

Use when the authenticated user is not permitted to perform a business operation.

Example:

```csharp
throw new ForbiddenException(
    "You do not have permission to modify this customer."
);
```

Normal role/permission authorization should still use ASP.NET Core authorization:

```csharp
[Authorize(Roles = "Admin,SalesManager")]
```

---

# 17. NotFoundException

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

Do not use `KeyNotFoundException` for normal API resource-not-found cases.

Use `NotFoundException` so the global middleware can reliably convert it into `404`.

---

# 18. ConflictException

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
if (existingCustomer != null)
{
    throw new ConflictException(
        "A customer with this email already exists."
    );
}
```

Typical uses:

```text
Duplicate customer
Duplicate email
Duplicate pipeline
Duplicate lead source
Duplicate contact
Duplicate organization data
```

---

# 19. UnprocessableEntityException

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

Use when the request is structurally valid but violates a business/domain rule.

Example:

```csharp
if (opportunity.Status == "CLOSED")
{
    throw new UnprocessableEntityException(
        "A closed opportunity cannot be moved back to an open stage."
    );
}
```

Other examples:

```text
Invalid opportunity state transition
Lead cannot be converted
Closed opportunity cannot be modified
Pipeline stage is incompatible with the pipeline
Business operation is not currently allowed
```

---

# 20. Unauthorized Requests

Authentication should normally be handled by ASP.NET Core authentication middleware.

For example:

```csharp
[Authorize]
[HttpGet]
public async Task<IActionResult> GetCustomers()
{
    // ...
}
```

ASP.NET Core should handle:

```text
Missing JWT
Invalid JWT
Expired JWT
Invalid authentication credentials
```

These should normally result in:

```text
401 Unauthorized
```

The application should not manually throw an exception for every missing or invalid JWT.

If the application has a specific application-level authentication failure that must be represented by the service layer, it may use an appropriate application exception, but normal JWT authentication should remain the responsibility of ASP.NET Core.

---

# 21. Exception-to-HTTP Mapping

The middleware should map application exceptions as follows:

| Exception                      | HTTP Status |
| ------------------------------ | ----------: |
| `BadRequestException`          |       `400` |
| `ForbiddenException`           |       `403` |
| `NotFoundException`            |       `404` |
| `ConflictException`            |       `409` |
| `UnprocessableEntityException` |       `422` |
| Unknown `Exception`            |       `500` |

Authentication failures from ASP.NET Core authentication middleware are normally handled separately as:

```text
401 Unauthorized
```

---

# 22. ExceptionHandlingMiddleware

The middleware is responsible for converting exceptions into standardized API responses.

Example structure:

```csharp
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
                    StatusCodes.Status400BadRequest,

                ForbiddenException =>
                    StatusCodes.Status403Forbidden,

                NotFoundException =>
                    StatusCodes.Status404NotFound,

                ConflictException =>
                    StatusCodes.Status409Conflict,

                UnprocessableEntityException =>
                    StatusCodes.Status422UnprocessableEntity,

                _ =>
                    StatusCodes.Status500InternalServerError
            };

            var message = exception switch
            {
                BadRequestException =>
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

# 23. Unexpected Exceptions

Unknown exceptions should never normally expose internal implementation details to the frontend.

Avoid returning:

```json
{
  "success": false,
  "message": "Npgsql.PostgresException: 23505..."
}
```

Instead:

```json
{
  "success": false,
  "message": "An unexpected error occurred.",
  "data": null
}
```

The actual exception should be logged server-side:

```csharp
_logger.LogError(
    ex,
    "An unhandled exception occurred while processing the request."
);
```

This protects sensitive implementation details while still allowing developers to diagnose the problem through server logs.

---

# 24. Controller Responsibility

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

Controllers should generally **not** contain:

```csharp
try
{
    // service call
}
catch (...)
{
    // convert exception
}
```

The exception middleware handles this globally.

---

# 25. Service Responsibility

Services are responsible for business logic and business validation.

Example:

```csharp
public async Task<CustomerResponseDto> GetCustomerById(
    Guid customerId)
{
    var organizationId =
        _currentUserServices.OrganizationId;

    var customer =
        await _customerRepository.GetByIdAsync(
            customerId,
            organizationId
        );

    if (customer == null)
    {
        throw new NotFoundException(
            "Customer not found."
        );
    }

    return customer.Adapt<CustomerResponseDto>();
}
```

Services may throw:

```text
BadRequestException
ForbiddenException
NotFoundException
ConflictException
UnprocessableEntityException
```

Services should **not** return:

```csharp
IActionResult
```

or HTTP-specific responses.

---

# 26. Multi-Tenant Resource Isolation

CRMSystem is a multi-tenant application.

Resources belonging to one organization must not be accessible by another organization.

Service and repository queries should enforce organization scope.

Example:

```csharp
var customer =
    await _customerRepository.GetByIdAsync(
        customerId,
        organizationId
    );
```

The application should not simply retrieve a resource by ID and assume the resource belongs to the current organization.

If a resource cannot be found within the current organization scope, the API should normally return:

```text
404 Not Found
```

This prevents exposing information about resources belonging to other organizations.

---

# 27. Business Logic Examples

Business rules should remain in the Service layer.

## Example: User Assignment

```csharp
var user =
    await _userRepository.GetByIdAsync(
        dto.AssignedUserId
    );

if (user == null)
{
    throw new NotFoundException(
        "Assigned user not found."
    );
}

if (user.OrganizationId != organizationId)
{
    throw new ForbiddenException(
        "The assigned user does not belong to your organization."
    );
}
```

---

## Example: Duplicate Customer

```csharp
var existingCustomer =
    await _customerRepository.GetByEmailAsync(
        dto.Email,
        organizationId
    );

if (existingCustomer != null)
{
    throw new ConflictException(
        "A customer with this email already exists."
    );
}
```

---

## Example: Invalid State Transition

```csharp
if (opportunity.Status == "CLOSED")
{
    throw new UnprocessableEntityException(
        "A closed opportunity cannot be moved back to an open state."
    );
}
```

---

# 28. HTTP Response Examples

## 200 OK

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

## 201 Created

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

## 400 Bad Request

```json
{
  "success": false,
  "message": "Validation failed.",
  "data": {
    "firstName": [
      "First name is required."
    ]
  }
}
```

---

## 403 Forbidden

```json
{
  "success": false,
  "message": "You do not have permission to modify this customer.",
  "data": null
}
```

---

## 404 Not Found

```json
{
  "success": false,
  "message": "Customer not found.",
  "data": null
}
```

---

## 409 Conflict

```json
{
  "success": false,
  "message": "A customer with this email already exists.",
  "data": null
}
```

---

## 422 Unprocessable Entity

```json
{
  "success": false,
  "message": "A closed opportunity cannot be moved back to an open state.",
  "data": null
}
```

---

## 500 Internal Server Error

```json
{
  "success": false,
  "message": "An unexpected error occurred.",
  "data": null
}
```

---

# 29. Recommended API Error Contract

The frontend can use the HTTP status code first and the response body second.

```text
HTTP Status
     │
     ├── 400 → Input/validation problem
     ├── 401 → Authentication problem
     ├── 403 → Permission problem
     ├── 404 → Resource not found
     ├── 409 → Duplicate/conflict
     ├── 422 → Business/domain rule
     └── 500 → Unexpected server error
```

This allows the frontend to handle errors predictably.

For example:

```text
400
  ↓
Show field validation messages

401
  ↓
Refresh/login

403
  ↓
Show permission message

404
  ↓
Show resource-not-found message

409
  ↓
Show duplicate/conflict message

422
  ↓
Show business-rule message

500
  ↓
Show generic server-error message
```

---

# 30. Implementation Checklist

When implementing a new CRM feature:

## Validation

* [ ] Create a FluentValidation validator for DTO input rules.
* [ ] Validate required fields.
* [ ] Validate string lengths.
* [ ] Validate formats.
* [ ] Validate numeric ranges.
* [ ] Validate basic request structure.
* [ ] Do not duplicate Service business rules in the validator.

## Business Logic

* [ ] Check entity existence.
* [ ] Check organization ownership.
* [ ] Check related entity ownership.
* [ ] Check user assignment rules.
* [ ] Check authorization/business permissions.
* [ ] Check duplicate records.
* [ ] Check state transitions.
* [ ] Check domain-specific rules.
* [ ] Throw the appropriate custom exception.

## Controllers

* [ ] Inherit from `BaseController`.
* [ ] Do not add repetitive `try/catch` blocks.
* [ ] Return appropriate success responses.
* [ ] Keep business logic out of controllers.

## Services

* [ ] Keep business logic in services.
* [ ] Perform organization-scoped queries.
* [ ] Throw custom application exceptions.
* [ ] Do not return `IActionResult`.

## Repositories

* [ ] Handle database access.
* [ ] Support organization-scoped queries where required.
* [ ] Do not expose database-specific exceptions directly to the frontend.

## Exceptions

* [ ] `BadRequestException` → `400`
* [ ] `ForbiddenException` → `403`
* [ ] `NotFoundException` → `404`
* [ ] `ConflictException` → `409`
* [ ] `UnprocessableEntityException` → `422`
* [ ] Unknown exception → `500`

## Security

* [ ] Do not expose stack traces.
* [ ] Do not expose database exception details.
* [ ] Do not expose sensitive information.
* [ ] Keep organization resources isolated.
* [ ] Use `[Authorize]` and policies/roles where appropriate.
* [ ] Log unexpected exceptions server-side.

---

# 31. Final Architecture

CRMSystem follows this responsibility model:

```text
┌─────────────────────────────────────┐
│              Request                │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│            Controller               │
│        HTTP/API concerns            │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│         FluentValidation            │
│          Input validation            │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│              Service                │
│         Business logic              │
│         Business validation         │
│         Authorization rules         │
│         Organization scope          │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│             Repository              │
│           Database access           │
└──────────────────┬──────────────────┘
                   │
                   ▼
               Database
```

For errors:

```text
Service / Application
        │
        ▼
 Custom Exception
        │
        ▼
ExceptionHandlingMiddleware
        │
        ├── 400
        ├── 403
        ├── 404
        ├── 409
        ├── 422
        └── 500
        │
        ▼
   ErrorResponse
        │
        ▼
      Frontend
```

The final API contract is:

### Success

```json
{
  "success": true,
  "message": "Operation completed successfully.",
  "data": {}
}
```

### Application Error

```json
{
  "success": false,
  "message": "Operation failed.",
  "data": null
}
```

### Validation Error

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

This standard should be applied consistently across all CRMSystem modules:

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

The objective is to maintain a **predictable frontend API contract** while keeping:

```text
Controllers
    → HTTP concerns

FluentValidation
    → Input validation

Services
    → Business logic and business rules

Repositories
    → Database access

ExceptionHandlingMiddleware
    → Global error handling
```

This keeps the CRMSystem backend maintainable, testable, and easier to integrate with the React frontend and future automation workflows.

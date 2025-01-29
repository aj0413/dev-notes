# REST API Coding Guidelines

An opinionated guidline of basic coding principles and standards to follow when creating a REST API service.

## Status Codes

Standardized HTTP status codes to return for API endpoints.

This will only cover those generally relied upon the developer to implement, i.e., this will not cover codes like `405 METHOD NOT ALLOWED` or `502 BAD GATEWAY`

#### 2xx Success

- `200 OK` : Basic successfully processed request.
- `201 CREATED` : Successfully created resource.
  - Generally expected to return representation of the created resource or a way to obtain details about it.
- `202 ACCEPTED` : Accepted request for processing; does not indicate status of job.
  - Normally you want return a way, either through body of response or header, for caller to check status of job.
- `204 NO CONTENT` : Successfully processed request, *but* nothing to return.
  - This would normally be used for something like deleting a resource, assuming you don't want to return the deleted resource in the response.

#### 4xx Bad Request

- `400 BAD REQUEST` : Basic failed to process request due to caller error.
- `401 UNAUTHORIZED` : Failed to process request due to invalid authentication credentials provided by caller.
- `403 FORBIDDEN` :  Failed to process request due to lack of privileges of caller.
- `404 NOT FOUND` : Failed to process request because resource or endpoint does not exist.
- `409 CONFLICT` : Failed to process request because it conflicts with an existing resource.
- `422 UNPROCESSABLE CONTENT` : Can't process request, but server understands it, i.e. the instructions/data is bad.

### Error Response

The error response for an HTTP API SHOULD be ContentType 'application/problem+json' and match the [Problem Details](https://www.rfc-editor.org/rfc/rfc9457) specification.

MSFT out-of-the-box model validation for ASP.NET Core returns a [ValidationProblemDetails](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.validationproblemdetails) object matching this specification, for instance. This is object returned when using DataAnnotations such as [Required](https://learn.microsoft.com/en-us/dotnet/api/system.componentmodel.dataannotations.requiredattribute) on c# objects that you bind to for requests. See [Validation failure error response](https://learn.microsoft.com/en-us/aspnet/core/web-api/handle-errors?view=aspnetcore-8.0#validation-failure-error-response) MSDN

Another example would be [FastEndpoints(FE)](https://fast-endpoints.com/) own implementation of the [ProblemDetails](https://github.com/FastEndpoints/FastEndpoints/blob/1c5db30c632abf53280a0a8ca1ea5e2f5b57bc53/Src/Library/DTOs/ProblemDetails.cs) class. We're currently refactoring a subset of our RESTful microservices to uses this library as an opinionated framework to help guide devs towards best practices. See [RFC7807 & RFC9457 Compatible Problem Details](https://fast-endpoints.com/docs/configuration-settings#rfc7807-rfc9457-compatible-problem-details) docs for FE

```
{
  "type": "https://www.rfc-editor.org/rfc/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "instance": "/api/route",
  "traceId": "0HMPNHL0JHL76:00000001",
  "detail": "string",
  "errors": [
    {
      "name": "Error or field name",
      "reason": "Error reason",
      "code": "string",
      "severity": "string"
    }
  ]
}
```

## Methods

- `POST` : Create a resource or kick off an action.
- `GET` : Request a resource or collection of resources.
  - Generally implemented with pagination, limits, and basic filters.
- `PUT` : Update/replace a resource.
  - Requires sending the entire resource representation in the request.
  - Could optionally create the resource, if not already created.
- `PATCH` : Update/modify a resource.
  - Accepts information to modify only part of a resource.
- `DELETE` : Deletes a resource.

**NOTE: `PUT`, `GET`, and `DELETE` methods should always be [idempotent](#what-is-idempotency). Consider the difference before implementing `PATCH` vs `PUT`. You SHOULD prefer idempotent methods where possible.**

## Endpoint Naming Coventions

Endpoints should always be the name of the resource, when dealing with it as a collection, or the name followed by `/{identifier}` when interacting with a specific one.

If you need to access the child of a resource, you *can* do the following `resource/{identifier}/resource/{identifier}` to interact with child resources of a specific parent resource. However, DO NOT ever create a URI going more than two resources deep, in the hierarchy tree, i.e. avoid `resource/{identifier}resource/{identifier}/resource`.

Additionally, URIs should use a plural, noun naming convention to refer to the resource.

Examples:
```
POST /accounts
GET  /accounts
GET  /accounts/{id}/users
GET  /users/{id}
```

**NOTE: The *type* of action you want to take should be indicated by the HTTP method used in the call; this is enough for CRUD operations. Avoid verbose and highly specific URIs, ie. use /clients/{id}/doSomething**

## Example RESTful endpoints

| HTTP Methods | Operation      | URI                                   | Description                                 | Basic Expected Responses                          |
|:-------------|:---------------|:--------------------------------------|:--------------------------------------------|:--------------------------------------------------|
| POST         | Create         | /clients                              | Creates a client                            | 201 (Created) with resource                       |
| GET          | Read           | /clients?page={page}&limit={limit}    | Return list of clients                      | 200 (Ok) with collection of resources             |
| GET          | Read           | /cients/{id}                          | Return specific client                      | 200 (Ok) with specified resource, 404 (Not Found) |
| GET          | Read           | /cients/{id}/documents                | Return specific client's documents          | 200 (Ok) with specified resources, 404 (Not Found)|
| PUT          | Update/Replace | /cients/{id}                          | Replace entire client resource              | 200 (Ok) with replaced resource, 404 (Not Found)  |
| PATCH        | Update/Modify  | /cients/{id}                          | Update part of client resource              | 200 (Ok) with updated resource, 404 (Not Found)   |
| DELETE       | Delete         | /cients/{id}                          | Delete client resource                      | 204 (No Content), 404 (Not Found)                 |
| GET          | Read           | /documents                            | Return list of documents                    | 200 (Ok) with specified resource                  |
| GET          | Read           | /documents/{id}                       | Return specific document                    | 200 (Ok) with specified resource, 404 (Not Found) |
| GET          | Read           | /documents/{id}/clients               | Return list of clients tied to a document   | 200 (Ok) with specified resource, 404 (Not Found) |

The above is only an example chart and does not cover all scenarios, i.e., creating/updating a client resource resulting in a 409 (Conflict) with an existing resource. 

You can see 404 (Not Found) can be returned for a number of endpoints when using a resource identifier since the resource may not exist.

## Query Parameters

Query parameters come in the form of `?param1=val1&param2=val2`, are most often used for GET requests, and SHOULD NOT be required.

For example, parameters on a GET request may help filter items based on a start and end date, so you should identify good defaults based on common business case.

Before ever marking a query parameter as required, you should first consider if the endpoint itself should be re-thought or if there exists a reasonable default that can be used.

## What is Idempotency?

> GET, HEAD, PUT, DELETE, OPTIONS, and TRACE are idempotent methods, meaning they are safe to be retried or executed multiple times without causing unintended side effects. In contrast, POST and PATCH are generally considered non-idempotent, as their outcomes may vary with each request

Defined by [RFC 9110 - HTTP Semantics (ietf.org) (specifically section 9.2.2-3)](https://datatracker.ietf.org/doc/html/rfc9110#section-9.2.2-3)

> Idempotent methods are distinguished because the request can be repeated automatically if a communication failure occurs before the client is able to read the server's response. For example, if a client sends a PUT request and the underlying connection is closed before any response is received, then the client can establish a new connection and retry the idempotent request. It knows that **repeating the request will have the same intended effect, even if the original request succeeded, though the response might differ.**

Further

> A client SHOULD NOT automatically retry a request with a non-idempotent method unless it has some means to know that the request semantics are actually idempotent, regardless of the method, or some means to detect that the original request was never applied. 

## Helpful Links`

- [Wikipedia on status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)
- [Summarized chart for HTTP methods and their responses](https://www.restapitutorial.com/lessons/httpmethods.html)
- [What is idempotency?](https://blog.dreamfactory.com/what-is-idempotency/#:~:text=performed%20only%20once.-,GET%2C%20HEAD%2C%20PUT%2C%20DELETE%2C%20OPTIONS%2C%20and%20TRACE,may%20vary%20with%20each%20request.)
- [Returning HTTP errors per RFC9457 (obseletes 7807)](https://www.rfc-editor.org/rfc/rfc9457)
- [Returning HTTP errors per RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807)
- [API URL Best Practices](https://apidog.com/blog/rest-api-url-best-practices-examples/)
- [What is the composition of an API URL?](https://www.numpyninja.com/post/what-is-the-composition-of-the-url-in-rest-api)
- [Why query parameters should not be required](https://apihandyman.io/what-s-the-problem-with-required-query-parameters/)
- [RESTful HTTP Methods](https://apihandyman.io/what-s-the-problem-with-required-query-parameters/)
- [RFC 9110 - HTTP Semantics (ietf.org)](https://datatracker.ietf.org/doc/html/rfc9110)
- [FastEndpoints(FE)](https://fast-endpoints.com/)
- [API Design Guide](https://apiguide.readthedocs.io/en/latest/build_and_publish/use_RESTful_urls.html)

## Throwing C# Exceptions

In most cases you won't need to be concerned with ServiceStack's error handling since it provides native support for the normal use-case of throwing C# Exceptions, e.g.:

```csharp 
public object Post(User request) 
{
    if (string.IsNullOrEmpty(request.Name))
        throw new ArgumentNullException("Name");
}
```

### Default Mapping of C# Exceptions to HTTP Errors

By Default C# Exceptions:

  - Inheriting from `ArgumentException` are returned with a HTTP StatusCode of **400 BadRequest**
  - `NotImplementedException` or `NotSupportedException ` is returned as a **405 MethodNotAllowed** 
  - `AuthenticationException` is returned as **401 Unauthorized**
  - `UnauthorizedAccessException` is returned as **403 Forbidden**
  - `OptimisticConcurrencyException` is returned as **409 Conflict**
  - Other normal C# Exceptions are returned as **500 InternalServerError**

This list can be extended with user-defined mappings on `Config.MapExceptionToStatusCode`.

### WebServiceException 

All Exceptions get injected into the `ResponseStatus` property of your Response DTO that is serialized into your ServiceClient's preferred Content-Type making error handling transparent regardless of your preferred format - i.e., the same C# Error handling code can be used for all ServiceClients.

```csharp 
try 
{
    var client = new JsonServiceClient(BaseUri);
    var response = client.Send<UserResponse>(new User());
} 
catch (WebServiceException webEx) 
{
    /*
      webEx.StatusCode  = 400
      webEx.ErrorCode   = ArgumentNullException
      webEx.Message     = Value cannot be null. Parameter name: Name
      webEx.StackTrace  = (your Server Exception StackTrace - in DebugMode)
      webEx.ResponseDto = (your populated Response DTO)
      webEx.ResponseStatus   = (your populated Response Status DTO)
      webEx.GetFieldErrors() = (individual errors for each field if any)
    */
}
```

### Enabling StackTraces

By default displaying StackTraces in Response DTOs are only enabled in Debug builds, although this behavior is overridable with:

```csharp
SetConfig(new HostConfig { DebugMode = true });
```

### Error Response Types

The Error Response that gets returned when an Exception is thrown varies on whether a conventionally-named `{RequestDto}Response` DTO exists or not. 

#### If it exists:
The `{RequestDto}Response` is returned, regardless of the service method's response type. If the `{RequestDto}Response` DTO has a **ResponseStatus** property, it is populated otherwise no **ResponseStatus** will be returned.  (If you have decorated the `{ResponseDto}Response` class and properties with `[DataContract]/[DataMember]` attributes, then **ResponseStatus** also needs to be decorated, to get populated).

#### Otherwise, if it doesn't:
A generic `ErrorResponse` gets returned with a populated **ResponseStatus** property.

The [Service Clients](https://github.com/ServiceStack/ServiceStack/wiki/Clients-overview) transparently handles the different Error Response types, and for schema-less formats like JSON/JSV/etc there's no actual visible difference between returning a **ResponseStatus** in a custom or generic `ErrorResponse` - as they both output the same response on the wire.

## Custom Exceptions

Ultimately all ServiceStack WebServiceExceptions are just Response DTO's with a populated [ResponseStatus](https://github.com/ServiceStack/ServiceStack/blob/master/src/ServiceStack.Interfaces/ResponseStatus.cs) that are returned with a HTTP Error Status. There are a number of different ways to customize how Exceptions are returned including:

### Custom mapping of C# Exceptions to HTTP Error Status

You can change what HTTP Error Status is returned for different Exception Types by configuring them with:

```csharp 
SetConfig(new HostConfig { 
    MapExceptionToStatusCode = {
        { typeof(CustomInvalidRoleException), 403 },
        { typeof(CustomerNotFoundException), 404 },
    }
});
```

### Returning a HttpError

If you want even finer grained control of your HTTP errors you can either **throw** or **return** an **HttpError** letting you customize the **Http Headers** and **Status Code** and HTTP Response **body** to get exactly what you want on the wire:

```csharp 
public object Get(User request) 
{
    throw HttpError.NotFound("User {0} does not exist".Fmt(request.Name));
}
```

The above returns a **404** NotFound StatusCode on the wire and is a short-hand for: 

```csharp 
new HttpError(HttpStatusCode.NotFound, 
    "User {0} does not exist".Fmt(request.Name)); 
```

### HttpError with a Custom Response DTO

The `HttpError` can also be used to return a more structured Error Response with:

```csharp
var responseDto = new ErrorResponse { 
    ResponseStatus = new ResponseStatus {
        ErrorCode = typeof(ArgumentException).Name,
        Message = "Invalid Request",
        Errors = new List<ResponseError> {
            new ResponseError {
                ErrorCode = "NotEmpty",
                FieldName = "Company",
                Message = "'Company' should not be empty."
            }
        }
    }
};

throw new HttpError(HttpStatusCode.BadRequest, "ArgumentException") {
    Response = responseDto,
}; 
```

### Implementing [IResponseStatusConvertible](https://github.com/ServiceStack/ServiceStack/blob/773aac107fc2e0fe6a823acef0c3bad20f686da0/src/ServiceStack.Interfaces/Model/IResponseStatusConvertible.cs#L9)

You can also override the serialization of Custom Exceptions by implementing the `IResponseStatusConvertible` interface to return your own populated ResponseStatus instead. This is how `ValidationException` allows customizing the Response DTO is by [having ValidationException implement][1] the [IResponseStatusConvertible][2] interface. 

E.g. Here's a custom Exception example that returns a populated Field Error:  

```csharp
public class CustomFieldException : Exception, IResponseStatusConvertible
{
  ...
    public string FieldErrorCode { get; set; }
    public string FieldName { get; set; }
    public string FieldMessage { get; set; }

    public ResponseStatus ToResponseStatus()
    {
        return new ResponseStatus {
            ErrorCode = GetType().Name,
            Message = Message,
            Errors = new List<ResponseError> {
                new ResponseError {
                    ErrorCode = FieldErrorCode,
                    FieldName = FieldName,
                    Message = FieldMessage
                }
            }
        }
    }    
}
```

### Implementing [IHasStatusCode](https://github.com/ServiceStack/ServiceStack/blob/42c93be28091e3023a1e9720eb5601d4c4fa01a0/src/ServiceStack.Interfaces/Model/IResponseStatusConvertible.cs#L13)

In addition to customizing the HTTP Response Body of C# Exceptions with 
[IResponseStatusConvertible](https://github.com/ServiceStack/ServiceStack/wiki/Error-Handling#implementing-iresponsestatusconvertible), 
you can also customize the HTTP Status Code by implementing `IHasStatusCode`:

```csharp
public class Custom401Exception : Exception, IHasStatusCode
{
    public int StatusCode 
    { 
        get { return 401; } 
    }
}
```

### Overriding OnExceptionTypeFilter in your AppHost

You can also catch and modify the returned `ResponseStatus` returned by overriding `OnExceptionTypeFilter` in your AppHost, e.g. [ServiceStack uses this](https://github.com/ServiceStack/ServiceStack/blob/master/src/ServiceStack/ServiceStackHost.Runtime.cs#L310) to customize the returned ResponseStatus to automatically add a custom field error for `ArgumentExceptions` with the specified `ParamName`, e.g:

```csharp
public virtual void OnExceptionTypeFilter(
    Exception ex, ResponseStatus responseStatus)
{
    var argEx = ex as ArgumentException;
    var isValidationSummaryEx = argEx is ValidationException;
    if (argEx != null && !isValidationSummaryEx && argEx.ParamName != null)
    {
        var paramMsgIndex = argEx.Message.LastIndexOf("Parameter name:");
        var errorMsg = paramMsgIndex > 0
            ? argEx.Message.Substring(0, paramMsgIndex)
            : argEx.Message;

        responseStatus.Errors.Add(new ResponseError
        {
            ErrorCode = ex.GetType().Name,
            FieldName = argEx.ParamName,
            Message = errorMsg,
        });
    }
}
```

### Custom HTTP Errors

In Any Request or Response Filter you can short-circuit the [Request Pipeline](https://github.com/ServiceStack/ServiceStack/wiki/Order-of-Operations) by emitting a Custom HTTP Response and Ending the request, e.g:

```csharp
this.PreRequestFilters.Add((req,res) => 
{
    if (req.PathInfo.StartsWith("/admin") && 
        !req.GetSession().HasRole("Admin")) 
    {
        res.StatusCode = (int)HttpStatusCode.Forbidden;
        res.StatusDescription = "Requires Admin Role";
        res.EndRequest();
    }
});
```

> To end the Request in a  Custom HttpHandler use `res.EndHttpHandlerRequest()`

### Fallback Error Pages

Use `IAppHost.GlobalHtmlErrorHttpHandler` for specifying a **fallback HttpHandler** for all error status codes, e.g.:

```csharp 
public override void Configure(Container container)
{
    this.GlobalHtmlErrorHttpHandler = new RazorHandler("/oops"),
}
```

For more fine-grained control, use `IAppHost.CustomHttpHandlers` for specifying custom HttpHandlers to use with specific error status codes, e.g.:

```csharp 
public override void Configure(Container container)
{
    this.CustomHttpHandlers[HttpStatusCode.NotFound] = 
        new RazorHandler("/notfound");
    this.CustomHttpHandlers[HttpStatusCode.Unauthorized] = 
        new RazorHandler("/login");
}
```

## Register handlers for handling Service and non-Service Exceptions

ServiceStack and its [[new API]] provides a flexible way to intercept exceptions. If you need a single entry point for all service exceptions, you can add a handler to `AppHost.ServiceExceptionHandler` in `Configure`. To handle exceptions occurring outside of services you can set the global `AppHost.UncaughtExceptionHandlers` handler, e.g.:

```csharp
public override void Configure(Container container)
{
    //Handle Exceptions occurring in Services:

    this.ServiceExceptionHandlers.Add((httpReq, request, exception) => {
        //log your exceptions here
        ...
        //call default exception handler or prepare your own custom response
        return DtoUtils.CreateErrorResponse(request, exception);
    });

    //Handle Unhandled Exceptions occurring outside of Services
    //E.g. Exceptions during Request binding or in filters:
    this.UncaughtExceptionHandlers.Add((req, res, operationName, ex) => {
         res.Write("Error: {0}: {1}".Fmt(ex.GetType().Name, ex.Message));
         res.EndRequest(skipHeaders: true);
    });
}
```

### Error handling using a custom ServiceRunner

If you want to provide different error handlers for different actions and services you can just tell ServiceStack to run your services in your own custom [IServiceRunner](https://github.com/ServiceStack/ServiceStack/blob/master/src/ServiceStack.Interfaces/Web/IServiceRunner.cs) and implement the **HandleExcepion** event hook in your AppHost:

```csharp
public override IServiceRunner<TRequest> CreateServiceRunner<TRequest>(
    ActionContext actionContext)
{           
    return new MyServiceRunner<TRequest>(this, actionContext); 
}
```

Where **MyServiceRunner** is just a custom class implementing the custom hooks you're interested in, e.g.:

```csharp
public class MyServiceRunner<T> : ServiceRunner<T> 
{
    public MyServiceRunner(IAppHost appHost, ActionContext actionContext) 
        : base(appHost, actionContext) {}

    public override object HandleException(IRequest request, 
        T request, Exception ex) {
      // Called whenever an exception is thrown in your Services Action
    }
}
```

# Community Resources

  - [ServiceStack Exceptions and Errors](http://www.philliphaydon.com/2012/03/09/service-stack-exceptions-and-errors/) by [@philliphaydon](https://twitter.com/philliphaydon)
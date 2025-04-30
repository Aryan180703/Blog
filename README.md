using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using System;
using System.IO;
using System.Text;
using System.Threading.Tasks;

namespace EShoppingZone.Middlewares
{
    public class RequestLoggingMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<RequestLoggingMiddleware> _logger;

        public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
        {
            _next = next;
            _logger = logger;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            try
            {
                // Log Request
                var request = await FormatRequest(context.Request);
                _logger.LogInformation("Incoming Request: {Method} {Path} {RequestBody}", 
                    context.Request.Method, context.Request.Path, request);

                // Capture Response
                var originalBodyStream = context.Response.Body;
                var responseBody = new MemoryStream(); // No 'using' to avoid premature dispose
                context.Response.Body = responseBody;

                try
                {
                    // Call next middleware
                    await _next(context);

                    // Log Response
                    var response = await FormatResponse(context.Response);
                    _logger.LogInformation("Outgoing Response: {StatusCode} {ResponseBody}", 
                        context.Response.StatusCode, response);

                    // Copy response to original stream
                    if (responseBody.CanSeek)
                    {
                        responseBody.Seek(0, SeekOrigin.Begin);
                        await responseBody.CopyToAsync(originalBodyStream);
                    }
                }
                finally
                {
                    // Restore original body and dispose responseBody
                    context.Response.Body = originalBodyStream;
                    responseBody.Dispose(); // Explicitly dispose
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in RequestLoggingMiddleware: {Message}", ex.Message);
                throw; // Let ExceptionMiddleware handle
            }
        }

        private async Task<string> FormatRequest(HttpRequest request)
        {
            try
            {
                if (!request.Body.CanRead)
                {
                    return $"{{ Query: {request.QueryString}, Body: <empty> }}";
                }

                request.EnableBuffering();
                using var reader = new StreamReader(
                    request.Body,
                    encoding: Encoding.UTF8,
                    detectEncodingFromByteOrderMarks: false,
                    leaveOpen: true);
                var body = await reader.ReadToEndAsync();
                request.Body.Seek(0, SeekOrigin.Begin);
                return $"{{ Query: {request.QueryString}, Body: {body} }}";
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Failed to read request body: {Message}", ex.Message);
                return $"{{ Query: {request.QueryString}, Body: <unreadable> }}";
            }
        }

        private async Task<string> FormatResponse(HttpResponse response)
        {
            try
            {
                if (!response.Body.CanRead || !response.Body.CanSeek)
                {
                    return "<empty>";
                }

                response.Body.Seek(0, SeekOrigin.Begin);
                using var reader = new StreamReader(response.Body, leaveOpen: true);
                var body = await reader.ReadToEndAsync();
                response.Body.Seek(0, SeekOrigin.Begin);
                return body.Length > 0 ? body : "{}";
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Failed to read response body: {Message}", ex.Message);
                return "<unreadable>";
            }
        }
    }
}




using System;
using System.Net;
using System.Text.Json;
using System.Threading.Tasks;
using EShoppingZone.DTOs;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;

namespace EShoppingZone.Middlewares
{
    public class ExceptionMiddleware
    {
        private readonly RequestDelegate _next;
        private readonly ILogger<ExceptionMiddleware> _logger;

        public ExceptionMiddleware(RequestDelegate next, ILogger<ExceptionMiddleware> logger)
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
                _logger.LogError(ex, "Unhandled exception: {Message} at {Path}", ex.Message, context.Request.Path);
                await HandleExceptionAsync(context, ex);
            }
        }

        private async Task HandleExceptionAsync(HttpContext context, Exception exception)
        {
            HttpStatusCode statusCode;
            string message;

            try
            {
                switch (exception)
                {
                    case ObjectDisposedException ode:
                        statusCode = HttpStatusCode.InternalServerError;
                        message = "Stream access error: " + ode.Message;
                        break;
                    case ArgumentNullException ane:
                        statusCode = HttpStatusCode.BadRequest;
                        message = ane.Message ?? "A required value was null.";
                        break;
                    case KeyNotFoundException knfe:
                        statusCode = HttpStatusCode.NotFound;
                        message = knfe.Message ?? "Resource not found.";
                        break;
                    case UnauthorizedAccessException uae:
                        statusCode = HttpStatusCode.Unauthorized;
                        message = uae.Message ?? "Unauthorized access.";
                        break;
                    case InvalidOperationException ioe:
                        statusCode = HttpStatusCode.BadRequest;
                        message = ioe.Message ?? "Invalid operation.";
                        break;
                    default:
                        statusCode = HttpStatusCode.InternalServerError;
                        message = "An unexpected error occurred.";
                        _logger.LogError(exception, "Detailed error: {Message}", exception.ToString());
                        break;
                }

                var response = new ResponseDTO<object>
                {
                    Success = false,
                    Message = message,
                    Data = null
                };

                context.Response.ContentType = "application/json";
                context.Response.StatusCode = (int)statusCode;

                // Ensure response body is writable
                if (!context.Response.HasStarted)
                {
                    var result = JsonSerializer.Serialize(response, new JsonSerializerOptions
                    {
                        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
                        WriteIndented = true
                    });
                    await context.Response.WriteAsync(result);
                }
                else
                {
                    _logger.LogWarning("Response already started, cannot write error response.");
                }
            }
            catch (Exception serializationEx)
            {
                _logger.LogCritical(serializationEx, "Serialization error in exception handling: {Message}", serializationEx.Message);
                if (!context.Response.HasStarted)
                {
                    context.Response.ContentType = "application/json";
                    context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
                    await context.Response.WriteAsync("{\"success\":false,\"message\":\"Critical error in exception handling.\"}");
                }
            }
        }
    }
}

# Minimal APIs

---

### 1. 🟢 Your team is starting a new microservice with 5–6 endpoints. One developer wants controllers, another prefers minimal APIs. How do you decide, and what are the trade-offs?

For a small, focused microservice with a handful of endpoints, minimal APIs are usually the better fit. They eliminate the ceremony of controller classes, attribute routing, and model-binding conventions — you define endpoints directly with `app.MapGet`, `app.MapPost`, etc. Controllers shine when the API is large, has many cross-cutting concerns handled by action filters, or when the team already follows MVC conventions. The key trade-offs are structure vs. speed: controllers enforce a consistent pattern out of the box, while minimal APIs are leaner but require discipline to keep organised. Both approaches can coexist in the same project, so this is not an all-or-nothing decision.

| | Minimal APIs | Controllers |
|---|---|---|
| Setup | Inline in `Program.cs` or extension methods | Class with `[ApiController]` attribute |
| Routing | `app.MapGet("/orders", handler)` | `[Route("api/[controller]")]` + `[HttpGet]` |
| Filters | Endpoint filters (`AddEndpointFilter`) | Action filters, result filters |
| Model binding | Explicit via `[FromBody]`, `[FromQuery]` | Convention-based (automatic) |
| Best for | Microservices, small APIs, performance-critical | Large APIs, teams familiar with MVC patterns |

**Hint:** A strong candidate discusses *when* to use each approach — not just syntax differences. Look for awareness that minimal APIs suit small, focused services while controllers suit complex APIs with many cross-cutting concerns. Follow up: "Can both coexist in one project? When would you mix them?"

**🚩 Red Signal:** Dismisses minimal APIs as "just for demos" or cannot articulate any concrete trade-off beyond surface-level syntax.

---

### 2. 🟡 Your minimal API project has grown to 50+ endpoints, all defined in `Program.cs`. It's becoming unmaintainable. How do you reorganise it?

The first step is to move related endpoints into dedicated static classes using extension methods — for example, `app.MapOrderEndpoints()` defined in `OrderEndpoints.cs`. Route groups (`app.MapGroup("/api/orders")`) let you share a common prefix, filters, and metadata across related endpoints without repeating yourself. For larger projects, the Carter library provides convention-based endpoint modules that auto-discover and register routes. Organising by feature (vertical slices) rather than by layer keeps each domain concept self-contained and easier to navigate.

```csharp
// OrderEndpoints.cs
public static class OrderEndpoints
{
    public static RouteGroupBuilder MapOrderEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/orders");
        group.MapGet("/", GetAllOrders);
        group.MapGet("/{id}", GetOrderById);
        group.MapPost("/", CreateOrder);
        return group;
    }
}
```

**Hint:** Look for awareness that minimal APIs do not enforce structure — the developer must be intentional about organisation. A good answer reflects real project experience. Follow up: "How would you share cross-cutting concerns like auth across a route group?"

**🚩 Red Signal:** Sees no problem with 50+ endpoints in `Program.cs`, or only knows to "move them to controllers."

---

### 3. 🟡 You need to add validation, logging, and auth checks on multiple minimal API endpoints. In MVC you'd use action filters. What's the equivalent approach in minimal APIs?

Endpoint filters (`AddEndpointFilter`) are the minimal API equivalent of action and result filters. They wrap the endpoint handler and execute logic before and after the handler runs — validation before, logging after. Filters are invoked in registration order, forming a pipeline just like MVC filters. For reusability, you can implement `IEndpointFilter` as a class and apply it to individual endpoints or an entire route group. Applying a filter to a `MapGroup` is especially powerful because every endpoint in that group inherits it, keeping cross-cutting concerns DRY.

```csharp
app.MapPost("/orders", CreateOrder)
   .AddEndpointFilter(async (context, next) =>
   {
       var order = context.GetArgument<OrderRequest>(0);
       if (string.IsNullOrEmpty(order.CustomerId))
           return Results.ValidationProblem(
               new Dictionary<string, string[]>
               { ["CustomerId"] = ["Customer ID is required"] });

       var result = await next(context);
       // After: log the outcome
       return result;
   });
```

**Hint:** Candidates should know filters are invoked in registration order and that `IEndpointFilter` can be a class for reusability. Follow up: "How would you apply the same filter to an entire route group instead of repeating it per endpoint?"

**🚩 Red Signal:** Unaware that minimal APIs have any filter mechanism, or suggests wrapping every handler in a manual try/catch.

---

### 4. 🟡 Your minimal API endpoint returns `Results.Ok(order)` for success and `Results.NotFound()` when an order is missing, but Swagger doesn't show the response types. How do you fix this?

The issue is that `Results.Ok()` and `Results.NotFound()` return `IResult`, which is untyped — Swagger cannot infer the response schema. The fix is to use `TypedResults` instead of `Results`. `TypedResults.Ok(order)` returns `Ok<Order>` and `TypedResults.NotFound()` returns `NotFound`, which carry compile-time type information. You can also declare the return type as `Results<Ok<Order>, NotFound>` to tell the OpenAPI generator exactly which responses are possible. This approach eliminates the need for manual `[ProducesResponseType]` attributes that MVC controllers often require.

```csharp
app.MapGet("/orders/{id}", async Task<Results<Ok<Order>, NotFound>> (int id, IOrderRepository repo) =>
    await repo.GetByIdAsync(id) is { } order
        ? TypedResults.Ok(order)
        : TypedResults.NotFound());
```

**Hint:** A strong answer distinguishes `TypedResults` (static class returning typed values) from `Results` (untyped factory), and explains how the union return type `Results<T1, T2>` drives automatic OpenAPI documentation. Follow up: "What happens if you add a third possible response like `Results<Ok<Order>, NotFound, ValidationProblem>`?"

**🚩 Red Signal:** Returns raw objects without status codes, uses `Results.Ok()` for everything including errors, or is unaware that typed results exist.

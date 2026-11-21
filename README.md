# Pageless ASP.NET Web Forms Architecture

Pure C# Rendering Engine bypassing .aspx lifecycle, ViewState, and server controls via direct Response.Write interception at BeginRequest.

Modernizing Web Forms workflows with fetch() and pure C# handlers.

Updated on October 07, 2026.

## Overview & Core Principles

**Pageless Architecture** renders full HTML documents entirely in C# using `StringBuilder` + `Response.Write`, intercepted at `Application_BeginRequest` in `Global.asax.cs`.

*   **Eliminated Features**: No `.aspx` files, `.master` files, ViewState, `<asp:*>` controls, `Page_Load`, or Postbacks.

### `web.config` Requirements
*   `<system.web><sessionState mode="Off" /></system.web>` — Disables `AcquireRequestState` lock for clean `BeginRequest` routing.
*   `<system.webServer><httpErrors existingResponse="PassThrough" /></system.webServer>` — Handlers remain single source of truth for 404/403/500 output.

### Architectural Constraints & Compensations
*   **Process-Local Session**: Anonymous session state lost on recycle --> `lsid` rehydration covers logged-in users.
*   **Synchronous `BeginRequest`**: Keep handler work short; offload long-running jobs to `HostingEnvironment.QueueBackgroundWorkItem`.
*   **`HttpContext.Current` Dependencies**: Put testable business logic in plain repo/service classes called by handlers.

---

## Pipeline Interception, Routing & Custom Session State

### Entry Points & Routing (`Global.asax.cs`)
Intercept all requests in `Application_BeginRequest` and route via a `switch(path)`. Every feature uses **two routes**: page handler (`/books`) and API handler (`/bookapi`).

```csharp
protected void Application_BeginRequest(object sender, EventArgs e)
{
    string path = Request.Path.ToLower().Trim().TrimEnd('/');

    switch (path)
    {
        case "/":
        case "/home":    RH.HomePage.HandleRequest();    return;
        case "/books":   RH.BookPage.HandleRequest();    return;
        case "/bookapi": RH.BookPageApi.HandleRequest(); return;
    }

    // Prefix routes for parameterized URLs (never query strings on page URLs)
    if (path.StartsWith("/u/"))    { RH.UserProfilePage.HandleRequest(path.Substring(3)); return; }
    if (path.StartsWith("/book/")) { RH.BookDetailPage.HandleRequest(path.Substring(6)); return; }
}
```

### Custom Session State (`AppSession`)
Never use `HttpContext.Session` or `IRequiresSessionState`.

*   **Architecture Model**: `ssid` cookie → in-process `ConcurrentDictionary<string, StateObject>` (RAM) → persistent `login_sessions` DB record referenced by the `lsid` cookie.
*   **Fast Path**: Normal requests resolve the random `ssid` token directly against the RAM dictionary; application session state is process-local and intentionally bypasses the built-in ASP.NET `SessionStateModule`.
*   **Persistent Login Recovery**: The RAM dictionary is lost on app-pool recycle. A logged-in user with a valid `lsid` cookie is rehydrated from the DB-backed login session by `AppSession.TryRestoreFromCookie()` on the next request. Anonymous RAM-only session state is not durable.
*   **Storage**:
    1.  `ssid` cookie: Hex key for in-memory `ConcurrentDictionary<string, StateObject>`.
    2.  `login_sessions` DB table: Persistent "Remember Me" record holding hashed `lsid` token (SHA-256).
*   **Cookie Flags**: `HttpOnly = true`, `Secure = ctx.Request.IsSecureConnection`, `SameSite = SameSiteMode.Lax`.
*   **Lifecycle**: `AppSession.TryRestoreFromCookie()` runs in `BeginRequest`. Issue a new `ssid` on login. Expiry rolls forward only when remaining lifetime < 1/12 of window.
*   **Facade Access**:
    ```csharp
    if (!AppSession.IsLoggedIn) { Response.Redirect("/login"); return; }
    obUser me = AppSession.LoginUser;
    ```

---

## Class Conventions & File Structure

*   **One Handler Per File**: Maintain exactly 1 request handler per `.cs` file.
*   **Data Model Convention**: Models use `snake_case` private fields matching DB columns, public `PascalCase` properties, and class names prefixed with `ob` (e.g., `obBook`).

```
/Global.asax.cs               <-- Routing switch table
/engine/
    config.cs                 <-- Connection string & settings
    ApiHelper.cs              <-- Shared response & termination helpers
    Guard.cs                  <-- Auth / ownership guards
    StaticAsset.cs            <-- Cache-busting asset tag generator
    PageTemplate.cs           <-- Master HTML shell
    Models/
        obBook.cs             <-- Data Model (ob prefix)
/RH/
    HomePage.cs               <-- Page Handlers
    BookPageApi.cs            <-- API Handlers
/css/ (site.css, books.css, components/book-picker.css)
/js/  (site.js, books.js, components/book-picker.js)
```

---

## API & JSON Standards

*   **Engine**: `Newtonsoft.Json` (`JsonConvert.SerializeObject`). Match C# properties (`PascalCase`) or DB columns (`snake_case`). **Never use `camelCase`**.
*   **Response Structure**: `{ "success": true, "message": "Success", "data": {}, "items": [] }`

### `ApiHelper` Utility Class
```csharp
public static class ApiHelper
{
    static HttpResponse Response => HttpContext.Current.Response;
    static HttpRequest Request => HttpContext.Current.Request;

    public static string GetBaseUrl() {
        Uri url = Request.Url;
        return $"{url.Scheme}://{url.Host}{(url.IsDefaultPort ? "" : ":" + url.Port)}";
    }

    public static void WriteJson(object obj) {
        Response.ContentType = "application/json";
        Response.Write(JsonConvert.SerializeObject(obj));
    }
    public static void WriteSuccess(string message = "Success") => WriteJson(new { success = true, message });
    public static void WriteError(string message, int statusCode = 400) {
        Response.StatusCode = statusCode;
        WriteJson(new { success = false, message });
    }
    public static void EndResponse() {
        Response.TrySkipIisCustomErrors = true;
        try { Response.Flush(); } catch { }
        Response.SuppressContent = true;
        HttpContext.Current.ApplicationInstance.CompleteRequest();
    }
}
```
> `EndResponse()` replaces `Response.End()` — flushes buffers and triggers `CompleteRequest()` cleanly without `ThreadAbortException`.

---

## HTML String Composition Rules

Compose HTML as multiline interpolated verbatim strings (`$@"..."`) using **single-quoted attributes** (`'`). Avoid long chains of one-line `sb.Append` calls built with escaped quotes and `+` concatenation; they hide the HTML shape and make encoding mistakes easy to miss.

```csharp
// ❌ Avoid — HTML shape hidden, logic tangled into markup
sb.Append("<div class='card-book' data-id='" + book.Id + "'>");
sb.Append("<strong>" + HttpUtility.HtmlEncode(book.Title) + "</strong><br>");
sb.Append("Status: " + (book.Stock > 0 ? (book.Stock < 5 ? "Low stock" : "In stock") : "Sold out"));
sb.Append("</div>");
```

Work in two phases: **collect** values first (run logic, encode), then **compose** the HTML in one interpolated block.

```csharp
// ✅ Approved pattern — phase 1: collect and encode values
string title  = HttpUtility.HtmlEncode(book.Title);
string author = HttpUtility.HtmlEncode(book.Author);
string status = book.Stock == 0 ? "Sold out"
              : book.Stock < 5  ? "Low stock"
              : "In stock";

// ✅ Approved pattern — phase 2: compose in one go
sb.Append($@"
<div class='card-book' data-id='{book.Id}'>
    <strong>{title}</strong><br>
    Author: {author}<br>
    Status: {status}<br>
    <button type='button' class='btn-edit'>Edit</button>
</div>");
```

*   **Default**: Write the whole fragment in a single multiline `sb.Append($@"...")`.
*   **When to split**: Use several `sb.Append($@"...")` blocks only when block-level control flow is needed — `if`/`else`, `switch`, or a loop emitting repeated items. Each block should still be a readable multiline fragment, not a one-line concatenation.

```csharp
// ✅ Split only around control flow
sb.Append($@"
<div class='book-list'>");

foreach (var book in books)
{
    string title = HttpUtility.HtmlEncode(book.Title);
    sb.Append($@"
    <div class='card-book' data-id='{book.Id}'>{title}</div>");
}

sb.Append(@"
</div>");
```

*   **Approved String Forms**:
    1. `$@"..."`: Default for multiline with dynamic values (leading newline after `$@"`).
    2. `@"..."`: Static multiline HTML or blocks containing inline CSS/JS object literals (no `$`).
    3. `$"..."`: Single-line tag emitted conditionally.
*   **Encoding Rules**: HTML content --> `HttpUtility.HtmlEncode(v)`, attributes --> `HttpUtility.HtmlAttributeEncode(v)`, inside `<script>` --> `JsonConvert.SerializeObject(v)`.
*   **Button Type Rule**: Any `<button>` whose purpose is to fire JavaScript must explicitly declare `type='button'`. Never rely on the browser default, because a button inside a `<form>` defaults to submit behavior. Use `type='submit'` only when native form submission is intentionally required.
*   **Separation**: Never interpolate raw user markup. Prefer `data-` attributes and delegated JS listeners over inline `onclick`.

---

## PageTemplate (Master Shell) & StaticAsset

### PageTemplate Surface
*   Set `Title`, `Description`, favicon, OG metadata, head tags, CSS links, navbar, container wrappers, script tags/links and `BodyClass` (e.g., `page-books`).
*   Inject page assets via `ExtraHeaderText` and `ExtraFooterText`.

```csharp
public class HomePage {
    public static void HandleRequest() {
        var pt = new PageTemplate { Title = "Home", Description = "Welcome", BodyClass = "page-home" };
        pt.ExtraHeaderText = StaticAsset.Css("/css/home.css") + "\n" + StaticAsset.Script("/js/home.js");

        var sb = new StringBuilder();
        sb.Append(pt.GenerateHtmlHeader());
        sb.Append(@"<section class='content'><h1>Welcome</h1></section>");
        sb.Append(pt.GenerateHtmlFooter());

        var res = HttpContext.Current.Response;
        res.ContentType = "text/html; charset=utf-8";
        res.Write(sb.ToString());
        ApiHelper.EndResponse();
    }
}
```

### Cache-Busting (`StaticAsset`)
```csharp
public static class StaticAsset {
    public static string Script(string path) => $"<script src='{path}?v={Version(path)}'></script>";
    public static string Css(string path) => $"<link rel='stylesheet' href='{path}?v={Version(path)}' />";
    public static string Component(string name) => Css($"/css/components/{name}.css") + "\n" + Script($"/js/components/{name}.js");
    static string Version(string path) {
        string phys = HostingEnvironment.MapPath("~" + path);
        return (phys != null && File.Exists(phys)) ? File.GetLastWriteTimeUtc(phys).ToString("yyyyMMddHHmmss") : "0";
    }
}
```

*   **Use `StaticAsset` for every CSS/JS reference**, including shared `site.css` / `site.js`, page files, and component files. Do not drop a raw resource URL and assume browser cache invalidation will take care of itself.
*   **Do not memoize `Version()`** into a static dictionary or long-lived cache. Editing a `.js` or `.css` file does not necessarily recycle the app pool; a memoized version can therefore keep emitting an old URL after the file changed. Reading the file write timestamp per request is an inexpensive metadata lookup and keeps cache busting automatic.

---

## JavaScript Organization Rules

1. **Data Island Only Inline**: C# emits dynamic server data into inline `<script>` globals. **All C# → JavaScript values must cross the boundary through `JsonConvert.SerializeObject(value)`**, except values already guaranteed to be numeric (for example a parsed `int`) which may be emitted directly. Never hand-build JavaScript string literals such as `const title = '{book.Title}';`. No inline functions or business logic!

   ```csharp
   const title = {JsonConvert.SerializeObject(book.Title)};
   const books = {JsonConvert.SerializeObject(lstBook)};
   const member_id = {member.Id}; // safe only because this is an int
   ```

2. **Static Behavior in External Files**: All logic and event listeners live in external `.js` files loaded via `StaticAsset.Script()`.
3. **File Scoping**:
   - `site.js`: App-wide logic (must guard element lookups `if (!el) return`).
   - `/js/<page>.js`: Page-specific logic.
   - `/js/components/<n>.js`: Reusable widgets (IIFE exposing constructor on `window`, receives data via `options`, DOM queries scoped to mount point, implements `destroy()`).
4. **Execution Timing**: Data islands precede script tags. Defer all DOM execution to `document.addEventListener('DOMContentLoaded', ...)` inside the external file.
5. **Print-Document Exception**: A route whose primary output is a printable HTML document should be server-rendered as final static HTML and should normally contain **no JavaScript at all**. Do not make printed content, layout, pagination, or styling depend on JavaScript execution. This avoids timing/race problems in browser and headless Chromium printing and keeps the document independently printable.

---

## CSS Organization Rules

1. **App-wide (`/css/site.css`)**: Shared layout, reset, colors, navbar/footer.
2. **Page-specific (`/css/<page>.css`)**: Scoped under `BodyClass` (e.g., `.page-books .card { ... }`).
3. **Component (`/css/components/<n>.css`)**: Scoped by component prefix (e.g., `.bookpicker-item`).
4. **Dynamic Styles**: Emit CSS values as custom properties in inline data islands (e.g., `<style>:root { --brand: #E0F3EF; }</style>`); keep rules in `.css` files.
5. **Print-Document Exception**: For HTML whose primary purpose is browser/headless-Chromium printing or PDF generation, prefer a **self-contained stylesheet inside the document `<head>`** (`<style>...</style>`) rather than external CSS resource files. A self-contained print document avoids extra stylesheet fetches and resource-readiness dependencies, making headless rendering more deterministic and print-resource-friendly. Keep this exception limited to print-oriented documents; normal interactive pages should continue using versioned external CSS through `StaticAsset.Css()`.

---

## Frontend Communication Patterns

### API Wrapper (`apiPost`) & Mutating Calls
```javascript
// site.js — Returns parsed JSON response object directly
async function apiPost(url, formData) {
    var res = await fetch(url, { method: 'POST', body: formData });
    if (!res.ok && res.status !== 400) throw new Error('Server error ' + res.status);
    return await res.json();
}

async function saveBook() {
    var fd = new FormData();
    fd.append('action', 'save-book');
    fd.append('id', document.getElementById('book-id').value);
    fd.append('title', document.getElementById('book-title').value);

    var data = await apiPost('/bookapi', fd);
    
    // Core Response Pattern: always branch on data.success
    if (data.success) {
        alert(data.message);
        // refresh list or update UI
    } else {
        alert('Validation error: ' + data.message);
    }
}
```

### Pre-rendered HTML Fragment Consumption (`apiPostText`)
```javascript
// site.js — Helper for pre-rendered HTML fragment endpoints
async function apiPostText(url, formData) {
    var res = await fetch(url, { method: 'POST', body: formData });
    if (!res.ok && res.status !== 400) throw new Error('Server error ' + res.status);
    return await res.text();
}

async function getAllBooksHtml() {
    var fd = new FormData();
    fd.append('action', 'get-books-html');
    var html = await apiPostText('/bookapi', fd);
    document.getElementById('div-my-books').innerHTML = html;
}
```

---

## Backend API Patterns

### Request Action Dispatcher
```csharp
public class BookPageApi {
    public static void HandleRequest() {
        var Req = HttpContext.Current.Request;
        string action = (Req["action"] + "").ToLower().Trim();
        try {
            switch (action) {
                case "get-books-json": GetBooksJson(); break;
                case "get-books-html": GetBooksHtml(); break;
                case "save-book":      Guard.Mutating(); SaveBook();   break;
                case "delete-book":    Guard.Mutating(); DeleteBook(); break;
                default: ApiHelper.WriteError($"Unknown action: {action}", 400); break;
            }
        }
        catch (GuardException gx) { ApiHelper.WriteError(gx.Message, gx.StatusCode); }
        catch (Exception ex)      { Log.Error(ex); ApiHelper.WriteError("An error occurred", 500); }
        ApiHelper.EndResponse();
    }
}
```

### Data Access & Ownership Scoping Rules
* **Row Ownership**: Scope entity operations by `userId` directly in SQL (`WHERE id = @id AND user_id = @uid`) or check ownership before updating. Return `404` for ownership check failures.
* **Unified Save Action**: `id <= 0` --> Insert; otherwise Update.
* **Keep HTTP Handlers Thin / HTTP-Free Core**: Static request handlers should handle only HTTP concerns — read request values and session/auth context, call application code, and write the response. Keep database access and reusable business logic in plain C# repository/service classes with **no `HttpContext` dependency**. This allows core code to be invoked and tested directly outside IIS/HTTP, including through PowerShell diagnostic/test scripts, which is especially useful for AI-assisted debugging.

```csharp
static void SaveBook() {
    var Req = HttpContext.Current.Request;
    int id = 0; int.TryParse(Req.Form["id"] + "", out id);
    string title = (Req.Form["title"] + "").Trim();
    if (string.IsNullOrEmpty(title)) { ApiHelper.WriteError("Title is required"); return; }

    int userId = AppSession.LoginUser.Id;
    if (id > 0 && !BookRepo.IsOwnedBy(id, userId)) { ApiHelper.WriteError("Not found", 404); return; }

    if (id <= 0) { BookRepo.Insert(userId, title); ApiHelper.WriteSuccess("Book added"); }
    else         { BookRepo.Update(id, title);     ApiHelper.WriteSuccess("Book updated"); }
}
```

---

## Security Baseline

### `Guard` Framework
```csharp
public static class Guard
{
    public static void LoggedIn() {
        if (!AppSession.IsLoggedIn) throw new GuardException("Not signed in", 401);
    }
    public static void Mutating() { LoggedIn(); }
}
```

### File Upload Safety Checklist
1. Store uploaded files **outside the web root** (`~/App_Data/uploads/`).
2. Whitelist allowed file extensions.
3. Generate stored filenames via `Guid.NewGuid()` + extension (never use client filenames as paths).
4. Enforce upload size limits in `web.config` and C# code.

---

## Background Tasks & State Tracking

### HostingEnvironment Background Tasks
```csharp
// Queue long work through ASP.NET host to prevent app pool termination mid-job
HostingEnvironment.QueueBackgroundWorkItem(ct => DoWork(taskId, ct));
```

### Task State Tracking
*   Track progress using `ConcurrentDictionary<int, TaskInfo>` keyed by task ID.
*   Authorize task actions by `OwnerUserId`.
*   Note: Dictionary is in-memory; polling clients must handle missing IDs gracefully after recycles.

---

## Request Lifecycle Diagram

```
HTTP Request (IIS)
  │
  ▼
Application_BeginRequest (Global.asax.cs)
  │
  ├──► AppSession.TryRestoreFromCookie() (ssid → RAM; lsid → DB rehydrate)
  │
  ├──► Route Switch → prefix-route fallthrough (dispatches to Handler)
  │      │
  │      ▼
  │    [API handler] Guard.Mutating() → validate → authorize row → repo call
  │      │
  │      ▼
  │    PageTemplate.GenerateHtmlHeader() (<head>, site.css, ExtraHeaderText, navbar, BodyClass)
  │      │
  │      ▼
  │    Handler builds Body HTML string via StringBuilder
  │      │
  │      ▼
  │    PageTemplate.GenerateHtmlFooter() (footer, site.js, ExtraFooterText)
  │      │
  │      ▼
  │    Response.Write(sb.ToString())
  │      │
  │      ▼
  │    ApiHelper.EndResponse() (Flushes & calls CompleteRequest())
  │
  ▼
EndRequest -> HTTP Response sent to Client
```

---

## C# Performance Constraints

*   **No `dynamic` Types**: Forbidden due to DLR performance overhead, loss of static analysis, and decompilation hostility.
*   **No LINQ in Hot Paths**: Avoid LINQ in hot rendering loops to prevent delegate and closure allocations. Use explicit `for`/`foreach`.
*   **Keep Handlers Short**: Handlers run synchronously on thread-pool threads. Offload long operations to background tasks.

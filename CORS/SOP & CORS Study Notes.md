## 1. Same-Origin Policy (SOP)

- **Same-Origin Policy (SOP)** is a browser security mechanism that restricts JavaScript from one origin from reading data belonging to another origin.
- SOP is still used today; **CORS does not replace SOP**.
- Cross-origin resources can often be loaded, but JavaScript normally cannot read their contents without permission.
- The main distinction is **loading vs reading**.

## 2. Origin

An origin consists of:
- **Scheme**: `http` or `https`
- **Host**: domain or IP address
- **Port**: e.g. `443`, `8080`

Example:
```
https://api.example.com:443
│       │               │
│       │               └── Port
│       └────────────────── Host
└────────────────────────── Scheme
```

Two URLs are same-origin only when their scheme, host, and port all match.
```
https://example.com
https://example.com/login
```
→ Same origin.

```
https://example.com
http://example.com
```
→ Different origin because the scheme differs.

```
https://example.com
https://api.example.com
```
→ Different origin because the host differs.

```
https://example.com
https://example.com:8080
```
→ Different origin because the port differs.

## 3. Origin vs Host

- **Host** = the domain/IP identifying the destination, e.g. `api.example.com`.
- **Origin** = scheme + host + port, e.g. `https://api.example.com:443`.
- The browser sends the requesting origin in the `Origin` HTTP header for relevant cross-origin requests.

## 4. SOP: Loading vs Reading

SOP does **not** prevent websites from loading every cross-origin resource.

Cross-origin loading is generally allowed:
```
<img src="https://cdn.example.com/logo.png">
<script src="https://cdn.example.com/app.js"></script>
<video src="https://media.example.com/video.mp4"></video>
```

However, JavaScript normally cannot read protected cross-origin data:
```
fetch("https://api.example.com/account")
    .then(r => r.json())
    .then(data => console.log(data));
```

If `api.example.com` does not permit the requesting origin through CORS, the browser prevents the JavaScript from accessing the response.

## 5. CORS

- **CORS (Cross-Origin Resource Sharing)** allows a server to explicitly permit JavaScript from another origin to read its responses.
- CORS is a controlled relaxation of the restrictions imposed by SOP.

Example:
```
Frontend: https://www.example.com
API:      https://api.example.com
```

JavaScript:
```
fetch("https://api.example.com/profile")
    .then(r => r.json())
    .then(data => console.log(data));
```

Request:
```
GET /profile HTTP/1.1
Host: api.example.com
Origin: https://www.example.com
```

Server:
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://www.example.com
```

Because the API explicitly allows `https://www.example.com`, the browser allows JavaScript from that origin to read the response.

Without the CORS permission, the server can still process the request and return a response, but the browser prevents the requesting JavaScript from reading it.

## 6. CORS Headers

### `Origin`

Identifies the origin that initiated the cross-origin request:
```
Origin: https://www.example.com
```

### `Access-Control-Allow-Origin`

Specifies which origin can read the response:
```
Access-Control-Allow-Origin: https://www.example.com
```

Wildcard:
```
Access-Control-Allow-Origin: *               //Allows any origin to read a non-credentialed CORS response.
```

### `Access-Control-Allow-Credentials`

Allows credentialed CORS:
```
Access-Control-Allow-Credentials: true
```

### `Access-Control-Allow-Methods`

Specifies permitted methods:
```
Access-Control-Allow-Methods: GET, POST, PUT
```

### `Access-Control-Allow-Headers`

Specifies permitted request headers:
```
Access-Control-Allow-Headers: Content-Type, Authorization
```

### Preflight request headers
```
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header
```

## 7. Credentials

Credentials can include:
- Cookies
- HTTP authentication
- Client certificates

Client:
```
fetch("https://api.example.com/profile", {
    credentials: "include"
});
```

Request may contain:
```
Cookie: session=abc123
```

Server:
```
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Credentials: true
```

Important:
- `credentials: "include"` tells the browser to include credentials in the request.
- `Access-Control-Allow-Credentials: true` tells the browser that credentialed CORS access is permitted.
- `Access-Control-Allow-Origin: *` cannot be used for credentialed CORS.

## 8. Preflight Request

A **preflight request** is an `OPTIONS` request sent by the browser before certain cross-origin requests.

Typical reasons include:
- Non-simple HTTP methods such as `PUT`, `PATCH`, `DELETE`
- Certain custom request headers
- Certain request configurations

Example:
```
fetch("https://api.example.com/account", {
    method: "PUT",
    headers: {
        "X-Custom-Header": "test"
    },
    credentials: "include"
});
```

Browser first sends:
```
OPTIONS /account HTTP/1.1
Host: api.example.com
Origin: https://www.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header
```

Server:
```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Methods: PUT
Access-Control-Allow-Headers: X-Custom-Header
Access-Control-Allow-Credentials: true
```

The browser checks whether the origin, method, and headers are permitted before sending the actual request.

## 9. CORS vs CSRF

### CORS
- Controls whether JavaScript from another origin can **read a response**.

### CSRF
- Concerns an attacker causing a victim's browser to **perform an authenticated action**.

Example:
```
<form action="https://bank.example/transfer" method="POST">
    <input name="amount" value="1000">
    <input name="to" value="attacker">
</form>
<script>document.forms[0].submit();</script>
```

The request may be sent even though the attacker cannot read the response.

### CSRF Token
A server can require a secret token:

```
<input type="hidden" name="csrf_token" value="8f72a9c1...">
```

The attacker normally cannot obtain the token because SOP prevents cross-origin JavaScript from reading the legitimate page.

**CORS ≠ CSRF protection.**

## 10. CORS Misconfiguration
CORS becomes a vulnerability when an untrusted origin is allowed to read sensitive responses.

### 10.1 Arbitrary Origin Reflection

Vulnerable code:
```
app.use((req, res, next) => {
    res.setHeader("Access-Control-Allow-Origin", req.headers.origin);
    res.setHeader("Access-Control-Allow-Credentials", "true");
    next();
});
```

The server effectively says:
```
Whatever Origin you send → I trust it
```

Attacker:
```
GET /api/profile HTTP/1.1
Host: api.example.com
Origin: https://evil.example
Cookie: session=abc123
```

Vulnerable response:
```
Access-Control-Allow-Origin: https://evil.example
Access-Control-Allow-Credentials: true
```

Attacker-controlled page:
```
fetch("https://api.example.com/api/profile", {
    credentials: "include"
})
.then(r => r.json())
.then(data => {
    console.log(data);
});
```

If the victim is authenticated, the attacker may receive sensitive information:
```
{
    "email": "victim@example.com",
    "balance": 5000
}
```

Attack flow:
```
Victim visits evil.example
        ↓
evil.example JavaScript requests api.example.com
        ↓
Browser sends victim's credentials
        ↓
API returns sensitive data
        ↓
API trusts evil.example through CORS
        ↓
Browser exposes response to evil.example
        ↓
Attacker reads victim's data
```

### 10.2 Overly Broad Wildcard

```
Access-Control-Allow-Origin: *
```

- Not automatically a vulnerability.
- May be appropriate for genuinely public resources.
- Should be investigated when used on sensitive APIs.
- `*` cannot be used with credentialed CORS.

### 10.3 Trusting `null`

Request:
```
Origin: null
```

Potentially dangerous response:
```
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
```

Some browser contexts can produce a `null` origin, so sensitive authenticated data should not blindly trust it.

### 10.4 Weak Origin Validation

Bad:
```
if (origin.endsWith("example.com")) {
    allow(origin);
}
```

Potential attacker-controlled origin:
```
https://evil-example.com
```

Another bad example:
```
if (origin.startsWith("https://app.example.com")) {
    allow(origin);
}
```

Potential attacker-controlled origin:
```
https://app.example.com.evil.com
```

Origin validation should compare the complete expected origin.

### 10.5 Insecure Trusted Origin

Avoid trusting an insecure HTTP origin for sensitive applications:
```
http://app.example.com
```

Prefer:
```
https://app.example.com
```

## 11. Secure Coding Practices

- Use an explicit allowlist of trusted origins.
- Compare the complete origin: scheme + host + port.
- Never blindly reflect the `Origin` header.
- Avoid prefix/suffix-based origin validation.
- Do not blindly trust `null`.
- Avoid `*` for sensitive authenticated APIs.
- Never allow credentials with arbitrary origins.
- Allow only required HTTP methods.
- Allow only required request headers.
- Use HTTPS for trusted origins.
- Enable CORS only where cross-origin access is required.
- Keep authentication and authorization enforced server-side.
- Use CSRF protection for state-changing operations.
- Do not use CORS as an authentication or authorization mechanism.
- Do not assume preflight protects every cross-origin request.

## 12. CORS Testing Methodology

### 1. Test an arbitrary origin

```
Origin: https://evil.example
```

Check for:
```
Access-Control-Allow-Origin: https://evil.example
```

### 2. Check credentials

Look for:
```
Access-Control-Allow-Credentials: true
```

### 3. Test `null`

```
Origin: null
```

### 4. Test origin validation

If the application trusts:
```
https://app.example.com
```

Test relevant variations:
```
https://app.example.com.evil.example
https://evil.example
http://app.example.com
```

### 5. Test preflight

```
OPTIONS /api/account HTTP/1.1
Host: api.example.com
Origin: https://evil.example
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Test
```

Inspect:
```
Access-Control-Allow-Origin
Access-Control-Allow-Methods
Access-Control-Allow-Headers
Access-Control-Allow-Credentials
```

### 6. Prove impact

A meaningful CORS vulnerability should demonstrate:
- An attacker-controlled origin is trusted.
- The endpoint contains sensitive data or privileged functionality.
- Credentials are involved when required.
- The attacker can actually read useful information or gain meaningful access.

## 13. CORS vs XSS
- **CORS** controls cross-origin JavaScript access to responses.
- **XSS** executes attacker-controlled JavaScript within the target origin.

With XSS on:
```
https://bank.example
```

The attacker's JavaScript executes as:
```
Origin = https://bank.example
```

Therefore SOP does not provide the normal cross-origin protection.

XSS can potentially access:
- Same-origin API responses
- DOM contents
- CSRF tokens
- Other resources available to scripts in that origin

```
CORS ≠ XSS protection
```

## 14. Final Mental Model

```
SOP
│
├── Same origin
│   └── JavaScript can normally access resources
│
└── Cross origin
    └── JavaScript cannot normally read protected responses
            │
            └── CORS
                └── Server explicitly allows an origin
```



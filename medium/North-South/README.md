# North-South

## Challenge Information

- **Platform:** CyLab Security Academy
- **Category:** Web Exploitation
- **Difficulty:** Medium
- **Challenge:** North-South
- **Author:** Darkraicg492
- **Flag:** `academy{g30_b453d_r0u71n9_5e58725f}`

---

# 1. Challenge Description

The challenge uses **geo-based routing** to control access to the application.

The challenge description explains that requests from a specific geographic region are routed to the server containing the flag, while everyone else is sent to another backend.

The objective is therefore to understand how the Nginx reverse proxy determines the client's geographic location and find a way to make the request appear to originate from the required region.

---

# 2. Initial Access

The initial instance was:

```text
http://chatelaine.cylabacademy.net:16276/
````

Later, the challenge instance was restarted and the URL changed to:

```text
http://xebec.cylabacademy.net:16994/
```

Requesting the application normally:

```bash
curl -i http://xebec.cylabacademy.net:16994/
```

returned:

```text
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)

Welcome!!

No flag in this region!
```

This indicated that we were being routed to the wrong backend.

---

# 3. Understanding the Nginx Configuration

The challenge provided the Nginx configuration.

Important sections:

```nginx
load_module /usr/lib/nginx/modules/ngx_http_geoip2_module.so;
```

This loads the GeoIP2 module.

The GeoIP database was configured as:

```nginx
geoip2 /etc/nginx/GeoLite2-Country.mmdb {
    auto_reload 5m;
    $geoip2_data_country_code default=ZZ country iso_code;
}
```

This creates the variable:

```text
$geoip2_data_country_code
```

which contains the ISO country code associated with the client's IP address.

Examples:

```text
LK = Sri Lanka
US = United States
IN = India
IS = Iceland
```

The configuration defined two upstream services:

```nginx
upstream north {
    server 127.0.0.1:8000;
}

upstream south {
    server 127.0.0.1:9000;
}
```

The important routing condition was:

```nginx
if ($geoip2_data_country_code = IS) {
    proxy_pass http://south;
}

proxy_pass http://north;
```

Therefore, the interesting condition was:

```text
Country code = IS
```

where:

```text
IS = Iceland
```

---

# 4. Understanding the Architecture

The application can be represented as:

```text
                    Internet
                       |
                       v
                +-------------+
                |    Nginx    |
                |     :80     |
                +------+------+
                       |
                 GeoIP lookup
                       |
              Country of source IP
                       |
              +--------+--------+
              |                 |
           IS / Iceland      Other
              |                 |
              v                 v
        127.0.0.1:9000   127.0.0.1:8000
            SOUTH             NORTH
              |                 |
              v                 v
           FLAG            No flag
```

The important point is that the routing decision happens based on the **source IP address**.

---

# 5. Testing Header Spoofing

A common idea when dealing with IP-based restrictions is to try headers such as:

```text
X-Forwarded-For
X-Real-IP
```

We tested:

```bash
curl -i http://xebec.cylabacademy.net:16994/ \
  -H 'X-Forwarded-For: 1.1.1.1'
```

The response was still:

```text
Welcome!!

No flag in this region!
```

We then tested:

```bash
curl -i http://xebec.cylabacademy.net:16994/ \
  -H 'X-Real-IP: 1.1.1.1'
```

Again:

```text
Welcome!!

No flag in this region!
```

Finally, both headers were supplied:

```bash
curl -i http://xebec.cylabacademy.net:16994/ \
  -H 'X-Forwarded-For: 1.1.1.1' \
  -H 'X-Real-IP: 1.1.1.1'
```

The result was still the North backend.

### Conclusion

The application was not trusting these headers for the GeoIP decision.

The important lesson is:

> Adding an IP-related HTTP header does not automatically change the IP address that GeoIP2 uses.

Nginx would need an appropriate `real_ip` configuration for forwarded headers to replace the client address.

---

# 6. Identifying the Required Region

From the configuration:

```nginx
if ($geoip2_data_country_code = IS)
```

we identified:

```text
IS = Iceland
```

Therefore, the request needed to reach the server from an IP address that the GeoIP database recognized as Iceland.

The required flow was:

```text
Client
   |
   v
Icelandic IP
   |
   v
Nginx
   |
   v
GeoIP2 = IS
   |
   v
South backend
   |
   v
Flag
```

---

# 7. Obtaining an Icelandic Exit IP

Instead of changing the Kali system's entire network configuration, an Iceland-capable VPN was used through the browser.

Urban VPN was connected to an Iceland endpoint.

After connecting, the browser request to:

```text
http://xebec.cylabacademy.net:16994/
```

was sent through the Icelandic exit IP.

This caused the GeoIP lookup to produce:

```text
IS
```

instead of the normal location.

---

# 8. Successful Routing

Before connecting through Iceland:

```text
Client
  |
  v
GeoIP = LK
  |
  v
NORTH :8000
  |
  v
No flag in this region!
```

After connecting through the Iceland endpoint:

```text
Client
  |
  v
Iceland VPN
  |
  v
GeoIP = IS
  |
  v
SOUTH :9000
  |
  v
FLAG
```

The browser then displayed:

```text
Welcome!!

academy{g30_b453d_r0u71n9_5e58725f}
```

---

# 9. Flag

```text
academy{g30_b453d_r0u71n9_5e58725f}
```

---

# 10. Key Technical Concepts Learned

## GeoIP

GeoIP databases map IP addresses to geographic information.

In this challenge:

```text
Client IP
    |
    v
GeoLite2-Country.mmdb
    |
    v
Country code
```

The Nginx configuration used:

```nginx
$geoip2_data_country_code
```

to obtain the country code.

---

## ISO Country Codes

Countries are represented using two-letter ISO codes.

Examples:

```text
LK → Sri Lanka
IN → India
US → United States
GB → United Kingdom
IS → Iceland
```

The challenge specifically required:

```text
IS
```

---

## Reverse Proxy Routing

Nginx was acting as a reverse proxy.

The client does not directly communicate with:

```text
127.0.0.1:8000
127.0.0.1:9000
```

Instead:

```text
Client → Nginx → Backend
```

Nginx decides which backend receives the request.

---

## Geographic Access Control

The challenge demonstrates a form of geographic access control:

```text
IP address
    ↓
Geolocation
    ↓
Country
    ↓
Access decision
```

This can be used by real applications for:

* regional content
* compliance requirements
* traffic routing
* localization
* access restrictions

---

# 11. Why Header Spoofing Failed

A useful lesson from this challenge is that these two headers:

```http
X-Forwarded-For
X-Real-IP
```

are not automatically authoritative.

We tried:

```http
X-Forwarded-For: 1.1.1.1
```

and:

```http
X-Real-IP: 1.1.1.1
```

but Nginx continued to identify us based on the actual connection IP.

This is because the supplied configuration did not contain something such as:

```nginx
set_real_ip_from ...
real_ip_header X-Forwarded-For;
```

Therefore, simply changing those headers did not change the GeoIP result.

---

# 12. Important CTF Reasoning

The challenge could initially look like an application-level authentication problem.

However, the configuration gave us the real clue:

```nginx
$geoip2_data_country_code = IS
```

Instead of attacking the application itself, we needed to satisfy the condition controlling the reverse proxy.

The reasoning process was:

```text
1. Access application
       ↓
2. Receive "No flag in this region"
       ↓
3. Inspect Nginx configuration
       ↓
4. Find GeoIP2
       ↓
5. Find country condition
       ↓
6. Decode IS = Iceland
       ↓
7. Test header-based IP spoofing
       ↓
8. Headers have no effect
       ↓
9. Obtain Icelandic exit IP
       ↓
10. Request application
       ↓
11. Nginx routes to South
       ↓
12. Retrieve flag
```

---

# 13. What Could Be Investigated Further

For a deeper understanding of Nginx security, the following could also be investigated in a lab environment:

### Alternative Host headers

Different `server_name` / virtual-host configurations can sometimes expose unintended services.

### Backend exposure

Check whether ports such as:

```text
8000
9000
```

are externally reachable.

### Alternative routes

Applications may expose paths such as:

```text
/admin
/debug
/internal
/api
```

that behave differently.

### Proxy configuration weaknesses

Misconfigured:

```text
proxy_pass
real_ip
Host
X-Forwarded-For
```

handling can sometimes create routing or access-control problems.

---

# 14. Main Takeaway

The most important lesson from this challenge is:

> **Understand the control point before trying to bypass it.**

The flag was not protected by a login or complicated application vulnerability.

The access decision was made here:

```nginx
if ($geoip2_data_country_code = IS)
```

Once that was understood, the entire challenge became:

```text
What does IS mean?
        ↓
     Iceland
        ↓
How can the request originate from Iceland?
        ↓
Iceland VPN exit
        ↓
GeoIP recognizes IS
        ↓
South backend
        ↓
FLAG
```

---

# 15. Tools Used

```text
curl
Nginx
GeoIP2
GeoLite2-Country
Urban VPN
Browser
```

---

# 16. Commands Used

Basic request:

```bash
curl -i http://xebec.cylabacademy.net:16994/
```

Test `X-Forwarded-For`:

```bash
curl -i http://xebec.cylabacademy.net:16994/ \
  -H 'X-Forwarded-For: 1.1.1.1'
```

Test `X-Real-IP`:

```bash
curl -i http://xebec.cylabacademy.net:16994/ \
  -H 'X-Real-IP: 1.1.1.1'
```

Test both:

```bash
curl -i http://xebec.cylabacademy.net:16994/ \
  -H 'X-Forwarded-For: 1.1.1.1' \
  -H 'X-Real-IP: 1.1.1.1'
```

---

# 17. Lessons for Real-World Security

GeoIP-based access control should not generally be treated as a strong authentication mechanism.

IP geolocation can be affected by:

* VPNs
* Proxies
* Cloud providers
* Mobile networks
* inaccurate GeoIP databases
* IP reassignment
* privacy networks

Therefore:

```text
GeoIP ≠ Identity
```

and:

```text
GeoIP ≠ Strong Authentication
```

GeoIP can be useful for **routing and regional policy**, but sensitive authorization should rely on stronger mechanisms such as authentication and authorization controls.

````





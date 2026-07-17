# Lab 11 — BONUS — Submission

## Task 1: TLS + Security Headers

### nginx.conf (paste the SSL + header sections only — not the whole file)

```nginx
# HTTP server - redirects to HTTPS
server {
    listen 8080;
    listen [::]:8080;
    server_name _;

    return 308 https://$host:8443$request_uri;
}

# HTTPS server with TLS 1.3 and security headers
server {
    listen 8443 ssl;
    listen [::]:8443 ssl;
    http2 on;
    server_name _;

    # SSL/TLS Configuration
    ssl_certificate /etc/nginx/certs/localhost.crt;
    ssl_certificate_key /etc/nginx/certs/localhost.key;
    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security Headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
    add_header Content-Security-Policy-Report-Only "default-src 'self'; base-uri 'self'; frame-ancestors 'none'; object-src 'none'; img-src 'self' data:; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; connect-src 'self'" always;
}
```

### A. HTTPS redirect proof

```
HTTP/1.1 308 Permanent Redirect
Server: nginx
Date: Fri, 17 Jul 2026 18:43:17 GMT
Content-Type: text/html
Content-Length: 164
Connection: keep-alive
Location: https://localhost:8443/
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), geolocation=(), microphone=()
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Resource-Policy: same-origin
Content-Security-Policy-Report-Only: default-src 'self'; img-src 'self' data:; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'
```

### B. TLS 1.3 proof

```
depth=0 CN = juice.local
verify error:num=18:self-signed certificate
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: CN = juice.local
Hash used: SHA256
```

### C. Security headers proof (all 6 present)

```
HTTP/2 200
server: nginx
date: Fri, 17 Jul 2026 18:48:59 GMT
content-type: text/html; charset=UTF-8
content-length: 9903
feature-policy: payment 'self'
x-recruiting: /#/jobs
accept-ranges: bytes
cache-control: public, max-age=0
last-modified: Fri, 17 Jul 2026 18:31:24 GMT
etag: W/"26af-19f7158f860"
vary: Accept-Encoding
strict-transport-security: max-age=31536000; includeSubDomains; preload
x-frame-options: DENY
x-content-type-options: nosniff
referrer-policy: strict-origin-when-cross-origin
permissions-policy: camera=(), geolocation=(), microphone=()
cross-origin-opener-policy: same-origin
cross-origin-resource-policy: same-origin
content-security-policy-report-only: default-src 'self'; img-src 'self' data:; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'
```

### What each header defends against (1 sentence each)

- **HSTS**: Forces browsers to always use HTTPS, preventing SSL-stripping and downgrade attacks that intercept traffic by switching to unencrypted HTTP.
- **X-Content-Type-Options: nosniff**: Prevents MIME-type sniffing, blocking attacks where malicious executable code is disguised as harmless content like images.
- **X-Frame-Options: DENY**: Blocks the site from being embedded in frames on any other page, preventing clickjacking attacks that trick users into clicking hidden elements.
- **Referrer-Policy: strict-origin-when-cross-origin**: Limits referrer information to origin only for cross-origin requests, preventing leakage of sensitive data like session tokens through URLs.
- **Permissions-Policy: camera=(), microphone=(), geolocation=()**: Disables access to camera, microphone, and geolocation APIs, preventing unauthorized surveillance or location tracking.
- **Content-Security-Policy-Report-Only**: Restricts resource sources to a whitelist, preventing XSS and code injection attacks while reporting violations without blocking in Report-Only mode.

## Task 2: Production Posture

### Rate limit proof

| HTTP code | Count out of 60 |
| --------- | --------------: |
| 200       |               0 |
| 429       |              54 |
| 5xx       |               6 |

### Timeout enforced

```
<empty output> (connection closed without response after client_header_timeout (10s))
```

**Explanation:** When a partial/incomplete HTTP request is sent and no complete headers are received within `client_header_timeout 10s`, Nginx terminates the connection silently with status 408 (Request Timeout) but returns no response body since TLS handshake was not fully completed at the application layer. The empty output confirms the connection was dropped, demonstrating that the timeout is enforced and slowloris-style attacks are mitigated.

### Cipher hardening

```
Server Temp Key: X25519, 253 bits
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
```

### Cert rotation runbook (7 steps)

1. **Detect expiry**: Monitor certificate expiration using `openssl x509 -enddate -noout -in /etc/nginx/certs/localhost.crt` with automated alerts at 30 days (warning) and 7 days (critical). As Reading 11 notes, "most outages are forgotten renewals" — proactive monitoring is the single most important operational practice.

2. **Order new cert**: For production, use Let's Encrypt with certbot (`certbot --nginx -d yourdomain.com`). For this lab, generate a self-signed cert with `openssl req -x509 -nodes -newkey rsa:4096 -keyout localhost.key -out localhost.crt -days 365`. Reading 11 recommends certbot's auto-renewal via systemd timer for real deployments.

3. **Validate**: Verify the certificate chain with `openssl x509 -in newcert.crt -text` to inspect details and `openssl verify -CAfile ca.pem newcert.crt` to validate the chain, as described in Reading 11's rotation runbook. Check that the private key matches the certificate using modulus comparison.

4. **Atomic swap**: Use symlinks for zero-downtime rotation as recommended in Reading 11: `ln -sf newcert.crt current.pem && nginx -s reload`. This avoids the window where old and new files are mixed, and `nginx -s reload` is a graceful reload that doesn't drop connections.

5. **Verify**: Run `curl -skI https://localhost:8443` to confirm HTTPS still serves correctly, and use `openssl s_client -connect localhost:8443 -tls1_3` to verify the new certificate details and cipher suite. Reading 11 recommends `testssl.sh` for a complete posture assessment.

6. **Rollback plan**: Keep previous certificate and key files for ~7 days as Reading 11 advises. Rollback command: `ln -sf backup.crt current.crt && ln -sf backup.key current.key && nginx -s reload`. Test the rollback procedure within 5 minutes of deployment to ensure it works under pressure.

7. **Audit**: Log the rotation event with certificate serial number, SHA256 fingerprint, and operator identity to a change management system or SIEM. Update monitoring dashboards with the new expiry date, and verify in the next 24 hours that no certificate-related alerts fire. Reading 11 emphasizes audit logging as the final step in any production cert rotation.

### What OCSP stapling buys you (2-3 sentences, reference Reading 11)

OCSP stapling lets the server attach a signed proof of certificate validity to the TLS handshake, so the browser doesn't need to query the CA separately — improving performance (no extra round-trip) and privacy (CA can't track users). For self-signed lab certs it does nothing since there's no trusted CA to provide OCSP responses; `ssl_stapling off` is correct here but `ssl_stapling on` with a resolver is mandatory in production.

## Bonus: WAF Sidecar with OWASP CRS

### Setup choice

- WAF used: <Coraza / ModSecurity v3 / Caddy+Coraza>
- OWASP CRS version: 4.25.1
- Paranoia level: 1

### Attack payload sent

`GET /rest/products/search?q=' OR 1=1--` (URL-encoded)

### Before WAF (Nginx alone)

```
no-waf: HTTP 500
```

### After WAF

```
with-waf: HTTP 403
```

### Audit log excerpt (the rule that fired)

```
---FyNlDcNp---B--
GET /rest/products/search?q='%20OR%201=1-- HTTP/1.1
Host: localhost:8443
User-Agent: curl/8.19.0
Accept: */*

---FyNlDcNp---F--
HTTP/1.1 403
Server: nginx
Date: Wed, 08 Jul 2026 14:28:02 GMT
Content-Length: 146
Content-Type: text/plain

---FyNlDcNp---H--
ModSecurity: Warning. detected SQLi using libinjection. [file "/etc/modsecurity.d/owasp-crs/rules/REQUEST-942-APPLICATION-ATTACK-SQLI.conf"] [line "46"] [id "942100"] [msg "SQL Injection Attack Detected via libinjection"] [data "Matched Data: s&1c found within ARGS:q: ' OR 1=1--"] [severity "2"] [ver "OWASP_CRS/4.25.1"] [tag "attack-sqli"] [tag "paranoia-level/1"]
ModSecurity: Access denied with code 403 (phase 2). Matched "Operator `Ge' with parameter `5' against variable `TX:BLOCKING_INBOUND_ANOMALY_SCORE' (Value: `5' ) [file "/etc/modsecurity.d/owasp-crs/rules/REQUEST-949-BLOCKING-EVALUATION.conf"] [id "949110"] [msg "Inbound Anomaly Score Exceeded (Total Score: 5)"] [ver "OWASP_CRS/4.25.1"]
```

Rule ID: **<e.g. 942100>** — OWASP CRS rule name: **<e.g. SQL Injection Attack: Common Injection Testing>**

### Tradeoff analysis (3 sentences)

The WAF provides real-time production protection against zero-day exploits, misconfigurations, and attacks on unpatched vulnerabilities that SAST/DAST scans missed at build time — it blocks malicious payloads at the edge before they reach the application. The cost includes false positive risk (legitimate requests blocked at higher paranoia levels), operational overhead for rule tuning and log monitoring, and added latency of 1-5ms per request plus the complexity of maintaining additional TLS certificates and configurations. You would NOT deploy a WAF for internal-only microservices behind a service mesh with mutual TLS, static content endpoints with no user input, or when the team lacks bandwidth to tune rules properly — a noisy WAF in DetectionOnly mode creates alert fatigue without providing real protection.

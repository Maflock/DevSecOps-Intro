# Lab 11 — BONUS — Reverse Proxy Hardening: Nginx Security Headers, TLS, and Rate Limiting

![difficulty](https://img.shields.io/badge/difficulty-advanced-red)
![topic](https://img.shields.io/badge/topic-Reverse%20Proxy-blue)
![points](https://img.shields.io/badge/points-10-orange)
![tech](https://img.shields.io/badge/tech-Nginx%20%2B%20OpenSSL-informational)

> **Goal:** Put a hardened Nginx reverse proxy in front of Juice Shop — TLS 1.3, full security-header set, request rate limiting, fail-closed timeouts. Demonstrate each control with curl evidence.
> **Deliverable:** A PR from `feature/lab11` with `submissions/lab11.md` + your hardened `nginx.conf`. Submit PR link via Moodle.

> 🌟 **This is a BONUS lab** — 10 pts total (Task 1: 6 + Task 2: 4), no separate bonus row.
> Bonus labs count toward a separate **30% weight** in your final grade (Lecture 1 README).
> Difficulty is **advanced** — you're expected to operate more independently than in the main labs.

---

## Overview

In this lab you will practice:
- **Production-grade Nginx config** — listen, upstream, ssl_protocols, http/2
- **TLS 1.3 + HSTS** — modern cert generation with `openssl`
- **Security headers** — CSP, X-Content-Type-Options, X-Frame-Options, Referrer-Policy, Permissions-Policy
- **Rate limiting + connection limits** — `limit_req_zone` + `limit_conn_zone`
- **Timeouts** — fail-closed behavior, not infinite hangs (Lecture 5/9 themes)

> Why this lab matters: every public-facing service runs behind a proxy. The default Nginx config has none of the protections in this lab. Real production hardening starts here.

---

## Project State

**You should have from Labs 1+8:**
- Juice Shop image (Lab 1) — it'll sit behind your proxy
- Familiarity with security headers / TLS posture from Lecture 7

**This lab adds:**
- A working `docker-compose` stack: nginx (in front) + juice-shop (behind)
- A hardened `nginx.conf` with all controls demonstrated
- Evidence (curl outputs + access logs) showing each control works

---

## Setup

You need:
- **Docker + docker-compose**
- **`openssl`** — for self-signed cert generation
- **`curl`** + **`jq`**

```bash
git switch main && git pull
git switch -c feature/lab11

# Verify
docker compose version && openssl version
```

> **Plumbing provided** (in `labs/lab11/`):
> - [`labs/lab11/docker-compose.yml`](lab11/docker-compose.yml) — stack with nginx + juice-shop containers wired together
> - [`labs/lab11/reverse-proxy/`](lab11/reverse-proxy/) — bare-bones starter Nginx config (NOT hardened — you'll harden it)

```bash
ls labs/lab11/docker-compose.yml labs/lab11/reverse-proxy/

# Generate a self-signed cert (3650 days = ~10 years, fine for lab)
mkdir -p labs/lab11/reverse-proxy/certs
# File names MUST be `localhost.crt` / `localhost.key` — the shipped nginx.conf
# expects those exact paths under /etc/nginx/certs/.
openssl req -x509 -nodes -newkey rsa:4096 \
  -keyout labs/lab11/reverse-proxy/certs/localhost.key \
  -out labs/lab11/reverse-proxy/certs/localhost.crt \
  -subj "/CN=juice.local" \
  -days 3650
```

---

## Task 1 — TLS + Security Headers + Rate Limiting (6 pts)

**Objective:** Configure Nginx with TLS 1.3 only, the full security-header set, and a working rate limit.

### 11.1: Harden the Nginx config

Edit `labs/lab11/reverse-proxy/nginx.conf`:

```nginx
# YOUR TASK: Hardened Nginx reverse proxy
# Requirements:
#
# 1. Listen on 443 only (HTTPS); redirect 80 → 443
#    - `listen 443 ssl http2;`
#    - `listen 80;` + `return 301 https://$host$request_uri;` in a separate server block
#
# 2. TLS 1.3 only (Lecture 7 misconfig list — old TLS = CKV_AWS_103 territory)
#    - `ssl_protocols TLSv1.3;`
#    - `ssl_prefer_server_ciphers off;` (TLS 1.3 ignores this)
#
# 3. HSTS (Strict-Transport-Security)
#    - `add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;`
#
# 4. Other required security headers (Lecture 1/5):
#    - X-Content-Type-Options: nosniff
#    - X-Frame-Options: DENY
#    - Referrer-Policy: strict-origin-when-cross-origin
#    - Permissions-Policy: <something reasonable — e.g., disable camera/microphone>
#    - Content-Security-Policy: default-src 'self' (start strict; relax only if needed)
#
# 5. Rate limit on /api/* paths (defaults to 10 req/s, burst 20)
#    - `limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;` in http {} block
#    - `limit_req zone=api burst=20 nodelay;` in the /api location block
#
# 6. Connection limit (max concurrent per IP)
#    - `limit_conn_zone $binary_remote_addr zone=conn:10m;`
#    - `limit_conn conn 50;`
#
# 7. Timeouts (fail-closed)
#    - `client_body_timeout 10s;`
#    - `client_header_timeout 10s;`
#    - `proxy_read_timeout 30s;`
#    - `proxy_connect_timeout 5s;`
#
# 8. Upstream Juice Shop
#    - `upstream juice_shop { server juice-shop:3000; }`
#    - All locations proxy_pass to it
#
# Hints:
#   - Use the Mozilla SSL Configuration Generator (https://ssl-config.mozilla.org)
#     "Modern" profile is TLS 1.3 only — that's what you want
#   - Always add `always` keyword to add_header (otherwise it disappears on 5xx responses)
```

### 11.2: Bring up the stack

```bash
cd labs/lab11
docker compose up -d
docker compose ps        # Both containers should be Healthy / Running

# Wait for Juice Shop to be ready (behind the proxy)
until curl -sk https://localhost/rest/products > /dev/null; do sleep 2; done
echo "✅ Juice Shop reachable via Nginx"

cd -
```

### 11.3: Test each control

```bash
mkdir -p labs/lab11/results

# A. HTTPS works, HTTP redirects
curl -sI http://localhost | tee labs/lab11/results/http-redirect.txt
# Should see: HTTP/1.1 301 Moved Permanently + Location: https://...

# B. TLS 1.3
echo | openssl s_client -connect localhost:443 -tls1_3 -brief 2>&1 | head -5 \
  | tee labs/lab11/results/tls13.txt

# C. Security headers
curl -skI https://localhost | tee labs/lab11/results/headers.txt
# Verify presence of: HSTS, X-Content-Type-Options, X-Frame-Options, Referrer-Policy, Permissions-Policy, CSP

# D. Rate limit kicks in (>10 req/s)
seq 1 60 | xargs -n1 -P 60 -I{} curl -sk -o /dev/null -w "%{http_code}\n" \
  https://localhost/api/ | sort | uniq -c | tee labs/lab11/results/ratelimit.txt
# Should see a mix of 200 + 429 — at least some 429s

# E. Timeout enforced (slowloris-style)
# This sends bytes very slowly; nginx should kill after client_header_timeout
echo "GET / HTTP/1.0" | timeout 15 nc localhost 443 2>&1 | head -5 \
  | tee labs/lab11/results/timeout.txt
```

### 11.4: Document in `submissions/lab11.md`

```markdown
# Lab 11 — BONUS — Submission

## Task 1: TLS + Security Headers + Rate Limiting

### nginx.conf (paste the SSL + HSTS + header + rate-limit + timeout sections)
```nginx
<paste relevant sections — not the whole file>
```

### A. HTTPS redirect proof
```
<paste labs/lab11/results/http-redirect.txt — must show 301 to https://>
```

### B. TLS 1.3 proof
```
<paste labs/lab11/results/tls13.txt — must show "Protocol  : TLSv1.3" line>
```

### C. Security headers proof
```
<paste labs/lab11/results/headers.txt — must show ALL 6 headers (HSTS, X-CTO, X-FO, Referrer-Policy, Permissions-Policy, CSP)>
```

### D. Rate limit proof
| HTTP code | Count out of 60 |
|-----------|----------------:|
| 200 | <n> |
| 429 | <n> |
| 5xx | <n> |

Excerpt of access log showing rate-limited requests:
```
<docker compose logs nginx | grep -E "limit|429" | head -5>
```

### E. Timeout enforced
```
<paste labs/lab11/results/timeout.txt — should show 408 Request Timeout or connection closed>
```

### What each header defends against (1 sentence each)
- HSTS: ...
- X-Content-Type-Options: nosniff: ...
- X-Frame-Options: DENY: ...
- Referrer-Policy: ...
- Permissions-Policy: ...
- Content-Security-Policy: ...
```

---

## Task 2 — Production Posture: Cipher Hardening, OCSP, Cert Rotation (4 pts)

**Objective:** Move beyond the lab cert to production-relevant TLS posture — explicit cipher list, OCSP stapling enabled, document the cert rotation process.

### 11.5: Strengthen cipher list

Edit `nginx.conf` to add (TLS 1.3 cipher suites, Mozilla Modern):

```nginx
# These are the cipher suites in the TLS 1.3 RFC
ssl_ciphers TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256;

# Curves (modern + safe)
ssl_ecdh_curve X25519:secp384r1;

# Session resumption (avoid renegotiating TLS for every connection)
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;        # disable to avoid leaking previous-session key material
```

### 11.6: OCSP stapling (informational — self-signed cert doesn't actually staple)

```nginx
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 1.1.1.1 valid=300s;
resolver_timeout 5s;
```

> Self-signed certs don't have a real CA to query OCSP from, so the directive is effectively documentation. In production with a real CA cert, this avoids cert-status round-trips on every TLS handshake.

### 11.7: Test with `testssl.sh` or manually

```bash
# Reload nginx
cd labs/lab11
docker compose restart nginx
cd -

# Test cipher list
echo | openssl s_client -connect localhost:443 -tls1_3 -ciphersuites TLS_AES_256_GCM_SHA384 2>&1 \
  | grep -E "Cipher|Server Temp Key" | tee labs/lab11/results/cipher.txt
```

### 11.8: Document cert rotation process

```markdown
## Task 2: Production Posture

### Cipher hardening (paste the ssl_ciphers + ssl_ecdh_curve + ssl_session_* lines)
```nginx
<paste>
```

### TLS handshake verification
```
<paste labs/lab11/results/cipher.txt>
```

### Cert rotation process (write the runbook — 5-7 steps)
1. **Detect expiry** — <e.g., "monitoring alert on cert expiry < 30 days">
2. **Order new cert** — <e.g., "Let's Encrypt via certbot, or vendor CA">
3. **Validate** — <how to verify the new cert is good BEFORE deploying>
4. **Deploy** — <atomic swap — rename old + new + reload nginx; or use a separate path>
5. **Verify** — <`curl -vk` to confirm new cert serves; `openssl s_client` to check chain>
6. **Rollback plan** — <what if the new cert breaks? Keep the old key/cert; one-line revert>
7. **Audit** — <log to your SIEM / DefectDojo / ticket system>

### What OCSP stapling buys you (2-3 sentences)
Why is OCSP stapling useful for production but not for a self-signed lab cert?
Reference Lecture 8 (signing/verification chain) — OCSP is the "fast revocation check"
piece of the chain.
```

---

## How to Submit

```bash
git add labs/lab11/reverse-proxy/nginx.conf
git add submissions/lab11.md
git commit -m "feat(lab11): hardened nginx reverse proxy in front of juice shop"
git push -u origin feature/lab11

# Cleanup
cd labs/lab11
docker compose down
cd -
```

> **Do NOT commit** `labs/lab11/reverse-proxy/certs/` — keys + certs should be gitignored. Submission paste-ins are the evidence.

PR checklist body:

```text
- [x] Task 1 — TLS 1.3 + 6 security headers + rate limit + timeouts (all with proof)
- [ ] Task 2 — Cipher list + session config + cert-rotation runbook
```

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ HTTPS serves; HTTP → HTTPS 301 redirect works
- ✅ TLS 1.3 negotiated (`openssl s_client -tls1_3` succeeds)
- ✅ All 6 security headers present (HSTS, X-CTO, X-FO, Referrer-Policy, Permissions-Policy, CSP)
- ✅ Rate limit demonstrably returns 429 on >10 req/s sustained load
- ✅ Slowloris-style request triggers timeout (408 or connection close)
- ✅ Each header's purpose explained in 1 sentence (no copy-paste from Mozilla docs)

### Task 2 (4 pts)
- ✅ `ssl_ciphers` configured to TLS 1.3 suite (no fallback to TLS 1.2 ciphers)
- ✅ `ssl_ecdh_curve` includes X25519
- ✅ Session resumption configured (`ssl_session_cache shared`)
- ✅ Cert rotation runbook has all 7 steps (detect → order → validate → deploy → verify → rollback → audit)
- ✅ OCSP-stapling explanation references the production-vs-lab gap honestly

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — TLS + headers + rate limit | **6** | All 5 controls with proof + per-header explanation |
| **Task 2** — Production posture | **4** | Cipher hardening + 7-step rotation runbook + OCSP explanation |
| **Total** | **10** | (No bonus row — this IS the bonus lab) |

---

## Resources

<details>
<summary>📚 Documentation</summary>

- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) — Pick "Modern", paste output, adjust
- [Nginx official `ssl_*` directives](https://nginx.org/en/docs/http/ngx_http_ssl_module.html) — Authoritative reference
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/) — Defends-against descriptions
- [Mozilla Web Security Cheatsheet](https://infosec.mozilla.org/guidelines/web_security) — Per-header reasoning
- [Let's Encrypt + certbot](https://certbot.eff.org/) — For real cert rotation

</details>

<details>
<summary>⚠️ Common Pitfalls</summary>

- 🚨 **`curl: (60) SSL certificate problem`** — your cert is self-signed. Use `-k` (insecure) flag for testing. In real production, the cert chain to a public CA root is what makes `curl` happy without `-k`.
- 🚨 **`add_header` directives disappear on 5xx responses** — always use the `always` keyword: `add_header Strict-Transport-Security "..." always;`.
- 🚨 **TLS 1.3 negotiation falls back to TLS 1.2** — you forgot `ssl_protocols TLSv1.3;` (without the `TLSv1.2` fallback). Some testing tools auto-fall-back; check `openssl s_client -tls1_3` explicitly.
- 🚨 **Rate limit always returns 503 instead of 429** — Nginx returns 503 by default. Set `limit_req_status 429;` in the http{} block.
- 🚨 **`docker compose up` fails with "host port already in use" on 80 or 443** — kill local Apache/nginx/other-proxy first: `sudo systemctl stop apache2 nginx`. Or remap to 8080/8443 in compose.
- 🚨 **CSP `default-src 'self'` breaks Juice Shop frontend** — Juice Shop loads inline scripts + remote fonts; full CSP requires extensive testing. For the lab, demonstrate it WORKS at strict; document the relax-as-needed approach for real deployments.
- 💡 **`testssl.sh`** is the industry-standard TLS posture tool. Install it (`brew install testssl`) and run `testssl.sh localhost:443` for a free A+ assessment.

</details>

<details>
<summary>🪜 Looking outside this course</summary>

The hardened proxy you built here is **production-grade** for a single-domain deployment:
- Add Let's Encrypt + certbot for real cert automation
- Add `fail2ban` + Nginx access log scraping for IP-based abuse blocking
- Add ModSecurity + OWASP Core Rule Set for a real WAF layer
- For multi-domain / cluster deployments: Envoy or Traefik often replace Nginx

Add this to your portfolio walkthrough (Lab 10 bonus) — "I built a hardened reverse proxy with the full security-header set, rate limiting, and TLS 1.3" is a strong interview line.

</details>

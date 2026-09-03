---
name: network-troubleshooting
description: "Systematic, safety-first diagnosis of developer network failures such as connection refused, DNS resolution errors, TLS certificate problems, proxy failures, and package registry timeouts. Use when a command fails with a network-shaped error (curl, npm, pip, git, docker), when a service is unreachable, or when the user asks to debug connectivity, DNS, TLS, or proxy issues."
---

# Network Troubleshooting

This skill provides a concise, safety-first decision guide for diagnosing developer network failures. Diagnostics must stay read-only and target-scoped — this is not an automated remediation toolkit.

## Safety Boundaries

These boundaries apply to every step of the workflow below and must never be relaxed, even if asked:

- Prefer read-only diagnostics and trusted project-provided diagnostic scripts.
- Use the failing host, URL, registry, or service as the default probe target — do not probe unrelated hosts.
- Ask before probing unrelated external services.
- Never print proxy URLs, credentials, tokens, auth headers, package index URLs, registry hostnames from config, or raw config values in shared output.
- Internal hosts and URLs may be collected for target-scoped local diagnostics, but replace them with placeholders before sharing logs or reports unless the user explicitly approves including them.
- Never dump local configuration from npm, pnpm, yarn, pip, Git, Docker, shell, OS proxy, VPN, or certificate stores.
- Never disable, bypass, or skip TLS or certificate verification (no `-k`/`--insecure`, no `NODE_TLS_REJECT_UNAUTHORIZED=0`, no `verify=False`), even temporarily "just to test."
- Never change OS networking, DNS, proxy, package manager, Git, Docker, shell, VPN, or trust-store settings without explicit user approval for the exact action being taken.

## Workflow

1. **Collect** — Capture the exact error text, the failing command, the target host/URL/port, OS/shell, proxy/VPN context, and whether the failure affects one target or many.
2. **Classify** — Match the symptom against the error classification table below to form a hypothesis.
3. **Diagnose** — Run only read-only checks scoped to the failing target, starting with the smallest relevant check.
4. **Explain** — Interpret each diagnostic result in plain language before suggesting any fix.
5. **Advise** — Present remediation options as choices; wait for explicit user approval before changing any state.
6. **Verify** — After an approved change, re-run the original failing command or an equivalent target-scoped check.

## Error Classification

| Error Pattern | Likely Category |
|---|---|
| `ECONNREFUSED`, `ERR_CONNECTION_REFUSED`, `Connection refused` | Target service or port is not listening |
| `ECONNRESET`, `socket hang up`, `Connection reset` | Connection dropped by target, proxy, firewall, or middlebox |
| `ETIMEDOUT`, `ERR_CONNECTION_TIMED_OUT`, `timed out` | Routing, firewall, proxy, or target availability |
| `ENOTFOUND`, `EAI_NONAME`, `ERR_NAME_NOT_RESOLVED`, `getaddrinfo` | DNS or hostname issue |
| `ERR_PROXY_CONNECTION_FAILED`, proxy tunnel errors, HTTP `407` | Proxy configuration or proxy authentication |
| `UNABLE_TO_VERIFY_LEAF_SIGNATURE`, `CERT_HAS_EXPIRED`, `self signed`, `ERR_CERT_*` | TLS certificate or local trust issue |
| HTTP `403` | Authorization, IP allowlist, CORS, or policy block |
| HTTP `502`, `503`, `504` | Upstream service, gateway, CDN, or transient server issue |
| `npm ERR! network`, package install timeout, `pip` timeout | Package registry, proxy, DNS, or network path issue |
| `fatal: unable to access`, Git fetch/push timeout | Git remote, proxy, DNS, TLS, or network path issue |

## Safe Target-Scoped Checks

Choose the smallest relevant check and explain what it does before running it. Always substitute the specific failing `<target-host>`, `<port>`, and `<path>` — never a broader or unrelated target.

### Connectivity

Linux/macOS:

```bash
ping -c 4 <target-host>
curl -v telnet://<target-host>:<port> --connect-timeout 5
```

Windows PowerShell:

```powershell
Test-Connection -ComputerName <target-host> -Count 4
Test-NetConnection -ComputerName <target-host> -Port <port>
```

### DNS

Linux/macOS:

```bash
nslookup <target-host>
dig <target-host>
```

Windows PowerShell:

```powershell
Resolve-DnsName <target-host>
```

### HTTP

Linux/macOS:

```bash
curl -vvv -o /dev/null -w "HTTP %{http_code}\nTime: %{time_total}s\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\n" https://<target-host>/<path>
curl -I https://<target-host>/<path>
```

Windows PowerShell:

```powershell
$uri = "https://<target-host>/<path>"
try {
  $resp = Invoke-WebRequest -Uri $uri -Method Head -TimeoutSec 10
  "HTTP status: $([int]$resp.StatusCode)"
} catch [Net.WebException] {
  if ($_.Exception.Response) {
    "HTTP status: $([int]$_.Exception.Response.StatusCode)"
  } else {
    "HTTP request failed: $($_.Exception.Message)"
  }
}
```

### TLS

Linux/macOS:

```bash
openssl s_client -connect <target-host>:<port> -servername <target-host> -showcerts </dev/null
echo | openssl s_client -connect <target-host>:<port> -servername <target-host> 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

Windows PowerShell — check the certificate and HTTP status separately so a non-2xx response is never mislabeled as a certificate failure:

```powershell
try {
  $req = [Net.HttpWebRequest]::Create("https://<target-host>:<port>/<path>")
  $req.Method = "HEAD"
  $req.Timeout = 5000
  try {
    $resp = $req.GetResponse()
  } catch [Net.WebException] {
    $resp = $_.Exception.Response
    if ($req.ServicePoint.Certificate) {
      $cert = $req.ServicePoint.Certificate
      "Cert subject: $($cert.Subject)"
      "Cert expires: $($cert.GetExpirationDateString())"
    }
    if ($resp) {
      "HTTP status: $([int]$resp.StatusCode) $($resp.StatusDescription)"
      $resp.Close()
    } else {
      "TLS/network error: $($_.Exception.Message)"
    }
    return
  }
  $cert = $req.ServicePoint.Certificate
  if ($cert) {
    "Cert subject: $($cert.Subject)"
    "Cert expires: $($cert.GetExpirationDateString())"
  }
  "HTTP status: $([int]$resp.StatusCode) $($resp.StatusDescription)"
  $resp.Close()
} catch {
  "TLS/network error: $($_.Exception.Message)"
}
```

### Proxy and Package Managers

- Avoid raw config reads for proxy, package manager, Git, Docker, or OS network configuration.
- Report only whether relevant settings appear present when this can be checked without printing values.
- If the available command would print a URL, token, internal hostname, auth header, or full config value, do not run it.
- Only perform package registry probes when the failed operation already targeted that registry, or after the user approves that exact probe target.

## User-Approved Remediation Options

Present these as choices for the user to approve — never apply them automatically.

| Diagnosis | Safe Advice |
|---|---|
| Target service is not listening | Check whether the local or remote service is running and whether the expected port is correct. |
| DNS lookup fails | Check the hostname, hosts-file expectations, DNS service status, or approved DNS changes. |
| Proxy appears required or unavailable | Ask whether the user wants to start or adjust the proxy/VPN, then verify only the approved target. |
| TLS certificate expired | Renew or replace the certificate, fix system time, install a trusted local development CA, or update the trust store. Never bypass TLS verification. |
| TLS unknown CA | Import the correct CA into the appropriate trust store after the user confirms the source and scope. |
| HTTP `407` | Ask the user to confirm proxy authentication requirements before changing credentials or tool settings. |
| HTTP `403` | Check authentication, API key scope, IP allowlist, CORS policy, or service policy. |
| HTTP `502`/`503`/`504` | Treat as upstream or gateway instability; check status pages when approved and retry with backoff. |
| Package install timeout | Discuss approved registry, proxy, or network-path options without printing or changing stored config values. |
| Git network failure | Discuss approved remote URL, proxy, credential, TLS, or network-path options without changing global Git settings automatically. |
| Docker pull failure | Discuss approved registry mirror, proxy, or daemon settings as a user-approved configuration change. |

## Verification

- After any user-approved change, re-run the original failing command whenever possible.
- If a smaller check is needed instead, keep it scoped to the same failing host, URL, registry, or service.
- Always explain what each diagnostic result means before recommending the next step.

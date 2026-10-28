# Cloudflare Tunnel Update

## The Update

Upgraded `cloudflared` on the Raspberry Pi from version `2026.3.0` to `2026.10.5`.

## Why the Update

In October 2026, Cloudflare disclosed a certificate security incident involving eight eMudhra intermediate certificate authority (CA) certificates. Their private keys were found to be cryptographically weak and recoverable. An attacker who recovered those keys could potentially create forged certificates to impersonate websites and intercept or modify HTTPS traffic.

Cloudflare's security guidance required Cloudflare Tunnel (`cloudflared`) installations to be upgraded to **version `2026.10.2` or later**. The Raspberry Pi was running `2026.3.0`, so it needed an update. Version `2026.10.5` meets that requirement.

**Scope note:** The incident concerned eMudhra's global TLS/code-signing certificate infrastructure. CERT-In and eMudhra stated that eMudhra's Indian digital-signature and eSign services were not affected. This update was performed to address Cloudflare's `cloudflared` mitigation guidance, not because the Raspberry Pi itself was known to have been compromised.

Sources:
- [Cloudflare security incident and mitigation guidance](https://blog.cloudflare.com/emudhra-certificate-compromise/)
- [CERT-In advisory: Compromise of eMudhra Intermediate CA Certificates](https://www.cert-in.org.in/s2cMainServlet?CACODE=CICA-2026-3649&pageid=PUBADV01)
- [eMudhra's incident statement](https://emudhra.com/en/post/certificate-security-incident)

## How the Update Was Done

1. Downloaded the ARM64 Debian package for `cloudflared` version `2026.10.5`.
2. Checked the package details to confirm the version and architecture:
   ```text
   Version: 2026.10.5
   Architecture: arm64
   ```
3. Installed the package:
   ```bash
   sudo dpkg -i /tmp/cloudflared-linux-arm64.deb
   ```
4. Verified the installed version and service status:
   ```bash
   cloudflared --version
   systemctl status cloudflared --no-pager
   ```
5. Restarted the Cloudflare Tunnel service:
   ```bash
   sudo systemctl restart cloudflared
   ```
6. Verified the version and service status again:
   ```bash
   cloudflared --version
   systemctl status cloudflared --no-pager
   ```
7. Tested Nextcloud and Grafana in a browser. Both were working after the restart.

## Result

- `cloudflared` version: `2026.10.5`
- Service status: `active (running)`
- Tunnel connections registered successfully.
- Nextcloud: working.
- Grafana: working.

**Status: Update completed successfully.**

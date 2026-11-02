# Cloudflare runbook

How domains and sites are moved to and run on Cloudflare: moving a domain's Domain Name
System (DNS) and its registration, the settings every zone gets, serving a site from a server
through a Cloudflare Tunnel, configuring nginx (by hand and through a Forge nginx template),
domains that only redirect to another domain, and a mail domain's Mail Transfer Agent Strict
Transport Security (MTA-STS) policy served from a Worker.

Every command block is self-contained: set the variables on its first lines, then copy the
whole block. Dashboard paths are as the Cloudflare and Forge dashboards showed them on
2026-11-01. They move things around; Cloudflare's search box at the top of the dashboard
finds a setting by name.

```
visitor ──https──> Cloudflare edge ──tunnel (outbound from the server)──> cloudflared
                                                                            │ http, Host header kept
                                                                            v
                                                              nginx on port 80 ──> PHP
```

Cloudflare ends the encrypted connection, Transport Layer Security (TLS), at its edge and
presents its own certificate to visitors. Cloudflare's dashboard still labels it "SSL", after
the older Secure Sockets Layer. The server makes an outbound connection to Cloudflare, so
nothing has to listen on ports 80 or 443 to the internet, and the server needs no certificate.

## Contents

1. [Before you start](docs/before-you-start.md)
2. [Move a domain's DNS to Cloudflare](docs/move-dns.md)
   - [Record the current records](docs/move-dns.md#record-the-current-records)
   - [Switch off DNSSEC at the old registrar](docs/move-dns.md#switch-off-dnssec-at-the-old-registrar)
   - [Add the zone](docs/move-dns.md#add-the-zone)
   - [Check the imported records](docs/move-dns.md#check-the-imported-records)
   - [Switch the nameservers](docs/move-dns.md#switch-the-nameservers)
3. [Transfer the registration](docs/transfer-registration.md)
   - [Domains ending in .uk](docs/transfer-registration.md#domains-ending-in-uk)
   - [.com and other generic domains](docs/transfer-registration.md#com-and-other-generic-domains)
   - [After the transfer: DNSSEC on](docs/transfer-registration.md#after-the-transfer-dnssec-on)
   - [Domains that stay at another registrar](docs/transfer-registration.md#domains-that-stay-at-another-registrar)
4. [Zone settings for every domain](docs/zone-settings.md)
5. [The tunnel (once per server)](docs/tunnel.md)
6. [Serve a site through the tunnel](docs/serve-site-through-tunnel.md)
   - [Routes](docs/serve-site-through-tunnel.md#routes)
   - [DNS records for the site](docs/serve-site-through-tunnel.md#dns-records-for-the-site)
   - [Redirect www to the bare domain](docs/serve-site-through-tunnel.md#redirect-www-to-the-bare-domain)
7. [nginx on the server](docs/nginx.md)
   - [The changes for each site](docs/nginx.md#the-changes-for-each-site)
   - [Every site listens the same way](docs/nginx.md#every-site-listens-the-same-way)
   - [The default server](docs/nginx.md#the-default-server)
   - [Certificates and closing the public ports](docs/nginx.md#certificates-and-closing-the-public-ports)
8. [Forge nginx template](docs/forge-nginx-template.md)
9. [Domains that only redirect](docs/redirect-only-domains.md)
   - [DNS records](docs/redirect-only-domains.md#dns-records)
   - [The redirect rule](docs/redirect-only-domains.md#the-redirect-rule)
10. [Serve an MTA-STS policy from a Worker](docs/mta-sts-policy.md)
    - [The records](docs/mta-sts-policy.md#the-records)
    - [The Worker](docs/mta-sts-policy.md#the-worker)
    - [The hostname](docs/mta-sts-policy.md#the-hostname)
    - [Reports without an inbox](docs/mta-sts-policy.md#reports-without-an-inbox)
11. [Checks](docs/checks.md)
12. [Troubleshooting](docs/troubleshooting.md)
13. [Sources](docs/sources.md)

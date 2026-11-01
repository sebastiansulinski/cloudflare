[Contents](../README.md)

# Before you start

Collect for each domain:

- [ ] Where it is registered (Namecheap, for the domains so far) and a login there.
- [ ] Whether it serves a site, only redirects, or does nothing.
- [ ] Whether it handles email, and with whom (Google Workspace, Amazon Simple Email Service).
- [ ] For a site: the server it runs on, and whether that server already has a tunnel.

Order of work, so nothing goes offline:

1. [Move the Domain Name System (DNS) records](move-dns.md), with the site or redirect
   ready in Cloudflare before the nameservers change.
2. Only then [transfer the registration](transfer-registration.md). Cloudflare
   needs an active zone first, and the transfer itself changes nothing visitors see.

---

Next: [Move a domain's DNS to Cloudflare](move-dns.md)

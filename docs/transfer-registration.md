[Contents](../README.md)

# Transfer the registration

Only once the zone is **Active**. Visitors notice nothing: the Domain Name System (DNS) is already
Cloudflare's.
A domain transferred to Cloudflare cannot move to another registrar for 60 days.

## Domains ending in .uk

`.uk` domains move by changing the domain's Internet Provider Security (IPS) tag at Nominet,
the `.uk` registry, not by an authorisation code. The order matters:

1. Cloudflare, account home: **Domain Registration, Transfer Domains**, select the domain and
   complete checkout. No fee, and no year is added.
2. **Only after checkout**, set the domain's IPS tag to the value below at the current
   registrar (Namecheap: under the domain's **Sharing & Transfer**, or ask its live chat
   support). A tag change before checkout makes Cloudflare reject the transfer.

   ```
   CLOUDFLARE
   ```

3. Cloudflare completes it when Nominet records the change, usually within hours. Ask the
   registrar again if nothing happens within 24 hours; the transfer cancels itself after
   30 days.

## .com and other generic domains

1. Namecheap: turn the domain's **Registrar Lock** off, then request the authorisation code
   under **Sharing & Transfer**. It arrives by email.
2. Cloudflare: **Domain Registration, Transfer Domains**, select the domain, enter the code,
   pay. The price is one year at Cloudflare's cost, added to the current expiry date.
3. Approve the transfer from Namecheap's confirmation email. Without approval it completes on
   its own after about 5 days.

```bash
DOMAIN=example.com
whois "$DOMAIN" | grep -iE "registrar:|status|dnssec" | sort -u
```

`Registrar: Cloudflare, Inc.` means it is done.

## After the transfer: DNSSEC on

Domain Name System Security Extensions (DNSSEC) signs the domain's records, so they cannot
be forged on the way to a visitor.

**Domain, DNS, Settings, DNSSEC: enable.** With Cloudflare as registrar, it publishes the
signing record at the registry itself, with nothing to copy.

---

Previous: [Move a domain's DNS to Cloudflare](move-dns.md) | Next: [Zone settings for every domain](zone-settings.md)

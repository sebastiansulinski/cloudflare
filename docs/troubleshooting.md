[Contents](../README.md)

# Troubleshooting

DNS is the Domain Name System, DNSSEC its Security Extensions, `CNAME` a canonical name
(alias) record and `TXT` a text record.

| Symptom | Cause | Fix |
|---|---|---|
| Lookups fail for everyone right after the nameserver change | DNSSEC was still on at the old registrar | [Switch off DNSSEC at the old registrar](move-dns.md#switch-off-dnssec-at-the-old-registrar); switching it off now fixes it once the cached signing record expires |
| `https://example.co.uk/` redirects to itself (301 to the same address, page signed "nginx") | nginx's port 80 block redirects to HTTPS; the tunnel arrives on port 80 | [The changes for each site](nginx.md#the-changes-for-each-site) step 1 |
| 502 error with an `https://localhost:443` route | No **Origin Server Name** on the route, so the certificate doesn't match | Set it ([Routes](serve-site-through-tunnel.md#routes)), or switch the route to `http://localhost:80` |
| Redirect-only domain times out, or shows 522 or 523 | The redirect rule isn't matching: both hostnames in one value, rule in another zone, rule not deployed, or the wrong rule type | [The redirect rule](redirect-only-domains.md#the-redirect-rule) |
| "Visitors cannot reach www" in **Recommendations** | The `www` route created no record | Add it ([DNS records for the site](serve-site-through-tunnel.md#dns-records-for-the-site)) |
| Adding a route fails: record already exists | An imported `A` or `CNAME` has the same name | Delete it, save the route again |
| Works for others, times out on your own computer | Your computer still caches the old address | Flush macOS's cache (below); in Chrome also `chrome://net-internals/#dns`, **Clear host cache** |
| Page unstyled, `http://` links in the page | PHP doesn't know the request is HTTPS | [The changes for each site](nginx.md#the-changes-for-each-site) step 4 |
| A contact form's limit hits every visitor at once; logs show `127.0.0.1` | nginx doesn't take the address from cloudflared | [The changes for each site](nginx.md#the-changes-for-each-site) step 3 |
| Two `x-frame-options` headers | nginx adds `SAMEORIGIN`, the application `DENY` | [The changes for each site](nginx.md#the-changes-for-each-site) step 2 |
| "[email protected]" instead of an email address | Email Address Obfuscation is on | [Zone settings](zone-settings.md) |
| Google mail fails DomainKeys Identified Mail checks | The key is split across two `TXT` records | [Check the imported records](move-dns.md#check-the-imported-records) |
| Certificate renewal failure emails after the move | The old certificate is still set to renew | [Certificates and closing the public ports](nginx.md#certificates-and-closing-the-public-ports) |
| One site's traffic lands on another site | One server block listens on `127.0.0.1:80`, the rest on `80` | [Every site listens the same way](nginx.md#every-site-listens-the-same-way) |
| Zone active but the site and email are gone | The zone has no records (nothing was imported or added) | Re-create them from the old provider's list ([Record the current records](move-dns.md#record-the-current-records)) |

Flush macOS's DNS cache:

```bash
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

Test a hostname through Cloudflare regardless of what your computer caches:

```bash
DOMAIN=example.co.uk
IP=$(dig +short A "$DOMAIN" @1.1.1.1 | head -1)
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" --resolve "$DOMAIN:443:$IP" "https://$DOMAIN/"
```

---

Previous: [Checks](checks.md) | Next: [Sources](sources.md)

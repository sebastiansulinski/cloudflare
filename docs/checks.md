[Contents](../README.md)

# Checks

After a site goes live through the tunnel. Each block sets its variables first. DNS is the
Domain Name System; `DS` is the Delegation Signer record, `MX` mail exchange, `TXT` text.

**DNS and registration:**

```bash
DOMAIN=example.co.uk
REGISTRY=$(dig +short NS "${DOMAIN#*.}" | head -1)
echo "== nameservers at the registry"; dig +nocmd NS "$DOMAIN" @"$REGISTRY" +norec +noall +authority | awk '{print $5}'
echo "== signing record at the registry"; dig +nocmd DS "$DOMAIN" @"$REGISTRY" +norec +noall +answer | grep DS
for t in A MX TXT; do echo "== $t"; dig +short $t "$DOMAIN"; done
echo "== www"; dig +short A "www.$DOMAIN"
```

**Redirects and status** (expects `301` to `https://` for the first three, `200`, then
`404`):

```bash
DOMAIN=example.co.uk
for u in "http://$DOMAIN/" "http://www.$DOMAIN/" "https://www.$DOMAIN/" "https://$DOMAIN/" "https://$DOMAIN/does-not-exist"; do
  printf "%-45s " "$u"; curl -s -o /dev/null -m 20 -w "%{http_code} %{redirect_url}\n" "$u"
done
```

**Headers** (expects `server: cloudflare`, one `x-frame-options`, no `x-xss-protection`,
cookies marked `secure`):

```bash
DOMAIN=example.co.uk
curl -sI "https://$DOMAIN/" | grep -iE "^(server|x-frame|x-xss|strict-transport|x-content-type|content-security|set-cookie)" | cut -c1-160
```

**No insecure links in the page** (expects nothing):

```bash
DOMAIN=example.co.uk
curl -s "https://$DOMAIN/" | grep -oE '(src|href|content)="http://[^"]+'
```

**Real visitor addresses** (on the server): the site's access log shows visitors'
addresses, not `127.0.0.1`.

**Email signing key is one record** (expects one line, starting `"v=DKIM1;`):

```bash
DOMAIN=example.co.uk
dig +short TXT "google._domainkey.$DOMAIN"
```

---

Previous: [Serve an MTA-STS policy from a Worker](mta-sts-policy.md) | Next: [Troubleshooting](troubleshooting.md)

[Contents](../README.md)

# Serve a site through the tunnel

Each site gets two routes on the server's tunnel and the Domain Name System (DNS) records
that point at them.

## Routes

**Tunnels**, the server's tunnel, tab **Routes**, **Add route**, **Published application**.
One route for the bare domain, one for `www`:

| Field | Bare domain | www |
|---|---|---|
| Subdomain | empty | `www` |
| Domain | `example.co.uk` | `example.co.uk` |
| Path | empty | empty |
| Service URL | `http://localhost:80` | `http://localhost:80` |

Leave **Additional application settings** on their defaults: cloudflared passes the
visitor's `Host` header through, so nginx picks the site by `server_name`.

Plain HTTP on port 80 is the standard here: the connection from cloudflared to nginx never
leaves the server, so encrypting it adds nothing, and there is no certificate to renew. It
needs [the nginx changes for each site](nginx.md#the-changes-for-each-site).

<details>
<summary>The alternative: <code>https://localhost:443</code></summary>

Only for a site whose nginx must keep its HTTPS block (a certificate already in place, no
nginx changes wanted). Each route then needs **Additional application settings, TLS, Origin
Server Name** set to that route's full hostname (`example.co.uk`, `www.example.co.uk`), and
**No TLS Verify** left off. Without it cloudflared asks nginx for `localhost`, gets the default
server's certificate, rejects it, and visitors see a 502 error. The certificate then has to
keep renewing, and Let's Encrypt's HTTP validation would have to pass through the tunnel.

</details>

## DNS records for the site

Adding a route creates a proxied record pointing at the tunnel (type **Tunnel**, or `CNAME`
to `<tunnel id>.cfargotunnel.com`). Check **DNS, Records**:

- The bare domain and `www` each have one, orange cloud.
- If a route's record is missing (the **Recommendations** box says "Visitors cannot reach
  www..."), add it: **Add record**, type **Tunnel** and pick the tunnel, or type `CNAME`,
  name `www`, target `<tunnel id>.cfargotunnel.com` (the id is on the tunnel's
  **Overview**), **Proxied**.
- A route cannot be saved while an `A` or `CNAME` record with the same name exists: delete
  the old record first. That deletion is the switch-over.
- The information icon on the bare domain's record ("CNAME flattening") is expected and needs
  nothing.

## Redirect www to the bare domain

Either in nginx (a `www` server block with `return 301 https://example.co.uk$request_uri;`)
or, tidier, in Cloudflare so the request never reaches the server: **Rules, Redirect Rules,
Create rule**, the template **Redirect from WWW to root**, or by hand:

- **When:** custom filter, **Hostname**, **equals**, `www.example.co.uk`
- **Then:** Dynamic, expression below, **301**, **Preserve query string** on

```
concat("https://example.co.uk", http.request.uri.path)
```

---

Previous: [The tunnel (once per server)](tunnel.md) | Next: [nginx on the server](nginx.md)

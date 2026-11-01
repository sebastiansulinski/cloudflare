[Contents](../README.md)

# Zone settings for every domain

In the zone (account home, then the domain). TLS is Transport Layer Security, the encryption
between visitors and Cloudflare; the dashboard labels it "SSL", after the older Secure
Sockets Layer.

| Setting | Where | Value | Cloudflare's default |
|---|---|---|---|
| SSL/TLS encryption mode | **SSL/TLS, Overview** | Full (strict) | varies |
| Always Use HTTPS | **SSL/TLS, Edge Certificates** | On | Off |
| Minimum TLS Version | same page | TLS 1.2 | TLS 1.0 |
| HTTP Strict Transport Security (HSTS) | same page | Not enabled | Not enabled |
| Email Address Obfuscation | **Security, Settings**, group **Client side abuse** | Off | On |
| Bot fight mode | **Security, Settings**, group **Bot traffic** | Off | Off |
| Rocket Loader | **Speed, Settings**, tab **Content Optimization** | Off | Off |

Why:

- **Encryption mode** does not affect tunnel traffic (the connection from Cloudflare to
  cloudflared is encrypted on its own, and the route's service address decides the rest).
  Full (strict) is the safe value should a record ever point straight at a server.
- **Always Use HTTPS** sends every `http://` request to `https://` at Cloudflare's edge, so
  nginx never needs its own redirect (and must not have one, see [the changes for each site](nginx.md#the-changes-for-each-site)).
- **Minimum TLS Version 1.2.** TLS 1.0 and 1.1 are retired; every
  browser in use supports 1.2.
- **Strict Transport Security stays off in Cloudflare.** The site sends its own header (from
  nginx or the application). Cloudflare's setting covers the whole zone, and with "include
  subdomains" or "preload" it forces HTTPS on every subdomain for the whole period, which
  preload makes very hard to undo.
- **Email Address Obfuscation, Bot fight mode and Rocket Loader inject Cloudflare scripts**
  into the page. A site with a strict Content Security Policy blocks them; obfuscation then
  shows "[email protected]" wherever the page has an email address.
- **Caching** stays at the default: static files only, never HTML, which carries form tokens.

---

Previous: [Transfer the registration](transfer-registration.md) | Next: [The tunnel (once per server)](tunnel.md)

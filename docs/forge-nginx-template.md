[Contents](../README.md)

# Forge nginx template

A Forge nginx template carries [the changes for each site](nginx.md#the-changes-for-each-site), so a site
created with it needs no hand edits.

**Create it once per server:** the server's dashboard, **Nginx Templates**, **New template**.
Forge fills in its default template; name the copy `Cloudflare Tunnel` and edit it:

1. Delete the `X-Frame-Options` and `X-XSS-Protection` lines.
2. Add the three `real_ip` lines below `add_header X-Content-Type-Options "nosniff";`.
3. Add `fastcgi_param HTTPS on;` inside the `location ~ \.php$` block, after its `include`
   line(s).
4. For Laravel sites that generate `robots.txt`: replace the `robots.txt` line as in
   [the changes for each site](nginx.md#the-changes-for-each-site).
5. Leave the `listen`, `server_name`, `root` and socket lines on Forge's variables, and
   leave Forge's `# FORGE ... (DO NOT REMOVE!)` comment lines in place: Forge uses them to
   find where its own includes go.

The changes are the same on every site; only Forge's variables differ. They must be written
exactly as below, with no spaces inside the braces:

| Variable | Becomes |
|---|---|
| `{{DOMAINS}}` | the site's domains, aliases included: `example.co.uk www.example.co.uk` |
| `{{SITE}}` | the site's name, without aliases: `example.co.uk` |
| `{{SITE_ID}}` | the site's id in Forge: `12345` |
| `{{PATH}}` | the web directory's full path: `/home/<user>/example.co.uk/public` |
| `{{ROOT_PATH}}` | the site's root: `/home/<user>/example.co.uk` |
| `{{DIRECTORY}}` | the web directory: `/public` |
| `{{PORT}}` | `80` (with `default_server` for the `default` site) |
| `{{PORT_V6}}` | `[::]:80` (likewise) |
| `{{PROXY_PASS}}` | the PHP socket: `unix:/var/run/php/php8.4-fpm.sock` |
| `{{USER}}` | the site's user: `forge`, or the isolated user |
| `{{SERVER_PUBLIC_IP}}`, `{{SERVER_PRIVATE_IP}}` | the server's addresses |

**Use it for each new site:** choose the template when creating the site in Forge, with **no
certificate** (Let's Encrypt cannot validate through the tunnel, and Cloudflare provides
HTTPS) and **no Domain Name System (DNS) change** (ignore any Forge prompt to point the domain at the server: an
`A` record bypasses the tunnel). Then add the site's routes ([Routes](serve-site-through-tunnel.md#routes)).

Things to know:

- **Editing a template does not change sites already created with it.** Edit those by hand
  ([The changes for each site](nginx.md#the-changes-for-each-site)).
- **An invalid template stops nginx**, taking every site on the server down. After creating
  the first site from a new or edited template, check straight away:

  ```bash
  sudo nginx -t
  ```

- Forge may rewrite a site's configuration when some of its settings change. After any
  change to a site in Forge, check [the changes for each site](nginx.md#the-changes-for-each-site) are still
  there:

  ```bash
  sudo nginx -T 2>/dev/null | grep -nE "real_ip_header|fastcgi_param HTTPS|X-Frame|X-XSS"
  ```

---

Previous: [nginx on the server](nginx.md) | Next: [Domains that only redirect](redirect-only-domains.md)

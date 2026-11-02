[Contents](../README.md)

# Domains that only redirect

A domain that sends every request to another domain (an old brand, a `.com` beside a
`.co.uk`) needs no server at all: Cloudflare answers with the redirect at its edge. Below, DNS is the
Domain Name System and TLS is Transport Layer Security.

## DNS records

Delete every imported record, then add exactly these. The generator in `site/index.html`
([Generator page](#generator-page)) writes them as a file to import.

| Type | Name | Content | Proxy | Why |
|---|---|---|---|---|
| `A` | `@` | `192.0.2.1` | **Proxied** | placeholder so Cloudflare answers for the bare domain |
| `A` | `www` | `192.0.2.1` | **Proxied** | placeholder for `www` |
| `MX` | `@` | `.` (priority `0`) | DNS only | "null MX": the domain accepts no email |
| `TXT` | `@` | `v=spf1 -all` | DNS only | no server may send email as this domain |
| `TXT` | `_dmarc` | `v=DMARC1; p=reject;` | DNS only | receivers reject anything claiming to be from it |

- `192.0.2.1` is an address reserved for documentation, which never reaches anything. A
  redirect rule only runs for proxied hostnames, and the placeholder is what makes them
  proxied. If the rule ever stops matching, visitors get a Cloudflare 522 or 523 error
  (Cloudflare trying to reach the placeholder), which shows in a browser as a timeout.
- The three email records are for a domain that **handles no email**. If it does, keep its
  real `MX`, Sender Policy Framework, DomainKeys Identified Mail and Domain-based Message
  Authentication, Reporting and Conformance (DMARC) records instead.

**Zone settings.** Cloudflare still answers every request itself, encryption included, so
set these as for any zone ([Zone settings for every domain](zone-settings.md)):

| Setting | Where | Value |
|---|---|---|
| Always Use HTTPS | **SSL/TLS, Edge Certificates** | On |
| Minimum TLS Version | same page | TLS 1.2 |
| HTTP Strict Transport Security (HSTS) | same page | Not enabled |

The settings that inject scripts (Email Address Obfuscation, Bot fight mode, Rocket Loader)
make no difference here: the domain serves no pages. The encryption mode doesn't matter
either, since nothing is passed on to a server.

## The redirect rule

**Rules, Redirect Rules, Create rule:**

- **Rule name:** `Redirect to example.co.uk`
- **If incoming requests match:** **Custom filter expression**: field **Hostname**,
  operator **is in**, and the two hostnames **as two separate values**: type one, press
  **Enter** so it becomes its own chip, then the other.
- **Then:** **Dynamic**, the expression below, status **301**, **Preserve query string**
  on.

The expression preview must show two separately quoted names with a space between them:

```
(http.host in {"old-domain.com" "www.old-domain.com"})
```

Or click **Edit expression** and paste that line, with the real names. One quoted string
containing a comma (`{"old-domain.com,www.old-domain.com"}`) matches nothing, so every
request falls through to the placeholder and times out.

The target, sending every path to the same path on the new domain:

```
concat("https://example.co.uk", http.request.uri.path)
```

Point it at the final address: if `example.co.uk` itself redirects to
`www.example.co.uk`, use that, so visitors make one hop instead of two.

Each zone has its own rules. A rule created in one domain's zone does nothing for another.

Check (expects `301` to the same path on the target, query string kept, for both
hostnames and both schemes):

```bash
FROM=old-domain.com
for u in "http://$FROM/" "https://$FROM/about?x=1" "http://www.$FROM/" "https://www.$FROM/about?x=1"; do
  printf "%-45s " "$u"; curl -s -o /dev/null -m 20 -w "%{http_code} %{redirect_url}\n" "$u"
done
```

## All of it in one command

`bin/setup-redirect-domain` does this whole page for a zone already on Cloudflare: the five DNS
records, the zone settings above (plus Full (strict) encryption, Email Address Obfuscation and
Rocket Loader off), and the redirect rule. Then it tests the four redirects (both hostnames,
both schemes, a path and a query string). Run it again any time: what is already right is left
alone.

```bash
~/code/cloudflare/bin/setup-redirect-domain --dry-run old-domain.com example.co.uk
~/code/cloudflare/bin/setup-redirect-domain old-domain.com example.co.uk
```

- `--dry-run` reads the zone and prints every change it would make, changing nothing.
- It stops if the zone has other DNS records or other redirect rules, and lists them.
  `--replace` deletes those records (in the same request that adds the five, so either all of
  it applies or none) and replaces the rules.
- `--no-check` skips the final test, for a zone whose nameservers have not switched yet.

**The token.** Cloudflare, **My Profile, API Tokens, Create Token, Create Custom Token**, with
these zone permissions (the labels in the token screen may read "Write" where these say
"Edit"):

| Permission | For |
|---|---|
| Zone, Zone, Read | finding the zone |
| Zone, DNS, Edit | the records |
| Zone, Zone Settings, Edit | encryption and the script-injecting features |
| Zone, Single Redirect, Edit | the redirect rule (also listed as "Dynamic URL Redirects") |

Limit **Zone Resources** to the zones it is for, or to all zones in the account if it is kept
for every new domain.

Put it in `.env` at this repository's root, once: copy `.env.example` to `.env` and paste the
token after `CLOUDFLARE_API_TOKEN=`. `.env` is gitignored, so it never reaches GitHub, and the
script reads only that one line from it. A token exported in the terminal takes precedence.
Keep a copy in the password manager too: Cloudflare shows a token only once.

The script needs `curl` and `jq`, and runs with macOS's own bash.

## Generator page

`site/index.html` is a single page: type the domain, then **Download** gives a file in the same shape as Cloudflare's own DNS export, with the five
records above. In the zone: **DNS, Records**, delete the imported records, then **Import and
Export**, **Import DNS records**, and choose the file. Leave **Proxy imported DNS records**
unticked: each `A` record carries `cf_tags=cf-proxied:true`, which sets its orange cloud. The
file has no `SOA` or `NS` records; Cloudflare manages those for every zone. A link such as
`?domain=old-domain.com` opens it filled in.

It is served by Cloudflare Pages from this repository, behind Cloudflare Access:

1. **Workers & Pages, Create, Pages, Connect to Git**: this repository, production branch
   `main`, framework preset **None**, no build command, build output directory `site`.
   Every push to `main` redeploys it.
2. **Zero Trust, Access, Applications, Add an application, Self-hosted**: the project's
   `<project>.pages.dev` hostname, plus `*.<project>.pages.dev` for preview deployments, with
   a policy that allows only your own email address. Without it, anyone with the address can
   open the page.

---

Previous: [Forge nginx template](forge-nginx-template.md) | Next: [Serve an MTA-STS policy from a Worker](mta-sts-policy.md)

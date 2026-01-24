---
name: sst-custom-domains
description: Configure custom domains and routing for SST components. Use when setting domain names on frontends/APIs/services/routers, managing DNS adapters (Route 53, Cloudflare, Vercel), or handling redirects and subdomain routing.
---
# SST custom domains

## Scope
- Configure custom domains for AWS-based components (frontends, APIs, services, routers).
- Select DNS adapters for Route 53, Cloudflare, or Vercel.
- Set up redirects and subdomain routing with `Router`.

## Workflow
1. Decide which component owns the domain (frontend, API, service, router).
2. Add the `domain` field (string or object) to the component configuration.
3. If using a supported DNS provider, select the appropriate adapter:
   - Route 53: default or `sst.aws.dns()` with optional hosted zone.
   - Cloudflare: add provider, then `sst.cloudflare.dns()`.
   - Vercel: add provider, set `VERCEL_API_TOKEN`, then `sst.vercel.dns()`.
4. For redirecting `www` to apex, use `domain.redirects` on a `Router`.
5. For subdomain routing, set `domain.aliases` on a `Router` and connect other components through it.
6. For unsupported DNS providers, document manual DNS record setup.

## Outputs
- Updated `sst.config.ts` with domain configuration and DNS adapters.
- Notes describing DNS setup, redirects, and subdomain routing decisions.

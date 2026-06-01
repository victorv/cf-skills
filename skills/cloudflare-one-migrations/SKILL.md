---
name: cloudflare-one-migrations
description: Plans migrations from Zscaler ZIA/ZPA, Palo Alto, legacy VPN, SWG, or SASE stacks to Cloudflare One. Use for migration assessments, policy mapping, rollout plans, and parity/gap analysis.
---

# Cloudflare One Migrations

Retrieve current Cloudflare docs, Cloudflare API schemas, and source-vendor export docs before generating exact configuration.

## Workflow

1. Identify the source stack: Zscaler ZIA, Zscaler ZPA, Palo Alto NGFW/Prisma/GlobalProtect, legacy VPN/SWG/SD-WAN, or other.
2. Request exports and logs before mapping. Prefer structured exports over screenshots or prose summaries.
3. Build an inventory: identities, groups, apps, destinations, connectors/tunnels, DNS/URL/firewall/DLP/TLS policies, objects/lists, locations/sites, exceptions, hit counts, and compliance logging.
4. Produce a mapping plan: source object, Cloudflare One target resource, confidence, prerequisites, unsupported/partial mappings, and manual decisions.
5. Create dependencies first: identity/[SCIM](https://developers.cloudflare.com/cloudflare-one/team-and-resources/users/scim/), connectors/on-ramps, routes/DNS, lists/objects, TLS bypasses, Access apps/policies, Gateway policies, DLP/CASB, logging.
6. Stage safely: use a migration prefix, create disabled/audit-mode rules by default, pilot with small groups/sites, compare logs, then expand rollout.
7. Account for every source rule. Each rule must map to a Cloudflare object or an explicit Not Migrated row with reason and security impact.

## Exports To Ask For

- ZIA: URL filtering, firewall filtering, SSL inspection, DLP, custom URL categories, IP groups, network services/service groups, users/groups/departments, locations, GRE tunnels, and static IPs.
- ZPA: app segments, segment groups, server groups, app connectors/connector groups, access policies, IdP/group mapping, private DNS domains, ports, and protocols.
- Palo Alto/Prisma: security/NAT/decryption rules, address/service objects and groups, URL categories, HIP profiles, GlobalProtect config, Prisma Access remote network/service connection config, zones, tags, logs, and hit counts.

## Mapping Heuristics

- ZIA/SWG policies usually map to [Gateway traffic policies](https://developers.cloudflare.com/cloudflare-one/traffic-policies/) and Gateway lists.
- ZPA private app access usually maps to [Access application types](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/choose-application-type/), [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/), WARP routing/DNS, and [Access policies](https://developers.cloudflare.com/cloudflare-one/access-controls/policies/).
- Palo Alto rules map only after understanding traffic direction, zones, objects, users, apps, decryption, and hit counts. Do not flatten zones blindly into lists.
- Legacy VPN replacement is usually Access + WARP + Tunnel for app access. Use [Cloudflare WAN](https://developers.cloudflare.com/cloudflare-wan/) only when site-to-site traffic is required; use the [Network VPN migration design guide](https://developers.cloudflare.com/reference-architecture/design-guides/network-vpn-migration/) and [Replace your VPN](https://developers.cloudflare.com/cloudflare-one/setup/replace-vpn/) docs for current patterns.

## Migration Assessment Prompts

- Source coverage: which products are in scope, which exports are available, and whether screenshots/prose summaries are hiding missing object files.
- Rule volume and hit data: counts by rule type, disabled/stale rules, no-hit rules, high-hit rules, and business-critical exceptions.
- Object dependencies: address objects, service objects, groups, custom categories, network services, app IDs, zones, tags, connectors, and server groups.
- Identity readiness: IdP, SCIM/group sync, group-name normalization, individual-user rules, local groups, service accounts, and contractor identities.
- TLS/DLP readiness: source decryption rules, certificate-pinned bypasses, [DLP](https://developers.cloudflare.com/cloudflare-one/data-loss-prevention/) engines/profiles, custom regex, exact-match data, and payload logging expectations.
- Connectivity readiness: source tunnels/connectors, private DNS, [Split Tunnels](https://developers.cloudflare.com/cloudflare-one/team-and-resources/devices/cloudflare-one-client/configure/route-traffic/split-tunnels/) or bypass behavior, source IP preservation, [egress IP](https://developers.cloudflare.com/cloudflare-one/traffic-policies/egress-policies/) allowlists, and site-to-site requirements.
- Rollout readiness: pilot groups/sites, parallel-run period, rollback owner, source-stack decommission criteria, and monitoring/log comparison plan.

## Source-Specific Traps

### Zscaler ZIA / SWG

- Custom URL categories often split into separate IP, domain, and URL lists. Count the generated lists, not just source categories.
- ZIA locations with IPs are useful as source IP lists; they are not automatically [Gateway DNS locations](https://developers.cloudflare.com/cloudflare-one/networks/resolvers-and-proxies/dns/locations/) for DNS policy scoping.
- GRE tunnel source IPs can inform policy conditions, but the transport migration is a separate WARP Connector or Cloudflare WAN workstream.
- CAUTION/warn behavior has no exact Gateway equivalent. Treat it as an explicit customer decision, not a silent allow/block choice.
- DLP engines and custom regex usually require manual Cloudflare DLP profile recreation. Placeholder policies must not be enabled as if DLP is complete.
- Network application groups and unsupported protocols are partial mappings. Review them before enablement.
- If SCIM is unavailable, identity-scoped source rules become overly broad unless you add an enforceable alternative such as user/email lists. Check [Gateway identity selectors](https://developers.cloudflare.com/cloudflare-one/traffic-policies/identity-selectors/) before creating those rules.

### Zscaler ZPA / Private Access

- ZPA app segments, server groups, and connector groups do not map 1:1. Cloudflare separates Access apps, tunnel routes, DNS, and policies.
- Creating tunnels through the API does not complete connector deployment. Plan cloudflared installation, authentication, and origin reachability separately.
- ZPA bypass means split-tunnel bypass in Cloudflare, not an Access `bypass` decision.
- Agentless/browser apps may become separate public-hostname Access apps per domain. WARP private apps remain private-destination apps.
- IP-anchored apps require an explicit egress choice: preserve source IP through customer egress or use Cloudflare [dedicated egress](https://developers.cloudflare.com/cloudflare-one/traffic-policies/egress-policies/) where available.
- Resolver policies can be account-wide. Be careful with overlapping private DNS namespaces across sites or virtual networks; retrieve [resolver policy](https://developers.cloudflare.com/cloudflare-one/traffic-policies/resolver-policies/) docs before making DNS changes.
- In combined ZIA and ZPA migrations, Gateway Network rules can accidentally block Access private-app traffic. Add and validate an explicit Access private-app allow path before enabling broad L4 blocks.

### Palo Alto / Prisma / NGFW

- One Palo Alto rule can produce multiple Cloudflare resources. Preserve rule intent, not rule count.
- App-ID, URL category, zone, HIP, schedule, and decryption behavior rarely translate exactly. Mark partial mappings rather than forcing false equivalence.
- Export address/service objects and groups with rules. Missing object exports cause silent-looking drops unless explicitly detected.
- Broad `any` destination/service rules and very broad CIDRs require manual review. Do not auto-create broad catchalls.
- HIP/device checks require Cloudflare [device posture](https://developers.cloudflare.com/cloudflare-one/reusable-components/posture-checks/) integrations before enforcement.

## Gotchas

- Source exports often split references across files. Resolve IDs against object, service, and group files before declaring a rule unmappable.
- Individual users, local groups, departments, and dynamic application IDs often need identity normalization. SCIM/group sync is the gating prerequisite for group selectors.
- Zscaler caution/warn behavior, Palo Alto App-ID behavior, and TLS/decryption exceptions may not have exact equivalents. Flag them as decision points instead of forcing a 1:1 mapping.
- Preserve source rule order and hit counts where available. Disable or delete stale/no-hit rules only with user approval.
- Never create broad allow-all catchalls to preserve connectivity unless explicitly requested and time-limited.

## Validation Gates

- After each migration stage, compare Cloudflare object counts against parsed source counts. Stop on mismatches.
- Review every `unsupported`, `partial`, `unmapped`, `needs_identity`, `needs_posture`, and `manual_review` item before enabling policies.
- Validate group matching with real pilot users after SCIM sync and re-authentication.
- Test TLS inspection and Do Not Inspect behavior before enabling HTTP/DLP blocks broadly.
- Keep rollback paths explicit: disable migrated rules by prefix, restore source routing, or revert the pilot group/site.
- Before declaring done, produce a source-rule accounting table: migrated object, partial mapping, not migrated reason, security impact, and owner for each manual action.

## Assessment Template

```markdown
## Migration Assessment

Source stack:
Artifacts reviewed:
Assumptions / missing exports:
Recommended Cloudflare One target:
Mapping summary:
Risks / partial mappings:
Not migrated:
Pilot plan:
Validation:
Rollback:
```

# ADR-0007: Vega-only MVP with a `sites` placeholder

**Status:** Accepted

## Context

Franchisee-operated stations are a real future plan, but building multi-tenancy now (tenant isolation, franchisee RBAC, revenue splits, white-labelling) would delay the MVP. The hardware manufacturer supplies hardware only and has no platform role.

## Decision

For MVP, all stations are Vega-owned and operated. No tenant model is built. A lightweight `sites` table is created now with `charge_points.site_id` as a foreign key, so that franchising later is an additive migration (`tenants`, `sites.tenant_id`) rather than re-parenting every charger. Hardware vendor is an attribute of the charger record, required and with no default.

## Consequences

- Phase 1 schema includes `sites` even though every row belongs to Vega.
- RBAC (Phase 2) is designed around Vega roles only. Station operators are scoped to assigned sites, which maps cleanly onto franchising later.
- Tenant isolation model, onboarding and white-labelling remain open.

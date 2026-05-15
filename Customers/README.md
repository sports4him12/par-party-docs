# Customers

Design docs for paying GolfSync customers — one doc per hosted
tournament. Each doc captures: what the customer's tournament actually
is, the architectural decisions (with alternatives considered), the
data model additions, what reuses existing infrastructure vs. what's
built fresh, scope-of-work phasing, open questions for the customer,
and cross-references.

These docs are the artifact for getting customer alignment **before**
any code lands. They double as the implementation playbook once
alignment is reached.

## Index

- [C2 Adopt @ The Federal Club — September 2026](c2-adopt-federal-2026.md)
  — First real hosted tournament. Captain's Choice scramble, AM/PM
  flights, role-gated pre-launch, full registration + sponsor + payment
  + reconciliation surface.

## Convention

When a new customer signs on:

1. Copy `c2-adopt-federal-2026.md` as the template
2. Fill in the customer-specific format, dates, payment needs, flight
   structure
3. Capture the architectural decisions (and what was considered + rejected)
4. List open questions for the customer interview
5. Save under `Customers/<customer-name>-<event-or-year>.md`
6. Link from this README

The first hosted tournament (C2 Adopt) was the chance to lay down the
patterns — every subsequent customer should be able to reuse most of
the surface (microsite + admin + mobile day-of + payment intake + role
gating). Only the customer-specific data + branding should change.

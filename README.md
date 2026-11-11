# Wiply Account (accountfederation)

Single account system for all Wiply products (Cassy AI, Wiply Assistant, Wiply Studio, the wiply.net social network and future SaaS).

- **Identity**: Keycloak (`auth.wiply.net`, realm `wiply`) — login, MFA, social login, sessions, Organizations.
- **Wiply Account** (`account.wiply.net`): source of truth for entitlements — profile, organizations and members, active services, plans, seats, limits, usage, invoices.
- **Billing**: Stripe (subscriptions, per-seat pricing, metered usage, EU VAT).

Design document (Italian): [docs/progetto.md](docs/progetto.md)

Status: design phase, no code yet.

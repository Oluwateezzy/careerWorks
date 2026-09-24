# Penetration Test Scope (Template)

Engage a qualified firm before B2B production launch. Minimum scope:

## In-scope

- Authentication (login, refresh, logout, OTP, session fixation)
- Authorization (escrow participant checks, admin routes, IDOR on `/escrows/:id`)
- Payment flows (initialize, webhook HMAC bypass, duplicate funding)
- Wallet withdrawal limits and idempotency
- KYC document upload (MIME bypass, path traversal)
- API key scoping (when enabled)

## Out-of-scope (unless agreed)

- Physical security
- Social engineering of employees
- DDoS load testing

## Deliverables

- Executive summary + severity-rated findings
- Retest confirmation after fixes
- Signed letter for enterprise customers (optional)

## Pre-test checklist

- [ ] Staging environment mirrors production security config
- [ ] Test accounts provisioned (buyer, seller, admin)
- [ ] Paystack test mode keys only

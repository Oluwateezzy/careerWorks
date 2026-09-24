# SOC 2 Preparation Checklist

## Security (CC6)

- [x] JWT RS256 + HttpOnly refresh cookies
- [x] Server-side refresh session revocation (Redis)
- [x] Rate limiting on auth endpoints
- [x] Idempotency keys on money mutations
- [x] Paystack webhook HMAC verification
- [ ] MFA enforced for all admin accounts in production
- [ ] Annual penetration test report archived

## Availability (A1)

- [ ] Database backups with tested restore
- [ ] Redis persistence / failover documented
- [ ] Incident response runbooks (`docs/security/runbooks.md`)

## Processing Integrity (PI1)

- [x] Double-entry ledger via `RecordEntry`
- [x] Unified funding path through `WalletService.FundEscrow`
- [x] Daily reconciliation job
- [ ] Automated alerting on reconciliation mismatches

## Confidentiality (C1)

- [x] AES-256-GCM for BVN/NIN/document numbers
- [x] KYC uploads validated (type, size) and stored privately
- [ ] KMS-managed encryption keys (not env plaintext in prod)

## Privacy (P1)

- [ ] Data retention policy documented
- [ ] PII access logged on admin KYC review

## Change Management

- [ ] PR review required for money-path changes
- [ ] Staging environment mirrors production security config

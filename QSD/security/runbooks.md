# Security Runbooks

## Webhook processing failure

1. Check `webhook_failures` table for recent Paystack errors.
2. Verify `PAYSTACK_WEBHOOK_SECRET` matches Paystack dashboard.
3. Replay failed events manually after fixing root cause.
4. Run daily reconciliation logs for payment/escrow mismatches.

## Suspected account compromise

1. Revoke user refresh sessions via logout-all (Redis `session:refresh:*` keys).
2. Freeze wallet (`is_frozen = true`) via admin DB action.
3. Review `audit_logs` for actor actions in last 24h.
4. Force password reset and enable MFA for admin accounts.

## Ledger mismatch

1. Run reconciliation worker output in server logs.
2. Compare Paystack settlement report vs `escrow_payments` vs ledger entries.
3. Do not manually adjust balances — trace `RecordEntry` chain for escrow ID.

## API key leak

1. Revoke key immediately via `DELETE /api/v1/orgs/:id/api-keys/:keyId`.
2. Rotate organization credentials and review audit logs for anomalous usage.
3. Issue new key with reduced scopes.

## Paystack outage

1. Payout outbox entries remain in `pending`/`processing` — do not double-pay.
2. Pause withdrawal UI messaging; monitor outbox worker logs.
3. Resume processing when Paystack transfer API is healthy.

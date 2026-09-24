# JWT Key Rotation

## When to rotate

- Every 90 days (scheduled)
- Immediately if private key may be compromised
- Before major production launch

## Steps

1. Generate new RSA key pair:
   ```bash
   openssl genrsa -out keys/private.new.pem 2048
   openssl rsa -in keys/private.new.pem -pubout -out keys/public.new.pem
   ```

2. Deploy new public key alongside old (support dual verification window if implementing `jwt_previous_public_key` — not yet wired).

3. Replace `JWT_PRIVATE_KEY_PATH` / `JWT_PUBLIC_KEY_PATH` in environment.

4. Restart API servers — all users must re-login (refresh sessions invalidated).

5. Archive old keys in secure vault; never commit to git.

6. Update `ENCRYPTION_KEY` separately — requires re-encryption migration for PII (plan maintenance window).

## Verification

- Login + refresh flow works
- Existing access tokens rejected after rotation
- Audit log shows no auth errors spike post-rotation

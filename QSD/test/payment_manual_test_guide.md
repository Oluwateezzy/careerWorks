# Manual Testing Guide — Phase 6: Payment Module (Paystack Integration)

This guide provides step-by-step instructions for manually testing the Paystack integration endpoints using the **Swagger UI** interactive interface.

---

## Prerequisites

1. **Start Services**: Ensure your backend server, PostgreSQL database, and Redis are running:
   ```bash
   go run cmd/server/main.go
   ```
2. **Environment Variables**: Verify your `.env` file contains valid Paystack test keys:
   ```env
   PAYSTACK_SECRET_KEY=sk_test_24443382189235a6b5e964248b2403d0fb604710
   PAYSTACK_PUBLIC_KEY=pk_test_54449b5d2c82244ca8b6a7b66bed5ffed098062c
   PAYSTACK_WEBHOOK_SECRET=sk_test_24443382189235a6b5e964248b2403d0fb604710
   ```
3. **Escrow State**: Ensure you have an escrow transaction in the database with status `accepted` (e.g. from Phase 5). Obtain the `escrow_id` and the JWT `access_token` of the buyer.

---

## Step 1: Open and Authorize Swagger UI

1. Open your browser and navigate to the Swagger UI page:
   [http://localhost:3000/swagger/index.html](http://localhost:3000/swagger/index.html)
2. Click the green **Authorize** button at the top-right of the page.
3. In the input field, simply paste the raw JWT token of the buyer (e.g. `eyJhbGciOiJSUzI1NiIs...`). The Swagger UI is now configured to automatically prepend the `Bearer ` prefix for you!
4. Click **Authorize** and then click **Close**.

---

## Step 2: Initialize Payment in Swagger UI

1. In the **Payments** section, expand the `POST /api/v1/payments/initialize` endpoint.
2. Click **Try it out** on the right side.
3. Replace the request body template with your details:
   ```json
   {
     "escrow_id": "YOUR_ESCROW_UUID",
     "callback_url": "http://localhost:3000/api/v1/health"
   }
   ```
4. Click the blue **Execute** button.
5. In the **Server response** section, verify a `200 OK` response code is returned.
6. Copy both the `authorization_url` and the `reference` values from the response body.

---

## Step 3: Complete Sandbox Payment

1. Open a new browser tab and navigate to the `authorization_url` copied in Step 2.
2. On the Paystack test checkout screen, choose **Card** and select a simulated payment method (e.g., click the **Success** button).
3. Complete the checkout. You will be redirected back to the healthcheck callback URL.

---

## Step 4: Verify Payment Status in Swagger UI

1. Return to the Swagger UI page.
2. Expand the `GET /api/v1/payments/verify/{reference}` endpoint.
3. Click **Try it out**.
4. Paste the transaction `reference` copied in Step 2 into the `reference` parameter input box.
5. Click **Execute**.
6. Verify a `200 OK` response is returned and the `status` field in the response data is `"confirmed"`.
7. Check the database to confirm that the related escrow's status has transitioned to `"funded"`.

---

## Step 5: Simulate Webhook Signature Ingestion

Since webhooks are signed using HMAC-SHA512 checksums of the request body, you can trigger a simulated webhook using the Swagger UI by calculating the signature locally:

1. **Construct the Webhook Payload**:
   ```json
   {"event":"charge.success","data":{"reference":"VESC_REF_xxxxxx","amount":150000,"status":"success","gateway_response":"Successful"}}
   ```
   *(Ensure to replace `VESC_REF_xxxxxx` with your actual payment reference).*

2. **Generate HMAC Signature**:
   Compute the signature of the exact payload using the webhook secret key:
   ```bash
   echo -n '{"event":"charge.success","data":{"reference":"VESC_REF_xxxxxx","amount":150000,"status":"success","gateway_response":"Successful"}}' | openssl dgst -sha512 -hmac "sk_test_24443382189235a6b5e964248b2403d0fb604710"
   ```
   *Copy the resulting hex string.*

3. **Trigger Webhook in Swagger UI**:
   - In Swagger UI, expand the `POST /api/v1/payments/webhook/paystack` endpoint.
   - Click **Try it out**.
   - Input the copied hex signature into the `X-Paystack-Signature` header parameter box.
   - Paste the webhook JSON payload in the request body text field.
   - Click **Execute**.
   - Verify a `200 OK` response code is returned.

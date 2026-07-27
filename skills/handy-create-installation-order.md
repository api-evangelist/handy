---
name: Create and track a Handy installation order
description: >-
  Attach a fixed-price Handy in-home installation/service to a retail order,
  then retrieve its booking and handle lifecycle updates. Uses the Handy Partner
  Orders API with RSA-SHA256 signed-request authentication.
api: openapi/handy-orders-openapi.json
operations:
- POST /api/v1/orders          # Create Order
- GET /api/v1/orders/{partner_order_id}   # Retrieve Order
- GET /api/v1/bookings/{id}    # Retrieve Booking
- PATCH /api/v1/orders/{partner_order_id} # Update Order
- DELETE /api/v1/orders/{partner_order_id} # Cancel Order
---

# Create and track a Handy installation order

Base URL: `https://partners.services.handy.com/api/v1` (sandbox:
`services.handy-sandbox.com`). Contact `partners-eng@handy.com` for sandbox
access and a signing keypair.

## Authentication (every request)
Sign each request and send three headers:
- `HDY-PARTNER-ID` — your partner id.
- `HDY-TIMESTAMP` — seconds since epoch (UTC).
- `HDY-SIGNATURE` — `base64(rsa_sha256(private_key, PARTNER_ID + "\n" + URL + "\n" + HTTP_METHOD + "\n" + TIMESTAMP + "\n" + PAYLOAD))`.

See `authentication/handy-authentication.yml` and the reference signing code at
https://github.com/Handybook/API-Request-Signing.

## Steps
1. **Create the order** — `POST /api/v1/orders`. Body includes a caller-supplied
   `partner_order_id`, the customer `user` (email, first/last name, phone), the
   service `address`, and `products` (each with a `sku`, `order_details`
   including `quantity` and `price_amount_in_cents`, and `scheduling_details`).
   A `201` returns the created order with its booking(s). Reuse the same
   `partner_order_id` to avoid creating a duplicate (natural de-duplication —
   there is no idempotency-key header).
2. **Retrieve the order** — `GET /api/v1/orders/{partner_order_id}` to read
   current state and the associated booking id(s).
3. **Retrieve the booking** — `GET /api/v1/bookings/{id}` to see `starts_at`,
   `time_zone`, `duration_seconds`, and the assigned `provider` (id, name,
   profile_url).
4. **Amend if needed** — `PATCH /api/v1/orders/{partner_order_id}` to update
   order details, or `DELETE /api/v1/orders/{partner_order_id}` to cancel.
5. **React to lifecycle events** — register a webhook callback endpoint. Handy
   POSTs `booking_rescheduled`, `booking_completed`, `booking_cancelled`,
   `service_fulfilled`, and `job_claim_assigned` events (see
   `asyncapi/handy-webhooks.yml`).

## Error handling
Errors return `{message, code, error_uuid, more_info}` (not RFC 9457). On `4xx`,
read `code` (e.g. `user_details_invalid`) and `more_info` for field-level detail;
capture `error_uuid` to share with Handy support. `401` means the request
signature failed; `409`/`422` indicate order-state or validation conflicts. See
`errors/handy-problem-types.yml`.

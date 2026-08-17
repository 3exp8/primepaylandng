# Prime Fast Pay — OpenAPI Specification

OpenAPI 3.1 spec for the Prime Fast Pay Developer API, derived from [`docs.html`](docs.html).

- **Version:** v1
- **Base URL (production):** `https://api.primefastpay.com/v1`
- **Base URL (sandbox):** `https://api.sandbox.primefastpay.com/v1`
- **Auth:** `Authorization: Bearer sk_live_xxx` / `sk_test_xxx`

> The `/v1` prefix lives in the `servers` URLs, so paths below are written without it
> (`/payouts`, not `/v1/payouts`). Resolved together they produce the URLs shown in the docs.

To use this with tooling, copy the YAML block below into `openapi.yaml`.

```yaml
openapi: 3.1.0

info:
  title: Prime Fast Pay API
  version: "1.0.0"
  summary: Send payouts globally on the rail that gets there first.
  description: |
    Send payouts globally via ACH, real-time payments (RTP), virtual cards, push-to-card,
    digital checks, and international wire. RESTful, JSON-based, and built for developers.

    ## Quick start

    1. Get your API keys from the dashboard.
    2. Create a recipient with bank or card details (`POST /recipients`).
    3. Send a payout (`POST /payouts`) — funds arrive in seconds.

    ## Transport

    All requests must use HTTPS. Unencrypted HTTP requests are rejected.

    ## Payment methods

    | Method           | Speed               | Fee     | Countries      |
    | ---------------- | ------------------- | ------- | -------------- |
    | `ach`            | 1–2 business days   | $0.25   | US             |
    | `rtp`            | Seconds (24/7)      | $0.50   | US             |
    | `push_to_card`   | Minutes             | 1.5%    | US, CA, UK     |
    | `virtual_card`   | Instant             | $1.00   | US, CA         |
    | `digital_check`  | 1–3 business days   | $0.50   | US             |
    | `wire`           | Same day            | $15.00  | 160+ countries |

    All fees are deducted from the payout amount unless sender-bears-cost is enabled in
    dashboard settings.

    ## Rate limits

    Rate limits are scoped to the API key environment: 100 req/min for `sk_test_` keys,
    1,000 req/min for `sk_live_` keys. Exceeding the limit returns `429` with a
    `Retry-After` header.
  contact:
    name: Prime Fast Pay Developer Support
    url: https://api.primefastpay.com

servers:
  - url: https://api.primefastpay.com/v1
    description: Production — use live keys (sk_live_*). 1,000 req/min.
  - url: https://api.sandbox.primefastpay.com/v1
    description: Sandbox — use test keys (sk_test_*). 100 req/min.

security:
  - bearerAuth: []

tags:
  - name: Payouts
    description: Initiate, inspect, and cancel payouts.
  - name: Recipients
    description: Store and manage tokenized recipient payout details.

paths:
  /payouts:
    post:
      operationId: createPayout
      summary: Create a payout
      description: |
        Initiate a payout to a recipient. Funds are disbursed according to the selected method.

        Pass an `Idempotency-Key` to make retries safe — reusing a key with different
        parameters returns `409 idempotency_conflict`.
      tags: [Payouts]
      parameters:
        - $ref: '#/components/parameters/IdempotencyKey'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PayoutCreateRequest'
            examples:
              rtpPayout:
                summary: $2,500.00 instant RTP payout
                value:
                  amount: 250000
                  currency: USD
                  recipient_id: rec_9x8y7z6w5v4u
                  method: rtp
                  description: "Invoice #4421"
                  metadata:
                    invoice_id: "4421"
                    project: website-redesign
      responses:
        '200':
          description: The created payout.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Payout'
              examples:
                completed:
                  value:
                    id: pay_1a2b3c4d5e6f
                    object: payout
                    amount: 250000
                    currency: USD
                    status: completed
                    method: rtp
                    recipient_id: rec_9x8y7z6w5v4u
                    description: "Invoice #4421"
                    metadata:
                      invoice_id: "4421"
                      project: website-redesign
                    estimated_arrival: "2026-08-11T12:22:15Z"
                    created_at: "2026-08-11T12:22:10Z"
                    completed_at: "2026-08-11T12:22:15Z"
        '400': { $ref: '#/components/responses/BadRequest' }
        '401': { $ref: '#/components/responses/Unauthorized' }
        '402': { $ref: '#/components/responses/InsufficientFunds' }
        '404': { $ref: '#/components/responses/NotFound' }
        '409': { $ref: '#/components/responses/IdempotencyConflict' }
        '422': { $ref: '#/components/responses/RecipientUnverified' }
        '429': { $ref: '#/components/responses/RateLimited' }
        '500': { $ref: '#/components/responses/ApiError' }

    get:
      operationId: listPayouts
      summary: List payouts
      description: Returns a paginated list of payouts. Filter by status, method, date range, or recipient.
      tags: [Payouts]
      parameters:
        - name: status
          in: query
          description: Filter by payout status.
          required: false
          schema:
            $ref: '#/components/schemas/PayoutStatus'
        - name: method
          in: query
          description: Filter by payout method.
          required: false
          schema:
            $ref: '#/components/schemas/PayoutMethod'
        - name: recipient_id
          in: query
          description: Filter to a single recipient.
          required: false
          schema:
            type: string
            examples: [rec_9x8y7z6w5v4u]
        - name: created_after
          in: query
          description: ISO 8601 timestamp. Inclusive lower bound on `created_at`.
          required: false
          schema:
            type: string
            format: date-time
        - name: created_before
          in: query
          description: ISO 8601 timestamp. Inclusive upper bound on `created_at`.
          required: false
          schema:
            type: string
            format: date-time
        - $ref: '#/components/parameters/Limit'
        - $ref: '#/components/parameters/StartingAfter'
      responses:
        '200':
          description: A paginated list of payouts.
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/ListResponse'
                  - type: object
                    properties:
                      data:
                        type: array
                        items:
                          $ref: '#/components/schemas/Payout'
        '400': { $ref: '#/components/responses/BadRequest' }
        '401': { $ref: '#/components/responses/Unauthorized' }
        '429': { $ref: '#/components/responses/RateLimited' }
        '500': { $ref: '#/components/responses/ApiError' }

  /payouts/{payout_id}:
    get:
      operationId: retrievePayout
      summary: Retrieve a payout
      description: Fetch a single payout by its ID.
      tags: [Payouts]
      parameters:
        - $ref: '#/components/parameters/PayoutId'
      responses:
        '200':
          description: The requested payout.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Payout'
        '401': { $ref: '#/components/responses/Unauthorized' }
        '404': { $ref: '#/components/responses/NotFound' }
        '429': { $ref: '#/components/responses/RateLimited' }
        '500': { $ref: '#/components/responses/ApiError' }

  /payouts/{payout_id}/cancel:
    post:
      operationId: cancelPayout
      summary: Cancel a payout
      description: |
        Cancel a payout that has not yet been submitted to the banking network. Once a payout is
        `processing` or `completed`, cancellation is impossible and the request fails with
        `payout_not_cancellable`.
      tags: [Payouts]
      parameters:
        - $ref: '#/components/parameters/PayoutId'
      responses:
        '200':
          description: The canceled payout.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Payout'
              examples:
                canceled:
                  value:
                    id: pay_1a2b3c4d5e6f
                    object: payout
                    amount: 250000
                    currency: USD
                    status: canceled
                    method: rtp
                    recipient_id: rec_9x8y7z6w5v4u
                    created_at: "2026-08-11T12:22:10Z"
        '401': { $ref: '#/components/responses/Unauthorized' }
        '404': { $ref: '#/components/responses/NotFound' }
        '409':
          description: The payout has already been submitted to the network.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorEnvelope'
              examples:
                notCancellable:
                  value:
                    error:
                      type: payout_not_cancellable
                      message: This payout has already been submitted to the network and cannot be canceled.
        '429': { $ref: '#/components/responses/RateLimited' }
        '500': { $ref: '#/components/responses/ApiError' }

  /recipients:
    post:
      operationId: createRecipient
      summary: Create a recipient
      description: |
        Store a recipient's payout details securely. Prime Fast Pay tokenizes sensitive data —
        you never handle raw account numbers after creation.

        Supply `bank_account` for `ach`, `rtp`, and `wire` payouts, or `debit_card` for
        `push_to_card` payouts.
      tags: [Recipients]
      parameters:
        - $ref: '#/components/parameters/IdempotencyKey'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RecipientCreateRequest'
            examples:
              individualWithBankAccount:
                summary: Individual with a US checking account
                value:
                  type: individual
                  name: John Doe
                  email: john@example.com
                  bank_account:
                    routing_number: "021000021"
                    account_number: "000123456789"
                    account_type: checking
                  address:
                    line1: 123 Main St
                    city: New York
                    state: NY
                    postal_code: "10001"
                    country: US
      responses:
        '200':
          description: The created recipient.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Recipient'
              examples:
                verified:
                  value:
                    id: rec_9x8y7z6w5v4u
                    object: recipient
                    type: individual
                    name: John Doe
                    email: john@example.com
                    status: verified
                    created_at: "2026-08-11T12:22:10Z"
        '400': { $ref: '#/components/responses/BadRequest' }
        '401': { $ref: '#/components/responses/Unauthorized' }
        '409': { $ref: '#/components/responses/IdempotencyConflict' }
        '429': { $ref: '#/components/responses/RateLimited' }
        '500': { $ref: '#/components/responses/ApiError' }

    get:
      operationId: listRecipients
      summary: List recipients
      description: Returns a paginated list of recipients.
      tags: [Recipients]
      parameters:
        - name: status
          in: query
          description: Filter by verification status.
          required: false
          schema:
            $ref: '#/components/schemas/RecipientStatus'
        - name: type
          in: query
          description: Filter by recipient type.
          required: false
          schema:
            $ref: '#/components/schemas/RecipientType'
        - name: email
          in: query
          description: Exact match filter on the recipient email.
          required: false
          schema:
            type: string
            format: email
        - $ref: '#/components/parameters/Limit'
        - $ref: '#/components/parameters/StartingAfter'
      responses:
        '200':
          description: A paginated list of recipients.
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/ListResponse'
                  - type: object
                    properties:
                      data:
                        type: array
                        items:
                          $ref: '#/components/schemas/Recipient'
        '400': { $ref: '#/components/responses/BadRequest' }
        '401': { $ref: '#/components/responses/Unauthorized' }
        '429': { $ref: '#/components/responses/RateLimited' }
        '500': { $ref: '#/components/responses/ApiError' }

  /recipients/{recipient_id}:
    get:
      operationId: retrieveRecipient
      summary: Retrieve a recipient
      description: Fetch a single recipient by its ID.
      tags: [Recipients]
      parameters:
        - $ref: '#/components/parameters/RecipientId'
      responses:
        '200':
          description: The requested recipient.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Recipient'
        '401': { $ref: '#/components/responses/Unauthorized' }
        '404': { $ref: '#/components/responses/NotFound' }
        '429': { $ref: '#/components/responses/RateLimited' }
        '500': { $ref: '#/components/responses/ApiError' }

webhooks:
  payout.created:
    post:
      operationId: onPayoutCreated
      summary: payout.created
      description: Sent when a payout is created.
      parameters:
        - $ref: '#/components/parameters/WebhookSignature'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PayoutEvent'
      responses:
        '200':
          description: Return 200 OK within 5 seconds to acknowledge receipt.

  payout.processing:
    post:
      operationId: onPayoutProcessing
      summary: payout.processing
      description: Sent when a payout is submitted to the banking network.
      parameters:
        - $ref: '#/components/parameters/WebhookSignature'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PayoutEvent'
      responses:
        '200':
          description: Acknowledged.

  payout.completed:
    post:
      operationId: onPayoutCompleted
      summary: payout.completed
      description: Sent when funds have settled with the recipient.
      parameters:
        - $ref: '#/components/parameters/WebhookSignature'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PayoutEvent'
      responses:
        '200':
          description: Acknowledged.

  payout.failed:
    post:
      operationId: onPayoutFailed
      summary: payout.failed
      description: Sent when a payout is rejected or returned by the network.
      parameters:
        - $ref: '#/components/parameters/WebhookSignature'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PayoutEvent'
      responses:
        '200':
          description: Acknowledged.

  payout.canceled:
    post:
      operationId: onPayoutCanceled
      summary: payout.canceled
      description: Sent when a payout is canceled before network submission.
      parameters:
        - $ref: '#/components/parameters/WebhookSignature'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PayoutEvent'
      responses:
        '200':
          description: Acknowledged.

  recipient.created:
    post:
      operationId: onRecipientCreated
      summary: recipient.created
      description: Sent when a recipient is created.
      parameters:
        - $ref: '#/components/parameters/WebhookSignature'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RecipientEvent'
      responses:
        '200':
          description: Acknowledged.

  recipient.verified:
    post:
      operationId: onRecipientVerified
      summary: recipient.verified
      description: Sent when a recipient passes KYC and bank verification.
      parameters:
        - $ref: '#/components/parameters/WebhookSignature'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RecipientEvent'
      responses:
        '200':
          description: Acknowledged.

  recipient.restricted:
    post:
      operationId: onRecipientRestricted
      summary: recipient.restricted
      description: Sent when a recipient is restricted and can no longer receive payouts.
      parameters:
        - $ref: '#/components/parameters/WebhookSignature'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RecipientEvent'
      responses:
        '200':
          description: Acknowledged.

  transfer.created:
    post:
      operationId: onTransferCreated
      summary: transfer.created
      description: Sent when a balance transfer is created.
      parameters:
        - $ref: '#/components/parameters/WebhookSignature'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Event'
      responses:
        '200':
          description: Acknowledged.

  transfer.completed:
    post:
      operationId: onTransferCompleted
      summary: transfer.completed
      description: Sent when a balance transfer settles.
      parameters:
        - $ref: '#/components/parameters/WebhookSignature'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Event'
      responses:
        '200':
          description: Acknowledged.

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      description: |
        Authenticate every request with your secret API key in the `Authorization` header:

        ```
        Authorization: Bearer sk_live_xxxxxxxx
        ```

        Keys are environment-scoped — `sk_test_` for sandbox, `sk_live_` for production.

        **Keep secret keys server-side.** Never expose them in client-side code. Publishable
        keys (`pk_`) are for tokenizing recipient data only and are not accepted here.

  parameters:
    PayoutId:
      name: payout_id
      in: path
      required: true
      description: Unique payout identifier.
      schema:
        type: string
        pattern: '^pay_[A-Za-z0-9]+$'
        examples: [pay_1a2b3c4d5e6f]

    RecipientId:
      name: recipient_id
      in: path
      required: true
      description: Unique recipient identifier.
      schema:
        type: string
        pattern: '^rec_[A-Za-z0-9]+$'
        examples: [rec_9x8y7z6w5v4u]

    IdempotencyKey:
      name: Idempotency-Key
      in: header
      required: false
      description: UUID. Prevents duplicate writes for 24 hours.
      schema:
        type: string
        format: uuid

    Limit:
      name: limit
      in: query
      required: false
      description: Number of objects to return.
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 10

    StartingAfter:
      name: starting_after
      in: query
      required: false
      description: Cursor for pagination — an object ID from the previous page.
      schema:
        type: string

    WebhookSignature:
      name: PrimeFastPay-Signature
      in: header
      required: true
      description: |
        Signature over the raw request body, e.g.
        `t=1723381330,v1=sha256=abc123def456...`

        Verify by computing the HMAC-SHA256 of `timestamp.payload` with your webhook secret
        and comparing it to `v1` using a constant-time comparison.
      schema:
        type: string
        examples: ["t=1723381330,v1=sha256=abc123def456"]

  schemas:
    PayoutMethod:
      type: string
      description: The rail used to disburse the payout.
      enum:
        - ach
        - rtp
        - push_to_card
        - virtual_card
        - digital_check
        - wire

    PayoutStatus:
      type: string
      description: Lifecycle state of the payout.
      enum:
        - pending
        - processing
        - completed
        - failed
        - canceled

    Currency:
      type: string
      description: ISO 4217 currency code.
      enum: [USD, EUR, GBP, CAD]

    Metadata:
      type: object
      description: Key-value pairs for internal tracking. Max 20 keys.
      maxProperties: 20
      additionalProperties:
        type: string

    PayoutCreateRequest:
      type: object
      required: [amount, currency, recipient_id, method]
      additionalProperties: false
      properties:
        amount:
          type: integer
          format: int64
          minimum: 100
          description: Amount in the smallest currency unit (cents). Minimum 100 ($1.00).
          examples: [250000]
        currency:
          $ref: '#/components/schemas/Currency'
        recipient_id:
          type: string
          description: ID of the recipient. Create one first via `POST /recipients`.
          examples: [rec_9x8y7z6w5v4u]
        method:
          $ref: '#/components/schemas/PayoutMethod'
        description:
          type: string
          maxLength: 255
          description: Internal memo.
          examples: ["Invoice #4421"]
        metadata:
          $ref: '#/components/schemas/Metadata'

    Payout:
      type: object
      required: [id, object, amount, currency, status, method, recipient_id, created_at]
      properties:
        id:
          type: string
          examples: [pay_1a2b3c4d5e6f]
        object:
          type: string
          const: payout
        amount:
          type: integer
          format: int64
          description: Amount in the smallest currency unit (cents).
          examples: [250000]
        currency:
          $ref: '#/components/schemas/Currency'
        status:
          $ref: '#/components/schemas/PayoutStatus'
        method:
          $ref: '#/components/schemas/PayoutMethod'
        recipient_id:
          type: string
          examples: [rec_9x8y7z6w5v4u]
        description:
          type: [string, 'null']
          examples: ["Invoice #4421"]
        metadata:
          $ref: '#/components/schemas/Metadata'
        estimated_arrival:
          type: [string, 'null']
          format: date-time
          description: Projected settlement time based on the selected method.
        created_at:
          type: string
          format: date-time
        completed_at:
          type: [string, 'null']
          format: date-time
          description: Set once the payout reaches `completed`; null otherwise.

    RecipientType:
      type: string
      enum: [individual, business]

    RecipientStatus:
      type: string
      description: Verification state of the recipient.
      enum: [pending, verified, restricted]

    BankAccount:
      type: object
      description: Required for `ach`, `rtp`, and `wire` payouts.
      required: [routing_number, account_number]
      properties:
        routing_number:
          type: string
          description: ABA routing number.
          examples: ["021000021"]
        account_number:
          type: string
          description: Bank account number. Tokenized on receipt and never returned.
          examples: ["000123456789"]
        account_type:
          type: string
          enum: [checking, savings]
          examples: [checking]

    DebitCard:
      type: object
      description: Required for `push_to_card` payouts.
      required: [number, exp_month, exp_year]
      properties:
        number:
          type: string
          description: Debit card number. Tokenized on receipt and never returned.
        exp_month:
          type: integer
          minimum: 1
          maximum: 12
        exp_year:
          type: integer
          examples: [2030]

    Address:
      type: object
      properties:
        line1:
          type: string
          description: Street address.
          examples: [123 Main St]
        city:
          type: string
          examples: [New York]
        state:
          type: string
          examples: [NY]
        postal_code:
          type: string
          examples: ["10001"]
        country:
          type: string
          description: ISO 3166-1 alpha-2 country code.
          examples: [US]

    RecipientCreateRequest:
      type: object
      required: [type, name, email]
      additionalProperties: false
      properties:
        type:
          $ref: '#/components/schemas/RecipientType'
        name:
          type: string
          description: Full legal name of the recipient.
          examples: [John Doe]
        email:
          type: string
          format: email
          description: Valid email for notifications.
          examples: [john@example.com]
        bank_account:
          $ref: '#/components/schemas/BankAccount'
        debit_card:
          $ref: '#/components/schemas/DebitCard'
        address:
          $ref: '#/components/schemas/Address'
        tax_id:
          type: string
          description: SSN or EIN for tax reporting (1099 / 1042-S).

    Recipient:
      type: object
      required: [id, object, type, name, email, status, created_at]
      properties:
        id:
          type: string
          examples: [rec_9x8y7z6w5v4u]
        object:
          type: string
          const: recipient
        type:
          $ref: '#/components/schemas/RecipientType'
        name:
          type: string
          examples: [John Doe]
        email:
          type: string
          format: email
          examples: [john@example.com]
        status:
          $ref: '#/components/schemas/RecipientStatus'
        created_at:
          type: string
          format: date-time

    ListResponse:
      type: object
      required: [object, data, has_more]
      properties:
        object:
          type: string
          const: list
        data:
          type: array
          items: {}
        has_more:
          type: boolean
          description: True when more objects exist past this page. Page again with `starting_after`.

    EventType:
      type: string
      description: Event name, following the pattern `resource.action`.
      enum:
        - payout.created
        - payout.processing
        - payout.completed
        - payout.failed
        - payout.canceled
        - recipient.created
        - recipient.verified
        - recipient.restricted
        - transfer.created
        - transfer.completed

    Event:
      type: object
      description: Webhook event envelope.
      required: [id, object, type, created_at, data]
      properties:
        id:
          type: string
          examples: [evt_7h6g5f4e3d2c]
        object:
          type: string
          const: event
        type:
          $ref: '#/components/schemas/EventType'
        created_at:
          type: string
          format: date-time
        data:
          type: object
          properties:
            object: {}

    PayoutEvent:
      allOf:
        - $ref: '#/components/schemas/Event'
        - type: object
          properties:
            data:
              type: object
              required: [object]
              properties:
                object:
                  $ref: '#/components/schemas/Payout'

    RecipientEvent:
      allOf:
        - $ref: '#/components/schemas/Event'
        - type: object
          properties:
            data:
              type: object
              required: [object]
              properties:
                object:
                  $ref: '#/components/schemas/Recipient'

    Error:
      type: object
      required: [type, message]
      properties:
        type:
          type: string
          description: Machine-readable error class.
          examples: [bad_request]
        message:
          type: string
          description: Human-readable explanation.
          examples: [Invalid routing number.]
        code:
          type: string
          description: More specific error code, when available.
          examples: [invalid_routing_number]
        param:
          type: string
          description: The offending request field, when the error is parameter-specific.
          examples: [bank_account.routing_number]

    ErrorEnvelope:
      type: object
      required: [error]
      properties:
        error:
          $ref: '#/components/schemas/Error'

  responses:
    BadRequest:
      description: Invalid parameters. Check `param` for the offending field.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorEnvelope'
          examples:
            badRequest:
              value:
                error:
                  type: bad_request
                  message: Invalid routing number.
                  code: invalid_routing_number
                  param: bank_account.routing_number

    Unauthorized:
      description: Invalid or missing API key.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorEnvelope'
          examples:
            unauthorized:
              value:
                error:
                  type: unauthorized
                  message: Invalid or missing API key.

    InsufficientFunds:
      description: Account balance too low for this payout.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorEnvelope'
          examples:
            insufficientFunds:
              value:
                error:
                  type: insufficient_funds
                  message: Account balance too low for this payout.

    NotFound:
      description: Payout or recipient ID does not exist.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorEnvelope'
          examples:
            notFound:
              value:
                error:
                  type: resource_not_found
                  message: Payout or recipient ID does not exist.

    IdempotencyConflict:
      description: Idempotency key reused with different parameters.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorEnvelope'
          examples:
            conflict:
              value:
                error:
                  type: idempotency_conflict
                  message: Idempotency key reused with different parameters.

    RecipientUnverified:
      description: Recipient failed KYC or bank verification.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorEnvelope'
          examples:
            unverified:
              value:
                error:
                  type: recipient_unverified
                  message: Recipient failed KYC or bank verification.

    RateLimited:
      description: Too many requests. Retry after the `Retry-After` header value.
      headers:
        Retry-After:
          description: Seconds to wait before retrying.
          schema:
            type: integer
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorEnvelope'
          examples:
            rateLimited:
              value:
                error:
                  type: rate_limit_exceeded
                  message: Too many requests.

    ApiError:
      description: Temporary server error. Retry with exponential backoff.
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorEnvelope'
          examples:
            apiError:
              value:
                error:
                  type: api_error
                  message: Temporary server error.
```

## Coverage map

| `docs.html` section    | Spec location                                          |
| ---------------------- | ------------------------------------------------------ |
| Quick start, base URLs | `info.description`, `servers`                          |
| Authentication         | `components.securitySchemes.bearerAuth`, root `security` |
| Payment methods        | `PayoutMethod` enum + fee/speed table in `info.description` |
| Create a payout        | `POST /payouts`                                        |
| Retrieve a payout      | `GET /payouts/{payout_id}`                             |
| List payouts           | `GET /payouts`                                         |
| Cancel a payout        | `POST /payouts/{payout_id}/cancel`                     |
| Create a recipient     | `POST /recipients`                                     |
| Retrieve a recipient   | `GET /recipients/{recipient_id}`                        |
| List recipients        | `GET /recipients`                                      |
| Receiving webhooks     | `webhooks`, `WebhookSignature` parameter               |
| Event types            | `EventType` enum, one `webhooks` entry per event        |
| Error codes            | `components.responses.*`, `Error` schema               |

## Gaps filled by inference

These were not specified in `docs.html` — confirm against the real implementation before publishing:

1. **List response envelope.** `ListResponse` (`{ object: "list", data: [...], has_more }`) is inferred from the cursor-pagination style. The docs describe `starting_after` but never show a list payload.
2. **Webhook payload shape.** The `Event` envelope (`id`, `object`, `type`, `created_at`, `data.object`) is inferred. The docs only document the signature header and event names.
3. **Cancel conflict status code.** `payout_not_cancellable` is shown as a body but with no HTTP status; mapped to `409`. The error table also omits this type.
4. **Success status codes.** All writes return `200`; the docs never state whether `POST /payouts` returns `200` or `201`.
5. **`transfer.*` events.** Two transfer events are listed with no documented transfer resource, so they use the generic `Event` schema with an untyped `data.object`.
6. **Field nullability and `additionalProperties: false`** on create requests are tightened choices, not documented constraints — loosen if the API accepts unknown fields.
7. **Address shape.** The parameter table says "Street"; the example uses `line1`. The spec follows the example.
8. **`Metadata` values** are typed as strings, matching the example; widen to `anyOf` if the API accepts numbers or booleans.

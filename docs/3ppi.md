---
id: 3ppi
title: 3rd Party Payment Initiation
---

The following use case demonstrates a 3rd Party Payment Initiation (3PPI) flow. This flow allows a user to authenticate 
a Payment Initiation Service Provider (PISP) to make payments on the users behalf without having to hold or be in the flow of money. Example use cases include:
* Authenticating an app for in-app payments
* Authentication existing 3PPI's like Google/Apple Pay.

```mermaid
sequenceDiagram
    participant User
    participant PISP
    participant Issuer as Issuer<br/>(Users wallet)
    participant Acquirer as Acquirer<br/>(receiving wallet)
    participant Receiver

    User->>PISP: Produce $PaymentPointer

    PISP->>Issuer: Get Open Payments Metadata

    PISP->>Issuer: POST /mandates
    activate PISP
    activate Issuer
    Issuer->>Issuer: Create Mandate
    Issuer-->>PISP: 201 Response
    deactivate Issuer
    deactivate Receiver

    PISP->>Issuer: Authorise Mandate
    activate PISP
    activate Issuer
    
        Issuer->>User: Mandate Consent
        activate User
        User-->>Issuer: Accept Mandate
        deactivate User

    Issuer-->>PISP: Respond Auth Tokens
    deactivate PISP
    deactivate Issuer

    alt Send Payment
        User->>PISP: Pay Receiver
    
        PISP->>Acquirer: POST /invoices
        activate Acquirer
        activate PISP
        Acquirer->>Acquirer: createInvoice(invoice)
        Acquirer-->>PISP: 201 paid=false
        deactivate PISP
        deactivate Acquirer
    else Pay Invoice
        Receiver->>PISP: Provide Invoice
    end
    
    PISP->>Issuer: POST /mandates/{id}/spend (invoice, auth_token)
    activate PISP
    activate Issuer

    Issuer->>Acquirer: GET invoice_url
    activate Acquirer
    Acquirer-->>Issuer: 200 destination_address, shared_secret
    deactivate Acquirer
    rect rgb(0, 255, 0)
        Issuer->>Acquirer: [ push payment ]
        activate Acquirer
    end
    Acquirer->>Acquirer: updateInvoiceBalance(amount)
    deactivate Acquirer

    Issuer-->>PISP: Response
    deactivate Receiver
    deactivate Issuer

    Receiver->>Acquirer: GET /invoices/{id}
    activate Receiver
    activate Acquirer
    Acquirer->>Receiver: 200 invoice
    deactivate Receiver
    deactivate Acquirer

```

# Flow

Based on the diagram above the flow is as follows:
1. User provides Payment Pointer
2. PISP gets Open Payments metadata
3. Authenticate PISP
4. Initiate payments

## 1.  User provides Payment Pointer

When a user would like to authenticate a new PISP it will provide them with their Payment Pointer

## 2. PISP gets Open Payments metadata

Using the Payment Pointer provided, the PISP will perform a GET against the correct url.

```http
GET /.well-known/open-payments-server HTTP/1.1
Host: issuer.wallet
```

A successful `200` response will return the Open Payments metadata

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "issuer": "https://issuer.wallet",
  "authorization_endpoint": "https://issuer.wallet/authorize",
  "token_endpoint": "https://issuer.wallet/token",
  "token_endpoint_auth_methods_supported": ["client_secret_basic","private_key_jwt"],
  "token_endpoint_auth_signing_alg_values_supported": ["RS256", "ES256"],
  "userinfo_endpoint": "https://issuer.wallet/userinfo",
  "jwks_uri": "https://issuer.wallet/jwks.json",
  "registration_endpoint": "https://issuer.wallet/register",
  "scopes_supported": ["openid","profile","email","address","phone","offline_access"],
  "response_types_supported": ["code", "token"],
  "service_documentation": "https://issuer.wallet/service_documentation.html",
  "ui_locales_supported": ["en-US", "en-GB", "en-CA", "fr-FR", "fr-CA"],
  "payment_invoices_endpoint": "https://issuer.wallet/invoices",
  "payment_mandates_endpoint": "https://issuer.wallet/mandates",
  "payment_sessions_endpoint": "https://issuer.wallet/sessions",
  "payment_assets_supported": [
    {"code": "USD", "scale": 2},
    {"code": "EUR", "scale": 2}
  ]
}
```

The PISP can now find `payment_mandates_endpoint` with which to create the necessary mandate.

## 3. Authenticate PISP

The PISP creates a mandate on the Users Issuer's wallet using the endpoint above. This mandate will contain the permissions
with which the PISP can operate on the Users behalf.

```http
POST /mandates HTTP/1.1
Host: issuer.wallet
Accept: application/json
Content-Type: application/json

{
  "amount": 200,
  "asset" : {
    "code": "USD",
    "scale": 2
  },
  "interval": "P1M",
  "scope": "issuer.wallet/alice"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "name": "//issuer.wallet/mandates/2fad69d0-7997-4543-8346-69b418c479a6",
  "amount": 200,
  "asset" : {
    "code": "USD",
    "scale": 2
  },
  "interval": "P1M",
  "start_at": "2020-01-22T00:00:00Z",
  "scope": "issuer.wallet/alice",
  "balance": 200 
}
```

The PISP submits the mandate to the AS of the Issuer in order get authorization from the User. Once the User has 
authorized the mandate, the PISP will receive an access_token with which to use against the mandate.

## 4. Initiate payments

Whenever the PISP wants to initiate a payment on the User behalf, it will submit an invoice to the `/spend` endpoint of 
the mandate it was authorized for. This will instruct the Issuer to complete the payment.

```http
POST /mandates/2fad69d0-7997-4543-8346-69b418c479a6/spend HTTP/1.1
Host: issuer.wallet
Authorization: Bearer random_auth_token
Accept: application/json
Content-Type: application/json

{
  "invoice": "//acquirer.wallet/invoices/2d24bd87-1afc-465e-a4ec-07cb4f70f7b0"
}

```

```http
HTTP/1.1 201 Created
```


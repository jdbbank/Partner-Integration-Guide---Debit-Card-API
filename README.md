# Partner Integration Guide - Debit Card API

Version: 1.0  
API version: v1  
Audience: Partner technical teams integrating debit card services  
Scope: Debit card inquiry, card lifecycle, card controls, PIN, CVV2, transactions, and debit-card Reset PIN.

## 1. Scope

This document covers only debit-card-related APIs exposed by this project. It intentionally excludes pure virtual-card-only functions unless the same endpoint is required to operate or inspect debit cards.

Included API groups:

| Group | Purpose |
| --- | --- |
| Authentication | Obtain a JWT bearer token. |
| Card list and card profile | Retrieve debit card identifiers, card status, account status, card flags, masked PAN, cardholder data, and product information. |
| Transactions | Retrieve card transaction history, formatted history, summary, analysis, and filtered statements. |
| Card profile maintenance | Lock and unlock cards. |
| Card option maintenance | Enable or disable e-commerce, contactless, and international usage. |
| PIN and security | Change PIN, reset PIN, clear PIN attempt counter, activate plastic card, and retrieve encrypted CVV2. |
| Limits | Retrieve transaction limiters for a customer. |

Excluded API groups:

| Group | Reason |
| --- | --- |
| Virtual card creation routes | Not part of debit card partner integration. |
| Prepaid top-up and withdrawal routes | Payment movement APIs, not debit-card lifecycle APIs. |
| Internal service routes | Not partner-facing unless explicitly provisioned by JDB. |

## 2. Environments and Base URL

Use the base URL provided during onboarding.

```text
Uat:    https://uat-api.example.com
Production: https://api.example.com
```


Always use the final gateway URL provided by JDB.

## 3. Data Format and Encoding

### 3.1 Transport

| Requirement | Value |
| --- | --- |
| Protocol | HTTPS only in production |
| Request body | JSON encoded as UTF-8 |
| Response body | JSON encoded as UTF-8 |
| Content type | `application/json` |
| Date/time format | ISO 8601 for date-time fields, for example `2026-05-13T03:30:00Z` |
| Timestamp header | Unix timestamp in seconds, for example `1778643000` |

### 3.2 URL Encoding

Path and query parameter values must be URL encoded according to RFC 3986.

| Value type | Encoding rule |
| --- | --- |
| Path parameters | Encode reserved characters before placing the value in the URL path. |
| Query parameters | Encode using standard query-string encoding. |
| Spaces | Use `%20`; do not rely on `+` in path parameters. |
| Date-time query values | Encode `:` as `%3A` if your HTTP client does not do this automatically. |

Example transaction URL:

```text
/v1/transactions/payment-method/6486410/318770?from=2026-05-01T00%3A00%3A00Z&to=2026-05-13T23%3A59%3A59Z&page_size=20
```

### 3.3 JSON Serialization for Request Signature

For endpoints requiring request signature verification, the server signs the parsed JSON body using:

```text
message = JSON.stringify(requestBody) + timestamp
signature = HMAC_SHA256(message, partnerSignatureSecret).hex
```

Important rules:

| Rule | Description |
| --- | --- |
| Body order | Use a stable JSON object structure. In JavaScript, `JSON.stringify(body)` preserves insertion order of object keys. |
| Empty GET body | Use `{}` as the body input when signing GET requests. The signed message becomes `{}` plus the timestamp. |
| Timestamp window | Timestamp must be within 5 minutes of server time. |
| Signature format | Lowercase hexadecimal HMAC-SHA256 string, 64 characters. |
| Secret | Use the partner signature secret provided by JDB. Do not use the encrypted `api-key` as the signature secret. |

Node.js example:

```javascript
const crypto = require('crypto');

function signRequest(body, timestamp, secret) {
  const message = JSON.stringify(body || {}) + String(timestamp);
  return crypto
    .createHmac('sha256', secret)
    .update(message, 'utf8')
    .digest('hex');
}
```

## 4. Authentication and Security

### 4.1 Credential Types

| Credential | Header / Usage | Description |
| --- | --- | --- |
| Encrypted API key | `api-key` | Required for all partner APIs. It identifies the partner and must be issued by JDB. |
| JWT access token | `Authorization: Bearer <token>` | Required for most debit card APIs after login. |
| Request timestamp | `timestamp` | Required for signed endpoints. Unix timestamp in seconds. |
| Request signature | `signature` | Required for signed endpoints. HMAC-SHA256 over `JSON.stringify(body) + timestamp`. |
| Response hash secret | JDB-provided secret | Used by partner to verify `_hash` on successful API responses. |

### 4.2 Login Flow

1. Call `POST /v1/login` with `api-key`, username, and password.
2. Store the returned JWT token until expiry.
3. For protected endpoints, send both `api-key` and `Authorization: Bearer <token>`.
4. For signed endpoints, also send `timestamp` and `signature`.

### 4.3 Required Headers by Endpoint Type

| Endpoint type | `api-key` | `Authorization` | `timestamp` | `signature` |
| --- | --- | --- | --- | --- |
| `POST /v1/login` | Mandatory | Not required | Not required | Not required |
| Standard partner debit APIs | Mandatory | Mandatory | Mandatory | Mandatory |
| Reset PIN APIs | Mandatory | Not currently enforced by route | Mandatory | Mandatory |
| Advanced debit card step APIs | Mandatory | Mandatory | Route-dependent | Route-dependent |

Note: Some advanced step routes currently enforce only API key and JWT in the application route configuration. JDB may still require signature at gateway level. Partners should support signed requests for all non-login calls unless JDB explicitly confirms otherwise.

### 4.4 Common Header Example

```http
Content-Type: application/json
Accept: application/json
api-key: U2FsdGVkX1+encryptedPartnerApiKey...
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...
timestamp: 1778643000
signature: 4d3f3be64d8e2f7a6b2c0d2b8e7f1b27a65a7d59c6f4c3d91a8df5b9f0f41a2c
```

### 4.5 Response Hash Verification

Successful responses are automatically extended with `_hash` and `_timestamp`.

The server calculates `_hash` from the response body before `_hash` and `_timestamp` are added:

```text
_hash = HMAC_SHA256(JSON.stringify(originalResponseBody), JDB_SECRET_KEY).hex
```

Partner-side verification example:

```javascript
const crypto = require('crypto');

function verifyResponseHash(response, responseSecret) {
  const { _hash, _timestamp, ...originalResponseBody } = response;
  const payload = JSON.stringify(originalResponseBody);
  const expected = crypto
    .createHmac('sha256', responseSecret)
    .update(payload, 'utf8')
    .digest('hex');
  return expected === _hash;
}
```

## 5. Standard Response Format

Most endpoints return this success envelope:

```json
{
  "code": 200,
  "responseCode": 0,
  "message": "Transaction Successful",
  "data": {},
  "transactionId": "9b03fe68-2ab1-4a70-a4e8-137130e7d297",
  "_hash": "7c9a1e0b7f2a...",
  "_timestamp": "2026-05-13T03:30:00.000Z"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `code` | number | HTTP status code returned in body. |
| `responseCode` | number/string | Business response code. `0` means success. |
| `message` | string | Human-readable result message. |
| `data` | object/array/null | Endpoint-specific payload. |
| `transactionId` | string | Unique ID generated by the API for tracing. |
| `_hash` | string | HMAC hash for successful response verification. |
| `_timestamp` | string | Server response timestamp. |

Error envelope:

```json
{
  "code": 400,
  "responseCode": 400,
  "message": "Invalid input or missing fields",
  "data": null,
  "transactionId": "9b03fe68-2ab1-4a70-a4e8-137130e7d297",
  "transactionDetall": "optional detail"
}
```

Validation error example:

```json
{
  "code": 400,
  "responseCode": 400,
  "message": [
    {
      "type": "field",
      "msg": "client_external_id is required",
      "path": "client_external_id",
      "location": "body"
    }
  ],
  "data": null
}
```

## 6. Common Error Codes

| HTTP | `responseCode` | Meaning | Recommended partner action |
| --- | ---: | --- | --- |
| 400 | 400 | Validation error or bad request. | Correct request body, path, or query parameters. |
| 400 | 1015 | Invalid encrypted API key format. | Confirm the issued `api-key` and preserve `+` characters. |
| 401 | 1020 | Invalid partner credentials or request signature. | Regenerate signature, check timestamp and secret. |
| 401 | 1026 | Missing `api-key`. | Send issued API key. |
| 401 | 401 | Missing, invalid, or expired JWT. | Login again and retry with new token. |
| 403 | 1031 | Partner is not active. | Contact JDB support. |
| 403 | 1521 | Partner not authorized for card type. | Contact JDB support to enable permissions. |
| 404 | 404 | Resource not found. | Verify IDs. |
| 409 | 409 | Conflict or duplicate operation. | Check idempotency key or wait for active operation to complete. |
| 429 | 429 | Rate limit exceeded. | Retry after the rate limit window. |
| 500 | 500 | Internal server error. | Retry later; contact support with `transactionId`. |
| 502 | 502 | External service failed. | Retry later. |
| 503 | 503 | External service unavailable or circuit breaker open. | Retry with backoff. |
| 504 | 504 | Timeout. | Retry with backoff. |

## 7. Rate Limiting

Most routes use the shared application limiter. Reset PIN has a stricter limiter of 10 requests per minute per IP.

Rate limit response example:

```json
{
  "code": 429,
  "responseCode": 429,
  "message": "Too many Reset PIN requests - please wait before retrying",
  "data": null
}
```

Partners should implement retry with exponential backoff and avoid retrying non-idempotent operations unless the endpoint explicitly supports idempotency.

## 8. API Summary

| No. | API | Method | Path | Primary use |
| ---: | --- | --- | --- | --- |
| 1 | Login | POST | `/v1/login` | Get JWT token. |
| 2 | Card list | POST | `/v1/card-list` | Find debit cards for a client. |
| 3 | Card profile | POST | `/v1/cards-profile` | Get detailed card profile. |
| 4 | Transaction history | GET | `/v1/transactions/payment-method/{client_ref_id}/{card_req_id}` | Raw transaction listing. |
| 5 | Formatted transactions | GET | `/v1/transactions/payment-method/{client_ref_id}/{card_req_id}/formatted` | Categorized transaction listing. |
| 6 | Unlock card | POST | `/v1/cards/unblock` | Unlock a blocked card. |
| 7 | Block by cardholder | POST | `/v1/cards/block-by-cardholder` | Temporarily block card by cardholder request. |
| 8 | Block by system | POST | `/v1/cards/block-by-system` | Temporarily block card by system request. |
| 9 | Card statuses | GET | `/v1/cards/statuses` | List supported card statuses. |
| 10 | E-commerce usage | POST | `/v1/card/online` | Enable or disable online/e-commerce usage. |
| 11 | Contactless usage | POST | `/v1/card/contactless` | Enable or disable contactless usage. |
| 12 | International usage | POST | `/v1/card/int_allow` | Enable or disable overseas/international usage. |
| 13 | Get CVV2 | POST | `/v1/card/cvv2` | Get encrypted CVV2. |
| 14 | Change PIN | POST | `/v1/pin-change` | Change PIN with old PIN. |
| 15 | Reset PIN preview | POST | `/v1/card/repin/preview` | Validate card and return fee quote. |
| 16 | Reset PIN confirm | POST | `/v1/card/repin/confirm` | Deduct fee and reset PIN. |
| 17 | Activate plastic | POST | `/v1/active-plastic` | Activate physical card. |


## 9. Authentication API

### 9.1 Login

```http
POST /v1/login
```

Headers:

| Header | Required | Description |
| --- | --- | --- |
| `Content-Type: application/json` | Mandatory | JSON request. |
| `api-key` | Mandatory | Encrypted partner API key. |

Request body:

| Field | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `username` | string | Mandatory | 3-50 characters | Partner username. |
| `password` | string | Mandatory | 6-100 characters | Partner password. |

Example request:

```json
{
  "username": "partner_user",
  "password": "secureP@ssw0rd"
}
```

Example response:

```json
{
  "code": 200,
  "responseCode": 0,
  "message": "Transaction Successful",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6...",
    "expiresIn": "1h",
    "tokenType": "Bearer"
  },
  "transactionId": "9b03fe68-2ab1-4a70-a4e8-137130e7d297"
}
```

## 10. Card List and Profile APIs

### 10.1 Find Card List

Retrieves debit card payment methods linked to a customer. The response filters CMS payment methods to card records and exposes card/account metadata needed by other APIs.

```http
POST /v1/card-list
```

Headers: standard signed partner headers.

Request body:

| Field | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `client_external_id` | string | Mandatory | Non-empty string | External customer/client identifier. |

Example request:

```json
{
  "client_external_id": "728715_11597107"
}
```

Important response fields:

| Field | Description |
| --- | --- |
| `card_req_id` | Payment method ID used by lock/unlock, profile, and option maintenance APIs. |
| `card_id` | Card ID used by PIN, CVV2, activation, and Reset PIN APIs. |
| `card_number_encryp` | Masked PAN. Do not treat as full PAN. |
| `card_number_full` | Full card contract number when returned by CMS custom data. Treat as sensitive. |
| `product_details` | Card product type. For debit card flows this should be `DEBIT`. |
| `card_status_code` | Card status code from CMS. `00` indicates active in several workflows. |
| `account_status_code` | Linked account status code. `00` indicates active. |
| `contactless_allow` | `Y` enabled, `N` disabled. |
| `ecom_allow` | `Y` enabled, `N` disabled. |
| `card_status_int_allow` | International/overseas usage flag. `Y` enabled, `N` disabled. |
| `agent_code` | Partner/agent classifier from card custom data. Used for authorization checks. |

Example response:

```json
{
  "code": 200,
  "responseCode": 0,
  "message": "Transaction Successful",
  "data": {
    "total_record": 1,
    "data": [
      {
        "card_req_id": "318770",
        "client_ref_id": "728715_11597107",
        "cif": "001-P-00004619",
        "card_id": "507470",
        "card_name": "MR JOHN DOE",
        "card_number_encryp": "6224********1234",
        "card_number_full": "6224123412341234",
        "card_currency": "LAK",
        "card_expiry_date_mm": "12",
        "card_expiry_date_yy": "29",
        "card_active": true,
        "card_status_code": "00",
        "card_valid": "Valid",
        "card_status_name": "Active",
        "product_display_name": "Debit Card",
        "bank_account_number": "0101200000012345",
        "account_status_name": "Active",
        "account_status_code": "00",
        "account_valid": "Valid",
        "plastic_status": "ACTIVE",
        "plastic_design": "DEFAULT",
        "product_details": "DEBIT",
        "payment_scheme": "VISA",
        "opened_at": "2026-05-01T04:00:00Z",
        "updated_at": "2026-05-13T03:00:00Z",
        "is_default_for_client": true,
        "card_status_int_allow": "Y",
        "physical_trx_allow": "Y",
        "contactless_allow": "Y",
        "ecom_allow": "N",
        "agent_code": "FUSION"
      }
    ]
  },
  "transactionId": "9b03fe68-2ab1-4a70-a4e8-137130e7d297"
}
```

### 10.2 Get Card Profile

```http
POST /v1/cards-profile
```

Headers: standard signed partner headers.

Request body:

| Field | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `card_req_id` | string | Mandatory | 1-50 characters | Payment method/card request ID from `card-list`. |

Example request:

```json
{
  "card_req_id": "318770"
}
```

```json
{
    "code": 200,
    "responseCode": 0,
    "message": "Transaction Successful",
    "data": {
        "total_record": 1,
        "data": [
            {
                "card_req_id": "360580",
                "client_ref_id": "6496520",
                "cif": "001-P-00006521",
                "card_id": "740090",
                "card_name": "MISS BOPBY VISA LGC",
                "card_number_encryp": "44661466____1873",
                "card_number_full": "4466146661441873",
                "card_currency": "USD",
                "card_expiry_date_mm": "05",
                "card_expiry_date_yy": "31",
                "card_active": true,
                "card_status_name": "Card OK",
                "card_status_code": "00",
                "card_valid": "Valid",
                "card_status_reason": "",
                "card_status_updated_at": "",
                "code": "VSDC_AG_C",
                "product_display_name": "001-JDB VISA Debit Classic - Agent",
                "bank_account_number": "001-P-00006521",
                "account_status_name": "Account OK",
                "account_status_code": "00",
                "account_valid": "Valid",
                "plastic_status": "Active",
                "plastic_design": "V0001",
                "product_details": "DEBIT",
                "balance_blocked": "0.00",
                "balance_total": "-32.00",
                "balance_available": "-32.00",
                "int_allow": "Y",
                "physical_trx_allow": "Y",
                "contactless_allow": "Y",
                "ecom_allow": "Y",
                "opened_at": "2026-05-05",
                "updated_at": "2026-05-13T03:12:00Z",
                "is_default_for_client": false
            }
        ]
    },
    "transactionId": "bab6744c-0d96-4101-877b-f874c591c440",
    "_hash": "38e1a9d69e1c000c36d849452b2efefae6dfa33c5f68a3636fa74db90e8000e7",
    "_timestamp": "2026-05-13T03:23:52.166Z"
}
```

## 11. Transaction APIs

All transaction APIs use:

```text
client_ref_id = customer/client ID
card_req_id = card payment method ID, usually `card_req_id` from /v1/card-list
```

Path parameter validation:

| Parameter | Required | Validation |
| --- | --- | --- |
| `client_ref_id` | Mandatory | 1-50 characters, `^[0-9A-Za-z_-]+$` |
| `card_req_id` | Mandatory | 1-50 characters, `^[0-9A-Za-z_-]+$` |

Common query parameters:

| Query | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `from` | ISO 8601 date-time | Optional | Valid ISO 8601 | Start date/time. |
| `to` | ISO 8601 date-time | Optional | Valid ISO 8601 | End date/time. |
| `page_size` | integer | Optional | 1-100 | Number of records per page. |
| `page_cursor` | string | Optional | 1-1000 characters | Cursor from previous response. |

### 11.1 Raw Transaction History

```http
GET /v1/transactions/payment-method/{client_ref_id}/{card_req_id}
```
Example respone:

```json
{
    "code": 200,
    "responseCode": 0,
    "message": "Transaction Successful",
    "data": {
        "pagination": {
            "cursor": "",
            "has_next_page": false,
            "total_in_current_page": 295
        },
        "summary": {
            "total_transactions": 295,
            "successful_transactions": 206,
            "failed_transactions": 89,
            "total_amount_processed": {
                "amount": 53862.81,
                "currency": "USD"
            },
            "transaction_types": [
                {
                    "type": "Fee",
                    "count": 17
                },
                {
                    "type": "ATM Withdrawal",
                    "count": 165
                },
                {
                    "type": "Other",
                    "count": 79
                },
                {
                    "type": "Purchase",
                    "count": 34
                }
            ],
            "countries": [
                {
                    "country": "Unknown",
                    "count": 1
                },
                {
                    "country": "CHN",
                    "count": 1
                },
                {
                    "country": "LAO",
                    "count": 1
                },
                {
                    "country": "THA",
                    "count": 1
                },
                {
                    "country": "HKG",
                    "count": 1
                }
            ],
            "status_breakdown": {
                "processed": 206,
                "rejected": 74,
                "reversed": 3,
                "waiting": 12,
                "other": 0
            }
        },
        "categorized_transactions": {
            "atm_withdrawals": [
                {
                    "id": "8318630",
                    "transaction_time": "2024-12-17T09:47:44",
                    "local_time": "2024-12-17T02:47:43",
                    "transaction_type": {
                        "code": "0719",
                        "name": "ATM Cash Withdrawal",
                        "category": "ATM Withdrawal"
                    },
                    "merchant": {
                        "name": "Mcdonalds",
                        "country": "CHN",
                        "category_code": "6011",
                        "category_description": "ATM/Financial Institution"
                    },
                    "amounts": {
                        "original_amount": "-40.00",
                        "settlement_amount": "-6.54",
                        "original_currency": "CNY",
                        "settlement_currency": "USD",
                        "exchange_rate": 0.1635
                    },
                    "card_info": {
                        "card_id": "466190",
                        "masked_pan": "622454______8537",
                        "display_name": "NOINA",
                        "payment_scheme": "UPI"
                    },
                    "status": {
                        "status": "Processed",
                        "result_code": "0",
                        "description": "Transaction completed successfully",
                        "is_successful": true
                    },
                    "fees": [
                        {
                            "description": "Service fee",
                            "amount": "-0.05",
                            "currency": "USD"
                        }
                    ],
                    "interaction": {
                        "card_present": true,
                        "cardholder_present": true,
                        "entry_mode": "ICCContact",
                        "authentication": "PIN"
                    },
                    "reference_numbers": {
                        "approval_code": "336042",
                        "retrieval_reference": "000000006457",
                        "transaction_reference": "O35211063S20",
                        "network_reference": "02001079681217094744"
                    },
                    "is_international": true,
                    "is_reversal": false,
                    "risk_indicators": [
                        "International Transaction",
                        "Foreign ATM Usage"
                    ]
                },

              
            ],
            "purchases": [
                {
                    "id": "8281350",
                    "transaction_time": "2024-12-12T11:44:41",
                    "local_time": "2024-12-12T04:44:41",
                    "transaction_type": {
                        "code": "0513 1",
                        "name": "CH Debit",
                        "category": "Purchase"
                    },
                    "merchant": {
                        "name": "TEST",
                        "city": "CITY",
                        "country": "LAO",
                        "category_code": "3390",
                        "category_description": "Government Services"
                    },
                    "amounts": {
                        "original_amount": "-10.00",
                        "settlement_amount": "-10.00",
                        "original_currency": "USD",
                        "settlement_currency": "USD"
                    },
                    "card_info": {
                        "card_id": "466190",
                        "masked_pan": "622454______8537",
                        "display_name": "NOINA",
                        "payment_scheme": "UPI"
                    },
                    "status": {
                        "status": "Reversed",
                        "result_code": "0",
                        "description": "Transaction was reversed/refunded",
                        "is_successful": false
                    },
                    "fees": [],
                    "interaction": {
                        "card_present": false,
                        "cardholder_present": false,
                        "entry_mode": "Unknown",
                        "authentication": "Unknown"
                    },
                    "reference_numbers": {
                        "approval_code": "466493",
                        "retrieval_reference": "434711028061",
                        "transaction_reference": "O3471102TD4G"
                    },
                    "is_international": false,
                    "is_reversal": true,
                    "risk_indicators": [
                        "Failed Transaction",
                        "Chained Transaction"
                    ]
                },
               
            ],
            "refunds": [],
            "fees": [
                {
                    "id": "8327990",
                    "transaction_time": "2024-12-18T20:30:51",
                    "local_time": "",
                    "transaction_type": {
                        "code": "MB_1_1_2_1_1_3",
                        "name": "Monthly Fee THB",
                        "category": "Fee"
                    },
                    "merchant": {
                        "name": "Unknown Merchant",
                        "country": "Unknown",
                        "category_code": "0000",
                        "category_description": "MCC 0000"
                    },
                    "amounts": {
                        "original_amount": "0.00",
                        "settlement_amount": "0.00",
                        "original_currency": "USD",
                        "settlement_currency": "USD"
                    },
                    "card_info": {
                        "card_id": "466190",
                        "masked_pan": "622454______8537",
                        "display_name": "NOINA",
                        "payment_scheme": "UPI"
                    },
                    "status": {
                        "status": "Processed",
                        "result_code": "0",
                        "description": "Transaction completed successfully",
                        "is_successful": true
                    },
                    "fees": [],
                    "interaction": {
                        "card_present": false,
                        "cardholder_present": false,
                        "entry_mode": "Unknown",
                        "authentication": "Unknown"
                    },
                    "reference_numbers": {},
                    "is_international": false,
                    "is_reversal": false,
                    "risk_indicators": []
                },
               
            ],
            "other": [
                {
                    "id": "8318350",
                    "transaction_time": "2024-12-16T14:54:31",
                    "local_time": "2024-12-16T07:54:31",
                    "transaction_type": {
                        "code": "0515",
                        "name": "Retail",
                        "category": "Other"
                    },
                    "merchant": {
                        "name": "Testing merchant 2",
                        "city": "ShenZhen",
                        "country": "CHN",
                        "category_code": "5311",
                        "category_description": "Department Stores"
                    },
                    "amounts": {
                        "original_amount": "-357.00",
                        "settlement_amount": "-357.00",
                        "original_currency": "USD",
                        "settlement_currency": "USD"
                    },
                    "card_info": {
                        "card_id": "466190",
                        "masked_pan": "622454______8537",
                        "display_name": "NOINA",
                        "payment_scheme": "UPI"
                    },
                    "status": {
                        "status": "Waiting",
                        "result_code": "0",
                        "description": "Transaction is being processed",
                        "is_successful": false
                    },
                    "fees": [],
                    "interaction": {
                        "card_present": true,
                        "cardholder_present": true,
                        "entry_mode": "ICCContactless",
                        "authentication": "Unknown"
                    },
                    "reference_numbers": {
                        "approval_code": "466497",
                        "retrieval_reference": "000000006435",
                        "transaction_reference": "O3511103DU66",
                        "network_reference": "02001079461216145431"
                    },
                    "is_international": true,
                    "is_reversal": false,
                    "risk_indicators": [
                        "International Transaction",
                        "High Amount Transaction",
                        "Failed Transaction"
                    ]
                },
               
               
            ]
        },
        "all_transactions": [
            {
                "id": "8327990",
                "transaction_time": "2024-12-18T20:30:51",
                "local_time": "",
                "transaction_type": {
                    "code": "MB_1_1_2_1_1_3",
                    "name": "Monthly Fee THB",
                    "category": "Fee"
                },
                "merchant": {
                    "name": "Unknown Merchant",
                    "country": "Unknown",
                    "category_code": "0000",
                    "category_description": "MCC 0000"
                },
                "amounts": {
                    "original_amount": "0.00",
                    "settlement_amount": "0.00",
                    "original_currency": "USD",
                    "settlement_currency": "USD"
                },
                "card_info": {
                    "card_id": "466190",
                    "masked_pan": "622454______8537",
                    "display_name": "NOINA",
                    "payment_scheme": "UPI"
                },
                "status": {
                    "status": "Processed",
                    "result_code": "0",
                    "description": "Transaction completed successfully",
                    "is_successful": true
                },
                "fees": [],
                "interaction": {
                    "card_present": false,
                    "cardholder_present": false,
                    "entry_mode": "Unknown",
                    "authentication": "Unknown"
                },
                "reference_numbers": {},
                "is_international": false,
                "is_reversal": false,
                "risk_indicators": []
            },
            
            {
                "id": "3998960",
                "transaction_time": "2024-10-23T16:30:06",
                "local_time": "",
                "transaction_type": {
                    "code": "MD_1",
                    "name": "Joining Fee USD",
                    "category": "Fee"
                },
                "merchant": {
                    "name": "Unknown Merchant",
                    "country": "Unknown",
                    "category_code": "0000",
                    "category_description": "MCC 0000"
                },
                "amounts": {
                    "original_amount": "0.00",
                    "settlement_amount": "0.00",
                    "original_currency": "USD",
                    "settlement_currency": "USD"
                },
                "card_info": {
                    "card_id": "466190",
                    "masked_pan": "622454______8537",
                    "display_name": "NOINA",
                    "payment_scheme": "UPI"
                },
                "status": {
                    "status": "Processed",
                    "result_code": "0",
                    "description": "Transaction completed successfully",
                    "is_successful": true
                },
                "fees": [
                    {
                        "description": "Service fee",
                        "amount": "-5.00",
                        "currency": "USD"
                    }
                ],
                "interaction": {
                    "card_present": false,
                    "cardholder_present": false,
                    "entry_mode": "Unknown",
                    "authentication": "Unknown"
                },
                "reference_numbers": {},
                "is_international": false,
                "is_reversal": false,
                "risk_indicators": []
            }
        ]
    },
    "transactionId": "13ee48df-396f-4212-8040-18d90b657994"
}
```

## 12. Card Lock and Unlock APIs

### 12.1 Unlock Card

```http
POST /v1/cards/unblock
```

Request body:

| Field | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `card_req_id` | string | Mandatory | 1-10 characters | Card request/payment method ID from `card-list`. |

Example request:

```json
{
  "card_req_id": "360580"
}
```
Example respone:

```json
{
    "code": 200,
    "responseCode": 0,
    "message": "Transaction Successful",
    "data": {
        "total_record": 1,
        "data": [
            {
                "card_req_id": "360580",
                "cif": "001-P-00006521",
                "client_ref_id": "6496520",
                "card_id": "740090",
                "card_name": "MISS BOPBY VISA LGC",
                "card_number_encryp": "44661466____1873",
                "card_active": true,
                "card_status_name": "Card OK",
                "card_status_code": "00",
                "card_valid": "Valid",
                "card_status_reason": "05 (Temporary Block by Cardholder) --> 00 (Card OK)  : card unblock by partner",
                "card_status_updated_at": "2026-05-12T17:00:00Z",
                "opened_at": "2026-05-05",
                "updated_at": "2026-05-13T03:12:00Z"
            }
        ]
    },
    "transactionId": "ab632f07-78f2-4407-ad4f-f1ddc5e34308",
    "_hash": "a38ea34ee6078d9a7d05606bb6e6a883c23d4b07124474e41548d33ccc942852",
    "_timestamp": "2026-05-13T04:30:39.104Z"
}
```

### 12.2 Block Card by Cardholder

```http
POST /v1/cards/block-by-cardholder
```

Use when the cardholder requests a temporary lock, for example lost card, suspicious use, or app-based lock.

Example request:

```json
{
  "card_req_id": "360580"
}
```

Example respone:

```json
{
    "code": 200,
    "responseCode": 0,
    "message": "Transaction Successful",
    "data": {
        "total_record": 1,
        "data": [
            {
                "card_req_id": "360580",
                "cif": "001-P-00006521",
                "client_ref_id": "6496520",
                "card_id": "740090",
                "card_name": "MISS BOPBY VISA LGC",
                "card_number_encryp": "44661466____1873",
                "card_active": true,
                "card_status_name": "Temporary Block by Cardholder",
                "card_status_code": "CH05",
                "card_valid": "Declined",
                "card_status_reason": "00 (Card OK) --> 05 (Temporary Block by Cardholder)  : block-by-cardholder",
                "card_status_updated_at": "2026-05-12T17:00:00Z",
                "opened_at": "2026-05-05",
                "updated_at": "2026-05-13T03:12:00Z"
            }
        ]
    },
    "transactionId": "037a972c-dfc6-46af-956e-ee0c0fabf275",
    "_hash": "1077ab3fbac351d5d7092d9671543f32e0e606d74cb9494a9faf8ea0fd5aef73",
    "_timestamp": "2026-05-13T04:27:42.884Z"
}
```

### 12.3 Block Card by System

```http
POST /v1/cards/block-by-system
```

Use when the partner system must block the card for operational or risk reasons.

Example request:

```json
{
    "card_req_id":"360580"
}
```
Example respone:

```json
{
    "code": 200,
    "responseCode": 0,
    "message": "Transaction Successful",
    "data": {
        "total_record": 1,
        "data": [
            {
                "card_req_id": "335010",
                "cif": "001-P-00005397",
                "client_ref_id": "6491440",
                "card_id": "689790",
                "card_name": "MR LODETO CNTT",
                "card_number_encryp": "495066______5291",
                "card_active": true,
                "card_status_name": "Temporary Block by System",
                "card_status_code": "S05",
                "card_valid": "Declined",
                "card_status_reason": "00 (Card OK) --> 05 (Temporary Block by System)  : block-by-system",
                "card_status_updated_at": "2025-12-15T17:00:00Z",
                "opened_at": "2025-12-15",
                "updated_at": "2025-12-15T09:49:11Z"
            }
        ]
    },
    "transactionId": "08c3934c-be7c-45f9-beb0-b47a076513d0"
}
```

### 12.4 Get Card Statuses

```http
GET /v1/cards/statuses
```
Example respone:

```json
{
    "code": 200,
    "responseCode": 0,
    "message": "Transaction Successful",
    "data": {
        "statuses": [
            {
                "status": "CH05",
                "description": "Temporary Block by Cardholder"
            },
            {
                "status": "S05",
                "description": "Temporary Block by System"
            },
            {
                "status": "00",
                "description": "Unknown Status"
            }
        ]
    },
    "transactionId": "a9398576-c6e9-485c-a0bc-d53e06888c52",
    "_hash": "a85e9a07ac0f995266b9f4d4f360c723dfa4803ba15bb2d638123a5da1fb725d",
    "_timestamp": "2026-05-13T03:33:25.698Z"
}
```
For GET requests requiring signature, sign `{}` plus the timestamp.

## 13. Card Option Maintenance APIs

These APIs update card usage flags. All use the same body structure.

Common request body:

| Field | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `card_req_id` | string | Mandatory | 1-50 characters, `^[0-9A-Za-z_-]+$` | Payment method/card request ID from `card-list`. |
| `custom_value` | string | Mandatory | 1 character | Desired flag value. Use `Y` to enable and `N` to disable. |
| `options` | object | Optional | Object | Optional compatibility field. |
| `options.contactless` | string | Optional | `Y` or `N` | Optional contactless value. |

Important: The route validates `custom_value` as a single character. Business logic expects `Y` or `N` for enable/disable behavior.

| API | Method | Path | Updated flag |
| --- | --- | --- | --- |
| E-commerce / online usage | POST | `/v1/card/online` | `ecom_allow` |
| Contactless usage | POST | `/v1/card/contactless` | `contactless_allow` |
| International usage | POST | `/v1/card/int_allow` | `int_allow` |

Example request:

```json
{
    "card_req_id": "360580",
    "custom_value": "Y"
}
```
Example respone:
```json
{
    "code": 200,
    "responseCode": 0,
    "message": "Transaction Successful",
    "data": {
        "total_record": 1,
        "data": [
            {
                "card_req_id": "360580",
                "client_ref_id": "6496520",
                "cif": "001-P-00006521",
                "card_id": "740090",
                "card_name": "MISS BOPBY VISA LGC",
                "card_number_encryp": "44661466____1873",
                "card_number_full": "4466146661441873",
                "card_currency": "USD",
                "card_active": true,
                "int_allow": "Y",
                "physical_trx_allow": "Y",
                "contactless_allow": "Y",
                "ecom_allow": "Y"
            }
        ]
    },
    "transactionId": "b7159481-944f-4b45-8b67-45a0e5cee00a",
    "_hash": "c8041fe7c88734ee42264abaf8015b8d329c5414300d5df1acf9390f1d4bc748",
    "_timestamp": "2026-05-13T03:12:40.389Z"
}
```
## 14. CVV2 API

### 14.1 Get Encrypted CVV2

```http
POST /v1/card/cvv2
```

This endpoint returns CVV2 encrypted by the API using the incoming `api-key` header as part of the encryption process. Treat the returned value as highly sensitive.

Request body:

| Field | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `card_id` | string | Mandatory | 3-10 characters, `^[A-Za-z0-9_\-@.]+$` | Card ID from `card-list`. |

Example request:

```json
{
  "card_id": "507470"
}
```

Example response:

```json
{
  "code": 200,
  "responseCode": 0,
  "message": "Transaction Successful",
  "data": {
    "cvv2encrypted": "U2FsdGVkX1+encryptedCvv2Value..."
  },
  "transactionId": "9b03fe68-2ab1-4a70-a4e8-137130e7d297"
}
```

Security requirements:

| Requirement | Description |
| --- | --- |
| Display | Show CVV2 only inside an authenticated secure session. |
| Storage | Do not store decrypted CVV2. |
| Logs | Never log decrypted CVV2 or encrypted CVV2 values. |
| Retry | Avoid repeated calls unless user explicitly requests CVV2 display again. |

## 15. PIN and Card Security APIs

### 15.1 Change PIN

```http
POST /v1/pin-change
```

Use when the customer knows the current PIN and wants to change it.

Request body:

| Field | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `client_external_id` | string | Mandatory | 1-50 chars, `^[0-9A-Za-z_-]+$` | Customer external ID. |
| `card_id` | string | Mandatory | 1-50 chars, `^[0-9A-Za-z_-]+$` | Card ID from `card-list`. |
| `old_pin` | string | Mandatory | Exactly 6 numeric digits | Current PIN. |
| `new_pin` | string | Mandatory | Exactly 6 numeric digits | New PIN. |

Example request:

```json
{
  "client_external_id": "728715_11597107",
  "card_id": "507470",
  "old_pin": "123456",
  "new_pin": "654321"
}
```

PIN rules:

| Rule | Description |
| --- | --- |
| Format | PIN must be exactly 6 digits. |
| Type | Send PIN as a JSON string, not a number, to preserve leading zeroes. |
| Logging | Never log PIN values. |
| UX | Confirm the customer intends to change PIN before calling this endpoint. |

### 15.2 Clear PIN Attempts

```http
POST /v1/clear-pin-attempts
```

Request body:

| Field | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `card_id` | string | Mandatory | 1-50 characters | Card ID from `card-list`. |

Example request:

```json
{
  "card_id": "507470"
}
```


## 16. Reset PIN Debit Card APIs

Reset PIN is a two-step flow:

1. `preview`: validate card eligibility, validate new PIN format, calculate fee, and return a quote.
2. `confirm`: use the quote to deduct fee and execute PIN reset.

### 16.1 Reset PIN Eligibility

| Check | Requirement |
| --- | --- |
| Card ownership | `card_id` must belong to `client_external_id`. |
| Product type | Card must be `DEBIT`. |
| Card scheme | `VISA`, `MASTERCARD`, or `UPI`. |
| Card status | Card status code must be `00`. |
| Account status | Account status code must be `00`. |
| PIN tries | PIN tries status must be `OK`. |
| Partner authorization | Partner agent code must match card custom data. |
| Fee configuration | Partner/card scheme/currency fee config must exist. |

### 16.2 Preview Reset PIN

```http
POST /v1/card/repin/preview
```

Request body:

| Field | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `client_external_id` | string | Mandatory | Non-empty string | Customer external ID. |
| `card_id` | string | Mandatory | Non-empty string | Card ID. |
| `new_pin` | string | Mandatory | Exactly 6 numeric digits | New PIN to set during confirm. |

Example request:

```json
{
  "client_external_id": "728715_11597107",
  "card_id": "507470",
  "new_pin": "123456"
}
```

Example response:

```json
{
  "code": 200,
  "responseCode": 0,
  "message": "Transaction Successful",
  "data": {
    "quote_id": "RPQ-20260513-000001",
    "transaction_type": "RESET_PIN",
    "masked_card_number": "6224********1234",
    "cardholder_name": "MR JOHN DOE",
    "card_scheme": "VISA",
    "fee_source": "CUSTOMER_ACCOUNT",
    "fee_account": "0101200000012345",
    "fee_amount": "10000.00",
    "fee_currency": "LAK",
    "eligible_for_repin": true,
    "expires_at": "2026-05-13T03:45:00.000Z"
  },
  "transactionId": "9b03fe68-2ab1-4a70-a4e8-137130e7d297"
}
```

### 16.3 Confirm Reset PIN

```http
POST /v1/card/repin/confirm
```

Request body:

| Field | Type | Required | Validation | Description |
| --- | --- | --- | --- | --- |
| `request_id` | string | Mandatory | Non-empty string | Partner-generated idempotency key. Must be unique per operation. |
| `quote_id` | string | Mandatory | Non-empty string | Quote ID from preview response. |
| `client_external_id` | string | Mandatory | Non-empty string | Same client used in preview. |
| `card_id` | string | Mandatory | Non-empty string | Same card used in preview. |

Example request:

```json
{
    "request_id": "TXT_2026051279",
    "quote_id": "Q2026051298843",
    "client_external_id": "001146456",
    "card_id": "740090"
}
```
Example response:

```json
{
    "code": 200,
    "responseCode": 0,
    "message": "Transaction Successful",
    "data": {
        "transaction_id": "c1bf12ab-7e27-40b8-90c2-d4c8beb7e37a",
        "request_id": "TXT_2026051279",
        "quote_id": "Q2026051298843",
        "status": "SUCCESS",
        "message": "PIN reset successful",
        "executed_at": "2026-05-12T10:29:00.558Z"
    },
    "transactionId": "1918ba4b-3101-499a-8900-f3fafbcfbd3c",
    "_hash": "779b0d3b8ef9cb28bb74adb0cc9f8225c7ba3dc240db6c60a00427c26f9ba7e5",
    "_timestamp": "2026-05-12T10:29:00.570Z"
}
```
Reset PIN statuses:

| Status | Meaning |
| --- | --- |
| `PENDING` | Transaction created but not completed. |
| `FEE_DEDUCTED` | Fee was deducted before PIN reset execution. |
| `SUCCESS` | PIN reset completed. |
| `FAILED` | PIN reset failed. |
| `PENDING_REPIN` | Fee deducted but PIN reset failed and is queued for retry/manual handling. |
| `CANCELLED` | Operation cancelled. |

Reset PIN error codes:

| HTTP | `responseCode` | Meaning |
| --- | ---: | --- |
| 400 | 2000 | Missing required fields. |
| 400 | 2001 | PIN must be exactly 6 numeric digits. |
| 400 | 2002 | Unsupported card scheme. |
| 400 | 2003 | Card must be a debit card. |
| 400 | 2004 | Card status is not active. |
| 400 | 2005 | Account status is not active. |
| 400 | 2006 | PIN tries status is not OK. |
| 400 | 2007 | Card does not belong to client. |
| 403 | 2010 | Partner agent mismatch. |
| 403 | 2011 | Partner not authorized. |
| 400 | 2020 | Quote not found. |
| 410 | 2021 | Quote expired. Create a new preview. |
| 400 | 2022 | Quote already used. |
| 400 | 2023 | Quote does not match card. |
| 409 | 2030 | Duplicate request ID. |
| 409 | 2031 | Card is currently being processed. |
| 400 | 2040 | Fee configuration not found. |
| 400 | 2041 | Fee source profile not found. |
| 400 | 2042 | Insufficient balance for Reset PIN fee. |
| 400 | 2043 | Fee deduction failed. |
| 400 | 2044 | Currency not supported. |
| 202 | 2050 | Fee deducted but PIN reset failed; queued for resolution. |
| 409 | 2051 | Existing pending Reset PIN request for this card. |
| 500 | 2099 | Internal Reset PIN module error. |



## 19. Idempotency and Retry Guidance

| API type | Idempotency guidance |
| --- | --- |
| Read/query APIs | Safe to retry with backoff. |
| Card option updates | Retry only after checking current card flags if the first result is unknown. |
| Lock/unlock | Retry only if the desired final state is still not reached. |
| Debit card issuance | Do not blindly retry after timeout. Query operation status or contact JDB with `transactionId` because duplicate issuance may occur. |
| Reset PIN confirm | Uses `request_id` as idempotency key. Reusing the same `request_id` returns duplicate/conflict behavior. |
| CVV2 | Avoid automatic retries unless required by user action. |

## 20. Partner Implementation Checklist

Before production launch:

| Item | Required |
| --- | --- |
| Store `api-key`, JWT token, HMAC secret, and response hash secret securely. | Yes |
| Implement login and token refresh. | Yes |
| Implement request signing for all non-login calls. | Yes |
| Verify `_hash` on successful responses. | Yes |
| Preserve PIN values as strings and never log them. | Yes |
| Never log CVV2, decrypted card number, PIN, API key, JWT, signature secret, or response hash secret. | Yes |
| Use `card-list` to resolve `card_req_id` and `card_id` before card operations. | Yes |
| Handle 400, 401, 403, 409, 429, 500, 502, 503, and 504 responses. | Yes |
| Use unique `request_id` for every Reset PIN confirm operation. | Yes |
| Implement retry/backoff and avoid duplicate card issuance. | Yes |

## 21. Support Information

When contacting JDB support, include:

| Field | Description |
| --- | --- |
| `transactionId` | API transaction ID from response. |
| Endpoint | Method and path. |
| Timestamp | Request timestamp and local timezone. |
| Partner request ID | For Reset PIN, include `request_id`; for issuance, include `regNumber` if supplied. |
| Environment | Sandbox or production. |
| Sanitized request | Remove API key, token, PIN, CVV2, full PAN, secrets, and signatures. |

Do not send sensitive values by email or chat unless JDB explicitly provides a secure channel.

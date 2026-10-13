# Cegecom MEF LSO Sonata (Billie) API

**Last updated:** 13 October 2026

Serviceability, quoting and ordering for Carrier Ethernet and internet access, to the
**MEF LSO Sonata Billie** release.

This is the reference for buyers integrating with us. **Reviewed against the current
implementation on 12 October 2026.** Examples use illustrative values and have been
checked with the request validators; they are not promises of serviceability or price.
The Broadband and Carrier Ethernet worked flows, retrieval, `externalId` filters and
quote decline were also verified against the staging API on **12 October 2026**.
Carrier Ethernet was tested with an authorised test ENNI; use your own assigned
handover reference when integrating.
Cegecom supplies your environment URL, credentials, product catalogue and handover
references. Confirm availability in your assigned environment before going live.

---

## Contents

- [What this API is](#what-this-api-is)
- [The flow](#the-flow)
- [Base URL and authentication](#base-url-and-authentication)
- [What we require of a request](#what-we-require-of-a-request)
- [Endpoints](#endpoints)
  - [Address validation](#address-validation)
  - [Product Offering Qualification](#product-offering-qualification)
  - [Quote](#quote)
  - [Decline a quote](#decline-a-quote)
  - [Product Order](#product-order)
- [Worked integration example](#worked-integration-example)
- [Retrieval and filtering](#retrieval-and-filtering)
- [Carrier Ethernet integration](#carrier-ethernet-integration)
- [Errors](#errors)
- [Products and terms](#products-and-terms)
- [Where we are narrower than MEF](#where-we-are-narrower-than-mef)
- [Not yet implemented](#not-yet-implemented)

---

## What this API is

Four MEF Sonata APIs, served from one endpoint:

| API | What it answers |
|---|---|
| **Address Validation** (MEF 79) | is this address one we know, and what is its id |
| **Product Offering Qualification** (MEF 79) | can you deliver this product, to this address |
| **Quote** (MEF 80) | what would it cost, and for how long does that stand |
| **Product Order** (MEF 57.2) | deliver it |

**POQ and Quote are synchronous:** the response contains the result, rather than a
pending job. An Order response acknowledges the request; it does not mean delivery
is complete. Retrieve the order to follow subsequent state changes. Push notifications
are not served; see [Where we are narrower than MEF](#where-we-are-narrower-than-mef).

---

## The flow

```
  POST /geographicAddressValidation   ->  addressId
  POST /productOfferingQualification  ->  POQ_...   + serviceability per item
  POST /quote                         ->  Quote_... + price per item
  POST /productOrder                  ->  Order_... and the quote becomes `accepted`
```

Each step references the one before it. The references are checked: a quote naming a
qualification that is not yours, or an order naming a quote item that does not exist,
is refused rather than accepted and reconciled later.

**Identifiers.** We assign `POQ_…`, `Quote_…` and `Order_…`, and those are what you send
back on the next step. You may also set your own `externalId` on POQ, Quote and Order; it is
optional, we store it, and you can filter listings by it. It is *not* an identifier we
resolve — `GET /quote/{id}` wants our id.

---

## Base URL and authentication

```
https://<host>/<stage>/billie
```

Use the host and stage listed in the documentation repository's README for your
chosen environment. Use its main or staging reference accordingly.

Every request needs an API key:

```
x-api-key: <your key>
```

The key identifies you. Everything you create is scoped to it: you cannot read, quote
against, or order another buyer's records, and an `externalId` only has to be unique
among **your** records.

Use the API key **value** in the header; do not send its identifier as a credential.
Send `Content-Type: application/json` on POST requests. Keep keys out of source code
and logs. Use the same buyer credentials throughout a linked flow.

A missing or unknown key is rejected with HTTP 403. Errors rejected before reaching
the service may have a different body from the MEF error examples below.

---

## What we require of a request

MEF states most of this in `required` lists; a few rules are in prose and no schema
validator will catch them, so they are collected here.

**On every create (`POQ`, `Quote`, `Order`):**

| Field | Where | Notes |
|---|---|---|
| item `id` | every item | yours, and what relationships point at |
| item `action` | every item | must be `add` — see below |
| `product.productConfiguration` | every item | required for `add`; `@type` selects the product |
| `relatedContactInformation` | body (POQ, Order) | all four of `role`, `name`, `emailAddress`, `number` on each contact |

**Per endpoint:**

| Field | Where | Notes |
|---|---|---|
| `instantSyncQualification` | POQ | must be `true` |
| `instantSyncQuote` | Quote | must be `true` |
| `buyerRequestedQuoteLevel` | Quote | `budgetary` or `firm` |
| `productOrderContact` | Order contacts | at least one contact with this role |
| `quoteItem` (`quoteId` + `id`) | every order item | we require the quote |

**`externalId` is optional** on all three. Send one and it must be unique among your
records; omit it and you get our id only.

**Two things that must *not* be sent on an `add`:**

- `product.id` — it identifies an **in-service** product, so it has nothing to name on
  a create. This is not the item's own `id`, which *is* required. See
  [the five ids](#the-five-ids).
- anything under [Not yet implemented](#not-yet-implemented) — we refuse rather than
  accept and ignore.

### The five ids

Easy to conflate, and only one of them is forbidden on an `add`:

| Field | Identifies | On `add` |
|---|---|---|
| `…Item[].id` | your own name for this item | **required** |
| `…ItemRelationship[].id` | another item in the same request | fine |
| `product.productRelationship[].id` | a referenced product, e.g. an ENNI | fine |
| `product.productOffering.id` / `productSpecification.id` | a catalogue entry | fine |
| `product.id` | an **in-service** product | **refused** |

---

## Endpoints

### Address validation

```
POST /geographicAddressValidation
```

Takes an address, returns the closest matches and an id for each.

```json
{
  "submittedGeographicAddress": {
    "streetName": "Rue de Gibraltar", "streetNr": "18",
    "postcode": "1624", "city": "Luxembourg", "country": "Luxembourg"
  }
}
```

Take `bestMatchGeographicAddress.id` and use it as a `place` on the next step.

---

### Product Offering Qualification

```
POST /productOfferingQualification
GET  /productOfferingQualification/{id}
GET  /productOfferingQualification?externalId=&limit=&offset=
```

Asks whether we can deliver a product at a location. Each item is answered
independently with a `serviceabilityConfidence`:

| Value | Meaning |
|---|---|
| `green` | serviceable — fiber in service, or capacity available |
| `yellow` | probably serviceable, subject to a feasibility check |
| `red` | not serviceable |

**Two ways to ask.** Name a `productOffering.id` from our catalogue, or name a
`productSpecification.id` and let us answer with the offering that fits — MEF's
*specification mode*, and the right choice when you want to know what we would sell you
rather than whether a specific SKU is available. Specification mode is currently
supported for the Carrier Ethernet shapes described below; Broadband and DIA requests
must name a product offering. The offering we choose comes back on the answer, and
that is what you quote against.

An item may inherit another item's location through
`qualificationItemRelationship: UNI_REFERENCE`, which is how a connection between two
ends is described without repeating the address.

---

### Quote

```
POST /quote
GET  /quote/{id}
GET  /quote?externalId=&limit=&offset=
```

Prices what a POQ said is serviceable. `productOfferingQualificationItem` on a quote
item links back to the qualification — both halves, `productOfferingQualificationId`
and `id`, because one qualification can answer several products.

The answer carries, per item, a `quoteItemPrice` (non-recurring and recurring),
`quoteItemTerm`, and an installation interval. The quote as a whole carries a
`validFor` window. Treat `endDateTime` as the end of the quoted validity and obtain a
new quote afterwards. **The current API does not automatically expire a quote when
that date passes; clients must not rely on automatic expiry enforcement.**

**A quote item comes back under the id you gave it.** We do not rename items.

**Some items are delivered but not charged.** A UNI port supplied with an Access E-Line
is priced as part of the connection, so its `quoteItemPrice` is an empty array.
This means no separate charge is quoted for the port, rather than a zero-priced
standalone product.

The port's offering is `CONNECT-ETHERNET-UNI`. It carries no rate, because a UNI item
states none — the bandwidth belongs to the Access E-Line that terminates on it, and
that is the item the price is on. Name it on the UNI quote item; the POQ answer returns
it, so you can take it from there rather than hard-coding it.

---

### Decline a quote

```
POST /rejectQuote
```

```json
{ "quoteId": "Quote_...", "reason": "sourced elsewhere" }
```

Moves an orderable quote to `declined`. `reason` is optional. The response is the same
shape as the request.

**The endpoint is named `rejectQuote` and the state is `declined`** — that is not a
mistake on either side. MEF 80 defines one operation, *Cancel or Decline Quote*, and the
API splits it by name; this is the Decline half. `rejected` in MEF is a seller-side
validation outcome on the deferred flow, which we do not serve.

`POST /cancelQuote` is **not served**: MEF 80 allows `cancelled` only from `inProgress`,
and a synchronous quote never passes through that state.

#### Quote states

| State | How it is reached |
|---|---|
| `approved.orderable` | we answered your quote request |
| `declined` | you called `POST /rejectQuote` |
| `accepted` | **you placed an order against it** |
| `expired` | an expired quote is refused for ordering; date-driven transition is not automatic |

`accepted`, `declined` and `expired` are final. An accepted quote cannot be declined and
cannot be ordered a second time.

---

### Product Order

```
POST /productOrder
GET  /productOrder/{id}
GET  /productOrder?externalId=&limit=&offset=
```

Orders one or more items of a quote. `MEFQuoteItemRef` names **one** quote item, so a
two-item quote needs two order items.

**Placing an order accepts its quote** — MEF 80: *"The ACCEPTED state is set by the
Seller. It is triggered by a Product Order."* The order and the acceptance are written
together; neither can happen without the other.

The order must describe the same product its quote priced. Stating less is fine and the
quote's values apply; stating something that **disagrees** — a different offering, a
different `@type`, different bandwidth, a different contract term — is refused.

A quote may be ordered once. A second order against the same quote is refused; request a
new quote.

---

## Worked integration example

This example requests Online-Business at 1000 Mbps down / 500 Mbps up for 12 months.
Confirm the offering and term with Cegecom; an example does not establish that a
product is available at your address. Requests and response excerpts are illustrative.

Replace `ADDRESS_ID`, `POQ_ID`, `QUOTE_ID` and `ORDER_ID` with values returned in
your own flow. They are placeholders, not ids to send unchanged. Treat returned ids
as opaque strings: preserve their case and the whole address id, including any `#`.
Use a new `externalId` for each new resource within its resource type.

For GET commands, also assign the returned resource ids to shell variables, for example
`POQ_ID="<returned POQ id>"`, `QUOTE_ID="<returned quote id>"` and
`ORDER_ID="<returned order id>"`. These variables do not substitute values inside
the JSON files; replace the placeholders in each file before sending it.

The shell commands below use Bash and `curl`. Set your assigned URL and API key:

```bash
BASE="https://<host>/<stage>/billie"
KEY="<your API key value>"
```

### 1. Validate the address

Save this as `address-request.json`, replacing the address with the installation
address you want to validate:

```json
{
  "submittedGeographicAddress": {
    "streetName": "Rue de Gibraltar",
    "streetNr": "18",
    "postcode": "1624",
    "city": "Luxembourg",
    "country": "Luxembourg"
  }
}
```

```bash
curl -sS -X POST "$BASE/geographicAddressValidation" \
  -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  --data-binary @address-request.json
```

For `validationResult: "success"`, copy `bestMatchGeographicAddress.id` to
`ADDRESS_ID`. For example, this response excerpt identifies the field to use:

```json
{
  "validationResult": "success",
  "bestMatchGeographicAddress": { "id": "ADDRESS_ID" }
}
```

An HTTP 200 alone does not mean the address matched. Check `validationResult` and
the returned match. You may request alternatives with `"provideAlternative": true`;
review an alternative before using its id. Do not proceed with a fabricated address id.

### 2. Qualify the product

Save this as `poq-request.json`, substituting the returned address id:

```json
{
  "externalId": "buyer-demo-poq-001",
  "instantSyncQualification": true,
  "provideAlternative": false,
  "relatedContactInformation": [
    {
      "number": "+352123456789",
      "emailAddress": "integration@example.com",
      "role": "buyerContactInformation",
      "name": "Integration Contact"
    }
  ],
  "productOfferingQualificationItem": [
    {
      "id": "BROADBAND",
      "action": "add",
      "product": {
        "productOffering": {
          "id": "ONLINE-BUSINESS-1000/500"
        },
        "productConfiguration": {
          "@type": "BROADBAND",
          "ipvc": {
            "ipvcEndPoint": {}
          },
          "ipUni": {
            "egressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 1000,
                "irUnits": "MBPS"
              }
            },
            "ingressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 500,
                "irUnits": "MBPS"
              }
            }
          },
          "ipUniAccessLink": {},
          "ipUniAccessLinkTrunk": {
            "ethernetPhysicalLink": [
              {
                "physicalLayer": "1000BASE_T"
              }
            ]
          }
        },
        "place": [
          {
            "role": "INSTALL_LOCATION",
            "id": "ADDRESS_ID",
            "@type": "GeographicAddressRef"
          }
        ]
      }
    }
  ]
}
```

```bash
curl -sS -X POST "$BASE/productOfferingQualification" \
  -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  --data-binary @poq-request.json
```

Copy the top-level response `id` to `POQ_ID` and retain item id `BROADBAND`.
Check the item's `serviceabilityConfidence`, `subjectToFeasibilityCheck` and
`installationInterval`. Read both `amount` and `units`; do not assume the interval
is always expressed in days. A completed POQ (`done.ready`) means the assessment
is finished; it does not mean every item is green. Discuss yellow results with
Cegecom before committing to delivery; a red result is not a serviceable offering.

### 3. Request a quote

Save this as `quote-request.json`, substituting `ADDRESS_ID` and `POQ_ID`:

```json
{
  "externalId": "buyer-demo-quote-001",
  "buyerRequestedQuoteLevel": "firm",
  "instantSyncQuote": true,
  "relatedContactInformation": [
    {
      "number": "+352123456789",
      "emailAddress": "integration@example.com",
      "role": "buyerContactInformation",
      "name": "Integration Contact"
    }
  ],
  "quoteItem": [
    {
      "action": "add",
      "id": "item-1",
      "requestedQuoteItemTerm": {
        "duration": {
          "amount": 12,
          "units": "calendarMonths"
        }
      },
      "product": {
        "place": [
          {
            "role": "INSTALL_LOCATION",
            "id": "ADDRESS_ID",
            "@type": "GeographicAddressRef"
          }
        ],
        "productOffering": {
          "id": "ONLINE-BUSINESS-1000/500"
        },
        "productConfiguration": {
          "@type": "BROADBAND",
          "ipvc": {
            "ipvcEndPoint": {}
          },
          "ipUni": {
            "egressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 1000,
                "irUnits": "MBPS"
              }
            },
            "ingressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 500,
                "irUnits": "MBPS"
              }
            }
          },
          "ipUniAccessLink": {},
          "ipUniAccessLinkTrunk": {
            "ethernetPhysicalLink": [
              {
                "physicalLayer": "1000BASE_T"
              }
            ]
          }
        }
      },
      "productOfferingQualificationItem": {
        "id": "BROADBAND",
        "productOfferingQualificationId": "POQ_ID"
      }
    }
  ]
}
```

```bash
curl -sS -X POST "$BASE/quote" \
  -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  --data-binary @quote-request.json
```

The POQ item id (`BROADBAND`) and quote item id (`item-1`) identify different
items. The POQ reference needs both `productOfferingQualificationId` and `id`.
Copy the quote's top-level `id` to `QUOTE_ID`; retain its item id `item-1`.
Review the returned prices, currency, term, quote level, installation interval and
validity window. Use the returned figures; no fixed price is implied by this example.

### 4. Place an order

**This step places an order and accepts its quote.** Run it only when you intend
to order, using the agreed integration environment for testing.

Save this as `order-request.json`, substituting `QUOTE_ID`:

```json
{
  "externalId": "buyer-demo-order-001",
  "relatedContactInformation": [
    {
      "role": "productOrderContact",
      "name": "Integration Contact",
      "emailAddress": "integration@example.com",
      "number": "+352123456789"
    }
  ],
  "productOrderItem": [
    {
      "id": "item-1",
      "action": "add",
      "product": {
        "productConfiguration": {
          "@type": "BROADBAND"
        }
      },
      "quoteItem": {
        "id": "item-1",
        "quoteId": "QUOTE_ID"
      }
    }
  ]
}
```

```bash
curl -sS -X POST "$BASE/productOrder" \
  -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  --data-binary @order-request.json
```

This request states the product type; other optional product details inherit from
the quote. If you send an offering, bandwidth or term, it must agree with the quote.
Keep the nonempty `productConfiguration`; a quote reference alone is insufficient.

The initial order response has `state: "acknowledged"`. Copy its top-level `id` to
`ORDER_ID`; acknowledgment means receipt, not completion of provisioning.
The referenced quote moves to `accepted`. Retrieve the order to follow delivery:

```bash
curl -sS "$BASE/productOrder/$ORDER_ID" -H "x-api-key: $KEY"
```

Common later order states include `inProgress`, `completed`, `failed` and `rejected`.
Use the returned `state` and `stateChange` information; do not interpret all HTTP 200
responses as completed orders. A buyer cannot set order state through this API.

### 5. Decline an unneeded quote instead of ordering it

For an orderable quote that you have not ordered, save this as `decline-request.json`:

```json
{ "quoteId": "QUOTE_ID", "reason": "Alternative offer selected" }
```

```bash
curl -sS -X POST "$BASE/rejectQuote" \
  -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  --data-binary @decline-request.json
```

Declining is an alternative to ordering, not a way to undo step 4. An accepted quote
cannot be declined. Retrieve the quote to confirm its resulting state.

---

## Retrieval and filtering

Use resource ids for individual GETs and your own `externalId` for collection filters:

```bash
curl -sS "$BASE/productOfferingQualification/$POQ_ID" -H "x-api-key: $KEY"
curl -sS "$BASE/quote/$QUOTE_ID" -H "x-api-key: $KEY"
curl -sS "$BASE/productOrder/$ORDER_ID" -H "x-api-key: $KEY"

curl -sS -G "$BASE/productOfferingQualification" -H "x-api-key: $KEY" \
  --data-urlencode "externalId=buyer-demo-poq-001"
curl -sS -G "$BASE/quote" -H "x-api-key: $KEY" \
  --data-urlencode "externalId=buyer-demo-quote-001"
curl -sS -G "$BASE/productOrder" -H "x-api-key: $KEY" \
  --data-urlencode "externalId=buyer-demo-order-001"
```

Filters match the complete reference exactly within your buyer's records. URL-encode
query values, especially spaces, `#`, `+`, `&` and `/`; `--data-urlencode` does this.
Do not put `externalId` in the resource-id path segment.

A collection returns an array of summaries; follow an entry's `href` or GET its `id`
for the full resource. No match returns `200 []`. An illustrative order listing is:

```json
[
  {
    "id": "ORDER_ID",
    "externalId": "buyer-demo-order-001",
    "state": "acknowledged",
    "orderDate": "2026-10-12T10:00:00Z",
    "href": "https://<host>/<stage>/billie/productOrder/ORDER_ID"
  }
]
```

To list a page, use nonnegative `offset` and a positive `limit` (default 50):

```bash
curl -sS -G "$BASE/productOrder" -H "x-api-key: $KEY" \
  --data-urlencode "limit=10" --data-urlencode "offset=0"
```

All three collections accept `externalId`, `state`, `projectId`, `limit` and `offset`.
Only send `projectId` when you assigned one on creation. **Current limitation:** when
`externalId` is present, `state` is not applied. Filter the returned summaries by state
on your side if needed. Pagination has no cursor or total-count response, and filtered
pages can be shorter than requested; do not treat a short page as a complete export.

### Recovering after a timeout

A lost response does not prove a create failed. Retain the `externalId` you sent and
query the relevant collection before resending. An empty result immediately after a
create may be temporary; allow a short delay and check again. A duplicate reference
is refused rather than returning the original create response, so this is not a
general idempotency-key mechanism. In particular, do not retry an uncertain order
with a new reference or quote without first confirming its outcome with Cegecom.

---

## Carrier Ethernet integration

An Access E-Line is quoted with the Operator UNI it terminates on. The POQ may ask
in specification mode; use Cegecom's agreed specification identifiers and handover
reference. The current names used by this API are `UNISpec` and `ELineSpec`.
The following 100 Mbps symmetric POQ is a complete request template. Replace
`ADDRESS_ID` and `ENNI_ID`; the VLAN values and physical link are examples
and must match the service agreed with Cegecom.

```json
{
  "productOfferingQualificationItem": [
    {
      "product": {
        "productSpecification": {
          "id": "UNISpec"
        },
        "productConfiguration": {
          "@type": "urn:mef:lso:spec:sonata:carrier-ethernet-operator-uni:v2.0.0:poq",
          "listofPhysicalLinks": [
            {
              "uniConnectorType": "LC",
              "id": "LC__1000BASE_LX",
              "physicalLink": "1000BASE_LX"
            }
          ]
        },
        "place": [
          {
            "@type": "GeographicSiteRef",
            "id": "ADDRESS_ID",
            "role": "INSTALL_LOCATION"
          }
        ]
      },
      "action": "add",
      "id": "1"
    },
    {
      "product": {
        "productSpecification": {
          "id": "ELineSpec"
        },
        "productConfiguration": {
          "@type": "urn:mef:lso:spec:sonata:access-eline:v2.0.0:poq",
          "listOfClassOfServiceNames": [
            "Basic"
          ],
          "uniEp": {
            "ovcEndPointMap": {
              "mapType": "FORM_U",
              "ovcEndPointMapFormU": [
                200
              ]
            },
            "ingressBandwidthProfilePerClassOfServiceName": [
              {
                "classOfServiceName": "Basic",
                "bwpFlow": {
                  "cir": {
                    "irValue": 100,
                    "irUnits": "MBPS"
                  },
                  "envelopeId": "1234567890"
                }
              }
            ]
          },
          "enniEp": {
            "ovcEndPointMap": {
              "mapType": "FORM_E",
              "ovcEndPointMapFormE": [
                100
              ]
            },
            "ingressBandwidthProfilePerClassOfServiceName": [
              {
                "classOfServiceName": "Basic",
                "bwpFlow": {
                  "cir": {
                    "irValue": 100,
                    "irUnits": "MBPS"
                  },
                  "envelopeId": "1234567890"
                }
              }
            ]
          }
        },
        "productRelationship": [
          {
            "relationshipType": "ENNI_REFERENCE",
            "id": "ENNI_ID"
          }
        ]
      },
      "qualificationItemRelationship": [
        {
          "relationshipType": "UNI_REFERENCE",
          "id": "1"
        }
      ],
      "action": "add",
      "id": "2"
    }
  ],
  "externalId": "buyer-demo-ce-poq-001",
  "instantSyncQualification": true,
  "relatedContactInformation": [
    {
      "number": "+352123456789",
      "emailAddress": "integration@example.com",
      "role": "buyerContactInformation",
      "name": "Integration Contact"
    }
  ],
  "provideAlternative": false
}
```

`qualificationItemRelationship` on item `2` points to item `1`, letting the E-Line
use the UNI's installation location. `productRelationship` names the existing ENNI,
not another request item. The ENNI must be authorised for the API key used for the
request; ask your Cegecom contact to associate the handover with your credentials.
You do not supply an ownership field in the payload. An unknown or unauthorised ENNI
returns `422 referenceNotFound`.

Save the POQ template as `ce-poq-request.json` and submit it:

```bash
curl -sS -X POST "$BASE/productOfferingQualification" \
  -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  --data-binary @ce-poq-request.json
```

Keep both item results and the top-level POQ `id` as `CE_POQ_ID`. Replace
`ENNI_ID` with the reference supplied by Cegecom, preserving its exact value.
Where Cegecom supplies a composite reference, use the complete `<buyer-ref>#<seller-ref>`
value. Do not confuse the ENNI reference with your API key or a request item id.

### Carrier Ethernet quote

Save this as `ce-quote-request.json`. Substitute `ADDRESS_ID`, `ENNI_ID` and
`CE_POQ_ID`. Take the actual offering ids from the POQ response; this template shows
the 100 Mbps offering. Configuration URNs now end in `:quote`, and the item
relationship is `quoteItemRelationship`.

```json
{
  "externalId": "buyer-demo-ce-quote-001",
  "instantSyncQuote": true,
  "buyerRequestedQuoteLevel": "firm",
  "quoteItem": [
    {
      "product": {
        "productConfiguration": {
          "@type": "urn:mef:lso:spec:sonata:carrier-ethernet-operator-uni:v2.0.0:quote",
          "listofPhysicalLinks": [
            {
              "uniConnectorType": "LC",
              "id": "LC__1000BASE_LX",
              "physicalLink": "1000BASE_LX"
            }
          ]
        },
        "place": [
          {
            "@type": "GeographicSiteRef",
            "id": "ADDRESS_ID",
            "role": "INSTALL_LOCATION"
          }
        ],
        "productOffering": {
          "id": "CONNECT-ETHERNET-UNI"
        }
      },
      "action": "add",
      "id": "1",
      "productOfferingQualificationItem": {
        "productOfferingQualificationId": "CE_POQ_ID",
        "id": "1"
      },
      "requestedQuoteItemTerm": {
        "duration": {
          "amount": 12,
          "units": "calendarMonths"
        }
      }
    },
    {
      "product": {
        "productConfiguration": {
          "@type": "urn:mef:lso:spec:sonata:access-eline:v2.0.0:quote",
          "listOfClassOfServiceNames": [
            "Basic"
          ],
          "uniEp": {
            "ovcEndPointMap": {
              "mapType": "FORM_U",
              "ovcEndPointMapFormU": [
                200
              ]
            },
            "ingressBandwidthProfilePerClassOfServiceName": [
              {
                "classOfServiceName": "Basic",
                "bwpFlow": {
                  "cir": {
                    "irValue": 100,
                    "irUnits": "MBPS"
                  },
                  "envelopeId": "1234567890"
                }
              }
            ]
          },
          "enniEp": {
            "ovcEndPointMap": {
              "mapType": "FORM_E",
              "ovcEndPointMapFormE": [
                100
              ]
            },
            "ingressBandwidthProfilePerClassOfServiceName": [
              {
                "classOfServiceName": "Basic",
                "bwpFlow": {
                  "cir": {
                    "irValue": 100,
                    "irUnits": "MBPS"
                  },
                  "envelopeId": "1234567890"
                }
              }
            ]
          }
        },
        "productRelationship": [
          {
            "relationshipType": "ENNI_REFERENCE",
            "id": "ENNI_ID"
          }
        ],
        "productOffering": {
          "id": "CONNECT-ETHERNET-100"
        }
      },
      "action": "add",
      "id": "2",
      "quoteItemRelationship": [
        {
          "relationshipType": "UNI_REFERENCE",
          "id": "1"
        }
      ],
      "productOfferingQualificationItem": {
        "productOfferingQualificationId": "CE_POQ_ID",
        "id": "2"
      },
      "requestedQuoteItemTerm": {
        "duration": {
          "amount": 12,
          "units": "calendarMonths"
        }
      }
    }
  ]
}
```

```bash
curl -sS -X POST "$BASE/quote" \
  -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  --data-binary @ce-quote-request.json
```

Retain the top-level `id` as `CE_QUOTE_ID` and item ids `1` and `2`. Each item
references its own POQ item, while both reference the same POQ resource. The
E-Line's `UNI_REFERENCE` points to the UNI item `1` within this quote. Review the
E-Line's price and term; the UNI's `quoteItemPrice: []` means no separate charge is
quoted for the port.

### Carrier Ethernet order

**Submitting this request places an order.** Save this as `ce-order-request.json`,
substituting `CE_QUOTE_ID` and the offering ids returned by the quote:

```json
{
  "externalId": "buyer-demo-ce-order-001",
  "relatedContactInformation": [
    {
      "role": "productOrderContact",
      "name": "Integration Contact",
      "emailAddress": "integration@example.com",
      "number": "+352123456789"
    }
  ],
  "productOrderItem": [
    {
      "id": "order-1",
      "action": "add",
      "quoteItem": {
        "quoteId": "CE_QUOTE_ID",
        "id": "1"
      },
      "product": {
        "productOffering": {
          "id": "CONNECT-ETHERNET-UNI"
        },
        "productConfiguration": {
          "@type": "urn:mef:lso:spec:sonata:carrier-ethernet-operator-uni:v2.0.0:order"
        }
      }
    },
    {
      "id": "order-2",
      "action": "add",
      "quoteItem": {
        "quoteId": "CE_QUOTE_ID",
        "id": "2"
      },
      "product": {
        "productOffering": {
          "id": "CONNECT-ETHERNET-100"
        },
        "productConfiguration": {
          "@type": "urn:mef:lso:spec:sonata:access-eline:v2.0.0:order"
        }
      }
    }
  ]
}
```

```bash
curl -sS -X POST "$BASE/productOrder" \
  -H "x-api-key: $KEY" -H "Content-Type: application/json" \
  --data-binary @ce-order-request.json
```

The order includes both items. Its own item ids (`order-1`, `order-2`) need not match
the quote's ids: `quoteItem.id` identifies the quote item being ordered. Both
references must use the same quote resource id. Configuration URNs end in `:order`;
the omitted optional configuration details inherit from the quote.

The initial order state is `acknowledged`, and the quote moves to `accepted`.
Retrieve the order by the id returned in its response, or find it by your reference:

```bash
curl -sS -G "$BASE/productOrder" -H "x-api-key: $KEY" \
  --data-urlencode "externalId=buyer-demo-ce-order-001"
```

---

## Errors

Service-generated errors carry `code`, `message` and `reason`. A `422` body **is a
JSON array**. An entry may also carry `propertyPath`, a JSON Pointer to the property
at fault; clients must handle entries where it is absent. Validation currently stops
at the first failure, so the array normally contains one entry. Authentication and
routing errors generated before the service may use a different body.

```json
[
  {
    "reason": "productOrderItem[0].id is required",
    "code": "missingProperty",
    "message": "productOrderItem[0].id is required: it is the identifier you will use to match our answer to this item.",
    "propertyPath": "/productOrderItem/0/id"
  }
]
```

Every other status is a single object.

| Status | Code | When |
|---|---|---|
| **400** | `invalidBody` | the body is not valid JSON |
| **403** | `accessDenied` | missing or unknown API key |
| **404** | `notFound` | `GET /{id}` — no such resource of yours |
| **422** | `missingProperty` | a required property is absent |
| **422** | `invalidValue` | a property has a value we cannot accept |
| **422** | `invalidFormat` | a value does not match the expected format |
| **422** | `referenceNotFound` | a reference in the body names nothing we hold |
| **422** | `unexpectedProperty` | a property we do not support was sent |
| **500** | `internalError` | our fault |
| **501** | `notImplemented` | method not served on this path |

**404 and 422 do not overlap.** A `GET /{id}` for an unknown id is `404 notFound` — the
URL names no resource. An unknown id *inside a body* is `422 referenceNotFound` — the
request is well formed and one of its references does not resolve. That is how MEF
declares them, and every one of our endpoints follows it.

An empty listing is `200 []`, not a 404: the collection exists and is empty.

---

## Products and terms

The catalogue decides what is sold, and it is keyed on `(offering, term)`:

| Offering | Product | Terms |
|---|---|---|
| `ONLINE-BUSINESS-*` | Broadband | 12 months |
| `ONLINE-CORPORATE-*` | DIA | 12 or 36 months |
| `CONNECT-ETHERNET-*` | Ethernet / Access E-Line | 12 or 36 months |
| `CONNECT-ETHERNET-UNI` | Operator UNI port | supplied with the connection, not charged |

The number in the name is the rate in Mbps, and the bandwidth you state must match it.
`ONLINE-BUSINESS-1000/500` is asymmetric — 1000 down, 500 up; the others are symmetric.
Available contract terms may change. Confirm the current terms with Cegecom Sales; 
the API validates requests against the current catalogue

Asking for a term an offering does not have is `422 referenceNotFound`, and the message
names the terms it does have.

---

## Where we are narrower than MEF

A seller may serve less than the specification allows. These are ours, and each is a
decision rather than a gap:

| Rule | Why |
|---|---|
| **`action` must be `add`** | `modify` and `delete` act on an in-service product; we do not implement either |
| **Synchronous only** | `instantSync*: false` is refused rather than answered as though it were true |
| **A quote reference on every order item** | we cannot price an order without the quote that priced it |
| **`productConfiguration` required on an `add`** | MEF requires it; we enforce it |
| **An Access E-Line must be quoted with its UNI** | we do not sell the port separately, and have no UNI inventory to reference |
| **No `POST /cancelQuote`** | unreachable for a synchronous seller — see [Decline a quote](#decline-a-quote) |

---

## Not yet implemented

These are **refused with `422 unexpectedProperty`** rather than accepted and ignored,
so that a successful create response never implies we agreed to something we do not do:

| Property | On | Why |
|---|---|---|
| `expediteIndicator` | order item | no expedite path and no expedited price |
| `coordinatedAction` | order item | we provision in no guaranteed order |
| `tspRestorationPriority` | order item | a United States scheme (ATIS-0404001) |

Only a *stated* request is refused — `expediteIndicator: false` asks for nothing and is
accepted.

**Carried but not acted on here** — `billingAccount`, `endCustomerName`,
`relatedBuyerPON`. They are stored with your request and echoed on the response; no part
of this API acts on them.

**Not served at all:** `POST /cancelQuote`, the notification hub (`POST /hub`),
`GET /geographicSite`, `GET /geographicAddress/{id}`, and product inventory.

---

*Questions, or behaviour that differs in your environment? Raise it with your
Cegecom integration contact, including the endpoint, HTTP status and resource id.
Do not include your API key.*

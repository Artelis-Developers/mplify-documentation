# mplify-documentation# MEF LSO Sonata API - Complete Documentation

Enterprise-grade implementation of the MEF LSO Sonata (Billie) specification for telecommunications service qualification and ordering in Luxembourg.

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Base URL](#base-url)
4. [API Endpoints](#api-endpoints)
   - [Address Validation](#1-address-validation)
   - [Product Offering Qualification](#2-product-offering-qualification-poq)
   - [Quote Management](#3-quote-management)
   - [Product Order Management](#4-product-order-management)
5. [Complete Workflow Example](#complete-workflow-example)
6. [Error Handling](#error-handling)
7. [Configuration](#configuration)

---

## Overview

This system provides a complete RESTful API implementation following MEF (Metro Ethernet Forum) LSO Sonata specifications for:

- **Address Validation** - Geographic address verification with alternatives
- **Product Qualification** - Service availability checking with confidence levels (GREEN/YELLOW/RED)
- **Quote Management** - Quote creation, retrieval, and lifecycle management
- **Product Order Management** - Order creation and tracking based on approved quotes

### Key Features

✅ **MEF Compliance** - Strict adherence to MEF LSO Sonata Billie specification  
✅ **Clean Architecture** - Separation of concerns with stable business logic  
✅ **Serverless** - AWS Lambda with API Gateway  
✅ **DynamoDB Storage** - High-performance NoSQL database for persistence  
✅ **Comprehensive Logging** - Audit trail for all requests/responses  
✅ **API Key Authentication** - Secure caller-based access control  
✅ **National Provider Integration** - Integration with POST Luxembourg FTTH API

---

## Authentication

All endpoints require an API key in the request header:

```bash
x-api-key: YOUR_API_KEY_HERE
```

The API key is validated via AWS API Gateway, and the `apiKeyId` is resolved to a `callerId` for access control.

---

## Base URL

```
https://<function-url>/<stage>/billie
```

Where:
- `<function-url>` is your Lambda Function URL or API Gateway URL
- `<stage>` is typically `staging` or `production`

Example: `https://abc123.lambda-url.eu-west-1.on.aws/staging/billie`

---

## API Endpoints

### 1. Address Validation

**Endpoint**: `POST /geographicAddressValidation`

Validates and standardizes geographic addresses, returning unique address identifiers for use in subsequent API calls.

#### Request Structure

```json
{
  "externalId": "CUSTOMER-REQ-001",
  "submittedGeographicAddress": {
    "@type": "FieldedAddress",
    "streetName": "rue de la Poste",
    "streetNr": "15",
    "postcode": "1234",
    "city": "Luxembourg",
    "country": "Luxembourg"
  },
  "provideAlternative": true
}
```

#### curl Example

```bash
curl -X POST 'https://<function-url>/<stage>/billie/geographicAddressValidation' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "provideAlternative": true,
    "submittedGeographicAddress": {
      "@type": "FieldedAddress",
      "streetName": "rue de la Poste",
      "streetNr": "15",
      "postcode": "1234",
      "city": "Luxembourg",
      "country": "Luxembourg"
    }
  }'
```

#### Response (Success - Address Found)

```json
{
  "id": "urn:mef:lso:sonata:geographicAddressValidation:12345",
  "externalId": "CUSTOMER-REQ-001",
  "validationResult": "success",
  "submittedGeographicAddress": {
    "@type": "FieldedAddress",
    "streetName": "rue de la Poste",
    "streetNr": "15",
    "postcode": "1234",
    "city": "Luxembourg",
    "country": "Luxembourg"
  },
  "bestMatchGeographicAddress": {
    "@type": "GeographicAddressRef",
    "id": "000000000048#000000000000",
    "href": "/geographicAddress/000000000048#000000000000",
    "hasPublicSite": false,
    "allowsNewSite": true
  },
  "alternateGeographicAddress": []
}
```

#### Response (No Match)

```json
{
  "validationResult": "noMatchingAddresses",
  "submittedGeographicAddress": {
    "@type": "FieldedAddress",
    "streetName": "rue de la Poste",
    "streetNr": "15",
    "postcode": "1234",
    "city": "Luxembourg",
    "country": "Luxembourg"
  },
  "alternateGeographicAddress": [
    {
      "@type": "GeographicAddressRef",
      "id": "ALT-ADDRESS-ID",
      "href": "/geographicAddress/ALT-ADDRESS-ID"
    }
  ]
}
```

#### Status Codes

- `200` - Success
- `400` - Validation error (missing fields, invalid format)
- `403` - Authentication error (missing/invalid API key)
- `500` - Internal server error

---

### 2. Product Offering Qualification (POQ)

**Endpoint**: `POST /productOfferingQualification`

Checks service availability and feasibility for a product offering at a specific address. Returns serviceability confidence levels (GREEN/YELLOW/RED) and installation intervals.

#### Request Structure

```json
{
  "externalId": "COLT_POQ_12345",
  "instantSyncQualification": true,
  "provideAlternative": false,
  "projectId": "COLT_Project_ID_67890",
  "requestedPOQCompletionDate": "2029-01-21T23:00:00Z",
  "relatedContactInformation": [
    {
      "number": "+352123456789",
      "emailAddress": "sales@colt.net",
      "role": "buyerContactInformation",
      "organization": "COLT",
      "name": "John Smith"
    }
  ],
  "productOfferingQualificationItem": [
    {
      "id": "DIA",
      "action": "add",
      "product": {
        "productOffering": {
          "id": "productOffering_default_dia-300mbps"
        },
        "productConfiguration": {
          "@type": "DIA",
          "ipvc": {
            "ipvcEndPoint": {}
          },
          "ipUni": {
            "egressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 300,
                "irUnits": "MBPS"
              }
            },
            "ingressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 300,
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
            "id": "000000000048#000000000000",
            "@type": "GeographicAddressRef"
          }
        ]
      },
      "relatedContactInformation": [
        {
          "number": "+352987654321",
          "emailAddress": "location.contact@colt.net",
          "role": "locationContact",
          "organization": "COLT",
          "name": "Jane Doe"
        }
      ]
    }
  ]
}
```

#### curl Example

```bash
curl -X POST 'https://<function-url>/<stage>/billie/productOfferingQualification' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "externalId": "COLT_POQ_12345",
    "instantSyncQualification": true,
    "provideAlternative": false,
    "projectId": "COLT_Project_ID_67890",
    "requestedPOQCompletionDate": "2029-01-21T23:00:00Z",
    "relatedContactInformation": [
      {
        "number": "+352123456789",
        "emailAddress": "sales@colt.net",
        "role": "buyerContactInformation",
        "organization": "COLT",
        "name": "John Smith"
      }
    ],
    "productOfferingQualificationItem": [
      {
        "id": "DIA",
        "action": "add",
        "product": {
          "productOffering": {
            "id": "productOffering_default_dia-300mbps"
          },
          "productConfiguration": {
            "@type": "DIA",
            "ipvc": {
              "ipvcEndPoint": {}
            },
            "ipUni": {
              "egressBandwidthProfileEnvelope": {
                "cir": {
                  "irValue": 300,
                  "irUnits": "MBPS"
                }
              },
              "ingressBandwidthProfileEnvelope": {
                "cir": {
                  "irValue": 300,
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
              "id": "000000000048#000000000000",
              "@type": "GeographicAddressRef"
            }
          ]
        },
        "relatedContactInformation": [
          {
            "number": "+352987654321",
            "emailAddress": "location.contact@colt.net",
            "role": "locationContact",
            "organization": "COLT",
            "name": "Jane Doe"
          }
        ]
      }
    ]
  }'
```

#### Response

```json
{
  "id": "urn:mef:lso:sonata:poq:12345",
  "externalId": "COLT_POQ_12345",
  "state": "done",
  "projectId": "COLT_Project_ID_67890",
  "requestedPOQCompletionDate": "2029-01-21T23:00:00Z",
  "productOfferingQualificationItem": [
    {
      "id": "DIA",
      "action": "add",
      "product": {
        "productOffering": {
          "id": "productOffering_default_dia-300mbps"
        },
        "place": [
          {
            "role": "INSTALL_LOCATION",
            "id": "000000000048#000000000000",
            "@type": "GeographicAddressRef"
          }
        ],
        "productConfiguration": {
          "@type": "DIA",
          "ipvc": {
            "ipvcEndPoint": {}
          },
          "ipUni": {
            "egressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 300,
                "irUnits": "MBPS"
              }
            },
            "ingressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 300,
                "irUnits": "MBPS"
              }
            }
          }
        }
      },
      "serviceabilityConfidence": "green",
      "serviceabilityConfidenceReason": "Serviceable (fiber in service)",
      "subjectToFeasibilityCheck": false,
      "installationInterval": {
        "amount": 45,
        "units": "calendarDays"
      },
      "qualificationDate": "2025-01-16T10:30:00Z"
    }
  ]
}
```

#### Serviceability Confidence Levels

- **GREEN**: Service is available, fiber in service, capacity available
  - Installation interval: 45 days (default)
  - `subjectToFeasibilityCheck`: `false`
  - Condition: `routeStatus="S"` AND `inServiceDate` exists AND `freeFibers >= 1`

- **YELLOW**: Capacity/feasibility check required
  - Installation interval: 90 days (default)
  - `subjectToFeasibilityCheck`: `true`
  - Condition: Default when GREEN conditions not met

- **RED**: Service not available (currently not returned, defaults to YELLOW)

#### Status Codes

- `200` - Success
- `400` - Validation error
- `403` - Authentication error
- `409` - Duplicate `externalId`
- `500` - Internal server error (including Critical Error 1129 for FTTH failures)

---

### 3. Quote Management

#### 3.1 Create Quote

**Endpoint**: `POST /quote`

Creates a new quote with pricing based on product offering, contract term, and caller discounts.

##### Request Structure

```json
{
  "externalId": "COLT_Quote_ID_12345",
  "buyerRequestedQuoteLevel": "firm",
  "projectId": "COLT_Project_ID_67890",
  "instantSyncQuote": true,
  "requestedQuoteCompletionDate": "2025-10-22T23:00:00Z",
  "relatedContactInformation": [
    {
      "number": "+352123456789",
      "emailAddress": "sales@colt.net",
      "role": "buyerContactInformation",
      "organization": "COLT",
      "name": "John Smith"
    }
  ],
  "quoteItem": [
    {
      "id": "DIA",
      "action": "add",
      "requestedQuoteItemTerm": {
        "duration": {
          "amount": 24,
          "units": "calendarMonths"
        },
        "name": "Standard Service Term",
        "endOfTermAction": "roll"
      },
      "product": {
        "productOffering": {
          "id": "productOffering_default_dia-300mbps"
        },
        "place": [
          {
            "role": "INSTALL_LOCATION",
            "id": "addr-12345-67890-abcdef",
            "@type": "GeographicAddressRef"
          }
        ],
        "productConfiguration": {
          "@type": "DIA",
          "ipvc": {
            "ipvcEndPoint": {}
          },
          "ipUni": {
            "egressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 300,
                "irUnits": "MBPS"
              }
            },
            "ingressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 300,
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
        "productOfferingQualificationId": "POQ_colt_2025-10-12T22-17-01_abc123def",
        "id": "DIA"
      }
    }
  ]
}
```

##### curl Example

```bash
curl -X POST 'https://<function-url>/<stage>/billie/quote' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "externalId": "COLT_Quote_ID_12345",
    "buyerRequestedQuoteLevel": "firm",
    "projectId": "COLT_Project_ID_67890",
    "instantSyncQuote": true,
    "requestedQuoteCompletionDate": "2025-10-22T23:00:00Z",
    "relatedContactInformation": [
      {
        "number": "+352123456789",
        "emailAddress": "sales@colt.net",
        "role": "buyerContactInformation",
        "organization": "COLT",
        "name": "John Smith"
      }
    ],
    "quoteItem": [
      {
        "id": "DIA",
        "action": "add",
        "requestedQuoteItemTerm": {
          "duration": {
            "amount": 24,
            "units": "calendarMonths"
          },
          "name": "Standard Service Term",
          "endOfTermAction": "roll"
        },
        "product": {
          "productOffering": {
            "id": "productOffering_default_dia-300mbps"
          },
          "place": [
            {
              "role": "INSTALL_LOCATION",
              "id": "addr-12345-67890-abcdef",
              "@type": "GeographicAddressRef"
            }
          ],
          "productConfiguration": {
            "@type": "DIA",
            "ipvc": {
              "ipvcEndPoint": {}
            },
            "ipUni": {
              "egressBandwidthProfileEnvelope": {
                "cir": {
                  "irValue": 300,
                  "irUnits": "MBPS"
                }
              },
              "ingressBandwidthProfileEnvelope": {
                "cir": {
                  "irValue": 300,
                  "irUnits": "MBPS"
                }
              }
            }
          }
        },
        "productOfferingQualificationItem": {
          "productOfferingQualificationId": "POQ_colt_2025-10-12T22-17-01_abc123def",
          "id": "DIA"
        }
      }
    ]
  }'
```

##### Response

```json
{
  "id": "urn:mef:lso:sonata:quote:COLT_Quote_ID_12345",
  "externalId": "COLT_Quote_ID_12345",
  "state": "approved.orderable",
  "quoteLevel": "firmSubjectToFeasibilityCheck",
  "quoteDate": "2025-01-16T10:30:00Z",
  "validUntil": "2025-02-15T10:30:00Z",
  "projectId": "COLT_Project_ID_67890",
  "relatedContactInformation": [
    {
      "number": "+352123456789",
      "emailAddress": "sales@colt.net",
      "role": "buyerContactInformation",
      "organization": "COLT",
      "name": "John Smith"
    }
  ],
  "quoteItem": [
    {
      "id": "DIA",
      "action": "add",
      "state": "approved.orderable",
      "installationInterval": {
        "amount": 90,
        "units": "calendarDays"
      },
      "quoteItemTerm": {
        "duration": {
          "amount": 24,
          "units": "calendarMonths"
        },
        "name": "Standard Service Term",
        "endOfTermAction": "roll"
      },
      "product": {
        "productOffering": {
          "id": "productOffering_default_dia-300mbps"
        },
        "place": [
          {
            "role": "INSTALL_LOCATION",
            "id": "addr-12345-67890-abcdef",
            "@type": "GeographicAddressRef"
          }
        ]
      },
      "productOfferingPrice": [
        {
          "name": "Installation Fee",
          "priceType": "nonRecurring",
          "price": {
            "dutyFreeAmount": {
              "value": 500.00,
              "unit": "EUR"
            }
          }
        },
        {
          "name": "Monthly Service Fee",
          "priceType": "recurring",
          "recurringChargePeriod": "month",
          "price": {
            "dutyFreeAmount": {
              "value": 150.00,
              "unit": "EUR"
            }
          }
        }
      ]
    }
  ]
}
```

#### 3.2 List Quotes

**Endpoint**: `GET /quote`

Lists quotes for the authenticated caller with optional filtering.

##### Query Parameters

- `state` - Filter by quote state (e.g., `approved.orderable`, `inProgress`)
- `externalId` - Filter by specific external ID
- `projectId` - Filter by project ID
- `limit` - Max results (default: 50)
- `offset` - Pagination offset (default: 0)

##### curl Example

```bash
# List all quotes
curl -X GET 'https://<function-url>/<stage>/billie/quote' \
  -H 'x-api-key: YOUR_API_KEY_HERE'

# List with filters
curl -X GET 'https://<function-url>/<stage>/billie/quote?state=approved.orderable&limit=20&offset=0' \
  -H 'x-api-key: YOUR_API_KEY_HERE'

# Filter by projectId
curl -X GET 'https://<function-url>/<stage>/billie/quote?projectId=COLT_Project_ID_67890' \
  -H 'x-api-key: YOUR_API_KEY_HERE'
```

##### Response

```json
[
  {
    "id": "urn:mef:lso:sonata:quote:COLT_Quote_ID_12345",
    "externalId": "COLT_Quote_ID_12345",
    "state": "approved.orderable",
    "quoteDate": "2025-01-16T10:30:00Z",
    "href": "/staging/billie/quote/COLT_Quote_ID_12345"
  },
  {
    "id": "urn:mef:lso:sonata:quote:COLT_Quote_ID_67890",
    "externalId": "COLT_Quote_ID_67890",
    "state": "accepted",
    "quoteDate": "2025-01-15T09:00:00Z",
    "href": "/staging/billie/quote/COLT_Quote_ID_67890"
  }
]
```

#### 3.3 Get Quote by ID

**Endpoint**: `GET /quote/{externalId}`

Retrieves full quote details by external ID.

##### curl Example

```bash
curl -X GET 'https://<function-url>/<stage>/billie/quote/COLT_Quote_ID_12345' \
  -H 'x-api-key: YOUR_API_KEY_HERE'
```

##### Response

Returns the complete quote object (same structure as POST response).

#### 3.4 Update Quote State

**Endpoint**: `PATCH /quote/{externalId}`

Updates the state of an existing quote.

##### Request Structure

```json
{
  "state": "accepted",
  "stateChangeReason": "Customer approved the quote"
}
```

##### curl Example

```bash
curl -X PATCH 'https://<function-url>/<stage>/billie/quote/COLT_Quote_ID_12345' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "state": "accepted",
    "stateChangeReason": "Customer approved the quote"
  }'
```

##### Response

Returns the updated quote object with new state and appended `stateChange` entry.

#### Status Codes (Quote Management)

- `200` - Success
- `400` - Validation error
- `403` - Authentication error
- `404` - Quote not found
- `409` - Duplicate `externalId` (POST only)
- `500` - Internal server error

---

### 4. Product Order Management

#### 4.1 Create Product Order

**Endpoint**: `POST /productOrder`

Creates a new product order based on an approved quote. **Quote ID is mandatory**.

##### Request Structure

```json
{
  "externalId": "COLT_Order_ID_12345",
  "projectId": "COLT_Project_ID_67890",
  "relatedContactInformation": [
    {
      "role": "productOrderContact",
      "name": "John Smith",
      "organization": "COLT",
      "emailAddress": "sales@colt.net",
      "number": "+352123456789"
    },
    {
      "role": "sellerContactInformation",
      "name": "Cegecom Sales",
      "organization": "Cegecom",
      "emailAddress": "sales@cegecom.lu",
      "number": "264991",
      "numberExtension": "352",
      "postalAddress": "3 Rue Jean Piret, 2350 Gasperich Luxembourg"
    }
  ],
  "productOrderItem": [
    {
      "id": "DIA-1",
      "action": "add",
      "requestedCompletionDate": "2025-10-22T23:00:00Z",
      "product": {
        "productOffering": {
          "id": "productOffering_default_dia-300mbps"
        },
        "place": [
          {
            "role": "INSTALL_LOCATION",
            "@type": "GeographicAddressRef",
            "id": "addr-12345-67890-abcdef"
          }
        ],
        "productConfiguration": {
          "@type": "urn:mef:lso:spec:carrier-ethernet:eaccess:v1.0.0:all",
          "ipvc": {
            "ipvcEndPoint": {}
          },
          "ipUni": {
            "egressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 300,
                "irUnits": "MBPS"
              }
            },
            "ingressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 300,
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
        "productOfferingQualificationId": "POQ_colt_2025-10-12T22-17-01_abc123def",
        "id": "DIA"
      },
      "quoteItem": {
        "quoteId": "COLT_Quote_ID_12345",
        "id": "DIA"
      },
      "relatedContactInformation": [
        {
          "role": "siteContact",
          "name": "On-Site Contact",
          "number": "+352000000000"
        }
      ]
    }
  ]
}
```

**Critical Requirement**: The `quoteId` field in `productOrderItem[].quoteItem.quoteId` is **MANDATORY**. The system will:
1. Validate that `quoteId` is present
2. Verify that the quote exists and belongs to the caller
3. Return specific errors if validation fails

##### curl Example

```bash
curl -X POST 'https://<function-url>/<stage>/billie/productOrder' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "externalId": "COLT_Order_ID_12345",
    "projectId": "COLT_Project_ID_67890",
    "relatedContactInformation": [
      {
        "role": "productOrderContact",
        "name": "John Smith",
        "organization": "COLT",
        "emailAddress": "sales@colt.net",
        "number": "+352123456789"
      },
      {
        "role": "sellerContactInformation",
        "name": "Cegecom Sales",
        "organization": "Cegecom",
        "emailAddress": "sales@cegecom.lu",
        "number": "264991",
        "numberExtension": "352",
        "postalAddress": "3 Rue Jean Piret, 2350 Gasperich Luxembourg"
      }
    ],
    "productOrderItem": [
      {
        "id": "DIA-1",
        "action": "add",
        "requestedCompletionDate": "2025-10-22T23:00:00Z",
        "product": {
          "productOffering": {
            "id": "productOffering_default_dia-300mbps"
          },
          "place": [
            {
              "role": "INSTALL_LOCATION",
              "@type": "GeographicAddressRef",
              "id": "addr-12345-67890-abcdef"
            }
          ],
          "productConfiguration": {
            "@type": "urn:mef:lso:spec:carrier-ethernet:eaccess:v1.0.0:all",
            "ipvc": {
              "ipvcEndPoint": {}
            },
            "ipUni": {
              "egressBandwidthProfileEnvelope": {
                "cir": {
                  "irValue": 300,
                  "irUnits": "MBPS"
                }
              },
              "ingressBandwidthProfileEnvelope": {
                "cir": {
                  "irValue": 300,
                  "irUnits": "MBPS"
                }
              }
            }
          }
        },
        "productOfferingQualificationItem": {
          "productOfferingQualificationId": "POQ_colt_2025-10-12T22-17-01_abc123def",
          "id": "DIA"
        },
        "quoteItem": {
          "quoteId": "COLT_Quote_ID_12345",
          "id": "DIA"
        },
        "relatedContactInformation": [
          {
            "role": "siteContact",
            "name": "On-Site Contact",
            "number": "+352000000000"
          }
        ]
      }
    ]
  }'
```

##### Response

```json
{
  "id": "COLT_Order_ID_12345",
  "href": "https://api.example.com/billie/productOrder/COLT_Order_ID_12345",
  "externalId": "COLT_Order_ID_12345",
  "projectId": "COLT_Project_ID_67890",
  "state": "acknowledged",
  "orderDate": "2025-01-16T09:12:33Z",
  "requestedCompletionDate": "2025-10-22T23:00:00Z",
  "relatedContactInformation": [
    {
      "role": "productOrderContact",
      "name": "John Smith",
      "organization": "COLT",
      "emailAddress": "sales@colt.net",
      "number": "+352123456789"
    }
  ],
  "productOrderItem": [
    {
      "id": "DIA-1",
      "action": "add",
      "state": "acknowledged",
      "requestedCompletionDate": "2025-10-22T23:00:00Z",
      "product": {
        "productOffering": {
          "id": "productOffering_default_dia-300mbps"
        },
        "place": [
          {
            "role": "INSTALL_LOCATION",
            "@type": "GeographicAddressRef",
            "id": "addr-12345-67890-abcdef"
          }
        ]
      },
      "quoteItem": {
        "quoteId": "COLT_Quote_ID_12345",
        "id": "DIA"
      }
    }
  ],
  "stateChange": [
    {
      "state": "acknowledged",
      "changeDate": "2025-01-16T09:12:33Z",
      "reason": "Order received"
    }
  ],
  "note": []
}
```

#### 4.2 List Orders

**Endpoint**: `GET /productOrder`

Lists orders for the authenticated caller with optional filtering.

##### Query Parameters

- `state` - Filter by order state (e.g., `acknowledged`, `inProgress`)
- `externalId` - Filter by specific external ID
- `projectId` - Filter by project ID
- `limit` - Max results (default: 50)
- `offset` - Pagination offset (default: 0)

##### curl Example

```bash
# List all orders
curl -X GET 'https://<function-url>/<stage>/billie/productOrder' \
  -H 'x-api-key: YOUR_API_KEY_HERE'

# List with filters
curl -X GET 'https://<function-url>/<stage>/billie/productOrder?state=acknowledged&limit=20' \
  -H 'x-api-key: YOUR_API_KEY_HERE'

# Filter by projectId
curl -X GET 'https://<function-url>/<stage>/billie/productOrder?projectId=COLT_Project_ID_67890' \
  -H 'x-api-key: YOUR_API_KEY_HERE'
```

##### Response

```json
[
  {
    "id": "COLT_Order_ID_12345",
    "externalId": "COLT_Order_ID_12345",
    "state": "acknowledged",
    "orderDate": "2025-01-16T09:12:33Z",
    "href": "/staging/billie/productOrder/COLT_Order_ID_12345"
  },
  {
    "id": "COLT_Order_ID_67890",
    "externalId": "COLT_Order_ID_67890",
    "state": "inProgress",
    "orderDate": "2025-01-15T08:00:00Z",
    "href": "/staging/billie/productOrder/COLT_Order_ID_67890"
  }
]
```

#### 4.3 Get Order by ID

**Endpoint**: `GET /productOrder/{externalId}`

Retrieves full order details by external ID.

##### curl Example

```bash
curl -X GET 'https://<function-url>/<stage>/billie/productOrder/COLT_Order_ID_12345' \
  -H 'x-api-key: YOUR_API_KEY_HERE'
```

##### Response

Returns the complete order object (same structure as POST response).

#### Status Codes (Product Order Management)

- `200` - Success
- `400` - Validation error
  - `quoteIdRequired` - Quote ID is missing from request
- `403` - Authentication error
- `404` - Resource not found
  - `quoteNotFound` - Referenced quote does not exist or doesn't belong to caller
- `409` - Duplicate `externalId`
- `500` - Internal server error

#### Quote Validation Rules

**Critical Requirement**: Every order must reference a valid quote.

1. **Missing Quote ID**:
   - Error Code: `quoteIdRequired`
   - HTTP Status: `400`
   - Message: "quoteId is required in productOrderItem[].quoteItem.quoteId"

2. **Invalid Quote ID**:
   - Error Code: `quoteNotFound`
   - HTTP Status: `404`
   - Message: "Quote with ID '{quoteId}' not found or does not belong to caller"

3. **Quote Verification**:
   - The system checks that the quote exists in the database
   - The system verifies the quote belongs to the requesting caller
   - Without a valid quote, no order can be created

---

## Complete Workflow Example

Here's a complete end-to-end workflow for ordering a service:

### Step 1: Validate Address

```bash
curl -X POST 'https://<function-url>/<stage>/billie/geographicAddressValidation' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "provideAlternative": true,
    "submittedGeographicAddress": {
      "@type": "FieldedAddress",
      "streetName": "rue de la Poste",
      "streetNr": "15",
      "postcode": "1234",
      "city": "Luxembourg",
      "country": "Luxembourg"
    }
  }'
```

**Save the returned `id` from `bestMatchGeographicAddress`** (e.g., `000000000048#000000000000`)

### Step 2: Check Product Qualification

```bash
curl -X POST 'https://<function-url>/<stage>/billie/productOfferingQualification' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "externalId": "COLT_POQ_12345",
    "instantSyncQualification": true,
    "productOfferingQualificationItem": [
      {
        "id": "DIA",
        "action": "add",
        "product": {
          "productOffering": {
            "id": "productOffering_default_dia-300mbps"
          },
          "place": [
            {
              "role": "INSTALL_LOCATION",
              "id": "000000000048#000000000000",
              "@type": "GeographicAddressRef"
            }
          ]
        }
      }
    ]
  }'
```

**Save the returned `id`** (e.g., `POQ_colt_2025-10-12T22-17-01_abc123def`)

### Step 3: Create Quote

```bash
curl -X POST 'https://<function-url>/<stage>/billie/quote' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "externalId": "COLT_Quote_ID_12345",
    "projectId": "COLT_Project_ID_67890",
    "quoteItem": [
      {
        "id": "DIA",
        "action": "add",
        "requestedQuoteItemTerm": {
          "duration": {
            "amount": 24,
            "units": "calendarMonths"
          }
        },
        "product": {
          "productOffering": {
            "id": "productOffering_default_dia-300mbps"
          },
          "place": [
            {
              "role": "INSTALL_LOCATION",
              "id": "000000000048#000000000000",
              "@type": "GeographicAddressRef"
            }
          ]
        },
        "productOfferingQualificationItem": {
          "productOfferingQualificationId": "POQ_colt_2025-10-12T22-17-01_abc123def",
          "id": "DIA"
        }
      }
    ]
  }'
```

**Save the returned `externalId`** (e.g., `COLT_Quote_ID_12345`)

### Step 4: Accept Quote

```bash
curl -X PATCH 'https://<function-url>/<stage>/billie/quote/COLT_Quote_ID_12345' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "state": "accepted",
    "stateChangeReason": "Customer approved the quote"
  }'
```

### Step 5: Create Order

```bash
curl -X POST 'https://<function-url>/<stage>/billie/productOrder' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "externalId": "COLT_Order_ID_12345",
    "projectId": "COLT_Project_ID_67890",
    "productOrderItem": [
      {
        "id": "DIA-1",
        "action": "add",
        "product": {
          "productOffering": {
            "id": "productOffering_default_dia-300mbps"
          },
          "place": [
            {
              "role": "INSTALL_LOCATION",
              "@type": "GeographicAddressRef",
              "id": "000000000048#000000000000"
            }
          ]
        },
        "quoteItem": {
          "quoteId": "COLT_Quote_ID_12345",
          "id": "DIA"
        }
      }
    ]
  }'
```

### Step 6: Check Order Status

```bash
curl -X GET 'https://<function-url>/<stage>/billie/productOrder/COLT_Order_ID_12345' \
  -H 'x-api-key: YOUR_API_KEY_HERE'
```

---

## Error Handling

All errors follow the MEF error response format:

```json
{
  "code": "missingProperty",
  "reason": "Missing required field: externalId",
  "message": "The request is missing a required field",
  "status": "400",
  "referenceError": "https://api.example.com/errors/400"
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `missingProperty` | 400 | Required field missing |
| `invalidValue` | 400 | Field contains invalid value |
| `accessDenied` | 403 | API key invalid or missing |
| `notFound` | 404 | Resource not found |
| `quoteIdRequired` | 400 | Quote ID is missing from order request |
| `quoteNotFound` | 404 | Referenced quote does not exist or doesn't belong to caller |
| `conflict` | 409 | Duplicate externalId |
| `internalError` | 500 | Internal server error |
| `1129` | 500 | Critical FTTH API error |
| `databaseFailure` | 500 | Critical database error |

### Error Response Examples

#### Missing API Key (403)

```json
{
  "code": "accessDenied",
  "reason": "Missing API key",
  "message": "Authentication required",
  "status": "403"
}
```

#### Duplicate External ID (409)

```json
{
  "code": "conflict",
  "reason": "Resource conflict: externalId already exists",
  "message": "A resource with this externalId already exists",
  "status": "409"
}
```

#### Quote Not Found (404)

```json
{
  "code": "quoteNotFound",
  "reason": "Quote with ID 'COLT_Quote_ID_12345' not found or does not belong to caller",
  "message": "The requested quote could not be found",
  "status": "404"
}
```

#### Missing Quote ID (400)

```json
{
  "code": "quoteIdRequired",
  "reason": "quoteId is required in productOrderItem[].quoteItem.quoteId",
  "message": "Quote ID is mandatory for creating product orders",
  "status": "400"
}
```

---

## Configuration

### Environment Variables

#### Required for FTTH Integration
- `POST_URL` - POST Luxembourg FTTH API URL
- `SYSTEM_ID` - System identifier
- `RSP_ID` - RSP identifier
- `CLIENT_ID` - API client ID
- `CLIENT_SECRET` - API client secret

#### Optional Configuration
- `CONFIG_S3_BUCKET` - S3 bucket for configuration overrides
- `CONFIG_S3_KEY` - Policy configuration key
- `MAPPING_S3_KEY` - MEF mapping configuration key

### DynamoDB Tables

#### Address Validation
- `X001-MPLIFY-AV-REQUEST` - Address validation requests
- `X001-MPLIFY-AV-ANSWER` - Address validation responses

#### Product Qualification
- `X001-MPLIFY-POQ-REQUEST` - POQ requests with FTTH payload
- `X001-MPLIFY-POQ-ANSWER` - POQ responses

#### Quote Management
- `X001-MPLIFY-QUOTE-REQUEST` - Quote creation and updates
- `X001-MPLIFY-QUOTE-ANSWER` - Complete quote records

#### Product Order Management
- `X001-MPLIFY-ORDER-REQUEST` - Order requests with quoteId
- `X001-MPLIFY-ORDER-ANSWER` - Complete order records

#### Pricing & Configuration
- `X001-BSS-QUOTE-PRICING-DIA` - DIA product pricing (PK: `productOfferingId`, SK: `requestedQuoteItemTerm`)
- `X001-BSS-QUOTE-PRICING-INTERNET` - Internet product pricing
- `X001-MPLIFY-API-KEYS` - API key configuration with discounts (PK: `apiKeyId`)

---

## MEF Compliance

This implementation strictly follows:
- MEF LSO Sonata Developer Guide
- MEF 79 - Address, Service Site, and Product Offering Qualification Management
- MEF 80 - Quote Management
- MEF 57.2 - Product Order Management

**Key Compliance Features:**
- Proper HTTP status codes (200, 400, 403, 404, 409, 500)
- Complete field coverage with explicit null values
- MEF error response format
- Correlation IDs for request tracing
- CORS support for browser clients

---

**Version**: 3.0  
**Last Updated**: January 2026  
**Architecture**: Clean Architecture with Hexagonal patterns  
**Status**: Production-ready

---

*This documentation reflects the actual code implementation. All curl examples have been tested and verified against the running API.*

# MEF LSO Sonata API - Complete Documentation

**Version:** 4.0  
**Last Updated:** January 26, 2026  
**Status:** Production-ready

---

## Table of Contents

1. [Overview](#overview)
2. [Technology Stack](#technology-stack)
3. [Architecture](#architecture)
4. [Authentication](#authentication)
5. [API Endpoints](#api-endpoints)
   - [Address Validation](#1-address-validation)
   - [Product Offering Qualification](#2-product-offering-qualification-poq)
   - [Quote Management](#3-quote-management)
   - [Product Order Management](#4-product-order-management)
6. [Complete Workflow Example](#complete-workflow-example)
7. [Error Handling](#error-handling)
8. [POQ Errors Explained](#poq-errors-explained)
9. [Product Configuration](#product-configuration)
10. [Data Persistence](#data-persistence)

---

## Overview

Enterprise-grade implementation of the MEF (Metro Ethernet Forum) LSO Sonata (Billie) specification for telecommunications service qualification and ordering. This system provides a complete RESTful API for:

- **Address Validation** - Geographic address verification with 96%+ building coverage
- **Product Qualification (POQ)** - Service availability checking with confidence levels (GREEN/YELLOW/RED)
- **Quote Management** - Quote creation, retrieval, and lifecycle management with dynamic pricing
- **Product Order Management** - Order creation and tracking based on approved quotes

### Key Features

✅ **MEF Compliance** - Strict adherence to MEF LSO Sonata (Billie) specification  
✅ **Clean Architecture** - Separation of concerns with stable business logic  
✅ **Serverless** - AWS Lambda with API Gateway for scalability  
✅ **High Performance** - DynamoDB for sub-millisecond data access  
✅ **Comprehensive Logging** - Full audit trail for all requests/responses  
✅ **Secure** - API key authentication with caller-based access control  
✅ **National Coverage** - Address validation for 96%+ of buildings  

### Base URL

```
https://<function-url>/<stage>/billie
```

Example: `https://abc123.lambda-url.eu-central-1.on.aws/staging/billie`

---

## Technology Stack

### Runtime & Deployment
- **Runtime**: Python 3.13
- **Deployment**: AWS Lambda with Function URL
- **Compute Model**: Serverless with VPC integration
- **API Gateway**: AWS API Gateway for routing and authentication

### Storage & Persistence
- **DynamoDB**: Event logging, quote persistence, order tracking, and pricing tables
- **S3**: Address database, configuration files, and static assets
- **SQLite**: Local address lookup cache for performance optimization

### Integration
- **National Fiber Network API**: Integration with national fiber infrastructure provider for availability checks
- **OAuth 2.0**: Secure API authentication with client credentials flow

### Development
- **Language**: Python 3.13 with type hints
- **Testing**: pytest for unit and integration tests
- **Deployment**: PowerShell and Bash scripts for automated deployment

---

## Architecture

Built on **Clean Architecture** principles with clear separation between stable business logic and volatile infrastructure:

```
┌─────────────────────────────────────────────────────────┐
│                   API Gateway Layer                      │
│                  (HTTP Handlers)                         │
├─────────────────────────────────────────────────────────┤
│              Application Services Layer                  │
│            (Orchestration & Workflows)                   │
├─────────────────────────────────────────────────────────┤
│                 Use Cases Layer                          │
│              (Business Workflows)                        │
├─────────────────────────────────────────────────────────┤
│                  Domain Layer                            │
│             (Core Business Rules)                        │
├─────────────────────────────────────────────────────────┤
│              Infrastructure Layer                        │
│     (External Integrations & Adapters)                   │
└─────────────────────────────────────────────────────────┘
```

### Directory Structure

```
src/
├── domain/         Core business logic (stable)
│   ├── exceptions/ Error definitions
│   ├── models/     Domain entities
│   └── rules/      Business rules
├── shared/         Shared utilities and configuration (stable)
│   ├── config/     Configuration files
│   ├── constants/  MEF constants
│   └── utils/      Helper functions
├── use_cases/      Application workflows (semi-stable)
│   ├── address/    Address validation
│   ├── qualification/ Product qualification
│   ├── quote/      Quote management
│   └── order/      Order management
├── adapters/       External integrations (volatile)
│   ├── http/       HTTP handlers
│   ├── logging/    Logging adapters
│   └── external/   External service integrations
└── services/       Infrastructure services
    ├── api_key_service.py
    ├── database_service.py
    ├── pricing_service.py
    └── post_service.py (National fiber network integration)
```

---

## Authentication

All endpoints require an API key in the request header:

```http
x-api-key: YOUR_API_KEY_HERE
```

The API key is validated via AWS API Gateway, and the `apiKeyId` is resolved to a `callerId` for access control. Each caller can only access their own resources (quotes, orders).

---

## API Endpoints

### 1. Address Validation

**Endpoint**: `POST /geographicAddressValidation`

Validates and standardizes geographic addresses, returning unique address identifiers for use in subsequent API calls. The system maintains an address database with 96%+ coverage of buildings.

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

**Important**: Save the `id` from `bestMatchGeographicAddress` (e.g., `000000000048#000000000000`) for use in subsequent API calls.

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

This endpoint integrates with the national fiber network provider's API to check fiber availability and route status.

#### Supported Product Types

- **CONNECT-ETHERNET** (Type: DIA - Dedicated Internet Access)
  - Symmetric bandwidth (upload = download)
  - SLA-backed service with guaranteed performance
  - Examples: `CONNECT-ETHERNET-100`, `CONNECT-ETHERNET-300`, `CONNECT-ETHERNET-1000`

- **ONLINE-BUSINESS** (Type: Internet)
  - Business-grade internet service
  - Best-effort delivery with high reliability
  - Examples: `ONLINE-BUSINESS-100`, `ONLINE-BUSINESS-300`, `ONLINE-BUSINESS-1000`

#### Request Structure (DIA Example)

```json
{
  "externalId": "COMPANYXYZ_POQ_12345",
  "instantSyncQualification": true,
  "provideAlternative": false,
  "projectId": "COMPANYXYZ_Project_ID_67890",
  "requestedPOQCompletionDate": "2029-01-21T23:00:00Z",
  "relatedContactInformation": [
    {
      "number": "+352123456789",
      "emailAddress": "sales@COMPANYXYZ.net",
      "role": "buyerContactInformation",
      "organization": "COMPANYXYZ",
      "name": "John Smith"
    }
  ],
  "productOfferingQualificationItem": [
    {
      "id": "DIA",
      "action": "add",
      "product": {
        "productOffering": {
          "id": "CONNECT-ETHERNET-1000"
        },
        "productConfiguration": {
          "@type": "DIA",
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
                "irValue": 1000,
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
          "emailAddress": "location.contact@COMPANYXYZ.net",
          "role": "locationContact",
          "organization": "COMPANYXYZ",
          "name": "Jane Doe"
        }
      ]
    }
  ]
}
```

#### Request Structure (Internet Example)

```json
{
  "externalId": "COMPANYXYZ_POQ_67890",
  "instantSyncQualification": true,
  "productOfferingQualificationItem": [
    {
      "id": "INTERNET",
      "action": "add",
      "product": {
        "productOffering": {
          "id": "ONLINE-BUSINESS-300"
        },
        "productConfiguration": {
          "@type": "INTERNET",
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
}
```

#### curl Example

```bash
curl -X POST 'https://<function-url>/<stage>/billie/productOfferingQualification' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "externalId": "COMPANYXYZ_POQ_12345",
    "instantSyncQualification": true,
    "productOfferingQualificationItem": [
      {
        "id": "DIA",
        "action": "add",
        "product": {
          "productOffering": {
            "id": "CONNECT-ETHERNET-1000"
          },
          "productConfiguration": {
            "@type": "DIA",
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
                  "irValue": 1000,
                  "irUnits": "MBPS"
                }
              }
            }
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

#### Response (Green Confidence)

```json
{
  "id": "urn:mef:lso:sonata:poq:12345",
  "externalId": "COMPANYXYZ_POQ_12345",
  "state": "done",
  "projectId": "COMPANYXYZ_Project_ID_67890",
  "requestedPOQCompletionDate": "2029-01-21T23:00:00Z",
  "productOfferingQualificationItem": [
    {
      "id": "DIA",
      "action": "add",
      "product": {
        "productOffering": {
          "id": "CONNECT-ETHERNET-1000"
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
                "irValue": 1000,
                "irUnits": "MBPS"
              }
            },
            "ingressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 1000,
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
      "qualificationDate": "2026-01-26T10:30:00Z"
    }
  ]
}
```

#### Serviceability Confidence Levels

| Level | Meaning | Installation Days | Feasibility Check | Conditions |
|-------|---------|-------------------|-------------------|------------|
| **GREEN** | Service available, fiber operational, capacity available | 45 days (default) | No | Route status = "S" AND in-service date exists AND free fibers ≥ 1 |
| **YELLOW** | Capacity/feasibility check required | 90 days (default) | Yes | Default when GREEN conditions not met |
| **RED** | Service not available at this location | N/A | Yes | Fiber point-of-presence (FO-POP) not activated |

**Important**: The system queries the national fiber network provider's API to determine:
- Route status (whether fiber is in service)
- Free fiber count (available capacity)
- In-service date (when the route became operational)
- FO-POP identifier (fiber point-of-presence location)

#### Status Codes

- `200` - Success
- `400` - Validation error (see [POQ Errors Explained](#poq-errors-explained))
- `403` - Authentication error
- `409` - Duplicate `externalId`
- `500` - Internal server error or Critical Error 1129 (fiber network API failure)

---

### 3. Quote Management

#### 3.1 Create Quote

**Endpoint**: `POST /quote`

Creates a new quote with pricing based on product offering, contract term, and caller-specific discounts. Pricing is dynamically retrieved from DynamoDB tables.

##### Request Structure

```json
{
  "externalId": "COMPANYXYZ_Quote_ID_12345",
  "buyerRequestedQuoteLevel": "firm",
  "projectId": "COMPANYXYZ_Project_ID_67890",
  "instantSyncQuote": true,
  "requestedQuoteCompletionDate": "2026-10-22T23:00:00Z",
  "relatedContactInformation": [
    {
      "number": "+352123456789",
      "emailAddress": "sales@COMPANYXYZ.net",
      "role": "buyerContactInformation",
      "organization": "COMPANYXYZ",
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
          "id": "CONNECT-ETHERNET-1000"
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
                "irValue": 1000,
                "irUnits": "MBPS"
              }
            },
            "ingressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 1000,
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
        "productOfferingQualificationId": "POQ_COMPANYXYZ_2026-01-26_abc123def",
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
    "externalId": "COMPANYXYZ_Quote_ID_12345",
    "buyerRequestedQuoteLevel": "firm",
    "projectId": "COMPANYXYZ_Project_ID_67890",
    "instantSyncQuote": true,
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
            "id": "CONNECT-ETHERNET-1000"
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
          "productOfferingQualificationId": "POQ_COMPANYXYZ_2026-01-26_abc123def",
          "id": "DIA"
        }
      }
    ]
  }'
```

##### Response

```json
{
  "id": "urn:mef:lso:sonata:quote:COMPANYXYZ_Quote_ID_12345",
  "externalId": "COMPANYXYZ_Quote_ID_12345",
  "state": "approved.orderable",
  "quoteLevel": "firmSubjectToFeasibilityCheck",
  "quoteDate": "2026-01-26T10:30:00Z",
  "validUntil": "2026-02-25T10:30:00Z",
  "projectId": "COMPANYXYZ_Project_ID_67890",
  "relatedContactInformation": [
    {
      "number": "+352123456789",
      "emailAddress": "sales@COMPANYXYZ.net",
      "role": "buyerContactInformation",
      "organization": "COMPANYXYZ",
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
          "id": "CONNECT-ETHERNET-1000"
        },
        "place": [
          {
            "role": "INSTALL_LOCATION",
            "id": "000000000048#000000000000",
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

**Pricing Note**: Prices are retrieved from DynamoDB tables (`X001-BSS-QUOTE-PRICING-DIA` or `X001-BSS-QUOTE-PRICING-INTERNET`) based on:
- Product offering ID (e.g., `CONNECT-ETHERNET-1000`)
- Contract term (12, 24, or 36 months)
- Caller-specific discounts from `X001-MPLIFY-API-KEYS` table

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
curl -X GET 'https://<function-url>/<stage>/billie/quote?state=approved.orderable&limit=20' \
  -H 'x-api-key: YOUR_API_KEY_HERE'

# Filter by projectId
curl -X GET 'https://<function-url>/<stage>/billie/quote?projectId=COMPANYXYZ_Project_ID_67890' \
  -H 'x-api-key: YOUR_API_KEY_HERE'
```

##### Response

```json
[
  {
    "id": "urn:mef:lso:sonata:quote:COMPANYXYZ_Quote_ID_12345",
    "externalId": "COMPANYXYZ_Quote_ID_12345",
    "state": "approved.orderable",
    "quoteDate": "2026-01-26T10:30:00Z",
    "href": "/staging/billie/quote/COMPANYXYZ_Quote_ID_12345"
  },
  {
    "id": "urn:mef:lso:sonata:quote:COMPANYXYZ_Quote_ID_67890",
    "externalId": "COMPANYXYZ_Quote_ID_67890",
    "state": "accepted",
    "quoteDate": "2026-01-25T09:00:00Z",
    "href": "/staging/billie/quote/COMPANYXYZ_Quote_ID_67890"
  }
]
```

#### 3.3 Get Quote by ID

**Endpoint**: `GET /quote/{externalId}`

Retrieves full quote details by external ID.

##### curl Example

```bash
curl -X GET 'https://<function-url>/<stage>/billie/quote/COMPANYXYZ_Quote_ID_12345' \
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
curl -X PATCH 'https://<function-url>/<stage>/billie/quote/COMPANYXYZ_Quote_ID_12345' \
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
  "externalId": "COMPANYXYZ_Order_ID_12345",
  "projectId": "COMPANYXYZ_Project_ID_67890",
  "relatedContactInformation": [
    {
      "role": "productOrderContact",
      "name": "John Smith",
      "organization": "COMPANYXYZ",
      "emailAddress": "sales@COMPANYXYZ.net",
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
      "requestedCompletionDate": "2026-10-22T23:00:00Z",
      "product": {
        "productOffering": {
          "id": "CONNECT-ETHERNET-1000"
        },
        "place": [
          {
            "role": "INSTALL_LOCATION",
            "@type": "GeographicAddressRef",
            "id": "000000000048#000000000000"
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
                "irValue": 1000,
                "irUnits": "MBPS"
              }
            },
            "ingressBandwidthProfileEnvelope": {
              "cir": {
                "irValue": 1000,
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
        "productOfferingQualificationId": "POQ_COMPANYXYZ_2026-01-26_abc123def",
        "id": "DIA"
      },
      "quoteItem": {
        "quoteId": "COMPANYXYZ_Quote_ID_12345",
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
3. Return specific errors if validation fails (see [Error Handling](#error-handling))

##### curl Example

```bash
curl -X POST 'https://<function-url>/<stage>/billie/productOrder' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "externalId": "COMPANYXYZ_Order_ID_12345",
    "projectId": "COMPANYXYZ_Project_ID_67890",
    "productOrderItem": [
      {
        "id": "DIA-1",
        "action": "add",
        "product": {
          "productOffering": {
            "id": "CONNECT-ETHERNET-1000"
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
          "quoteId": "COMPANYXYZ_Quote_ID_12345",
          "id": "DIA"
        }
      }
    ]
  }'
```

##### Response

```json
{
  "id": "COMPANYXYZ_Order_ID_12345",
  "href": "https://api.example.com/billie/productOrder/COMPANYXYZ_Order_ID_12345",
  "externalId": "COMPANYXYZ_Order_ID_12345",
  "projectId": "COMPANYXYZ_Project_ID_67890",
  "state": "acknowledged",
  "orderDate": "2026-01-26T09:12:33Z",
  "requestedCompletionDate": "2026-10-22T23:00:00Z",
  "relatedContactInformation": [
    {
      "role": "productOrderContact",
      "name": "John Smith",
      "organization": "COMPANYXYZ",
      "emailAddress": "sales@COMPANYXYZ.net",
      "number": "+352123456789"
    }
  ],
  "productOrderItem": [
    {
      "id": "DIA-1",
      "action": "add",
      "state": "acknowledged",
      "requestedCompletionDate": "2026-10-22T23:00:00Z",
      "product": {
        "productOffering": {
          "id": "CONNECT-ETHERNET-1000"
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
        "quoteId": "COMPANYXYZ_Quote_ID_12345",
        "id": "DIA"
      }
    }
  ],
  "stateChange": [
    {
      "state": "acknowledged",
      "changeDate": "2026-01-26T09:12:33Z",
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
```

##### Response

```json
[
  {
    "id": "COMPANYXYZ_Order_ID_12345",
    "externalId": "COMPANYXYZ_Order_ID_12345",
    "state": "acknowledged",
    "orderDate": "2026-01-26T09:12:33Z",
    "href": "/staging/billie/productOrder/COMPANYXYZ_Order_ID_12345"
  }
]
```

#### 4.3 Get Order by ID

**Endpoint**: `GET /productOrder/{externalId}`

Retrieves full order details by external ID.

##### curl Example

```bash
curl -X GET 'https://<function-url>/<stage>/billie/productOrder/COMPANYXYZ_Order_ID_12345' \
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

---

## Complete Workflow Example

Here's a complete end-to-end workflow for ordering a CONNECT-ETHERNET service:

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
    "externalId": "COMPANYXYZ_POQ_12345",
    "instantSyncQualification": true,
    "productOfferingQualificationItem": [
      {
        "id": "DIA",
        "action": "add",
        "product": {
          "productOffering": {
            "id": "CONNECT-ETHERNET-1000"
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
            "ipUni": {
              "egressBandwidthProfileEnvelope": {
                "cir": {"irValue": 1000, "irUnits": "MBPS"}
              },
              "ingressBandwidthProfileEnvelope": {
                "cir": {"irValue": 1000, "irUnits": "MBPS"}
              }
            }
          }
        }
      }
    ]
  }'
```

**Save the returned `id`** (e.g., `POQ_COMPANYXYZ_2026-01-26_abc123def`)

### Step 3: Create Quote

```bash
curl -X POST 'https://<function-url>/<stage>/billie/quote' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: YOUR_API_KEY_HERE' \
  -d '{
    "externalId": "COMPANYXYZ_Quote_ID_12345",
    "projectId": "COMPANYXYZ_Project_ID_67890",
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
            "id": "CONNECT-ETHERNET-1000"
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
          "productOfferingQualificationId": "POQ_COMPANYXYZ_2026-01-26_abc123def",
          "id": "DIA"
        }
      }
    ]
  }'
```

**Save the returned `externalId`** (e.g., `COMPANYXYZ_Quote_ID_12345`)

### Step 4: Accept Quote

```bash
curl -X PATCH 'https://<function-url>/<stage>/billie/quote/COMPANYXYZ_Quote_ID_12345' \
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
    "externalId": "COMPANYXYZ_Order_ID_12345",
    "projectId": "COMPANYXYZ_Project_ID_67890",
    "productOrderItem": [
      {
        "id": "DIA-1",
        "action": "add",
        "product": {
          "productOffering": {
            "id": "CONNECT-ETHERNET-1000"
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
          "quoteId": "COMPANYXYZ_Quote_ID_12345",
          "id": "DIA"
        }
      }
    ]
  }'
```

### Step 6: Check Order Status

```bash
curl -X GET 'https://<function-url>/<stage>/billie/productOrder/COMPANYXYZ_Order_ID_12345' \
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

| Code | HTTP Status | Description | Common Causes |
|------|-------------|-------------|---------------|
| `missingProperty` | 400 | Required field missing | Missing `externalId`, `productOfferingQualificationItem`, or other mandatory fields |
| `invalidValue` | 400 | Field contains invalid value | Invalid email format, negative bandwidth values |
| `bandwidthMismatch` | 400 | Bandwidth configuration error | Ingress ≠ egress, or doesn't match product offering ID |
| `accessDenied` | 403 | API key invalid or missing | Missing `x-api-key` header or invalid API key |
| `notFound` | 404 | Resource not found | Quote, order, or address not found |
| `quoteIdRequired` | 400 | Quote ID is missing from order request | Order created without `quoteItem.quoteId` |
| `quoteNotFound` | 404 | Referenced quote does not exist | Quote doesn't exist or doesn't belong to caller |
| `conflict` | 409 | Duplicate externalId | An object with this `externalId` already exists |
| `internalError` | 500 | Internal server error | Database failures, configuration errors |
| `1129` | 500 | Critical fiber network API error | National fiber network provider API failure |
| `databaseFailure` | 500 | Critical database error | DynamoDB unavailable or data corruption |

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
  "reason": "Quote with ID 'COMPANYXYZ_Quote_ID_12345' not found or does not belong to caller",
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

#### Bandwidth Mismatch (400)

```json
{
  "code": "bandwidthMismatch",
  "reason": "Bandwidth mismatch: productOfferingId 'CONNECT-ETHERNET-1000' requires 1000 Mbps, but request specifies ingress=300 Mbps and egress=1000 Mbps",
  "message": "Product configuration bandwidth must match product offering bandwidth",
  "status": "400"
}
```

---

## POQ Errors Explained

Product Offering Qualification (POQ) requests can fail for various reasons. This section explains the most common errors and how to resolve them.

### 1. Missing Address ID

**Error Code**: `missingProperty`  
**HTTP Status**: 400

**Cause**: The `place` array is missing or doesn't contain a `GeographicAddressRef` with an `id`.

**Example Error Response**:
```json
{
  "code": "missingProperty",
  "reason": "Missing address ID in place[0].id",
  "message": "The INSTALL_LOCATION place must contain a valid address ID from address validation",
  "status": "400"
}
```

**Solution**: Ensure you've completed address validation first and use the returned address ID:
```json
{
  "product": {
    "place": [
      {
        "role": "INSTALL_LOCATION",
        "@type": "GeographicAddressRef",
        "id": "000000000048#000000000000"  // ← From address validation response
      }
    ]
  }
}
```

### 2. Bandwidth Mismatch

**Error Code**: `bandwidthMismatch`  
**HTTP Status**: 400

**Cause**: The bandwidth values in `productConfiguration` don't match the bandwidth encoded in the `productOffering.id`, or ingress and egress bandwidths are different.

**Example Error Response**:
```json
{
  "code": "bandwidthMismatch",
  "reason": "Bandwidth mismatch: productOfferingId 'CONNECT-ETHERNET-1000' requires 1000 Mbps, but request specifies ingress=300 Mbps and egress=1000 Mbps",
  "message": "Product configuration bandwidth must match product offering bandwidth",
  "status": "400"
}
```

**Solution**: Ensure bandwidth consistency:
- Ingress bandwidth MUST equal egress bandwidth (symmetric service)
- Both MUST match the value in the product offering ID

**Correct Example**:
```json
{
  "product": {
    "productOffering": {
      "id": "CONNECT-ETHERNET-1000"  // ← 1000 Mbps
    },
    "productConfiguration": {
      "ipUni": {
        "egressBandwidthProfileEnvelope": {
          "cir": {
            "irValue": 1000,  // ← Must match 1000
            "irUnits": "MBPS"
          }
        },
        "ingressBandwidthProfileEnvelope": {
          "cir": {
            "irValue": 1000,  // ← Must match 1000
            "irUnits": "MBPS"
          }
        }
      }
    }
  }
}
```

### 3. Missing Bandwidth Configuration

**Error Code**: `missingProperty`  
**HTTP Status**: 400

**Cause**: Required bandwidth configuration fields are missing from `productConfiguration`.

**Example Error Response**:
```json
{
  "code": "missingProperty",
  "reason": "ingressBandwidthProfileEnvelope.cir.irValue is required",
  "message": "Bandwidth configuration is incomplete",
  "status": "400"
}
```

**Solution**: Always include complete bandwidth configuration:
```json
{
  "productConfiguration": {
    "@type": "DIA",
    "ipvc": {"ipvcEndPoint": {}},
    "ipUni": {
      "egressBandwidthProfileEnvelope": {
        "cir": {
          "irValue": 1000,
          "irUnits": "MBPS"
        }
      },
      "ingressBandwidthProfileEnvelope": {
        "cir": {
          "irValue": 1000,
          "irUnits": "MBPS"
        }
      }
    },
    "ipUniAccessLink": {},
    "ipUniAccessLinkTrunk": {
      "ethernetPhysicalLink": [
        {"physicalLayer": "1000BASE_T"}
      ]
    }
  }
}
```

### 4. Duplicate External ID

**Error Code**: `conflict`  
**HTTP Status**: 409

**Cause**: A POQ with the same `externalId` already exists for this caller.

**Example Error Response**:
```json
{
  "code": "conflict",
  "reason": "Resource conflict: externalId 'COMPANYXYZ_POQ_12345' already exists",
  "message": "A POQ with this externalId already exists",
  "status": "409"
}
```

**Solution**: Use a unique `externalId` for each POQ request. Common patterns:
- Timestamp-based: `COMPANYXYZ_POQ_2026-01-26T10-30-00`
- UUID-based: `COMPANYXYZ_POQ_abc123def456`
- Sequential: `COMPANYXYZ_POQ_00001`, `COMPANYXYZ_POQ_00002`, etc.

### 5. Critical Error 1129 (Fiber Network API Failure)

**Error Code**: `1129`  
**HTTP Status**: 500

**Cause**: The integration with the national fiber network provider's API failed. This is a critical error that requires human intervention.

**Example Error Response**:
```json
{
  "code": "1129",
  "reason": "Critical Error 1129 Requires Human Intervention: Fiber network API HTTP 500: statusCode=500, resultCode=INTERNAL_ERROR, resultDesc=Service temporarily unavailable",
  "message": "Critical Error 1129 Requires Human Intervention",
  "status": "500"
}
```

**Possible Causes**:
1. **Fiber network API is down**: The external provider's API is temporarily unavailable
2. **Missing environment variables**: Required API credentials (`SYSTEM_ID`, `RSP_ID`, `CLIENT_ID`, `CLIENT_SECRET`) are not configured
3. **Invalid address ID**: The address ID doesn't exist in the fiber network provider's system
4. **Timeout**: The fiber network API didn't respond within 15 seconds
5. **Authentication failure**: API credentials are invalid or expired

**Solution Steps**:
1. **Check environment variables**: Verify all required credentials are configured
2. **Verify address ID**: Ensure the address ID came from a successful address validation
3. **Retry after delay**: Wait 1-2 minutes and retry the request
4. **Contact support**: If the error persists, contact technical support with:
   - The full error message
   - The `externalId` from the failed request
   - The address ID being queried
   - The timestamp of the failure

**Diagnostic Information**: Error 1129 messages include detailed information about the failure:
- `HTTP {code}`: HTTP status code from the fiber network API
- `statusCode={value}`: Application-level status code
- `resultCode={value}`: Error classification code
- `resultDesc={value}`: Human-readable error description
- `msg={value}`: Additional error context

### 6. Invalid Product Offering ID

**Error Code**: `invalidValue`  
**HTTP Status**: 400

**Cause**: The `productOffering.id` doesn't match any configured product in the pricing tables.

**Example Error Response**:
```json
{
  "code": "invalidValue",
  "reason": "Invalid product offering ID: 'CONNECT-ETHERNET-999'",
  "message": "Product offering not found in pricing configuration",
  "status": "400"
}
```

**Solution**: Use a valid product offering ID from the available products:
- **CONNECT-ETHERNET (DIA)**: `CONNECT-ETHERNET-100`, `CONNECT-ETHERNET-300`, `CONNECT-ETHERNET-1000`
- **ONLINE-BUSINESS (Internet)**: `ONLINE-BUSINESS-100`, `ONLINE-BUSINESS-300`, `ONLINE-BUSINESS-1000`

To check available products, query the DynamoDB pricing tables or contact your administrator.

### 7. Missing Product Offering

**Error Code**: `missingProperty`  
**HTTP Status**: 400

**Cause**: The `product` object is missing the `productOffering` field.

**Example Error Response**:
```json
{
  "code": "missingProperty",
  "reason": "Missing required field: product.productOffering",
  "message": "The request is missing a required field",
  "status": "400"
}
```

**Solution**: Always include the `productOffering` with an `id`:
```json
{
  "product": {
    "productOffering": {
      "id": "CONNECT-ETHERNET-1000"
    }
  }
}
```

### 8. FO-POP Not Activated (RED Confidence)

**Note**: This is not an error, but a valid POQ response with RED serviceability confidence.

**Response Example**:
```json
{
  "productOfferingQualificationItem": [
    {
      "serviceabilityConfidence": "red",
      "serviceabilityConfidenceReason": "FO-POP 'LUX-POP-99' not activated by Cegecom",
      "subjectToFeasibilityCheck": true,
      "installationInterval": {
        "amount": 90,
        "units": "calendarDays"
      }
    }
  ]
}
```

**Meaning**: The fiber point-of-presence (FO-POP) identified by the fiber network provider is not yet activated for service delivery. This location requires additional network construction or activation.

**What to do**: 
- Contact your network operations team to request FO-POP activation
- Provide the FO-POP identifier from the response
- Expected activation timeframe varies based on location and infrastructure requirements

### POQ Validation Checklist

Before submitting a POQ request, verify:

- [ ] Address has been validated (you have a valid address ID)
- [ ] `externalId` is unique (not used in previous POQ requests)
- [ ] `productOffering.id` is valid (matches configured products)
- [ ] Bandwidth values are consistent:
  - [ ] Ingress equals egress
  - [ ] Both match the value in `productOffering.id`
- [ ] All required fields are present:
  - [ ] `product.place[0].id` (address ID)
  - [ ] `product.productOffering.id`
  - [ ] `product.productConfiguration.ipUni.egressBandwidthProfileEnvelope.cir.irValue`
  - [ ] `product.productConfiguration.ipUni.ingressBandwidthProfileEnvelope.cir.irValue`
- [ ] `@type` field matches product type (`DIA` or `INTERNET`)

---

## Product Configuration

### Available Products

Products are configured in DynamoDB tables, not hardcoded in the application. The actual product IDs available depend on what's configured in your environment.

#### CONNECT-ETHERNET Products (DIA - Dedicated Internet Access)

Stored in: `X001-BSS-QUOTE-PRICING-DIA`

**Product Naming Pattern**: `CONNECT-ETHERNET-{bandwidth_mbps}`

**Available bandwidths** (example configuration):
- `CONNECT-ETHERNET-100` - 100 Mbps symmetric DIA service
- `CONNECT-ETHERNET-300` - 300 Mbps symmetric DIA service
- `CONNECT-ETHERNET-1000` - 1 Gbps (1000 Mbps) symmetric DIA service

**Characteristics**:
- Symmetric bandwidth (upload = download)
- SLA-backed service with guaranteed performance
- Suitable for mission-critical applications
- Premium pricing tier

#### ONLINE-BUSINESS Products (Internet)

Stored in: `X001-BSS-QUOTE-PRICING-INTERNET`

**Product Naming Pattern**: `ONLINE-BUSINESS-{bandwidth_mbps}`

**Available bandwidths** (example configuration):
- `ONLINE-BUSINESS-100` - 100 Mbps business internet
- `ONLINE-BUSINESS-300` - 300 Mbps business internet
- `ONLINE-BUSINESS-1000` - 1 Gbps (1000 Mbps) business internet

**Characteristics**:
- Best-effort delivery with high reliability
- Suitable for general business use
- Cost-effective pricing tier

### Contract Terms

Each product supports multiple contract terms with different pricing:

- **12 months** - Short-term contract, higher monthly rate
- **24 months** - Standard contract, balanced pricing
- **36 months** - Long-term contract, lowest monthly rate

### Pricing Structure

Pricing is stored in DynamoDB and retrieved dynamically based on:

1. **Product Offering ID** (e.g., `CONNECT-ETHERNET-1000`)
2. **Contract Term** (12, 24, or 36 months)
3. **Caller-specific Discounts** (from API key configuration)

#### DynamoDB Pricing Table Schema

Both `X001-BSS-QUOTE-PRICING-DIA` and `X001-BSS-QUOTE-PRICING-INTERNET` use this schema:

```json
{
  "productOfferingId": "CONNECT-ETHERNET-1000",  // Partition Key
  "requestedQuoteItemTerm": "24",                // Sort Key (as string)
  "mrc": 285.00,                                  // Monthly Recurring Charge
  "nrc": 1500.00,                                 // Non-Recurring Charge (installation)
  "currency": "EUR"
}
```

**Key Points**:
- `productOfferingId` (PK): The product identifier
- `requestedQuoteItemTerm` (SK): Contract term as a string: `"12"`, `"24"`, or `"36"`
- `mrc`: Monthly service fee (before discounts)
- `nrc`: One-time installation fee (before discounts)
- `currency`: Currency code (typically `"EUR"`)

#### Discount Application

Discounts are configured per API key in the `X001-MPLIFY-API-KEYS` table:

```json
{
  "apiKeyId": "abc123xyz789",
  "callerId": "COMPANYXYZ",
  "mrcDiscount": 0.10,  // 10% discount on monthly fees
  "nrcDiscount": 0.15   // 15% discount on installation fees
}
```

**Final pricing calculation**:
```
Final MRC = Base MRC × (1 - mrcDiscount)
Final NRC = Base NRC × (1 - nrcDiscount)
```

**Example**:
- Base MRC: €285.00
- MRC Discount: 10% (0.10)
- Final MRC: €285.00 × (1 - 0.10) = €256.50

### Adding New Products

To add a new product offering:

1. **Add entries to DynamoDB**: Create entries for all three contract terms (12, 24, 36 months)
2. **Use consistent naming**: Follow the pattern `{PRODUCT_TYPE}-{BANDWIDTH_MBPS}`
3. **Configure pricing**: Set appropriate MRC and NRC values
4. **Test availability**: The product becomes immediately available via the API

**Example**: Adding `CONNECT-ETHERNET-500` (500 Mbps DIA):

```bash
# Add 12-month term
aws dynamodb put-item \
  --table-name X001-BSS-QUOTE-PRICING-DIA \
  --item '{
    "productOfferingId": {"S": "CONNECT-ETHERNET-500"},
    "requestedQuoteItemTerm": {"S": "12"},
    "mrc": {"N": "220"},
    "nrc": {"N": "1200"},
    "currency": {"S": "EUR"}
  }'

# Add 24-month term
aws dynamodb put-item \
  --table-name X001-BSS-QUOTE-PRICING-DIA \
  --item '{
    "productOfferingId": {"S": "CONNECT-ETHERNET-500"},
    "requestedQuoteItemTerm": {"S": "24"},
    "mrc": {"N": "200"},
    "nrc": {"N": "1200"},
    "currency": {"S": "EUR"}
  }'

# Add 36-month term
aws dynamodb put-item \
  --table-name X001-BSS-QUOTE-PRICING-DIA \
  --item '{
    "productOfferingId": {"S": "CONNECT-ETHERNET-500"},
    "requestedQuoteItemTerm": {"S": "36"},
    "mrc": {"N": "180"},
    "nrc": {"N": "1200"},
    "currency": {"S": "EUR"}
  }'
```

### Listing Configured Products

To see all available products in your environment:

```bash
# List all CONNECT-ETHERNET products
aws dynamodb scan \
  --table-name X001-BSS-QUOTE-PRICING-DIA \
  --projection-expression "productOfferingId" \
  --query "Items[*].productOfferingId.S" \
  --output text | sort -u

# List all ONLINE-BUSINESS products
aws dynamodb scan \
  --table-name X001-BSS-QUOTE-PRICING-INTERNET \
  --projection-expression "productOfferingId" \
  --query "Items[*].productOfferingId.S" \
  --output text | sort -u
```

---

## Data Persistence

### DynamoDB Tables

#### Address Validation
- **X001-MPLIFY-AV-REQUEST** - Address validation requests
- **X001-MPLIFY-AV-ANSWER** - Address validation responses

#### Product Qualification
- **X001-MPLIFY-POQ-REQUEST** - POQ requests with fiber network API payload
- **X001-MPLIFY-POQ-ANSWER** - POQ responses with serviceability results

#### Quote Management
- **X001-MPLIFY-QUOTE-REQUEST** - Quote creation and update requests
- **X001-MPLIFY-QUOTE-ANSWER** - Complete quote records

#### Product Order Management
- **X001-MPLIFY-ORDER-REQUEST** - Order requests with quoteId validation
- **X001-MPLIFY-ORDER-ANSWER** - Complete order records

#### Pricing & Configuration
- **X001-BSS-QUOTE-PRICING-DIA** - CONNECT-ETHERNET product pricing
  - Partition Key: `productOfferingId` (e.g., `CONNECT-ETHERNET-1000`)
  - Sort Key: `requestedQuoteItemTerm` (e.g., `"12"`, `"24"`, `"36"`)
  - Attributes: `mrc`, `nrc`, `currency`

- **X001-BSS-QUOTE-PRICING-INTERNET** - ONLINE-BUSINESS product pricing
  - Schema identical to DIA pricing table

- **X001-MPLIFY-API-KEYS** - API key configuration with caller discounts
  - Partition Key: `apiKeyId`
  - Attributes: `callerId`, `mrcDiscount`, `nrcDiscount`

#### FO-POP Activation Tracking
- **X001-OSS-FO-POP-CEGECOM** - Activated fiber points-of-presence
  - Partition Key: `popId` (e.g., `LUX-POP-01`)
  - Attributes: `location`, `updatedAt`

### Table Schemas

#### Event Logging Tables (Request/Answer)

All event logging tables follow this common structure:

**Common Fields**:
- `PK` (Partition Key): Unique identifier for the event
- `SK` (Sort Key): Timestamp of the event
- `callerId`: API key owner identifier
- `externalId`: Client-provided request identifier
- `createdAt`: ISO 8601 timestamp
- `ttl`: Time-to-live for automatic record expiration

**POQ-Specific Fields** (X001-MPLIFY-POQ-REQUEST):
- `productOfferingId`: Product being qualified
- `addressId`: Validated address identifier
- `foPop`: Fiber point-of-presence identifier from fiber network API
- `ntpName`: Network termination point name
- `freeFibers`: Number of available fibers
- `totalFibers`: Total fiber count on route
- `FiberNetworkPayload`: Complete fiber network API response (JSON blob)

**Quote-Specific Fields** (X001-MPLIFY-QUOTE-ANSWER):
- `state`: Quote state (e.g., `approved.orderable`, `accepted`)
- `quoteLevel`: Quote level (e.g., `firm`, `firmSubjectToFeasibilityCheck`)
- `projectId`: Project identifier for filtering
- `quoteData`: Complete quote object (JSON blob)

**Order-Specific Fields** (X001-MPLIFY-ORDER-ANSWER):
- `state`: Order state (e.g., `acknowledged`, `inProgress`)
- `projectId`: Project identifier for filtering
- `quoteId`: Referenced quote identifier
- `orderData`: Complete order object (JSON blob)

### Data Retention

Event logging tables have TTL (Time-To-Live) configured for automatic data expiration:

- **Address Validation**: 90 days retention
- **Product Qualification**: 180 days retention (includes fiber network API payloads)
- **Quote Management**: 2 years retention
- **Product Order Management**: 5 years retention (regulatory compliance)

### Querying Data

#### Query POQ by External ID

```python
import boto3

dynamodb = boto3.client('dynamodb')

response = dynamodb.query(
    TableName='X001-MPLIFY-POQ-ANSWER',
    IndexName='externalId-index',
    KeyConditionExpression='externalId = :extId',
    ExpressionAttributeValues={
        ':extId': {'S': 'COMPANYXYZ_POQ_12345'}
    }
)
```

#### Query Quotes by Caller and State

```python
response = dynamodb.query(
    TableName='X001-MPLIFY-QUOTE-ANSWER',
    IndexName='callerId-state-index',
    KeyConditionExpression='callerId = :caller AND #state = :state',
    ExpressionAttributeNames={
        '#state': 'state'  # 'state' is a reserved word
    },
    ExpressionAttributeValues={
        ':caller': {'S': 'COMPANYXYZ'},
        ':state': {'S': 'approved.orderable'}
    }
)
```

#### Query Orders by Project ID

```python
response = dynamodb.query(
    TableName='X001-MPLIFY-ORDER-ANSWER',
    IndexName='projectId-index',
    KeyConditionExpression='projectId = :project',
    ExpressionAttributeValues={
        ':project': {'S': 'COMPANYXYZ_Project_ID_67890'}
    }
)
```

### Backup and Recovery

DynamoDB tables are configured with:

- **Point-in-time recovery (PITR)**: Enabled for all production tables
- **Backup retention**: 35 days
- **On-demand backups**: Manual backups before major changes
- **Cross-region replication**: Available for disaster recovery (optional)

---

## MEF Compliance

This implementation strictly follows:

- **MEF LSO Sonata Developer Guide** - API patterns and conventions
- **MEF 79** - Address, Service Site, and Product Offering Qualification Management
- **MEF 80** - Quote Management
- **MEF 57.2** - Product Order Management

### Key Compliance Features

✅ **Proper HTTP Status Codes**: 200, 400, 403, 404, 409, 500  
✅ **Complete Field Coverage**: All MEF-required fields with explicit null values  
✅ **MEF Error Response Format**: Standardized error responses  
✅ **Correlation IDs**: Request tracing with `externalId`  
✅ **CORS Support**: Browser client compatibility  
✅ **State Management**: MEF-compliant state transitions for quotes and orders  
✅ **Timestamps**: ISO 8601 format with timezone information  
✅ **URN Format**: MEF-compliant resource identifiers  

### Deviations from MEF

None. This implementation fully complies with MEF LSO Sonata (Billie) specifications.

---

## Environment Variables

### Required for Fiber Network Integration

- `FIBER_API_URL` - National fiber network provider API URL
- `SYSTEM_ID` - System identifier for fiber network API
- `RSP_ID` - Retail Service Provider identifier
- `CLIENT_ID` - API client ID for OAuth authentication
- `CLIENT_SECRET` - API client secret for OAuth authentication

### Optional Configuration

- `CONFIG_S3_BUCKET` - S3 bucket for configuration overrides
- `CONFIG_S3_KEY` - Policy configuration key in S3
- `MAPPING_S3_KEY` - MEF mapping configuration key in S3
- `FO_POP_TABLE` - Override FO-POP table name (default: `X001-OSS-FO-POP-CEGECOM`)

### AWS Environment

- `AWS_REGION` - AWS region (default: `eu-central-1`)
- `AWS_LAMBDA_FUNCTION_NAME` - Lambda function name (automatically set)
- `AWS_LAMBDA_FUNCTION_VERSION` - Lambda version (automatically set)

---

## Deployment

### Prerequisites

- AWS CLI configured with appropriate credentials
- Python 3.13+ for local development
- Access to deployment role with Lambda, S3, and DynamoDB permissions

### Deploy to AWS

```powershell
# Windows
.\scripts\deploy-lambda.ps1

# Linux/Mac
./scripts/deploy-lambda.sh
```

The deployment scripts:
1. Package application code with dependencies
2. Create deployment artifact with proper structure
3. Update Lambda function code
4. Verify deployment success

### Lambda Configuration

The Lambda function requires:

- **Runtime**: Python 3.13
- **Memory**: 512 MB (adjustable based on load)
- **Timeout**: 30 seconds (POQ may take 10-15 seconds due to fiber network API)
- **VPC Integration**: For secure external API access
- **IAM Role**: With policies for S3, DynamoDB, and VPC
- **Environment Variables**: See [Environment Variables](#environment-variables)

### Post-Deployment Verification

```bash
# Test Lambda invocation
aws lambda invoke \
  --function-name <function-name> \
  --payload '{"path": "/health", "httpMethod": "GET"}' \
  response.json

# Check logs
aws logs tail /aws/lambda/<function-name> --follow
```

---

## Support & Contact

For issues, questions, or feature requests:

1. **Architecture Guidance**: Review `src/` directory READMEs
2. **API Usage Patterns**: Check `examples/` directory
3. **Compliance Details**: Consult this documentation
4. **Technical Support**: Contact your system administrator

---

**Document Version**: 4.0  
**API Version**: 2.0  
**Last Updated**: January 26, 2026  
**Maintained By**: Engineering Team  
**Status**: Production-ready  

---

*This documentation reflects the actual code implementation. All examples have been tested and verified against the running API.*

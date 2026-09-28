# MyAccess – Catalog-Agnostic Data Access Request Service

## 1. Objective

The objective is to provide a **standardized and reusable access-request process** for BigQuery tables and views through **MyAccess**, using the underlying SailPoint APIs.

The solution should be **catalog-agnostic**, allowing users to request access to data from any supported data catalog, such as:

* Atlan
* Google Knowledge Catalog
* Future data catalogs

The catalog should not need to understand the internal SailPoint implementation. It should only need to provide the required information about the user, data resource, requested access, and justification.

---

# 2. High-Level Architecture

```text
                    DATA CATALOGS
        ┌──────────────┬──────────────┐
        │              │              │
      Atlan      Knowledge Catalog   Future
        │              │            Catalogs
        └──────────────┼──────────────┘
                       │
                       │ Access Request
                       ▼
             ┌─────────────────────┐
             │   MyAccess Layer    │
             │                     │
             │ Catalog Integration │
             │ Request Validation  │
             │ Resource Resolution │
             │ Entitlement Mapping │
             │ Request Processing  │
             └──────────┬──────────┘
                        │
                        │ SailPoint API
                        ▼
             ┌─────────────────────┐
             │      SailPoint      │
             │   Access Platform   │
             └──────────┬──────────┘
                        │
                        ▼
                Access Workflow
                        │
             ┌──────────┴──────────┐
             │                     │
          Approved              Rejected
             │                     │
             ▼                     ▼
        Provisioning          Rejection Reason
```

**MyAccess is the service through which access requests are handled. SailPoint APIs are used by MyAccess to perform the underlying identity/access-management operations.**

---

# 3. Core Use Case

The initial scope is:

> Allow users to request access to **BigQuery tables and views** from any integrated data catalog through MyAccess.

Example:

```text
User
 ↓
Opens BigQuery table in Atlan
 ↓
Clicks "Request Access"
 ↓
Atlan sends request to MyAccess
 ↓
MyAccess identifies the BigQuery resource
 ↓
MyAccess resolves the corresponding access/entitlement
 ↓
MyAccess calls SailPoint API
 ↓
SailPoint/MyAccess processes the request
 ↓
Approval / Rejection
 ↓
Access is provisioned or rejection is returned
```

Future scope may include **column-level access**.

---

# 4. Catalog-Agnostic Design

The system should not be tightly coupled to a particular catalog.

For example, Atlan may provide an asset in one format while another catalog provides the same BigQuery resource in a different format.

Each catalog should therefore have an integration/adapter that converts its request into a **common access-request model**.

### Common request model

Conceptually:

```json
{
  "requester": {
    "email": "user@company.com"
  },
  "resource": {
    "type": "BIGQUERY_TABLE",
    "project": "analytics-prod",
    "dataset": "customer",
    "name": "customers"
  },
  "access": {
    "level": "READ"
  },
  "justification": "Required for analytics",
  "source": {
    "catalog": "atlan"
  }
}
```

The actual API contract will be finalized based on the MyAccess and SailPoint requirements.

---

# 5. Supported Resources

## Initial Scope

### BigQuery Table

```text
Project
   ↓
Dataset
   ↓
Table
```

Example:

```text
analytics-prod.customer.customers
```

### BigQuery View

```text
Project
   ↓
Dataset
   ↓
View
```

Example:

```text
analytics-prod.customer.customer_summary
```

---

# 6. Future Column-Level Access

The design should allow the resource model to be extended to support column-level access without redesigning the entire service.

For example:

```json
{
  "resource": {
    "type": "BIGQUERY_TABLE",
    "project": "analytics-prod",
    "dataset": "customer",
    "name": "customers"
  },
  "scope": {
    "type": "COLUMN",
    "columns": [
      "customer_id",
      "customer_name"
    ]
  },
  "access": {
    "level": "READ"
  }
}
```

The initial implementation does not need to implement column-level access, but the API and internal architecture should avoid preventing it later.

---

# 7. Main Components

## 7.1 Catalog Integration / Adapter

Responsible for translating a catalog-specific request into the common MyAccess request format.

Example:

```text
Atlan
  ↓
Atlan Adapter
  ↓
Common MyAccess Request
```

Similarly:

```text
Knowledge Catalog
  ↓
Knowledge Catalog Adapter
  ↓
Common MyAccess Request
```

This prevents catalog-specific logic from spreading throughout the application.

---

## 7.2 Request Validation

MyAccess validates:

* Requester information
* BigQuery project
* Dataset
* Table/View
* Requested access level
* Required justification
* Required catalog information
* Other mandatory fields

Invalid requests should be rejected before making unnecessary calls to SailPoint.

---

## 7.3 Identity Resolution

The requester from the catalog must be mapped to the corresponding identity recognized by MyAccess/SailPoint.

Conceptually:

```text
Catalog User
     ↓
Email / User Identifier
     ↓
MyAccess
     ↓
SailPoint Identity
```

The exact identity-resolution mechanism will depend on the organization's SailPoint/MyAccess implementation.

---

# 8. Entitlement Resolution

This is a key part of the service.

A catalog provides the **data resource**, but SailPoint generally needs the appropriate access object/entitlement configured in the organization's access-management system.

The service therefore needs to determine:

> Which MyAccess/SailPoint entitlement corresponds to this BigQuery resource and requested access level?

Conceptually:

```text
BigQuery Resource
       ↓
Entitlement Resolver
       ↓
MyAccess/SailPoint Access Object
```

Example:

```text
analytics-prod.customer.customers
                +
               READ
                ↓
       BQ_CUSTOMER_READ
                ↓
        SailPoint entitlement
```

The actual mapping mechanism needs to be confirmed with the MyAccess/SailPoint administrators.

Possible implementations include:

* Existing MyAccess/SailPoint mappings
* Configuration
* A mapping database
* An entitlement lookup API
* Another existing enterprise mapping mechanism

The implementation should use the organization's existing mechanism wherever possible rather than introducing a duplicate source of truth.

---

# 9. Existing Access Check

Before creating a new access request, the service should determine whether the user already has the required access, where supported by the APIs.

Conceptually:

```text
User + Resource + Access Level
              ↓
       Existing Access?
         /          \
       YES           NO
        │             │
        ▼             ▼
Return existing    Create request
access/status
```

This can help prevent unnecessary duplicate requests.

---

# 10. Access Request Creation

After validation and entitlement resolution:

```text
Requester
    +
BigQuery Resource
    +
Access Level
    +
Entitlement
    +
Justification
        ↓
MyAccess
        ↓
SailPoint API
        ↓
Access Request
```

MyAccess should encapsulate the SailPoint-specific API details so that catalog integrations do not need to know how SailPoint works internally.

---

# 11. Authentication

MyAccess will communicate with the SailPoint APIs using **OAuth 2.0**.

The SailPoint administrator will provide:

* `clientId`
* `clientSecret`

The exact token endpoint, grant type, and scope will be obtained from the organization's configuration/documentation.

Conceptually:

```text
MyAccess
   │
   │ clientId + clientSecret
   ▼
OAuth 2.0 Token Endpoint
   │
   │ access_token
   ▼
MyAccess
   │
   │ Authorization: Bearer <access_token>
   ▼
SailPoint API
```

The client secret must not be stored in source code.

For the GCP deployment, credentials should be managed using **GCP Secret Manager** and appropriate GKE workload identity/access mechanisms.

---

# 12. Request Lifecycle

The complete lifecycle should be considered during API testing.

### Successful flow

```text
REQUESTED
    ↓
PENDING_APPROVAL
    ↓
APPROVED
    ↓
PROVISIONED
```

### Rejected flow

```text
REQUESTED
    ↓
PENDING_APPROVAL
    ↓
REJECTED
    ↓
Rejection Reason
```

The actual status values will depend on the MyAccess/SailPoint APIs.

---

# 13. Error Handling

The service should distinguish between different classes of failures.

### Authentication failure

```text
401
Invalid/expired OAuth token
```

### Authorization failure

```text
403
Client does not have required permission
```

### Invalid request

```text
400
Invalid resource/request parameters
```

### Resource/entitlement not found

```text
Requested BigQuery resource or corresponding
access object cannot be resolved
```

### Business rejection

The request was successfully created but the access workflow subsequently rejected it.

### Duplicate/existing request

The user already has the access or an equivalent request is already pending.

The service should normalize these responses where appropriate so that catalog integrations don't have to understand SailPoint-specific error formats.

---

# 14. API Design

The external MyAccess interface should be designed around **data access**, not around SailPoint.

For example:

```http
POST /v1/access-requests
```

The catalog should communicate:

```text
WHO
WHAT RESOURCE
WHAT ACCESS
WHY
SOURCE CATALOG
```

The catalog should not need to provide SailPoint-specific implementation details.

A status endpoint can also be exposed:

```http
GET /v1/access-requests/{requestId}
```

Conceptually:

```json
{
  "requestId": "123456",
  "status": "PENDING_APPROVAL"
}
```

The exact API contract will be finalized after the SailPoint/MyAccess API behavior is fully tested.

---

# 15. Technology Stack

The organization's existing technology stack can be reused.

### Application

* Python
* FastAPI
* Pydantic
* Python HTTP client for external APIs

### Deployment

* Docker
* Google Kubernetes Engine (GKE)

### GCP

* Secret Manager
* Cloud Logging
* Cloud Monitoring
* Load Balancer / GKE Ingress
* Workload Identity

### Persistence

A database such as PostgreSQL/Cloud SQL should only be introduced if the service needs to maintain:

* Resource-to-entitlement mappings
* Request state
* Configuration
* Audit information

The source of truth should first be confirmed with the MyAccess/SailPoint team.

---

# 16. Proposed Internal Structure

A possible Python project structure:

```text
app/
│
├── api/
│   └── access_requests.py
│
├── services/
│   ├── access_request_service.py
│   ├── entitlement_resolver.py
│   └── identity_resolver.py
│
├── integrations/
│   ├── sailpoint/
│   │   ├── client.py
│   │   └── models.py
│   │
│   └── catalogs/
│       ├── base.py
│       ├── atlan.py
│       └── knowledge_catalog.py
│
├── models/
│   └── access_request.py
│
├── config/
│   └── settings.py
│
└── main.py
```

The exact structure can be simplified depending on the size of the first implementation.

---

# 17. API Testing Strategy

Before implementing the complete service, the SailPoint/MyAccess API should be tested independently using Postman.

### Authentication

* Obtain OAuth token
* Test expired/invalid token
* Test insufficient scope/permission

### Valid requests

* Valid BigQuery table
* Valid BigQuery view
* Different supported access levels

### Validation

* Invalid user
* Invalid resource
* Invalid entitlement
* Missing required fields
* Invalid access level
* Malformed payload

### Business scenarios

* User already has access
* Existing pending request
* User not eligible
* Request rejected
* Request approved
* Request successfully provisioned

### For every test, capture

```text
Request
HTTP Status
Response
Error Code
Error Message
Request ID
Resulting Status
```

This API behavior becomes the contract that the MyAccess integration needs to handle.

---

# 18. Target Architecture

The intended long-term architecture is:

```text
                         ┌───────────────┐
                         │     Atlan     │
                         └───────┬───────┘
                                 │
                         ┌───────▼───────┐
                         │ Atlan Adapter │
                         └───────┬───────┘
                                 │
                                 │
┌─────────────────┐              │
│ Knowledge       │              │
│ Catalog         │              │
└────────┬────────┘              │
         │                       │
┌────────▼─────────┐             │
│ Knowledge        │             │
│ Catalog Adapter  │             │
└────────┬─────────┘             │
         │                       │
         └───────────┬───────────┘
                     ▼
          ┌─────────────────────┐
          │      MyAccess       │
          │                     │
          │ Request Validation  │
          │ Identity Resolution │
          │ Resource Resolution │
          │ Entitlement         │
          │ Resolution          │
          │ Access Check        │
          │ Request Creation    │
          │ Status Handling     │
          │ Error Normalization │
          └──────────┬──────────┘
                     │
                     ▼
          ┌─────────────────────┐
          │  SailPoint Client   │
          │   OAuth 2.0         │
          └──────────┬──────────┘
                     │
                     ▼
              SailPoint APIs
                     │
                     ▼
              Access Workflow
```

The fundamental design principle is:

> **Catalogs should integrate with MyAccess, while MyAccess handles the SailPoint-specific access-management logic.**

This allows additional catalogs to be integrated without rebuilding the SailPoint integration each time.

---

# 19. Open Questions

Before implementation, the following should be confirmed with the MyAccess/SailPoint team:

1. What SailPoint object represents access to a BigQuery table/view?
2. How is a BigQuery resource mapped to that object?
3. Does an entitlement lookup API already exist?
4. Does MyAccess already expose any of this functionality?
5. How is the requester mapped to a SailPoint identity?
6. How can existing access be checked?
7. How can existing/pending requests be checked?
8. How are approvals and rejections represented?
9. How is request status retrieved?
10. Is provisioning synchronous or asynchronous?
11. Are callbacks/webhooks available for status changes?
12. What OAuth 2.0 grant type and scopes are required?
13. What are the exact DEV endpoints?
14. What is the expected behavior for duplicate requests?
15. How should column-level access be represented when it becomes available?

These answers will determine the final API contract and the amount of logic that needs to exist inside the MyAccess integration.

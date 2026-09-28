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

# Catalog-Agnostic Data Access Request Middleware

## 1. Overview

The objective is to build a reusable **Middleware** that enables users to request access to BigQuery tables and views directly from any supported data catalog.

The Middleware acts as the integration layer between data catalogs such as **Atlan**, **Google Knowledge Catalog**, and future catalogs, and the organization's **SailPoint API**.

The Middleware will be catalog-agnostic. Catalog-specific implementations should only be responsible for translating their native request format into the common Middleware request format.

### Primary goal

Provide a standardized process:

**Data Catalog → Middleware → SailPoint API → Access Request / Approval / Provisioning**

The initial scope is:

* BigQuery tables
* BigQuery views
* Table/view-level access
* SailPoint-based access requests
* OAuth 2.0 authentication with SailPoint

Future scope may include:

* Column-level access
* Additional data catalogs
* Additional access levels
* Additional data platforms

---

# 2. High-Level Architecture

```text
                         DATA CATALOGS

              ┌──────────────┐
              │    Atlan     │
              └──────┬───────┘
                     │
              ┌──────▼───────┐
              │   Google     │
              │   Knowledge  │
              │   Catalog    │
              └──────┬───────┘
                     │
              ┌──────▼───────┐
              │    Future    │
              │   Catalogs   │
              └──────┬───────┘
                     │
                     │ HTTPS / REST
                     ▼
            ┌──────────────────────┐
            │      MIDDLEWARE      │
            │                      │
            │ Request Validation   │
            │ User Resolution      │
            │ Asset Resolution     │
            │ Entitlement Mapping  │
            │ Access Validation    │
            │ SailPoint Request    │
            │ Status Handling      │
            │ Error Handling       │
            └──────────┬───────────┘
                       │
                       │ OAuth 2.0
                       ▼
            ┌──────────────────────┐
            │     SailPoint API    │
            │                      │
            │ Access Request       │
            │ Approval Workflow    │
            │ Provisioning         │
            └──────────────────────┘
```

---

# 3. Core Concept

The Middleware should not be tightly coupled to any particular catalog.

For example, Atlan may represent a BigQuery table differently from Google Knowledge Catalog.

The Middleware should convert both into a common internal representation.

### Example

A catalog sends:

```json
{
  "assetId": "12345",
  "assetType": "table",
  "qualifiedName": "analytics-prod.customer.customers"
}
```

The Middleware converts this into a canonical resource:

```json
{
  "resourceType": "BIGQUERY_TABLE",
  "project": "analytics-prod",
  "dataset": "customer",
  "name": "customers"
}
```

The same principle applies to BigQuery views.

---

# 4. End-to-End Process

## Step 1 — User selects an asset

A user discovers a BigQuery table or view through a data catalog.

Example:

```text
Catalog
  ↓
analytics-prod.customer.customers
```

The user selects:

**Request Access**

---

## Step 2 — Catalog calls Middleware

The catalog sends an access-request request to the Middleware.

Conceptually:

```http
POST /v1/access-requests
```

The request contains:

* Requester
* Catalog
* Resource
* Resource type
* Requested access level
* Justification
* Catalog-specific reference, if required

Example:

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

The exact external API contract can be finalized during implementation.

---

# 5. Middleware Processing

After receiving the request, the Middleware performs several steps.

## 5.1 Request validation

Validate:

* Required fields
* User identity
* Resource type
* Project
* Dataset
* Table/view name
* Access level
* Justification
* Source catalog

Invalid requests should be rejected before calling SailPoint.

---

## 5.2 User / Identity Resolution

The Middleware determines the corresponding SailPoint identity for the requester.

Conceptually:

```text
Catalog User
     ↓
Email / Employee ID
     ↓
SailPoint Identity
     ↓
SailPoint Identity ID
```

The exact identity-resolution mechanism depends on the organization's SailPoint configuration.

---

# 6. Resource Resolution

The Middleware determines the actual BigQuery resource being requested.

### Table

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

### View

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

The Middleware should maintain a canonical representation independent of the catalog.

---

# 7. Entitlement Resolution

This is one of the most important responsibilities of the Middleware.

The Middleware must determine which SailPoint access object corresponds to the requested BigQuery resource and access level.

Conceptually:

```text
BigQuery Resource
        ↓
Access Level
        ↓
Entitlement Resolver
        ↓
SailPoint Entitlement / Access Profile
```

Example:

```text
analytics-prod.customer.customers
             +
            READ
             ↓
     BQ_CUSTOMER_READ
             ↓
      SailPoint Object
```

The exact SailPoint object used for the mapping must be confirmed with the SailPoint/IAM team.

The mapping should not be hard-coded into catalog-specific code.

---

# 8. Existing Access Validation

Before creating a new request, the Middleware should determine whether the user already has the requested access.

Conceptually:

```text
Does user already have entitlement?
          │
     ┌────┴────┐
     │         │
    YES        NO
     │         │
     ▼         ▼
 Return       Continue
 existing     request
 access
```

The actual implementation depends on the SailPoint APIs available in the organization's environment.

---

# 9. Duplicate Request Handling

The Middleware should also handle cases where the user already has an active/pending request for the same entitlement.

Example:

```text
User
 ↓
Request BQ_CUSTOMER_READ
 ↓
Existing pending request found
```

The Middleware should avoid unnecessarily creating another identical request.

The exact duplicate behavior should be finalized based on SailPoint's capabilities and organizational requirements.

---

# 10. SailPoint Authentication

The Middleware will authenticate with SailPoint using **OAuth 2.0**.

The SailPoint administrator will provide:

* Client ID
* Client Secret
* OAuth token endpoint
* Required scope, if applicable
* Required permissions

Conceptually:

```text
Middleware
     │
     │ client_id
     │ client_secret
     ▼
SailPoint OAuth Endpoint
     │
     ▼
Access Token
     │
     ▼
Authorization: Bearer <token>
```

Client credentials must not be stored in source code.

They should be securely managed using the organization's approved secret-management mechanism.

For the GCP environment, **GCP Secret Manager** is the preferred option.

---

# 11. SailPoint API Request

Once the Middleware has resolved:

* SailPoint identity
* BigQuery resource
* Access level
* SailPoint entitlement/access object
* Justification

it constructs the SailPoint API request according to the organization's SailPoint API documentation.

Conceptually:

```text
Canonical Access Request
          ↓
SailPoint-specific request
          ↓
SailPoint Access Request API
```

The Middleware should isolate SailPoint-specific payload construction inside the SailPoint integration layer.

---

# 12. Request Lifecycle

Creating an access request and completing the access request are separate stages.

A typical lifecycle may look like:

```text
REQUESTED
    ↓
PENDING_APPROVAL
    ↓
APPROVED
    ↓
PROVISIONED
```

or:

```text
REQUESTED
    ↓
PENDING_APPROVAL
    ↓
REJECTED
```

The actual statuses depend on the SailPoint implementation.

The Middleware should expose a catalog-neutral representation of the status.

---

# 13. Status Retrieval

The Middleware should provide a way for the originating catalog to determine the current state of a request.

Conceptually:

```http
GET /v1/access-requests/{requestId}
```

Example response:

```json
{
  "requestId": "123456",
  "status": "PENDING_APPROVAL"
}
```

Possible normalized states could include:

```text
PENDING
APPROVED
REJECTED
PROVISIONED
FAILED
```

The exact mapping should be finalized after testing the SailPoint API.

---

# 14. Error Handling

The Middleware should hide SailPoint-specific errors from catalog integrations wherever possible.

For example:

```text
SailPoint
    ↓
HTTP 400
SailPoint-specific error
    ↓
Middleware
    ↓
Normalized error
    ↓
Catalog
```

Example:

```json
{
  "status": "REJECTED",
  "code": "ACCESS_NOT_ELIGIBLE",
  "message": "The requested access is not available for this user."
}
```

The Middleware should distinguish between:

### Authentication failures

```text
401 Unauthorized
```

### Authorization failures

```text
403 Forbidden
```

### Invalid requests

```text
400 Bad Request
```

### Resource/entitlement issues

```text
Resource does not exist
Entitlement does not exist
User is not eligible
```

### Duplicate requests

```text
Existing request/access found
```

### SailPoint/system failures

```text
5xx / timeout / unavailable
```

The exact status and error mapping should be based on actual SailPoint API behavior discovered during testing.

---

# 15. Technology Stack

The organization primarily uses GCP, Python and GKE. The proposed stack therefore follows the existing ecosystem.

### Application

```text
Python
FastAPI
```

FastAPI provides:

* REST API implementation
* Request validation
* OpenAPI documentation
* Type-safe request/response models
* Good support for asynchronous operations

### Deployment

```text
Docker
   ↓
GKE
```

The Middleware will run as a containerized service on GKE.

### Secrets

```text
GCP Secret Manager
```

Used for:

* SailPoint client secret
* Other sensitive integration credentials

Client IDs may also be configuration-managed depending on organizational standards.

### Logging

```text
Google Cloud Logging
```

### Monitoring

```text
Google Cloud Monitoring
```

### Optional persistence

If entitlement mappings, request state, or configuration need to be persisted outside SailPoint:

```text
Cloud SQL
PostgreSQL
```

This should only be introduced if required by the final architecture.

---

# 16. Proposed Internal Structure

A possible Python project structure:

```text
middleware/
│
├── main.py
│
├── api/
│   └── access_requests.py
│
├── services/
│   ├── access_request_service.py
│   ├── entitlement_resolver.py
│   ├── identity_resolver.py
│   └── resource_resolver.py
│
├── integrations/
│   │
│   ├── sailpoint/
│   │   ├── client.py
│   │   ├── auth.py
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
└── config/
    └── settings.py
```

The exact structure can change based on the team's existing Python standards.

---

# 17. Catalog Integration Model

The Middleware should expose a **common API contract**.

The catalogs should not need to understand SailPoint's API.

```text
                  Middleware API
                       ▲
                       │
       ┌───────────────┼────────────────┐
       │               │                │
     Atlan       Knowledge Catalog   Future Catalog
       │               │                │
       └───────────────┴────────────────┘
```

Each catalog only needs to translate its asset/request information into the common Middleware contract.

---

# 18. Future Column-Level Access

The initial implementation focuses on table/view-level access.

However, the request model should be designed so that column-level access can be introduced without redesigning the entire system.

### Current

```json
{
  "resource": {
    "type": "BIGQUERY_TABLE",
    "project": "analytics-prod",
    "dataset": "customer",
    "name": "customers"
  },
  "access": {
    "level": "READ"
  }
}
```

### Future

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

This keeps the core architecture extensible.

---

# 19. Initial Testing Strategy

Before implementing the complete Middleware, the SailPoint API should be tested independently using Postman.

### Authentication

* Obtain OAuth 2.0 token
* Verify token
* Test expired/invalid token
* Test insufficient permissions

### Access request

* Valid table request
* Valid view request
* Invalid user
* Invalid entitlement
* Missing required fields
* Invalid access level
* User already has access
* Duplicate/pending request
* User not eligible
* Request rejected
* SailPoint API failure

For every scenario record:

```text
Request
HTTP status
Response
Error code
Error message
Request ID
Final status
```

These results will define the behavior that the Middleware needs to implement.

---

# 20. V1 Scope

### Included

* BigQuery tables
* BigQuery views
* Catalog-agnostic request API
* OAuth 2.0 authentication
* SailPoint API integration
* User/identity resolution
* Resource resolution
* Entitlement resolution
* Access request creation
* Request status
* Error handling
* Atlan integration capability
* Google Knowledge Catalog integration capability
* GKE deployment

### Future

* Column-level access
* Additional catalogs
* Additional data platforms
* Advanced policy/eligibility evaluation
* Event-driven status updates
* Additional access types

---

# 21. Key Open Questions

The following should be clarified with the SailPoint/IAM team before finalizing the implementation:

1. What SailPoint object represents BigQuery table/view access?

   * Entitlement?
   * Access Profile?
   * Role?
   * Other?

2. How is a BigQuery resource mapped to that SailPoint object?

3. Can the SailPoint API resolve an entitlement from a resource, or must the Middleware maintain the mapping?

4. How do we resolve a catalog user to a SailPoint identity?

5. How do we check whether the user already has the entitlement?

6. How do we check for an existing pending request?

7. What are the exact SailPoint request lifecycle states?

8. How can the Middleware retrieve the final request status?

9. What happens when an approval is rejected?

10. Is there a SailPoint webhook/event mechanism available for request-status changes?

11. What OAuth 2.0 scopes and permissions are required?

12. Are there API rate limits that the Middleware needs to handle?

---

# 22. Target Architecture

The intended final architecture is:

```text
                    ┌─────────────────┐
                    │      Atlan      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Knowledge       │
                    │ Catalog         │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Future Catalogs │
                    └────────┬────────┘
                             │
                             ▼
                 ┌────────────────────────┐
                 │       MIDDLEWARE       │
                 │                        │
                 │ FastAPI / Python       │
                 │                        │
                 │ ┌────────────────────┐ │
                 │ │ Request Validation │ │
                 │ ├────────────────────┤ │
                 │ │ Identity Resolver  │ │
                 │ ├────────────────────┤ │
                 │ │ Resource Resolver  │ │
                 │ ├────────────────────┤ │
                 │ │ Entitlement        │ │
                 │ │ Resolver           │ │
                 │ ├────────────────────┤ │
                 │ │ Access Validation  │ │
                 │ ├────────────────────┤ │
                 │ │ Request Management │ │
                 │ ├────────────────────┤ │
                 │ │ Status Management  │ │
                 │ └────────────────────┘ │
                 │            │           │
                 │      SailPoint Client  │
                 └────────────┬───────────┘
                              │
                       OAuth 2.0
                              │
                              ▼
                   ┌───────────────────┐
                   │   SailPoint API   │
                   └───────────────────┘
```

The fundamental design principle is:

**Catalogs should know how to request access. The Middleware should know how to translate that request into an access decision/request for SailPoint. SailPoint remains the access-management backend.**

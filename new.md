# Number Reservation - Low Level Design (LLD)
# Sales App Number Reservation & Reserved Number Activation

## Document Information
| Item | Value |
|------|-------|
| Version | 1.0 |
| Date | 2025-02-08 |
| Author | Architecture Team |
| Status | **Draft - Pending Review** |
| Feature | Number Reservation (FRS 2.11) |
| Target Platform | Java Microservices (New Sales App Service) |

> **Note**: This document is based on analysis of:
> - Java `ordermgmtservice` codebase (activation flow, entities, strategy pattern)
> - Java `digital-external-omni-auth` codebase (User, Role, ChannelType, permissions)
> - Java `digitalconfigmanagement` codebase (configuration patterns)
> - Java `digitallookup` codebase (shared enums, DTOs)
> - Legacy .NET `RedBullSalesPortalRestSharp` codebase (existing reservation logic, BSS integration)
> - Legacy .NET `ConcurrentSalesApp` codebase (ReservationCode, UnpickNumberService, BssApiHelper)

---

## 1. Executive Summary

### 1.1 Overview
Number Reservation enables the Sales Portal team and authorized partners to **reserve MSISDNs** (phone numbers) for specific entities (organization, user, customer, channel, or package type). Reserved numbers can then only be activated by the designated entity during the activation flow. This feature introduces a new **`salesapp-service`** Java microservice dedicated to Sales team operations, following the same architecture as the existing `.NET SalesPortal` but built on the Java microservices stack.

### 1.2 Key Characteristics
- **Target Users**: Sales team and authorized partner users (NOT end customers)
- **Reservation Types**: Organization, Username, Customer ID, Channel, Package Type (Prepaid/Postpaid)
- **Configurable**: Who can reserve and which reservation options are available is configuration-driven
- **Activation Integration**: Reserved numbers flow through the existing `ordermgmtservice` activation with an additional reservation validation step
- **New Service**: `salesapp-service` — a new Java microservice dedicated to Sales App operations

### 1.3 Reservation Types Summary

| Reservation Type | Scope | Who Can Activate |
|------------------|-------|-------------------|
| **Organization** | Numbers belong to an organization | Any user under this organization |
| **Username** | Numbers assigned to a specific user | Only this specific user |
| **Customer ID** | Numbers locked to a customer | Only for this customer's activation |
| **Channel** | Numbers visible in a sales channel | Any user in this channel |
| **Package Type** | Numbers locked to Prepaid or Postpaid | Anyone, but only with matching package type |

### 1.4 Legacy .NET Code References (Existing Logic)

| Component | File Path | Relevance |
|-----------|-----------|-----------|
| ReservationCode field | `SalesOrder.cs:137` | `ReservationCode` field on SalesOrder entity |
| Reservation Report | `CompletedOrdersWithReservationCode/` | Completed orders with reservation tracking |
| BSS OperateMSISDN | `BssApiHelper.cs:3044` | Pick (1029) / Unpick (1030) operations |
| BSS GetSimStatus | `BssApiHelper.cs:2644` | Status code 33 = "Reserved", 34 = "Pick" |
| UnpickNumberService | `UnpickNumberService.cs` | Release reserved/picked numbers |
| MSISDN Pool | `SalesDbContext.cs:94` | MSISDNPool DbSet |
| SalesAPIController | `SalesAPIController.cs:1300-1301` | ReservationCode set on order creation |

### 1.5 Existing Java Code References

| Component | File Path | Relevance |
|-----------|-----------|-----------|
| CustomerProductOrder | `ordermgmtservice/.../entity/CustomerProductOrder.java` | Main order entity (needs reservation fields) |
| PickedMsisdnLog | `ordermgmtservice/.../entity/PickedMsisdnLog.java` | MSISDN pick audit trail |
| ClientMsisdn | `ordermgmtservice/.../entity/ClientMsisdn.java` | MSISDN-to-customer mapping |
| NumberTiers enum | `digitallookup/.../enums/NumberTiers.java` | Number tier classification |
| PaymentType enum | `digitallookup/.../enums/PaymentType.java` | PREPAID / POSTPAID / HYBRID |
| SimType enum | `digitallookup/.../enums/SimType.java` | Regular / eSIM |
| User entity | `digital-external-omni-auth/.../entity/User.java` | User with role, channel, partyRoleId |
| ChannelType entity | `digital-external-omni-auth/.../entity/ChannelType.java` | Sales channel types |
| Configurations entity | `digitalconfigmanagement/.../entity/Configurations.java` | Key-value config with JSON support |
| ActivationProcessingChain | `ordermgmtservice/.../chain/ActivationProcessingChain.java` | Chain of handlers for activation |
| BssMediationController | `bssmediationcontroller/` | BSS SOAP integration layer |

---

## 2. Architecture Overview

### 2.1 Microservices Involved

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                   NUMBER RESERVATION FEATURE ARCHITECTURE                        │
│                        Microservices Involvement                                 │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│                    ┌─────────────────────────┐                                   │
│                    │   Sales App Frontend    │                                   │
│                    │   (Angular / Web)       │                                   │
│                    └───────────┬─────────────┘                                   │
│                                │                                                 │
│              ┌─────────────────┼──────────────────┐                              │
│              │                 │                   │                              │
│              ▼                 ▼                   ▼                              │
│  ┌───────────────────┐  ┌──────────────────┐  ┌──────────────────────┐           │
│  │  salesapp-service  │  │ ordermgmtservice │  │ digital-external-   │           │
│  │  *** NEW ***       │  │   Port: 8073     │  │ omni-auth           │           │
│  │  Port: 8078        │  │                  │  │                     │           │
│  ├───────────────────┤  ├──────────────────┤  ├──────────────────────┤           │
│  │                    │  │                  │  │                     │           │
│  │  Tables (PgSQL):   │  │ Tables (PgSQL):  │  │ Tables (PgSQL):     │           │
│  │  ├── number_       │  │ ├── customer_    │  │ ├── "user"          │           │
│  │  │   reservation   │  │ │   product_     │  │ ├── role            │           │
│  │  ├── reservation_  │  │ │   order        │  │ ├── channel_type    │           │
│  │  │   config        │  │ ├── picked_      │  │ ├── business_role   │           │
│  │  ├── reservation_  │  │ │   msisdn_log   │  │ ├── configurations  │           │
│  │  │   audit_log     │  │ └── client_      │  │ └── attribute       │           │
│  │  └── reservation_  │  │     msisdn       │  │     (pages/screens) │           │
│  │      code          │  │                  │  │                     │           │
│  │                    │  │ Modified:         │  │                     │           │
│  │  Components:       │  │ ├── Reservation- │  │ Read-only access:   │           │
│  │  ├── Reservation   │  │ │   Validation   │  │ ├── User lookup     │           │
│  │  │   Controller    │  │ │   Handler      │  │ ├── Role check      │           │
│  │  ├── Reservation   │  │ └── (new handler │  │ └── Channel check   │           │
│  │  │   Service       │  │     in chain)    │  │                     │           │
│  │  ├── Reservation   │  │                  │  │                     │           │
│  │  │   Validator     │  │                  │  │                     │           │
│  │  └── BSS Client    │  │                  │  │                     │           │
│  │                    │  │                  │  │                     │           │
│  └────────┬───────────┘  └────────┬─────────┘  └──────────┬─────────┘           │
│           │                       │                        │                     │
│           ▼                       ▼                        │                     │
│  ┌────────────────────────────────────────────┐            │                     │
│  │         bssmediationcontroller             │            │                     │
│  │         (Huawei BSS SOAP)                  │◄───────────┘                     │
│  │         Port: 8071                         │                                  │
│  ├────────────────────────────────────────────┤                                  │
│  │  OperateMSISDN (Pick: 1029, Unpick: 1030) │                                  │
│  │  GetResourceDetailInfo (SIM Status)        │                                  │
│  └────────────────────────────────────────────┘                                  │
│                                                                                  │
│  ┌────────────────────────────────────────────┐                                  │
│  │      digitalconfigmanagement               │                                  │
│  │      Port: 8074                            │                                  │
│  ├────────────────────────────────────────────┤                                  │
│  │  Reservation feature configs               │                                  │
│  │  (who can reserve, enabled options)         │                                  │
│  └────────────────────────────────────────────┘                                  │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Service Communication

| From | To | Protocol | Purpose |
|------|----|----------|---------|
| `salesapp-service` | `bssmediationcontroller` | REST/HTTP | Pick/Unpick MSISDN on BSS |
| `salesapp-service` | `digital-external-omni-auth` | REST/HTTP | Validate user, role, channel permissions |
| `salesapp-service` | `digitalconfigmanagement` | REST/HTTP | Read reservation feature configs |
| `ordermgmtservice` | `salesapp-service` | REST/HTTP | Validate reservation during activation |
| Frontend | `salesapp-service` | REST/HTTP | Reserve/Release/Search numbers |
| Frontend | `ordermgmtservice` | REST/HTTP | Activate reserved number (existing flow + reservation step) |

### 2.3 Table Distribution

| Table | Service / Database | Purpose |
|-------|--------------------|---------|
| `number_reservation` | salesapp-service DB | Core reservation records |
| `reservation_config` | salesapp-service DB | Reservation feature configuration per user/channel/role |
| `reservation_audit_log` | salesapp-service DB | Audit trail for all reservation actions |
| `reservation_code` | salesapp-service DB | Unique reservation codes generation & tracking |
| `customer_product_order` | ordermgmtservice DB | Existing orders table (add `reservation_code` column) |
| `configurations` | digitalconfigmanagement DB | Global feature toggles |

---

## 3. Database Schema Design

### 3.1 New Tables (salesapp-service)

#### 3.1.1 `number_reservation`

```sql
CREATE TABLE number_reservation (
    id                  BIGSERIAL PRIMARY KEY,
    msisdn              VARCHAR(20) NOT NULL,
    reservation_code    VARCHAR(50) NOT NULL UNIQUE,
    reservation_type    VARCHAR(30) NOT NULL,  -- ORGANIZATION, USERNAME, CUSTOMER_ID, CHANNEL, PACKAGE_TYPE

    -- Reservation target (depends on type)
    organization_id     BIGINT,                -- When type = ORGANIZATION
    username            VARCHAR(100),           -- When type = USERNAME
    customer_id         VARCHAR(50),            -- When type = CUSTOMER_ID
    channel_id          BIGINT,                 -- When type = CHANNEL
    package_type        VARCHAR(20),            -- When type = PACKAGE_TYPE (PREPAID / POSTPAID)

    -- Metadata
    number_tier         INT DEFAULT 0,          -- Maps to NumberTiers enum
    sim_type            VARCHAR(20),            -- Regular / eSIM (nullable = any)

    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',  -- ACTIVE, USED, RELEASED, EXPIRED

    -- BSS State
    bss_status          VARCHAR(20),            -- Pick status from BSS (Reserved=33, Pick=34)

    -- Audit
    reserved_by         VARCHAR(100) NOT NULL,  -- Username who reserved
    reserved_at         TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMP,              -- Nullable: null = never expires
    used_at             TIMESTAMP,              -- When used in activation
    used_by             VARCHAR(100),           -- Who activated it
    used_order_id       BIGINT,                 -- FK to order that consumed it
    released_at         TIMESTAMP,              -- When released/unpicked
    released_by         VARCHAR(100),           -- Who released it
    release_reason      VARCHAR(500),           -- Why it was released

    -- Standard audit
    created_at          TIMESTAMP NOT NULL DEFAULT NOW(),
    last_modified_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by          VARCHAR(100),
    last_modified_by    VARCHAR(100),
    archived            BOOLEAN DEFAULT FALSE,

    -- Indexes
    CONSTRAINT uq_msisdn_active UNIQUE (msisdn, status) -- Only one active reservation per MSISDN
);

CREATE INDEX idx_reservation_msisdn ON number_reservation(msisdn);
CREATE INDEX idx_reservation_code ON number_reservation(reservation_code);
CREATE INDEX idx_reservation_type ON number_reservation(reservation_type);
CREATE INDEX idx_reservation_customer ON number_reservation(customer_id);
CREATE INDEX idx_reservation_username ON number_reservation(username);
CREATE INDEX idx_reservation_org ON number_reservation(organization_id);
CREATE INDEX idx_reservation_channel ON number_reservation(channel_id);
CREATE INDEX idx_reservation_status ON number_reservation(status);
CREATE INDEX idx_reservation_expires ON number_reservation(expires_at) WHERE status = 'ACTIVE';
```

#### 3.1.2 `reservation_config`

```sql
CREATE TABLE reservation_config (
    id                      BIGSERIAL PRIMARY KEY,
    config_scope            VARCHAR(30) NOT NULL,   -- GLOBAL, CHANNEL, USER, PARTNER, ROLE
    scope_value             VARCHAR(100),            -- Channel ID, Username, Partner ID, Role name

    -- Which reservation types are allowed
    allow_by_organization   BOOLEAN DEFAULT FALSE,
    allow_by_username       BOOLEAN DEFAULT FALSE,
    allow_by_customer_id    BOOLEAN DEFAULT FALSE,
    allow_by_channel        BOOLEAN DEFAULT FALSE,
    allow_by_package_type   BOOLEAN DEFAULT FALSE,

    -- Limits
    max_reservations        INT DEFAULT 100,         -- Max active reservations per user
    reservation_expiry_hours INT DEFAULT 720,         -- Default: 30 days

    -- OTP Configuration (for activation of reserved numbers)
    otp_enabled             BOOLEAN DEFAULT TRUE,

    -- Feature toggle
    enabled                 BOOLEAN DEFAULT TRUE,

    -- Audit
    created_at              TIMESTAMP NOT NULL DEFAULT NOW(),
    last_modified_at        TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by              VARCHAR(100),
    last_modified_by        VARCHAR(100),

    CONSTRAINT uq_config_scope UNIQUE (config_scope, scope_value)
);
```

#### 3.1.3 `reservation_audit_log`

```sql
CREATE TABLE reservation_audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    reservation_id      BIGINT REFERENCES number_reservation(id),
    msisdn              VARCHAR(20) NOT NULL,
    reservation_code    VARCHAR(50),
    action              VARCHAR(30) NOT NULL,   -- RESERVE, RELEASE, ACTIVATE, EXPIRE, TRANSFER, VALIDATE
    action_result       VARCHAR(20) NOT NULL,   -- SUCCESS, FAILED
    performed_by        VARCHAR(100) NOT NULL,
    performed_at        TIMESTAMP NOT NULL DEFAULT NOW(),
    details             TEXT,                    -- JSON details of the action
    error_message       VARCHAR(1000),
    ip_address          VARCHAR(50),
    user_agent          VARCHAR(500)
);

CREATE INDEX idx_audit_reservation ON reservation_audit_log(reservation_id);
CREATE INDEX idx_audit_msisdn ON reservation_audit_log(msisdn);
CREATE INDEX idx_audit_performed_by ON reservation_audit_log(performed_by);
CREATE INDEX idx_audit_action ON reservation_audit_log(action);
```

#### 3.1.4 `reservation_code`

```sql
CREATE TABLE reservation_code (
    id              BIGSERIAL PRIMARY KEY,
    code            VARCHAR(50) NOT NULL UNIQUE,
    reservation_id  BIGINT REFERENCES number_reservation(id),
    generated_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    is_used         BOOLEAN DEFAULT FALSE,
    used_at         TIMESTAMP
);

CREATE INDEX idx_rcode_code ON reservation_code(code);
```

### 3.2 Modified Tables (ordermgmtservice)

#### 3.2.1 `customer_product_order` — Add Columns

```sql
ALTER TABLE customer_product_order ADD COLUMN reservation_code VARCHAR(50);
ALTER TABLE customer_product_order ADD COLUMN reservation_id BIGINT;
```

---

## 4. Entity Design (Java)

### 4.1 New Entities (salesapp-service)

#### 4.1.1 NumberReservation Entity

```java
package com.segmatek.salesappservice.entity;

import com.segmatek.salesappservice.entity.enums.ReservationStatus;
import com.segmatek.salesappservice.entity.enums.ReservationType;
import com.segmatek.DigitalLookup.enums.NumberTiers;
import com.segmatek.DigitalLookup.enums.PaymentType;
import com.segmatek.DigitalLookup.enums.SimType;
import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(name = "number_reservation",
       uniqueConstraints = @UniqueConstraint(columnNames = {"msisdn", "status"}))
public class NumberReservation extends BaseModel<Long> {

    @Column(nullable = false, length = 20)
    private String msisdn;

    @Column(name = "reservation_code", nullable = false, unique = true, length = 50)
    private String reservationCode;

    @Enumerated(EnumType.STRING)
    @Column(name = "reservation_type", nullable = false, length = 30)
    private ReservationType reservationType;

    // -- Reservation target fields (polymorphic based on type) --
    @Column(name = "organization_id")
    private Long organizationId;

    @Column(name = "username", length = 100)
    private String username;

    @Column(name = "customer_id", length = 50)
    private String customerId;

    @Column(name = "channel_id")
    private Long channelId;

    @Column(name = "package_type", length = 20)
    private String packageType;  // PREPAID / POSTPAID

    // -- Number metadata --
    @Enumerated(EnumType.ORDINAL)
    @Column(name = "number_tier")
    private NumberTiers numberTier;

    @Enumerated(EnumType.STRING)
    @Column(name = "sim_type", length = 20)
    private SimType simType;

    // -- Status --
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private ReservationStatus status;

    @Column(name = "bss_status", length = 20)
    private String bssStatus;

    // -- Reservation lifecycle --
    @Column(name = "reserved_by", nullable = false, length = 100)
    private String reservedBy;

    @Column(name = "reserved_at", nullable = false)
    private LocalDateTime reservedAt;

    @Column(name = "expires_at")
    private LocalDateTime expiresAt;

    // -- Usage tracking --
    @Column(name = "used_at")
    private LocalDateTime usedAt;

    @Column(name = "used_by", length = 100)
    private String usedBy;

    @Column(name = "used_order_id")
    private Long usedOrderId;

    // -- Release tracking --
    @Column(name = "released_at")
    private LocalDateTime releasedAt;

    @Column(name = "released_by", length = 100)
    private String releasedBy;

    @Column(name = "release_reason", length = 500)
    private String releaseReason;

    // -- Business Methods --

    public boolean isActive() {
        return status == ReservationStatus.ACTIVE && !isExpired();
    }

    public boolean isExpired() {
        return expiresAt != null && LocalDateTime.now().isAfter(expiresAt);
    }

    public void markUsed(String activatingUser, Long orderId) {
        this.status = ReservationStatus.USED;
        this.usedAt = LocalDateTime.now();
        this.usedBy = activatingUser;
        this.usedOrderId = orderId;
    }

    public void release(String releasingUser, String reason) {
        this.status = ReservationStatus.RELEASED;
        this.releasedAt = LocalDateTime.now();
        this.releasedBy = releasingUser;
        this.releaseReason = reason;
    }

    public void expire() {
        this.status = ReservationStatus.EXPIRED;
    }

    @PrePersist
    void onCreate() {
        if (reservedAt == null) reservedAt = LocalDateTime.now();
        if (status == null) status = ReservationStatus.ACTIVE;
    }
}
```

#### 4.1.2 ReservationConfig Entity

```java
package com.segmatek.salesappservice.entity;

import com.segmatek.salesappservice.entity.enums.ConfigScope;
import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(name = "reservation_config",
       uniqueConstraints = @UniqueConstraint(columnNames = {"config_scope", "scope_value"}))
public class ReservationConfig extends BaseModel<Long> {

    @Enumerated(EnumType.STRING)
    @Column(name = "config_scope", nullable = false, length = 30)
    private ConfigScope configScope;  // GLOBAL, CHANNEL, USER, PARTNER, ROLE

    @Column(name = "scope_value", length = 100)
    private String scopeValue;

    @Column(name = "allow_by_organization")
    private Boolean allowByOrganization = false;

    @Column(name = "allow_by_username")
    private Boolean allowByUsername = false;

    @Column(name = "allow_by_customer_id")
    private Boolean allowByCustomerId = false;

    @Column(name = "allow_by_channel")
    private Boolean allowByChannel = false;

    @Column(name = "allow_by_package_type")
    private Boolean allowByPackageType = false;

    @Column(name = "max_reservations")
    private Integer maxReservations = 100;

    @Column(name = "reservation_expiry_hours")
    private Integer reservationExpiryHours = 720;  // 30 days

    @Column(name = "otp_enabled")
    private Boolean otpEnabled = true;

    @Column
    private Boolean enabled = true;
}
```

#### 4.1.3 ReservationAuditLog Entity

```java
package com.segmatek.salesappservice.entity;

import com.segmatek.salesappservice.entity.enums.AuditAction;
import com.segmatek.salesappservice.entity.enums.AuditResult;
import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(name = "reservation_audit_log")
public class ReservationAuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "reservation_id")
    private Long reservationId;

    @Column(nullable = false, length = 20)
    private String msisdn;

    @Column(name = "reservation_code", length = 50)
    private String reservationCode;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30)
    private AuditAction action;

    @Enumerated(EnumType.STRING)
    @Column(name = "action_result", nullable = false, length = 20)
    private AuditResult actionResult;

    @Column(name = "performed_by", nullable = false, length = 100)
    private String performedBy;

    @Column(name = "performed_at", nullable = false)
    private LocalDateTime performedAt;

    @Column(columnDefinition = "TEXT")
    private String details;

    @Column(name = "error_message", length = 1000)
    private String errorMessage;

    @Column(name = "ip_address", length = 50)
    private String ipAddress;

    @Column(name = "user_agent", length = 500)
    private String userAgent;

    @PrePersist
    void onCreate() {
        if (performedAt == null) performedAt = LocalDateTime.now();
    }
}
```

### 4.2 New Enums (digitallookup — shared)

```java
// ReservationType.java
package com.segmatek.DigitalLookup.enums;

public enum ReservationType {
    ORGANIZATION,
    USERNAME,
    CUSTOMER_ID,
    CHANNEL,
    PACKAGE_TYPE
}

// ReservationStatus.java
package com.segmatek.DigitalLookup.enums;

public enum ReservationStatus {
    ACTIVE,
    USED,
    RELEASED,
    EXPIRED
}

// ConfigScope.java
package com.segmatek.DigitalLookup.enums;

public enum ConfigScope {
    GLOBAL,
    CHANNEL,
    USER,
    PARTNER,
    ROLE
}

// AuditAction.java
package com.segmatek.DigitalLookup.enums;

public enum AuditAction {
    RESERVE,
    RELEASE,
    ACTIVATE,
    EXPIRE,
    TRANSFER,
    VALIDATE
}

// AuditResult.java
package com.segmatek.DigitalLookup.enums;

public enum AuditResult {
    SUCCESS,
    FAILED
}
```

### 4.3 Modified Entity (ordermgmtservice)

```java
// Add to CustomerProductOrder.java
@Column(name = "reservation_code", length = 50)
private String reservationCode;

@Column(name = "reservation_id")
private Long reservationId;
```

---

## 5. API Design

### 5.1 salesapp-service APIs

**Base Path:** `/api/v1/reservations`

#### 5.1.1 Reserve a Number

```
POST /api/v1/reservations
```

**Request Body:**
```json
{
  "msisdn": "5XXXXXXXX",
  "reservationType": "CUSTOMER_ID",
  "organizationId": null,
  "username": null,
  "customerId": "1234567890",
  "channelId": null,
  "packageType": null,
  "simType": "Regular",
  "expiresInHours": 720,
  "notes": "VIP customer reservation"
}
```

**Response (201 Created):**
```json
{
  "id": 1,
  "msisdn": "5XXXXXXXX",
  "reservationCode": "RES-20250208-ABC123",
  "reservationType": "CUSTOMER_ID",
  "customerId": "1234567890",
  "status": "ACTIVE",
  "reservedBy": "seller_username",
  "reservedAt": "2025-02-08T10:30:00",
  "expiresAt": "2025-03-10T10:30:00",
  "numberTier": "GOLD",
  "simType": "Regular"
}
```

**Error Responses:**
- `400` — MSISDN invalid or already reserved
- `403` — User not authorized for this reservation type
- `404` — MSISDN not found in BSS inventory
- `409` — MSISDN already has an active reservation
- `422` — BSS operation failed (cannot pick number)

#### 5.1.2 Bulk Reserve Numbers

```
POST /api/v1/reservations/bulk
```

**Request Body:**
```json
{
  "msisdns": ["5XXXXXXX1", "5XXXXXXX2", "5XXXXXXX3"],
  "reservationType": "CHANNEL",
  "channelId": 5,
  "expiresInHours": 720
}
```

**Response (200 OK):**
```json
{
  "successful": [
    { "msisdn": "5XXXXXXX1", "reservationCode": "RES-20250208-ABC123", "status": "ACTIVE" },
    { "msisdn": "5XXXXXXX2", "reservationCode": "RES-20250208-DEF456", "status": "ACTIVE" }
  ],
  "failed": [
    { "msisdn": "5XXXXXXX3", "error": "MSISDN already reserved", "errorCode": "ALREADY_RESERVED" }
  ],
  "summary": { "total": 3, "reserved": 2, "failed": 1 }
}
```

#### 5.1.3 Release a Reservation

```
DELETE /api/v1/reservations/{reservationId}
```

**Request Body:**
```json
{
  "reason": "Customer cancelled request"
}
```

**Response (200 OK):**
```json
{
  "id": 1,
  "msisdn": "5XXXXXXXX",
  "reservationCode": "RES-20250208-ABC123",
  "status": "RELEASED",
  "releasedBy": "seller_username",
  "releasedAt": "2025-02-08T15:00:00",
  "releaseReason": "Customer cancelled request"
}
```

#### 5.1.4 Get Reservation by Code

```
GET /api/v1/reservations/code/{reservationCode}
```

**Response (200 OK):**
```json
{
  "id": 1,
  "msisdn": "5XXXXXXXX",
  "reservationCode": "RES-20250208-ABC123",
  "reservationType": "CUSTOMER_ID",
  "customerId": "1234567890",
  "status": "ACTIVE",
  "reservedBy": "seller_username",
  "reservedAt": "2025-02-08T10:30:00",
  "expiresAt": "2025-03-10T10:30:00"
}
```

#### 5.1.5 Search/List Reservations

```
GET /api/v1/reservations?status=ACTIVE&reservationType=CUSTOMER_ID&customerId=1234567890&page=0&size=20
```

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `status` | String | Filter by status (ACTIVE, USED, RELEASED, EXPIRED) |
| `reservationType` | String | Filter by type |
| `msisdn` | String | Filter by MSISDN |
| `customerId` | String | Filter by customer ID |
| `username` | String | Filter by assigned username |
| `channelId` | Long | Filter by channel |
| `organizationId` | Long | Filter by organization |
| `reservedBy` | String | Filter by who reserved |
| `page` | Int | Page number (0-based) |
| `size` | Int | Page size (default 20) |

**Response (200 OK):**
```json
{
  "content": [
    {
      "id": 1,
      "msisdn": "5XXXXXXXX",
      "reservationCode": "RES-20250208-ABC123",
      "reservationType": "CUSTOMER_ID",
      "customerId": "1234567890",
      "status": "ACTIVE",
      "reservedBy": "seller_username",
      "reservedAt": "2025-02-08T10:30:00",
      "expiresAt": "2025-03-10T10:30:00"
    }
  ],
  "totalElements": 1,
  "totalPages": 1,
  "number": 0,
  "size": 20
}
```

#### 5.1.6 Validate Reservation (Called by ordermgmtservice)

```
POST /api/v1/reservations/validate
```

**Request Body (internal service-to-service):**
```json
{
  "reservationCode": "RES-20250208-ABC123",
  "msisdn": "5XXXXXXXX",
  "customerId": "1234567890",
  "username": "seller_username",
  "channelId": 5,
  "packageType": "PREPAID"
}
```

**Response (200 OK):**
```json
{
  "valid": true,
  "reservationId": 1,
  "reservationCode": "RES-20250208-ABC123",
  "reservationType": "CUSTOMER_ID",
  "msisdn": "5XXXXXXXX"
}
```

**Error Responses:**
- `400` — Reservation code doesn't match MSISDN
- `403` — Activation not allowed for this user/customer/channel
- `404` — Reservation code not found
- `410` — Reservation expired

#### 5.1.7 Mark Reservation as Used (Called by ordermgmtservice after activation)

```
PUT /api/v1/reservations/{reservationId}/use
```

**Request Body:**
```json
{
  "orderId": 456,
  "activatedBy": "seller_username"
}
```

#### 5.1.8 Get Reservation Configuration

```
GET /api/v1/reservations/config?username=seller1&channelId=5
```

**Response:**
```json
{
  "allowByOrganization": true,
  "allowByUsername": true,
  "allowByCustomerId": true,
  "allowByChannel": false,
  "allowByPackageType": true,
  "maxReservations": 100,
  "reservationExpiryHours": 720,
  "otpEnabled": true
}
```

#### 5.1.9 Update Reservation Configuration (Admin)

```
PUT /api/v1/reservations/config
```

**Request Body:**
```json
{
  "configScope": "CHANNEL",
  "scopeValue": "5",
  "allowByOrganization": true,
  "allowByUsername": true,
  "allowByCustomerId": true,
  "allowByChannel": true,
  "allowByPackageType": true,
  "maxReservations": 200,
  "reservationExpiryHours": 1440,
  "otpEnabled": false
}
```

---

## 6. Service Layer Design

### 6.1 salesapp-service Components

```
salesapp-service/
├── src/main/java/com/segmatek/salesappservice/
│   ├── SalesAppServiceApplication.java
│   ├── controller/
│   │   ├── NumberReservationController.java
│   │   └── ReservationConfigController.java
│   ├── service/
│   │   ├── NumberReservationService.java
│   │   ├── ReservationValidationService.java
│   │   ├── ReservationCodeGenerator.java
│   │   ├── ReservationExpiryScheduler.java
│   │   └── ReservationConfigService.java
│   ├── client/
│   │   ├── BssMediationClient.java          (REST client to bssmediationcontroller)
│   │   ├── AuthServiceClient.java           (REST client to omni-auth)
│   │   └── ConfigManagementClient.java      (REST client to config service)
│   ├── entity/
│   │   ├── NumberReservation.java
│   │   ├── ReservationConfig.java
│   │   ├── ReservationAuditLog.java
│   │   ├── ReservationCode.java
│   │   ├── BaseModel.java
│   │   └── enums/ (if not placed in digitallookup)
│   ├── repository/
│   │   ├── NumberReservationRepository.java
│   │   ├── ReservationConfigRepository.java
│   │   ├── ReservationAuditLogRepository.java
│   │   └── ReservationCodeRepository.java
│   ├── dto/
│   │   ├── request/
│   │   │   ├── ReserveNumberRequest.java
│   │   │   ├── BulkReserveRequest.java
│   │   │   ├── ReleaseReservationRequest.java
│   │   │   ├── ValidateReservationRequest.java
│   │   │   └── UseReservationRequest.java
│   │   └── response/
│   │       ├── ReservationResponse.java
│   │       ├── BulkReservationResponse.java
│   │       ├── ValidationResult.java
│   │       └── ReservationConfigResponse.java
│   ├── mapper/
│   │   └── ReservationMapper.java           (MapStruct)
│   ├── exception/
│   │   ├── ReservationNotFoundException.java
│   │   ├── ReservationExpiredException.java
│   │   ├── MsisdnAlreadyReservedException.java
│   │   ├── UnauthorizedReservationException.java
│   │   └── BssOperationFailedException.java
│   └── config/
│       └── AppConfig.java
├── src/main/resources/
│   ├── application.properties
│   └── application-local.properties
├── pom.xml
└── Dockerfile
```

### 6.2 NumberReservationService — Core Logic

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class NumberReservationService {

    private final NumberReservationRepository reservationRepository;
    private final ReservationAuditLogRepository auditLogRepository;
    private final ReservationConfigService configService;
    private final ReservationValidationService validationService;
    private final ReservationCodeGenerator codeGenerator;
    private final BssMediationClient bssClient;
    private final AuthServiceClient authClient;
    private final ReservationMapper mapper;

    /**
     * Reserve a number with the given parameters.
     *
     * Flow:
     * 1. Validate user has permission for this reservation type
     * 2. Validate MSISDN is available in BSS (status must be "Normal" or "Pick")
     * 3. Check no active reservation exists for this MSISDN
     * 4. Check user has not exceeded max reservations
     * 5. Pick the MSISDN in BSS (OperateMSISDN with code 1029)
     * 6. Generate unique reservation code
     * 7. Save reservation record
     * 8. Log audit entry
     * 9. Return reservation details
     */
    @Transactional
    public ReservationResponse reserveNumber(ReserveNumberRequest request, String username) {
        // Step 1: Permission check
        // Step 2: BSS status check
        // Step 3: Duplicate check
        // Step 4: Limit check
        // Step 5: BSS Pick
        // Step 6: Generate code
        // Step 7: Save
        // Step 8: Audit
        // Step 9: Return
    }

    /**
     * Release a reservation — unpick from BSS and mark as released.
     *
     * Flow:
     * 1. Find reservation by ID
     * 2. Validate reservation is ACTIVE
     * 3. Validate user has permission to release
     * 4. Unpick from BSS (OperateMSISDN with code 1030)
     * 5. Update reservation status to RELEASED
     * 6. Log audit entry
     */
    @Transactional
    public ReservationResponse releaseReservation(Long reservationId, ReleaseReservationRequest request, String username) {
        // ...
    }

    /**
     * Validate reservation during activation flow (called by ordermgmtservice).
     *
     * Validation Rules per Reservation Type:
     * - ORGANIZATION: Check user belongs to this organization
     * - USERNAME: Check activating user matches reserved username
     * - CUSTOMER_ID: Check customer ID matches
     * - CHANNEL: Check user belongs to this channel
     * - PACKAGE_TYPE: Check selected package type matches (PREPAID/POSTPAID)
     */
    public ValidationResult validateReservation(ValidateReservationRequest request) {
        // ...
    }

    /**
     * Mark reservation as used after successful activation.
     */
    @Transactional
    public void markAsUsed(Long reservationId, UseReservationRequest request) {
        // ...
    }

    /**
     * Search reservations with filters and pagination.
     */
    public Page<ReservationResponse> searchReservations(/* filters */, Pageable pageable) {
        // ...
    }
}
```

### 6.3 ReservationExpiryScheduler — Background Job

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class ReservationExpiryScheduler {

    private final NumberReservationRepository reservationRepository;
    private final BssMediationClient bssClient;
    private final ReservationAuditLogRepository auditLogRepository;

    /**
     * Runs every 30 minutes to expire reservations past their expiry date.
     *
     * Flow:
     * 1. Find all ACTIVE reservations where expires_at < NOW()
     * 2. For each expired reservation:
     *    a. Unpick from BSS (OperateMSISDN 1030)
     *    b. Update status to EXPIRED
     *    c. Log audit entry
     * 3. Log summary of processed expirations
     */
    @Scheduled(cron = "0 0/30 * * * *")  // Every 30 minutes
    @Transactional
    public void expireReservations() {
        // ...
    }
}
```

---

## 7. Activation Flow with Reservation

### 7.1 Modified Activation Flow (ordermgmtservice)

The existing activation flow in `ordermgmtservice` needs a new handler in the `ActivationProcessingChain` to validate reservations.

#### 7.1.1 New Handler: ReservationValidationHandler

```java
package com.segmatek.ordermgmtservice.activation.chain.handlers;

/**
 * New handler in the ActivationProcessingChain.
 * Position: After OrderValidationHandler, Before PaymentValidationHandler.
 *
 * Handles:
 * - If order has a reservationCode → validate with salesapp-service
 * - If MSISDN is reserved in BSS (status=33) but no code provided → reject
 * - If no reservation → pass through (normal flow)
 */
@Component
@RequiredArgsConstructor
public class ReservationValidationHandler extends ActivationProcessingHandler {

    private final SalesAppServiceClient salesAppClient;

    @Override
    protected void doHandle(ActivationProcessingContext context) {
        CustomerProductOrder order = context.getOrder();

        // Case 1: No reservation code → check MSISDN isn't reserved
        if (order.getReservationCode() == null) {
            // Normal flow — no reservation validation needed
            // Optionally: Check BSS status to ensure MSISDN isn't reserved by someone else
            return;
        }

        // Case 2: Has reservation code → validate with salesapp-service
        ValidateReservationRequest request = ValidateReservationRequest.builder()
            .reservationCode(order.getReservationCode())
            .msisdn(order.getMsisdn())
            .customerId(/* from context */)
            .username(/* from context */)
            .channelId(order.getSalesChannelId())
            .packageType(/* from order type */)
            .build();

        ValidationResult result = salesAppClient.validateReservation(request);

        if (!result.isValid()) {
            context.markFailed("RESERVATION_INVALID", result.getErrorMessage(), "ReservationValidationHandler");
            return;
        }

        // Store reservation ID in context for later use
        context.addMetadata("reservationId", result.getReservationId());
    }
}
```

#### 7.1.2 New Handler: ReservationCompletionHandler

```java
/**
 * Runs after ActivationCompletionHandler.
 * Marks the reservation as USED after successful activation.
 */
@Component
@RequiredArgsConstructor
public class ReservationCompletionHandler extends ActivationProcessingHandler {

    private final SalesAppServiceClient salesAppClient;

    @Override
    protected void doHandle(ActivationProcessingContext context) {
        Long reservationId = context.getMetadata("reservationId", Long.class);

        if (reservationId == null) return; // No reservation to complete

        salesAppClient.markReservationUsed(
            reservationId,
            context.getOrderId(),
            /* activating username */
        );
    }
}
```

#### 7.1.3 Updated Chain Order

```
EXISTING CHAIN:
1. OrderValidationHandler
2. PaymentValidationHandler
3. CustomerValidationHandler
4. IccidValidationHandler
5. SematiValidationHandler
6. CustomerDataUpdateHandler
7. BssProvisioningHandler
8. NotificationHandler
9. ActivationCompletionHandler

MODIFIED CHAIN:
1. OrderValidationHandler
2. *** ReservationValidationHandler ***     ← NEW
3. PaymentValidationHandler
4. CustomerValidationHandler
5. IccidValidationHandler
6. SematiValidationHandler
7. CustomerDataUpdateHandler
8. BssProvisioningHandler
9. NotificationHandler
10. ActivationCompletionHandler
11. *** ReservationCompletionHandler ***     ← NEW
```

### 7.2 Full Activation Flow with Reservation (Normal Flow from FRS)

```
┌──────────────────────────────────────────────────────────────────────────┐
│          RESERVED NUMBER ACTIVATION FLOW (FRS 2.11 Normal Flow)         │
└──────────────────────────────────────────────────────────────────────────┘

Step 1: Open New Activation Page
    │
    ▼
Step 2: Enter ID Number
    ├── Saudi (auto-detect) → Enter contact number
    ├── Iqama → Select nationality + Enter contact number
    └── Visitor → Select ID type + Enter contact number
    │
    ▼
Step 3: Press Verification Button
    │   → TCC/Semati query customer info
    │   → Returns: Nationality, First Name, Family Name
    │
    ▼
Step 4: Select Activation Line Options
    │   → User selects: PrePaid Calls line + Physical SIM
    │
    ▼
Step 5: Enter/Scan SIM Serial (ICCID)
    │   → System verifies Physical SIM status immediately (early validation)
    │
    ▼
Step 6: MSISDN Selection Page (RESERVATION ENTRY POINT)
    │   ┌─────────────────────────────────────────────────┐
    │   │  User enters Reservation Code                    │
    │   │  → POST /api/v1/reservations/validate            │
    │   │    {                                              │
    │   │      reservationCode: "RES-xxx",                 │
    │   │      msisdn: "5XXXXXXXX",                        │
    │   │      customerId: "entered_id",                   │
    │   │      username: "current_user",                   │
    │   │      channelId: user_channel,                    │
    │   │      packageType: "PREPAID"                      │
    │   │    }                                              │
    │   │                                                   │
    │   │  Validation checks:                               │
    │   │  ✓ Reservation code exists and is ACTIVE         │
    │   │  ✓ MSISDN matches the reservation                │
    │   │  ✓ Customer ID matches (if type=CUSTOMER_ID)     │
    │   │  ✓ User matches (if type=USERNAME)               │
    │   │  ✓ Channel matches (if type=CHANNEL)             │
    │   │  ✓ Package type matches (if type=PACKAGE_TYPE)   │
    │   │  ✓ Organization matches (if type=ORGANIZATION)   │
    │   │                                                   │
    │   │  → If valid: Show reserved MSISDN, press Next    │
    │   │  → If invalid: Show error message                │
    │   └─────────────────────────────────────────────────┘
    │
    ▼
Step 7: Offers Page
    │   → Show available offers/plans for user to choose
    │
    ▼
Step 8: OTP Verification (configurable per screen/user/channel)
    │   → Send OTP to customer contact number
    │   → User enters OTP → System validates
    │   → (Skip if OTP disabled in reservation config)
    │
    ▼
Step 9: Customer Verification (Nafath)
    │   → Nafath code appears
    │   → Customer approves on Nafath app
    │   → Nafath verifies → sends to TCC → TCC stores data
    │
    ▼
Step 10: Activation Processing (ordermgmtservice chain)
    │   1. OrderValidationHandler         → Validate order
    │   2. ReservationValidationHandler   → Validate reservation ← NEW
    │   3. PaymentValidationHandler       → Verify payment
    │   4. CustomerValidationHandler      → Validate customer
    │   5. IccidValidationHandler         → Validate SIM card
    │   6. SematiValidationHandler        → TCC/Semati check
    │   7. CustomerDataUpdateHandler      → Update customer data
    │   8. BssProvisioningHandler         → Create subscriber in BSS
    │   9. NotificationHandler            → Send notifications
    │   10. ActivationCompletionHandler   → Mark complete
    │   11. ReservationCompletionHandler  → Mark reservation USED ← NEW
    │
    ▼
Step 11: Completion Screen
    │   → Show success message
    │   → Download contract button
    │
    ▼
Done ✓
```

---

## 8. Reservation Flow (Pre-Activation)

### 8.1 Reserve a Number Flow

```
┌──────────────────────────────────────────────────────────────────┐
│              RESERVE A NUMBER FLOW                                │
└──────────────────────────────────────────────────────────────────┘

Sales User → Opens Number Reservation Page
    │
    ▼
1. Load Reservation Config
   │   GET /api/v1/reservations/config?username=X&channelId=Y
   │   → Returns allowed reservation types for this user
   │
   ▼
2. User Selects MSISDN(s)
   │   → Browse available numbers (BSS query)
   │   → Select one or more MSISDNs
   │
   ▼
3. User Selects Reservation Type
   │   (Only shows options allowed by config)
   │   ├── By Organization → Select organization from dropdown
   │   ├── By Username → Enter/search username
   │   ├── By Customer ID → Enter customer ID (national/iqama)
   │   ├── By Channel → Select channel from dropdown
   │   └── By Package Type → Select Prepaid/Postpaid
   │
   ▼
4. Submit Reservation
   │   POST /api/v1/reservations
   │   {
   │     msisdn, reservationType, target value, expiresInHours
   │   }
   │
   ▼
5. Backend Processing (NumberReservationService)
   │
   ├── 5a. Permission Check
   │   │   → Verify user has reservation page access (via omni-auth BusinessRoleAttribute)
   │   │   → Verify reservation type is allowed for user's config scope
   │   │   → Fail 403 if not authorized
   │   │
   │   ▼
   ├── 5b. MSISDN Availability Check
   │   │   → GET BSS GetResourceDetailInfo (via bssmediationcontroller)
   │   │   → Status must be "Normal" (2) or "Pick" (34)
   │   │   → Fail 404 if MSISDN not found
   │   │   → Fail 409 if already Reserved (33)
   │   │
   │   ▼
   ├── 5c. Active Reservation Check
   │   │   → Query number_reservation WHERE msisdn = X AND status = 'ACTIVE'
   │   │   → Fail 409 if exists
   │   │
   │   ▼
   ├── 5d. User Limit Check
   │   │   → Count active reservations by this user
   │   │   → Compare with maxReservations from config
   │   │   → Fail 422 if limit exceeded
   │   │
   │   ▼
   ├── 5e. BSS Pick Operation
   │   │   → POST OperateMSISDN(msisdn, oprType="1029")
   │   │   → MSISDN status changes to "Reserved" (33) in BSS
   │   │   → Fail 422 if BSS operation fails
   │   │
   │   ▼
   ├── 5f. Generate Reservation Code
   │   │   → Format: "RES-{YYYYMMDD}-{6-char-alphanumeric}"
   │   │   → Ensure uniqueness
   │   │
   │   ▼
   ├── 5g. Save Reservation Record
   │   │   → Insert into number_reservation
   │   │
   │   ▼
   └── 5h. Audit Log
       │   → Insert into reservation_audit_log (action=RESERVE, result=SUCCESS)
       │
       ▼
6. Return ReservationResponse with code
   │   → Share reservation code with user
   │
   ▼
Done ✓
```

### 8.2 Release a Number Flow

```
┌──────────────────────────────────────────────────────────────────┐
│              RELEASE A RESERVED NUMBER FLOW                       │
└──────────────────────────────────────────────────────────────────┘

Sales User → Opens Reservation Management Page
    │
    ▼
1. View Active Reservations
   │   GET /api/v1/reservations?status=ACTIVE&reservedBy=current_user
   │
   ▼
2. Select Reservation to Release
   │
   ▼
3. Enter Release Reason
   │
   ▼
4. Submit Release
   │   DELETE /api/v1/reservations/{id}
   │   { reason: "Customer cancelled" }
   │
   ▼
5. Backend Processing
   │
   ├── 5a. Find reservation → Fail 404 if not found
   ├── 5b. Validate ACTIVE status → Fail 400 if not ACTIVE
   ├── 5c. Validate permission (reserved_by matches or admin)
   ├── 5d. BSS Unpick → OperateMSISDN(msisdn, "1030")
   ├── 5e. Update status → RELEASED, set released_at, released_by, release_reason
   └── 5f. Audit log → action=RELEASE, result=SUCCESS
   │
   ▼
Done ✓
```

---

## 9. Validation Rules Matrix

### 9.1 Reservation Validation During Activation

| Reservation Type | Validation Rule | Error Message |
|------------------|----------------|---------------|
| **ORGANIZATION** | User's organization (from omni-auth) must match `reservation.organizationId` | "Reserved number does not belong to your organization" |
| **USERNAME** | Activating user's username must match `reservation.username` | "This number is reserved for another user" |
| **CUSTOMER_ID** | Entered customer ID must match `reservation.customerId` | "Reserved number does not match customer ID" |
| **CHANNEL** | User's channel (from omni-auth) must match `reservation.channelId` | "This number is not available in your channel" |
| **PACKAGE_TYPE** | Selected package type must match `reservation.packageType` | "Reserved number does not match the package type" |

### 9.2 Alternative Flow Error Conditions (from FRS)

| Condition | Error Code | Message |
|-----------|-----------|---------|
| Reserved number doesn't match customer ID | `CUSTOMER_MISMATCH` | "Reserved number does not match customer ID" |
| User cannot view reserved MSISDN | `ACCESS_DENIED` | "User not able to view the reserved MSISDN — number not assigned to user or channel" |
| Package type mismatch | `PACKAGE_MISMATCH` | "Reserved number does not match the package type" |
| Reservation code invalid/expired | `RESERVATION_EXPIRED` | "Reservation code is invalid or has expired" |
| No permission on reservation screen | `FORBIDDEN` | "Don't have permission on the screen" |
| Virtual credit insufficient | `INSUFFICIENT_CREDIT` | "Virtual credit does not cover the price" |

---

## 10. Configuration Design

### 10.1 Global Configurations (digitalconfigmanagement)

```json
// Key: "number_reservation_feature"
{
  "enabled": true,
  "defaultExpiryHours": 720,
  "maxBulkReservations": 50,
  "codePrefix": "RES",
  "codeFormat": "RES-{date}-{random6}"
}
```

### 10.2 Reservation Config Hierarchy (Resolution Order)

```
1. USER-level config     (most specific — overrides all below)
2. ROLE-level config
3. CHANNEL-level config
4. PARTNER-level config
5. GLOBAL-level config   (least specific — fallback)
```

When resolving a user's effective configuration, the system checks from most-specific to least-specific and merges:

```java
public ReservationConfigResponse getEffectiveConfig(String username, Long channelId) {
    // 1. Try USER-level
    Optional<ReservationConfig> userConfig = repository.findByScopeAndValue(USER, username);

    // 2. Try ROLE-level (from user's role)
    // 3. Try CHANNEL-level
    Optional<ReservationConfig> channelConfig = repository.findByScopeAndValue(CHANNEL, channelId.toString());

    // 4. Try PARTNER-level (from user's partner)
    // 5. Fallback to GLOBAL
    ReservationConfig globalConfig = repository.findByScopeAndValue(GLOBAL, null);

    // Merge: most specific wins for each field
    return merge(userConfig, roleConfig, channelConfig, partnerConfig, globalConfig);
}
```

### 10.3 OTP Configuration

Per the FRS: "OTP step will be enabled/disabled based on a new configuration per Screen per user/Channel/Partner."

This is handled by the `otp_enabled` field in `reservation_config`, following the same hierarchy:

| Level | Example |
|-------|---------|
| Per User | User "seller1" → OTP disabled |
| Per Channel | Channel "Retail" → OTP enabled |
| Per Partner | Partner "PartnerX" → OTP disabled |
| Global | Default → OTP enabled |

---

## 11. New Service Setup (salesapp-service)

### 11.1 Maven Project (pom.xml)

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.0.2</version>
</parent>

<groupId>com.segmatek</groupId>
<artifactId>salesapp-service</artifactId>
<version>0.0.1-SNAPSHOT</version>

<properties>
    <java.version>17</java.version>
</properties>

<dependencies>
    <!-- Spring Boot -->
    <dependency>spring-boot-starter-web</dependency>
    <dependency>spring-boot-starter-data-jpa</dependency>
    <dependency>spring-boot-starter-validation</dependency>
    <dependency>spring-boot-starter-actuator</dependency>

    <!-- Database -->
    <dependency>postgresql</dependency>

    <!-- Shared Library -->
    <dependency>
        <groupId>com.segmatek</groupId>
        <artifactId>digitalLookup</artifactId>
    </dependency>
    <dependency>
        <groupId>com.segmatek</groupId>
        <artifactId>digitalCommonLibrary</artifactId>
    </dependency>

    <!-- Utilities -->
    <dependency>lombok</dependency>
    <dependency>mapstruct</dependency>

    <!-- OpenAPI -->
    <dependency>springdoc-openapi-starter-webmvc-ui</dependency>
</dependencies>
```

### 11.2 Application Properties

```properties
spring.application.name=salesapp-service
server.port=8078

# Database
spring.datasource.url=jdbc:postgresql://${DB_HOST:localhost}:5432/salesapp_db
spring.datasource.username=${DB_USER:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# Service URLs
service.url.bss.mediation=http://digitalmediationbss-svc:8071/dot
service.url.auth=http://omni-auth-manager-svc:8080
service.url.config.management=http://digitalconfigmanagement-svc:8074

# Reservation Defaults
reservation.default.expiry-hours=720
reservation.code.prefix=RES
reservation.expiry.scheduler.cron=0 0/30 * * * *
reservation.max.bulk=50

# Common config
common.config.profile=${COMMON_ENV:common.properties}
```

### 11.3 Dockerfile

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/salesapp-service-*.jar app.jar
EXPOSE 8078
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 11.4 Kubernetes Service Configuration (digital-configuration)

```yaml
# salesapp-service deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: salesapp-service
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: salesapp-service
          image: salesapp-service:latest
          ports:
            - containerPort: 8078
          env:
            - name: DB_HOST
              value: "postgresql-svc"
            - name: COMMON_ENV
              value: "common.properties"
---
apiVersion: v1
kind: Service
metadata:
  name: salesapp-svc
spec:
  ports:
    - port: 8078
      targetPort: 8078
  selector:
    app: salesapp-service
```

---

## 12. Security & Authorization

### 12.1 Permission Model

A new page and attributes must be created in the `digital-external-omni-auth` permission system:

```
Module: Sales App
  └── Page: Number Reservation
       ├── SubPage: Reserve Numbers
       │    ├── Attribute: "reserve_number"         → AccessRight: CREATE
       │    ├── Attribute: "reserve_by_org"          → AccessRight: CREATE
       │    ├── Attribute: "reserve_by_username"     → AccessRight: CREATE
       │    ├── Attribute: "reserve_by_customer"     → AccessRight: CREATE
       │    ├── Attribute: "reserve_by_channel"      → AccessRight: CREATE
       │    └── Attribute: "reserve_by_package"      → AccessRight: CREATE
       ├── SubPage: Manage Reservations
       │    ├── Attribute: "view_reservations"       → AccessRight: VIEW
       │    ├── Attribute: "release_reservation"     → AccessRight: DELETE
       │    └── Attribute: "view_all_reservations"   → AccessRight: VIEW (admin)
       └── SubPage: Reservation Config
            ├── Attribute: "view_config"             → AccessRight: VIEW
            └── Attribute: "edit_config"             → AccessRight: EDIT (admin)
```

### 12.2 API Security

All `salesapp-service` endpoints should be secured via:
- **Keycloak JWT token** — Same as other Java services
- **`@PreAuthorize`** annotations for role-based access
- **BusinessRoleAttribute** checks for fine-grained permission

### 12.3 Service-to-Service Authentication

The internal validation endpoint (`POST /api/v1/reservations/validate`) called by `ordermgmtservice` should use:
- Service-to-service token (Keycloak client credentials grant)
- Or internal network trust (within Kubernetes cluster)

---

## 13. Pre-Conditions Mapping (from FRS)

| FRS Pre-Condition | Implementation |
|-------------------|----------------|
| User created on portal and BSS | `digital-external-omni-auth` User entity + BSS subscriber |
| User has permission to Number Reservation page | `BusinessRoleAttribute` with key `reserve_number` → AccessRight `CREATE` |
| User authorized from Nafath | Nafath verification at activation step (existing flow) |
| User has account on Manafeth, refreshes location every 10 min | Pre-existing Manafeth integration (not in reservation scope) |
| User has virtual credit on virtual Wallet | Wallet balance check at activation (existing flow via PaymentValidationHandler) |
| Number already reserved for customer by user | `number_reservation` table with status=ACTIVE |

---

## 14. Testing Strategy

### 14.1 Unit Tests

| Test Class | Coverage |
|------------|----------|
| `NumberReservationServiceTest` | Reserve, release, validate, expire, bulk operations |
| `ReservationValidationServiceTest` | All 5 reservation type validations |
| `ReservationCodeGeneratorTest` | Code format, uniqueness |
| `ReservationConfigServiceTest` | Config hierarchy resolution |
| `ReservationExpirySchedulerTest` | Expiry detection and BSS unpick |

### 14.2 Integration Tests

| Test | Scope |
|------|-------|
| Reserve → Activate → Complete | Full E2E flow with ordermgmtservice |
| Reserve → Release | Reserve and unpick from BSS |
| Reserve → Expire | Scheduler-based expiry |
| Bulk Reserve | Multiple MSISDNs at once |
| Permission denied scenarios | All 5 reservation types with wrong user/customer/channel |
| BSS failure scenarios | BSS Pick/Unpick failures, retry behavior |

---

## 15. Migration Considerations

### 15.1 Data Migration from .NET

If there are existing reservations in the .NET system (`SalesOrder.ReservationCode`):

1. Extract completed orders with `ReservationCode` from SQL Server
2. Map to `number_reservation` records with `status = USED`
3. Preserve reservation codes for audit trail continuity

### 15.2 Parallel Operation

During migration, both systems may operate:
- .NET Sales Portal continues for existing users
- Java `salesapp-service` serves new reservation requests
- BSS is the source of truth for MSISDN status (both systems interact via BSS)

---

## 16. Summary of Changes Per Service

| Service | Change Type | Changes |
|---------|------------|---------|
| **salesapp-service** | **NEW** | Entire new microservice for number reservation |
| **digitallookup** | MODIFY | Add new enums: ReservationType, ReservationStatus, ConfigScope, AuditAction, AuditResult |
| **ordermgmtservice** | MODIFY | Add `reservation_code`, `reservation_id` to CustomerProductOrder; Add ReservationValidationHandler and ReservationCompletionHandler to chain |
| **digital-external-omni-auth** | DATA | Insert new Module, Page, SubPage, Attribute records for permissions |
| **digitalconfigmanagement** | DATA | Insert new configuration key `number_reservation_feature` |
| **digital-configuration** | MODIFY | Add Kubernetes deployment for salesapp-service |
| **bssmediationcontroller** | NONE | No changes — existing Pick/Unpick operations already support reservation |

---

## Appendix A: BSS Operation Codes Reference

| Operation Code | Name | Description | Used In |
|---------------|------|-------------|---------|
| 1029 | Pick/Reserve | Picks an MSISDN from pool → status changes to Reserved (33) or Pick (34) | Reserve number |
| 1030 | Unpick/Release | Releases a picked MSISDN → status returns to Normal (2) | Release reservation, expiry |

## Appendix B: BSS MSISDN Status Codes

| Status Code | Name | Meaning for Reservation |
|-------------|------|-------------------------|
| 0 | Init Lock | Cannot reserve |
| 2 | Normal | Available — can be reserved |
| 3 | Locked | Cannot reserve |
| 4 | Sold | Already sold — cannot reserve |
| 33 | Reserved | Already reserved in BSS |
| 34 | Pick | Picked/Reserved via OperateMSISDN |
| 36 | Used | In use — cannot reserve |

## Appendix C: Reservation Code Format

```
Format: RES-{YYYYMMDD}-{6-char-alphanumeric}
Example: RES-20250208-A3X7K9

Components:
- "RES" — Fixed prefix
- Date — Reservation date (YYYYMMDD)
- Random — 6 alphanumeric characters (uppercase + digits)

Uniqueness: Guaranteed by unique constraint on reservation_code column + retry logic
```

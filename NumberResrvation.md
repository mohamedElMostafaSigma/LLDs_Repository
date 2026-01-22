# Low-Level Design: Number Reservation & SIM Activation Flow

| **Document Info** | |
|-------------------|-----------------|
| **Version** | 1.0 |
| **Created Date** | January 19, 2026 |
| **Last Updated** | January 19, 2026 |
| **Status** | Verified from Source Code |
| **Application** | RedBull Sales Portal |

---

## Table of Contents

1. [Overview](#1-overview)
2. [Source Code References](#2-source-code-references)
3. [Database Schema](#3-database-schema)
4. [Entity Definitions](#4-entity-definitions)
5. [API Endpoints](#5-api-endpoints)
6. [Complete Flow Diagram](#6-complete-flow-diagram)
7. [BSS API Integration](#7-bss-api-integration)
8. [TCC API Integration](#8-tcc-api-integration)
9. [Activation State Machine](#9-activation-state-machine)
10. [Configuration](#10-configuration)
11. [Key Business Rules](#11-key-business-rules)
12. [Error Handling](#12-error-handling)

---

## 1. Overview

### 1.1 Purpose

This document describes the complete **Number Reservation and SIM Activation** flow in the RedBull Sales Portal, covering the process from customer ID validation through number selection, plan selection, and final SIM activation.

### 1.2 Scope

- Customer identity validation (TCC eligibility)
- MSISDN (phone number) query and reservation
- Plan and addon selection
- SIM card activation (physical and eSIM)
- Wallet deduction and commission calculation
- Port-in (MNP) handling

### 1.3 Key Components

| Component | Description |
|-----------|-------------|
| **TCC** | Telecom Compliance Center - Government regulatory system for SIM registration |
| **BSS** | Huawei Business Support System - CRM and inventory management |
| **MSISDN** | Mobile Station International Subscriber Directory Number (phone number) |
| **ICCID** | Integrated Circuit Card Identifier (SIM card serial) |
| **Vanity Level** | Number tier classification (1-5) affecting pricing |

### 1.4 Activation Types

| Value | Enum | Description |
|-------|------|-------------|
| 0 | `PrepaidNewSIM` | New prepaid SIM activation |
| 1 | `PrepaidPortIn` | Prepaid number port-in from another operator |
| 2 | `PrepaidDataSIM` | Prepaid data-only SIM |
| 3 | `PostpaidNewSIM` | New postpaid SIM activation |
| 4 | `PostpaidPortIn` | Postpaid number port-in |
| 5 | `PostpaidDataSIM` | Postpaid data-only SIM |

---

## 2. Source Code References

| File | Purpose | Key Lines |
|------|---------|-----------|
| `Modules/SalesAPI/SalesAPIController.cs` | Main API controller | 925-1099, 1169-1908, 3351-3425 |
| `Modules/Common/Helpers/BssApiHelper.cs` | Huawei BSS integration | 3044-3076, 3298-3339 |
| `Modules/Common/Helpers/TCCHelper.cs` | TCC API integration | CheckEligibility, AddNumber, NumberMNP |
| `Models/Tables/SalesOrder.cs` | Order entity | 77-148 |
| `Models/Tables/VanityLevels.cs` | Number pricing tiers | 6-18 |
| `Models/SalesDbContext.cs` | Database context | DbSet definitions |

---

## 3. Database Schema

### 3.1 Entity Relationship Diagram

```
┌─────────────────────┐
│      Seller         │
├─────────────────────┤
│ UserId (PK)         │
│ WalletBalance       │
│ PartnerId           │
│ SellerChannel       │
│ SellerUserName      │
└──────────┬──────────┘
           │ 1:N
           ▼
┌─────────────────────┐         ┌─────────────────────┐
│    SalesOrder       │         │    VanityLevels     │
├─────────────────────┤         ├─────────────────────┤
│ Id (PK)             │         │ Id (PK)             │
│ SellerId (FK)       │◄────────│ Tier                │
│ MSISDN              │  Lookup │ Price               │
│ VanityLevel         │─────────│ OldPrice            │
│ ICCID               │         │ NameEn              │
│ Status              │         └─────────────────────┘
│ ActivationState     │
│ ReservationCode     │         ┌─────────────────────┐
│ PlanId              │         │     ExtraSIM        │
│ OrderTotal          │         ├─────────────────────┤
│ WalletBalanceBefore │◄────────│ OrderId (FK)        │
│ WalletBalanceAfter  │   1:N   │ MSISDN              │
│ EligibilityTCN      │         │ ICCID               │
│ ActivationTCN       │         │ Status              │
│ IsPostpaid          │         │ IsESIM              │
│ IsMNP               │         └─────────────────────┘
└─────────────────────┘
           │
           │ Lookup
           ▼
┌─────────────────────┐         ┌─────────────────────┐
│      PlanArt        │         │       Addon         │
├─────────────────────┤         ├─────────────────────┤
│ Id (PK)             │         │ Id (PK)             │
│ NameEn              │         │ NameEn              │
│ Price               │         │ Price               │
│ IsPostpaid          │         │ OfferId             │
│ OfferId             │         └─────────────────────┘
│ Commission          │
└─────────────────────┘
```

### 3.2 SalesOrder Table

```sql
CREATE TABLE [SalesOrder] (
    [Id] INT IDENTITY(1,1) PRIMARY KEY,
    [SellerId] INT NOT NULL,

    -- Customer Information
    [IdNumber] NVARCHAR(MAX),
    [IdType] INT,
    [Nationality] INT,
    [FirstName] NVARCHAR(MAX),
    [LastName] NVARCHAR(MAX),
    [ContactNumber] NVARCHAR(MAX),
    [Email] NVARCHAR(MAX),
    [CRMCustomerId] NVARCHAR(MAX),

    -- Number Information
    [MSISDN] NVARCHAR(MAX),              -- Reserved phone number (5xxxxxxxx format)
    [VanityLevel] INT,                   -- Number tier (1-5)
    [MSISDNCost] DECIMAL(18,2),          -- Number price including VAT
    [ReservationCode] NVARCHAR(MAX),     -- Pre-reservation code (optional)

    -- SIM Information
    [ICCID] NVARCHAR(MAX),               -- SIM card serial
    [IMSI] NVARCHAR(MAX),                -- International Mobile Subscriber Identity
    [IsESim] INT,                        -- 0 = Physical SIM, 1 = eSIM

    -- Plan Information
    [PlanId] NVARCHAR(MAX),
    [PlanName] NVARCHAR(MAX),
    [PlanCost] DECIMAL(18,2),
    [Addons] NVARCHAR(MAX),              -- JSON array of addon IDs
    [AddonsCost] DECIMAL(18,2),
    [ExtraSIMCost] DECIMAL(18,2),

    -- Order Totals
    [OrderTotal] DECIMAL(18,2),
    [WalletBalanceBefore] DECIMAL(18,2),
    [WalletBalanceAfter] DECIMAL(18,2),
    [Commission] DECIMAL(18,2),

    -- Status & State
    [Status] INT,                        -- OrderStatus enum
    [ActivationState] TINYINT,           -- State machine: 0,1,2,3,4
    [IsMNP] INT,                         -- ActivationType enum
    [MNPOperator] NVARCHAR(MAX),
    [IsPostpaid] INT,                    -- 0 = Prepaid, 1 = Postpaid

    -- TCC Integration
    [EligibilityTCN] NVARCHAR(MAX),      -- TCC eligibility transaction number
    [ActivationTCN] NVARCHAR(MAX),       -- TCC activation transaction number

    -- Authentication
    [OTP] NVARCHAR(MAX),
    [OTPExpiry] DATETIME,
    [UseIAMToken] INT,
    [IAMToken] NVARCHAR(MAX),
    [FingerIndex] INT,
    [FingerImage] NVARCHAR(MAX),
    [AbsherToken] NVARCHAR(MAX),

    -- Location
    [OrderLat] NVARCHAR(MAX),
    [OrderLng] NVARCHAR(MAX),
    [OrderCity] NVARCHAR(MAX),
    [OrderAddress] NVARCHAR(MAX),

    -- Metadata
    [OrderDate] DATETIME,
    [PartnerId] INT,
    [SubType] INT,
    [Remarks] NVARCHAR(MAX),
    [RowVersion] ROWVERSION,             -- Concurrency token
    [IsRefunded] BIT
)
```

### 3.3 VanityLevels Table

```sql
CREATE TABLE [VanityLevels] (
    [Id] INT PRIMARY KEY,
    [Tier] INT,                          -- Level 1-5
    [Price] DECIMAL(18,2),               -- Current price
    [OldPrice] DECIMAL(18,2),            -- Previous price (for discounts)
    [Color] NVARCHAR(MAX),               -- UI color code
    [NameAr] NVARCHAR(MAX),              -- Arabic name
    [NameEn] NVARCHAR(MAX)               -- English name
)
```

### 3.4 ExtraSIM Table

```sql
CREATE TABLE [ExtraSIM] (
    [Id] INT IDENTITY(1,1) PRIMARY KEY,
    [OrderId] INT,                       -- FK to SalesOrder
    [MSISDN] NVARCHAR(MAX),              -- Data number
    [ICCID] NVARCHAR(MAX),
    [IsESIM] INT,
    [Status] INT,
    [OrderType] INT,
    [AddedDate] DATETIME
)
```

---

## 4. Entity Definitions

### 4.1 SalesOrder Entity

```csharp
[Table("SalesOrder")]
public class SalesOrder
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }
    public int SellerId { get; set; }

    // Number reservation fields
    public string MSISDN { get; set; }
    public int? VanityLevel { get; set; }
    public decimal? MSISDNCost { get; set; } = 0;
    public string? ReservationCode { get; set; }

    // SIM fields
    public string ICCID { get; set; }
    public string IMSI { get; set; }
    public int? IsESim { get; set; }

    // Plan fields
    public string PlanId { get; set; }
    public string PlanName { get; set; }
    public decimal? PlanCost { get; set; } = 0;
    public string Addons { get; set; }
    public decimal? AddonsCost { get; set; } = 0;

    // Status fields
    public OrderStatus Status { get; set; }
    public byte? ActivationState { get; set; }  // State machine
    public ActivationType IsMNP { get; set; }
    public int? IsPostpaid { get; set; }

    // TCC fields
    public string EligibilityTCN { get; set; }
    public string ActivationTCN { get; set; }

    // Concurrency
    [Timestamp]
    public byte[] RowVersion { get; set; }
}
```

### 4.2 Enums

```csharp
public enum OrderStatus
{
    New = 0,
    InProgress = 1,
    Completed = 2,
    Canceled = 4,
    Failed = 5,
    AwaitingActivation = 6
}

public enum ActivationType
{
    PrepaidNewSIM = 0,
    PrepaidPortIn = 1,
    PrepaidDataSIM = 2,
    PostpaidNewSIM = 3,
    PostpaidPortIn = 4,
    PostpaidDataSIM = 5,
    Device = 6
}

public enum IdType
{
    Citizen = 1,
    Resident = 2,
    Visitor = 3,
    GCC_Passport = 4,
    GCC_National_Id = 5,
    Pilgrim_Passport = 6,
    Pilgrim_Border = 7,
    Umrah_Passport = 8,
    Visitor_Visa = 9,
    Umrah_Visa = 10,
    Haj_Visa = 11,
    Premium_Resident = 12,
    Diplomat = 99
}
```

---

## 5. API Endpoints

### 5.1 ValidateIdNew

| Property | Value |
|----------|-------|
| **Method** | POST |
| **Route** | `/api/Sales/ValidateIdNew` |
| **Auth** | JWT Required |
| **Source** | SalesAPIController.cs:925-1099 |

**Request:**
```json
{
    "IdNumber": "1234567890",
    "IdType": 1,
    "Nationality": 113,
    "ContactNumber": "0512345678",
    "IsMNP": "0",
    "IsESim": "0"
}
```

**Response:**
```json
{
    "custInfo": {
        "idNumber": "1234567890",
        "firstName": "Mohammed",
        "lastName": "Ahmed",
        "customerId": "CRM001",
        "status": "Active"
    },
    "tccEligibility": {
        "tcn": "e81e370a-c021-49ca-af46-62e4285fe3d7",
        "code": 600,
        "message": "success"
    },
    "order": {
        "Id": 12345,
        "Status": 0,
        "EligibilityTCN": "e81e370a-c021-49ca-af46-62e4285fe3d7"
    }
}
```

**Logic Flow:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ ValidateIdNew                                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. Check if service is disabled (DisableActivation config)                │
│                                                                             │
│   2. Check BlackList table for ID                                           │
│      └─> If found: Return "Sorry, this Person is BlackListed"               │
│                                                                             │
│   3. Get seller from JWT token                                              │
│                                                                             │
│   4. Call BSS: GetCustomerInformation(IdNumber, IdType)                     │
│      └─> Returns: firstName, lastName, customerId                           │
│                                                                             │
│   5. Check if Postpaid request from POS channel                             │
│      └─> If sellerType == 1 && isPostpaid: Return error                     │
│                                                                             │
│   6. For MNP/Port-In requests (IsMNP = 1 or 4):                             │
│      └─> Skip TCC eligibility check                                         │
│      └─> Create order directly                                              │
│                                                                             │
│   7. For New SIM requests:                                                  │
│      └─> Call TCC: CheckEligibility(IdNumber, Nationality, IdType, SubType) │
│      └─> If TCC code = 605: Try alternative SubType                         │
│      └─> If TCC code = 600: Create SalesOrder                               │
│                                                                             │
│   8. Generate OTP and set expiry (5 minutes)                                │
│                                                                             │
│   9. Save SalesOrder with Status = New                                      │
│                                                                             │
│   10. Return customer info, TCC result, and order                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 5.2 GetMSISDNs

| Property | Value |
|----------|-------|
| **Method** | POST |
| **Route** | `/api/Sales/GetMSISDNs` |
| **Auth** | JWT Required |
| **Source** | SalesAPIController.cs:3351-3393 |

**Request:**
```json
{
    "filter": "055",
    "count": 10,
    "vanity": 3
}
```

**Response:**
```json
[
    {
        "ServiceNumber": "551234567",
        "ResDeptId": "RD001",
        "Level": 3,
        "Price": 100.00,
        "OldPrice": 150.00
    },
    {
        "ServiceNumber": "551234568",
        "ResDeptId": "RD001",
        "Level": 3,
        "Price": 100.00,
        "OldPrice": 150.00
    }
]
```

**Logic Flow:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ GetMSISDNs                                                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. Get seller from JWT token                                              │
│                                                                             │
│   2. Parse parameters:                                                      │
│      - filter: Search pattern (e.g., "055")                                 │
│      - count: Number of results (default: 10)                               │
│      - vanity: Tier level (default: 4)                                      │
│                                                                             │
│   3. Load VanityLevels pricing table                                        │
│                                                                             │
│   4. Call BSS: GetAvailableMSISDNs(seller, vanity, filter, count)           │
│      └─> SOAP Action: QueryAvailableNumber                                  │
│      └─> Returns: ServiceNumber, ResDeptId, Level                           │
│                                                                             │
│   5. Map Level to Price from VanityLevels                                   │
│                                                                             │
│   6. Filter out:                                                            │
│      - Numbers with Level 1 or 5 (reserved/special)                         │
│      - Numbers in MobileAlreadyExists table                                 │
│      - Numbers in MSISDNPool table                                          │
│                                                                             │
│   7. Return filtered list with pricing                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 5.3 UpdateOrder

| Property | Value |
|----------|-------|
| **Method** | POST |
| **Route** | `/api/Sales/UpdateOrder` |
| **Auth** | JWT Required |
| **Source** | SalesAPIController.cs:1169-1908 |

This is a **multi-step endpoint** that handles different phases of the order process based on the `step` parameter.

#### Step: "select-number"

**Request:**
```json
{
    "OrderId": 12345,
    "step": "select-number",
    "MSISDN": "0551234567",
    "MSISDNCost": 100.00,
    "MSISDNLevel": 3,
    "ReservationCode": "RES123",
    "IsMNP": "0",
    "MNPOperator": "",
    "ExtraSIM": "[{\"enabled\": true, \"eSIM\": 0}]"
}
```

**Logic:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ UpdateOrder (step="select-number")                                          │
│ Source: SalesAPIController.cs:1277-1365                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. Normalize MSISDN: Remove prefixes (05, +9665, etc.) → 5xxxxxxxx        │
│                                                                             │
│   2. If MSISDN changed from previous selection:                             │
│      └─> Clone order with new MSISDN (preserves history)                    │
│                                                                             │
│   3. Calculate costs:                                                       │
│      - MSISDNCost = MSISDNCost × (1 + 0.15)  [Add 15% VAT]                  │
│      - OrderTotal = MSISDNCost + PlanCost + AddonsCost                      │
│                                                                             │
│   4. Set ReservationCode if provided                                        │
│                                                                             │
│   5. Validate seller wallet balance                                         │
│      └─> If OrderTotal > WalletBalance: Throw "Insufficient Wallet Balance" │
│                                                                             │
│   6. For Postpaid with ExtraSIMs:                                           │
│      └─> For each enabled ExtraSIM:                                         │
│          - Call BSS: GetDataMSISDN(seller) → Get data number                │
│          - Call BSS: OperateMSISDN(MSISDN, seller) → Reserve number         │
│          - Create ExtraSIM record                                           │
│      └─> Calculate ExtraSIMCost                                             │
│                                                                             │
│   7. For New SIM activations (PrepaidNewSIM or PostpaidNewSIM):             │
│      └─> Call BSS: OperateMSISDN(ord.MSISDN, seller) → RESERVE NUMBER       │
│                                                                             │
│   8. Save order                                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

#### Step: "select-plan"

**Request:**
```json
{
    "OrderId": 12345,
    "step": "select-plan",
    "PlanId": "PLAN001",
    "PlanName": "Unlimited 200",
    "Addons": "[\"ADDON001\", \"ADDON002\"]"
}
```

**Logic:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ UpdateOrder (step="select-plan")                                            │
│ Source: SalesAPIController.cs:1367-1394                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. Set PlanId, PlanName, Addons                                           │
│                                                                             │
│   2. Calculate PlanCost:                                                    │
│      - Postpaid: PlanCost = Plan.Price (no VAT)                             │
│      - Prepaid:  PlanCost = Plan.Price × (1 + 0.15) [Add 15% VAT]           │
│                                                                             │
│   3. Calculate AddonsCost:                                                  │
│      - Loop through addons JSON array                                       │
│      - Sum prices (with VAT for prepaid)                                    │
│                                                                             │
│   4. Update OrderTotal = MSISDNCost + PlanCost + AddonsCost                 │
│                                                                             │
│   5. Validate wallet balance again                                          │
│                                                                             │
│   6. Calculate Commission (for POS sellers):                                │
│      - Commission based on plan and addon commission rates                  │
│                                                                             │
│   7. Check for PlanDevice (bundled device)                                  │
│      └─> Set DeviceType if applicable                                       │
│                                                                             │
│   8. Update Status = AwaitingActivation                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

#### Step: "authenticate"

**Request:**
```json
{
    "OrderId": 12345,
    "step": "authenticate",
    "ICCID": "8996612345678901234",
    "Email": "customer@email.com",
    "AuthMethod": "1",
    "FingerIndex": 2,
    "FingerImage": "base64...",
    "ExtraSIM": "[{\"ICCID\": \"8996612345678901235\"}]"
}
```

**Logic:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ UpdateOrder (step="authenticate")                                           │
│ Source: SalesAPIController.cs:1396-1900                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. IDEMPOTENCY CHECK: Verify order not already processed                  │
│      └─> If ActivationState >= 2: Return "Order already in progress"        │
│                                                                             │
│   2. Validate wallet balance                                                │
│                                                                             │
│   3. Get/Validate ICCID:                                                    │
│      - Physical SIM: Use provided ICCID                                     │
│      - eSIM: Call BSS PickESim() to allocate ICCID                          │
│      - Validate format: ^899661\d{13,14}$                                   │
│                                                                             │
│   4. Get SIM details from BSS                                               │
│      └─> Returns IMSI for the ICCID                                         │
│                                                                             │
│   5. Process ExtraSIMs (if any):                                            │
│      └─> Set ICCID for each extra SIM                                       │
│                                                                             │
│   6. Set authentication fields:                                             │
│      - FingerIndex, FingerImage (biometric)                                 │
│      - IAMToken or AbsherToken (digital auth)                               │
│                                                                             │
│   7. Validate location (OrderLat, OrderLng)                                 │
│                                                                             │
│   8. SET ActivationState = 1 (Add_InFlight) with concurrency check          │
│                                                                             │
│   9. CALL TCC:                                                              │
│      - Port-In: tcc.NumberMNP(...)                                          │
│      - New SIM: tcc.AddNumber(...)                                          │
│                                                                             │
│   10. If TCC Success (code = 600):                                          │
│       └─> SET ActivationState = 2 (Added)                                   │
│       └─> SET ActivationState = 3 (CRM_InFlight)                            │
│                                                                             │
│   11. CALL BSS CreateSalesOrder:                                            │
│       - With Device: CreateSalesOrderWithDevice()                           │
│       - Without Device: CreateSalesOrder()                                  │
│       - Port-In: CreateSalesMNPOrder()                                      │
│                                                                             │
│   12. DEDUCT FROM WALLET:                                                   │
│       - WalletBalanceBefore = seller.WalletBalance                          │
│       - seller.WalletBalance -= OrderTotal                                  │
│       - WalletBalanceAfter = seller.WalletBalance                           │
│                                                                             │
│   13. Update Status = Completed                                             │
│                                                                             │
│   14. Generate ZATCA eInvoice (for applicable channels)                     │
│                                                                             │
│   15. Process ExtraSIMs activation                                          │
│                                                                             │
│   16. Return completed order                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Complete Flow Diagram

### 6.1 High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    NUMBER RESERVATION & ACTIVATION FLOW                      │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
    │ VALIDATE │────▶│   GET    │────▶│  SELECT  │────▶│  SELECT  │
    │   ID     │     │  NUMBERS │     │  NUMBER  │     │   PLAN   │
    └──────────┘     └──────────┘     └──────────┘     └──────────┘
         │                                                   │
         │  TCC Check                                        │
         │  BSS Customer                                     ▼
         │                                            ┌──────────┐
         │                                            │AUTHENTI- │
         │                                            │  CATE    │
         │                                            └────┬─────┘
         │                                                 │
         │                                                 │  TCC AddNumber
         │                                                 │  BSS CreateOrder
         │                                                 │  Wallet Deduct
         │                                                 ▼
         │                                            ┌──────────┐
         └────────────────────────────────────────────│ COMPLETE │
                                                      └──────────┘
```

### 6.2 Sequence Diagram

```
┌────────┐     ┌─────────────┐     ┌────────┐     ┌────────┐     ┌────────┐
│ Client │     │SalesAPICtrl │     │  BSS   │     │  TCC   │     │Database│
└───┬────┘     └──────┬──────┘     └───┬────┘     └───┬────┘     └───┬────┘
    │                 │                │              │               │
    │ ValidateIdNew   │                │              │               │
    │────────────────▶│                │              │               │
    │                 │ GetCustomerInfo│              │               │
    │                 │───────────────▶│              │               │
    │                 │◀───────────────│              │               │
    │                 │                │              │               │
    │                 │ CheckEligibility              │               │
    │                 │───────────────────────────────▶               │
    │                 │◀───────────────────────────────               │
    │                 │                │              │               │
    │                 │ Save SalesOrder│              │               │
    │                 │───────────────────────────────────────────────▶
    │◀────────────────│                │              │               │
    │                 │                │              │               │
    │ GetMSISDNs      │                │              │               │
    │────────────────▶│                │              │               │
    │                 │ QueryAvailable │              │               │
    │                 │───────────────▶│              │               │
    │                 │◀───────────────│              │               │
    │◀────────────────│                │              │               │
    │                 │                │              │               │
    │ UpdateOrder     │                │              │               │
    │ (select-number) │                │              │               │
    │────────────────▶│                │              │               │
    │                 │ OperateMSISDN  │              │               │
    │                 │ (RESERVE)      │              │               │
    │                 │───────────────▶│              │               │
    │                 │◀───────────────│              │               │
    │◀────────────────│                │              │               │
    │                 │                │              │               │
    │ UpdateOrder     │                │              │               │
    │ (select-plan)   │                │              │               │
    │────────────────▶│                │              │               │
    │◀────────────────│                │              │               │
    │                 │                │              │               │
    │ UpdateOrder     │                │              │               │
    │ (authenticate)  │                │              │               │
    │────────────────▶│                │              │               │
    │                 │ GetSIMDetails  │              │               │
    │                 │───────────────▶│              │               │
    │                 │◀───────────────│              │               │
    │                 │                │              │               │
    │                 │ AddNumber      │              │               │
    │                 │───────────────────────────────▶               │
    │                 │◀───────────────────────────────               │
    │                 │                │              │               │
    │                 │ CreateSaleOrder│              │               │
    │                 │───────────────▶│              │               │
    │                 │◀───────────────│              │               │
    │                 │                │              │               │
    │                 │ Deduct Wallet  │              │               │
    │                 │───────────────────────────────────────────────▶
    │◀────────────────│                │              │               │
    │                 │                │              │               │
```

---

## 7. BSS API Integration

### 7.1 GetAvailableMSISDNs

| Property | Value |
|----------|-------|
| **SOAP Action** | `QueryAvailableNumber` |
| **Endpoint** | `/apiaccess/InventoryService/InventoryService` |
| **Source** | BssApiHelper.cs:3298-3339 |

**SOAP Request Structure:**
```xml
<soapenv:Envelope>
  <soapenv:Body>
    <inv:QueryAvailableNumberReqMsg>
      <com:ReqHeader>
        <com:Version>1</com:Version>
        <com:TransactionId>{timestamp}</com:TransactionId>
        <com:Channel>{BSSChannel}</com:Channel>
        <com:PartnerId>101</com:PartnerId>
        <com:AccessUser>{BSSUser}</com:AccessUser>
        <com:AccessPassword>{encrypted}</com:AccessPassword>
        <com:OperatorId>{SellerUserName}</com:OperatorId>
      </com:ReqHeader>
      <inv:ResType>10</inv:ResType>
      <inv:Level>{vanity}</inv:Level>
      <inv:FuzzyCode>{filter}</inv:FuzzyCode>
      <inv:RecordNum>{count}</inv:RecordNum>
    </inv:QueryAvailableNumberReqMsg>
  </soapenv:Body>
</soapenv:Envelope>
```

**Response Fields:**
- `ServiceNumber`: Phone number (5xxxxxxxx format)
- `ResDeptId`: Resource department ID
- `Level`: Vanity level (1-5)

---

### 7.2 OperateMSISDN (Reserve Number)

| Property | Value |
|----------|-------|
| **SOAP Action** | `OperateMsisdn` |
| **Endpoint** | `/apiaccess/InventoryService/InventoryService` |
| **Source** | BssApiHelper.cs:3044-3076 |
| **Operation Type** | `1029` (Reserve/Pick) |

**SOAP Request Structure:**
```xml
<soapenv:Envelope>
  <soapenv:Body>
    <inv:OperateMsisdnReqMsg>
      <com:ReqHeader>
        <com:Version>1</com:Version>
        <com:TransactionId>{timestamp}</com:TransactionId>
        <com:Channel>{BSSChannel}</com:Channel>
        <com:PartnerId>101</com:PartnerId>
        <com:AccessUser>{BSSUser}</com:AccessUser>
        <com:AccessPassword>{encrypted}</com:AccessPassword>
        <com:OperatorId>{SellerUserName}</com:OperatorId>
      </com:ReqHeader>
      <inv:ResType>10</inv:ResType>
      <inv:OperType>1029</inv:OperType>
      <inv:ResCodeList>
        <inv:ResCode>{MSISDN}</inv:ResCode>
      </inv:ResCodeList>
    </inv:OperateMsisdnReqMsg>
  </soapenv:Body>
</soapenv:Envelope>
```

**Error Handling:**
- HTTP != 200: `"Failed to Reserve MSISDN, Connection to CRM failed"`
- ReturnCode != 0: `"Cannot Pick MSISDN"`

---

### 7.3 CreateSalesOrder (Final Activation)

| Property | Value |
|----------|-------|
| **SOAP Action** | `CreateSaleOrder` |
| **Endpoint** | `/apiaccess/OrderServices/OrderServices` |
| **Source** | BssApiHelper.cs:3468+ |

This API is called during the `authenticate` step to finalize the activation in BSS/CRM.

---

## 8. TCC API Integration

### 8.1 CheckEligibility

Called during `ValidateIdNew` to verify customer can register a new SIM.

**Parameters:**
- IdNumber
- Nationality
- IdType
- SubType (0 = Standard, 1 = Alternative)
- Seller info

**Response Codes:**
| Code | Meaning |
|------|---------|
| 600 | Success - Customer eligible |
| 605 | Customer not eligible (try alternative SubType) |
| Other | Error - Show message to user |

---

### 8.2 AddNumber

Called during `authenticate` step for new SIM activations.

**Parameters:**
- User, Seller, Order
- TCC Region (from province)
- Source Type (from channel)
- ExtraSIMs list

**Response:**
```json
{
    "person": {
        "first": "Mohammed",
        "father": "Ali",
        "family": "Ahmed",
        "gender": "M",
        "birthdate": "1990-01-01",
        "borderNumber": "1234567890"
    },
    "tcn": "transaction-number",
    "code": 600,
    "message": "success"
}
```

---

### 8.3 NumberMNP

Called during `authenticate` step for port-in (MNP) requests.

Same structure as AddNumber but handles number transfer from another operator.

---

## 9. Activation State Machine

### 9.1 State Definitions

| State | Value | Description |
|-------|-------|-------------|
| Pending | 0 (null) | Initial state, no activation started |
| Add_InFlight | 1 | TCC API call in progress |
| Added | 2 | TCC registration successful |
| CRM_InFlight | 3 | BSS CreateSalesOrder in progress |
| Completed | 4 | Activation fully complete |

### 9.2 State Transition Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ACTIVATION STATE MACHINE                            │
└─────────────────────────────────────────────────────────────────────────────┘

     ┌─────────┐                                      ┌─────────┐
     │ Pending │──────────────────────────────────────│ Failed  │
     │  (0)    │                                      │         │
     └────┬────┘                                      └─────────┘
          │                                                ▲
          │ authenticate step                              │
          │ (optimistic lock)                              │ TCC/BSS Error
          ▼                                                │
     ┌─────────────┐                                       │
     │ Add_InFlight │──────────────────────────────────────┘
     │     (1)      │
     └──────┬───────┘
            │
            │ TCC AddNumber/NumberMNP
            │ (code = 600)
            ▼
     ┌─────────────┐
     │   Added     │
     │    (2)      │
     └──────┬──────┘
            │
            │ Start BSS call
            ▼
     ┌─────────────┐
     │ CRM_InFlight│
     │     (3)     │
     └──────┬──────┘
            │
            │ BSS CreateSalesOrder success
            │ Wallet deducted
            ▼
     ┌─────────────┐
     │  Completed  │
     │    (4)      │
     └─────────────┘
```

### 9.3 Concurrency Control

The system uses **optimistic concurrency** with `RowVersion` to prevent duplicate activations:

```csharp
// Source: SalesAPIController.cs:1509-1523
ord.ActivationState = 1; // Add_InFlight
db.Entry(ord).Property(o => o.ActivationState).IsModified = true;
db.Entry(ord).Property(o => o.RowVersion).OriginalValue = ord.RowVersion;
try
{
    await db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException)
{
    return BadRequest("Order already in progress, please wait until it is completed.");
}
```

---

## 10. Configuration

### 10.1 appsettings.json

```json
{
    "ApiSettings": {
        "BSSChannel": "72",
        "BSSUser": "salesApp",
        "BSSURL": "https://bss.redbullmobile.sa"
    },
    "AppSettings": {
        "BypassTCC": "0",
        "BypassBSS": "0",
        "DisableActivation": "0"
    }
}
```

### 10.2 Configuration Keys

| Key | Description | Used In |
|-----|-------------|---------|
| `ApiSettings:BSSChannel` | BSS channel ID | All BSS calls |
| `ApiSettings:BSSUser` | BSS username | All BSS calls |
| `AppSettings:BypassTCC` | Skip TCC calls (dev) | ValidateIdNew, Authenticate |
| `AppSettings:BypassBSS` | Skip BSS calls (dev) | All BSS calls |
| `AppSettings:DisableActivation` | Disable activation service | ValidateIdNew |

---

## 11. Key Business Rules

### 11.1 Vanity Level Pricing

| Level | Description | Price Range |
|-------|-------------|-------------|
| 1 | Reserved (not sellable) | N/A |
| 2 | Premium | High |
| 3 | Standard | Medium |
| 4 | Basic | Low |
| 5 | Reserved (not sellable) | N/A |

Only levels 2, 3, 4 are available for sale. Levels 1 and 5 are filtered out.

### 11.2 MSISDN Format Normalization

All MSISDNs are normalized to `5xxxxxxxx` format:

```csharp
Regex.Replace(param.MSISDN.Value, @"^(05|\+9665|5|009665|9665)", "5")
```

| Input | Output |
|-------|--------|
| `0551234567` | `551234567` |
| `+9665512345` | `5512345` |
| `9665512345` | `5512345` |
| `551234567` | `551234567` |

### 11.3 VAT Calculation

- **VAT Rate:** 15%
- **Prepaid:** VAT included in prices
- **Postpaid:** VAT not included (billed separately)

```csharp
// Prepaid
PlanCost = Plan.Price * (1 + 0.15);
MSISDNCost = MSISDNCost * (1 + 0.15);

// Postpaid
PlanCost = Plan.Price;
MSISDNCost = MSISDNCost * (1 + 0.15); // Number cost still has VAT
```

### 11.4 Wallet Validation

Wallet balance is validated at multiple steps:
1. `select-number` - After calculating number cost
2. `select-plan` - After adding plan and addons
3. `authenticate` - Final validation before deduction

### 11.5 POS Restrictions

POS sellers (SellerChannel = 1) cannot sell postpaid products:

```csharp
if ((sellerType == 1) && (isPostpaid))
    return BadRequest("POS are not allowed to sell Postpaid");
```

---

## 12. Error Handling

### 12.1 Common Error Scenarios

| Scenario | Error Message | Step |
|----------|---------------|------|
| Blacklisted customer | `"Sorry, this Person is BlackListed"` | ValidateIdNew |
| TCC eligibility failed | `"TCC{code}: {message}"` | ValidateIdNew |
| Insufficient balance | `"Insufficient Wallet Balance"` | select-number, select-plan, authenticate |
| Invalid ICCID | `"ICCID Is Invalid"` | authenticate |
| Duplicate activation | `"Order already in progress"` | authenticate |
| BSS connection failed | `"Failed to Reserve MSISDN, Connection to CRM failed"` | select-number |
| Cannot pick number | `"Cannot Pick MSISDN"` | select-number |
| Service disabled | `"Service is not available now"` | ValidateIdNew |

### 12.2 Recovery Mechanisms

1. **Activation Recovery:** System supports recovering failed activations via `IsActivationRecovery` flag
2. **Order Cloning:** When MSISDN changes, order is cloned to preserve history
3. **Idempotency:** ActivationState prevents duplicate processing

---

## Appendix A: ICCID Format

ICCID (Integrated Circuit Card Identifier) format for RedBull Mobile:

```
899661xxxxxxxxxxxxxx
│││││└──────────────── 13-14 digit serial
│││││
└┴┴┴┴───────────────── Fixed prefix: 899661 (Saudi Arabia)
```

Validation regex: `^899661\d{13,14}$`

---

## Appendix B: Phone Number Formats

| Format | Example | Region |
|--------|---------|--------|
| Local (05) | 0551234567 | Saudi Arabia |
| International (+966) | +966551234567 | Saudi Arabia |
| Without prefix | 551234567 | Internal format |

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-19 | Generated from Source Code | Initial version |

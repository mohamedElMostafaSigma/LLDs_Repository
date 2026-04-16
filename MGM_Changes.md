# MGM (Member Get Member) Feature - Complete Documentation
## For Java Microservices Re-Implementation

---

## 1. Git History

**Primary Commits (across branches, merged to preprod):**
- `40b5c4244` — feat(MGM): Implement Member Get Member with Reward Wallet
- `14afce651` — GetReferalCode-Fix-preprod (PR #2614)
- `a1d6f7fa9` — P3-ReferralCode-CR (PR #2613)
- `06d0a30c4` — Fix: MaxReferrals check
- `b6db1df06` — P3-ReferralCode-CR (PR #2612)
- `1953463ba` — P3-ReferralCode-CR (PR #2611)
- `8f85f4069` — P3-ReferralCode-CR (PR #2610)
- `d161af6c5` — Add InvalidMGMServiceNumber + CustomerAndRefferalSubscriberAreSame exceptions

---

## 2. Database Schema

### 2.1 `ReferralCodeLog` — Event log per referral
| Column | Type | Purpose |
|--------|------|---------|
| Id | int (PK) | Auto-increment |
| RefferalCode | string | The code applied (e.g., "RB1a2B3c") |
| AddedDate | datetime | When event occurred |
| ClientId | int | Master/Referrer ClientId |
| Msisdn | string | Master/Referrer phone |
| ReferredClientId | int | Referee ClientId |
| ReferredClientMsisdn | string | Referee phone |
| OfferId | string | Primary plan offering ID |
| ReferredPrice | decimal | Full plan price |
| ReferredDiscountAmount | decimal | Price after discount |
| BonusBalance | decimal | Bonus for the referrer |
| Status | enum | Applied / Redeemed / Paid / Activated / Canceled |
| OrderId | int | Linked order ID |

**Migration:** `M202408280244_AddReferralCodeLogTable.cs`

### 2.2 `ReferralCodeBonus` — Bonus per tier (max 5)
| Column | Type | Purpose |
|--------|------|---------|
| Id | int (PK) | Auto-increment |
| MemberNum | int | 1, 2, 3, 4, 5 |
| BonusAmount | decimal | Reward for that tier |

**Migration:** `M202408280401_AddReferralCodeBonusTable.cs`

### 2.3 `ReferralCodeOffers` — Default discount config
| Column | Type | Purpose |
|--------|------|---------|
| Id | int (PK) | Auto-increment |
| DiscountPercentage | int | e.g., 20 |
| OfferId | string | Huawei secondary offering ID |
| OfferName | string | Display name |
| PaymentType | enum | Prepaid / Postpaid |
| Validity | int | Days |
| IsDefault | bool | Use as fallback |

**Migrations:** Initial + `M202501151315_UpdateReferralCodeOffersTable` (added Validity)

### 2.4 `ReferralCodeTerms` — Multilingual T&Cs
| Column | Type | Purpose |
|--------|------|---------|
| Id | int (PK) | Auto-increment |
| TermsAr | string | Arabic T&Cs |
| TermsEn | string | English T&Cs |

**Migration:** `M202409040755_AddReferralCodeTermsTable.cs`

### 2.5 `ClientReferralCodeInfo` — Per-customer custom discount
| Column | Type | Purpose |
|--------|------|---------|
| Id | int (PK) | Auto-increment |
| AddedDate | datetime | Created |
| ClientId | int | Referrer client |
| Msisdn | string | Referrer phone (indexed) |
| MsisdnId | int | FK to ClientMsisdn |
| PrimaryOfferingId | string? | NULL = applies to all plans |
| PrimaryPlanId | int? | Optional |
| DiscountOfferId | string | Huawei secondary offering ID |
| DiscountPercentage | int | Custom % |

**Migration:** `M202411050244_AddClientReferralCodeInfoTable.cs`

### 2.6 `ClientMsisdn` (modified)
Added `referralcode` column — stores each customer's personal code.

### 2.7 Enum: `ReferralCodeStatus`
```
Applied = 1     // Code applied in cart/order
Redeemed = 2    // Code accepted in order
Paid = 3        // Payment processed
Activated = 4   // SIM activation done — referral complete
Canceled = 5    // Canceled before activation
```

State machine: `Applied -> Redeemed -> Paid -> Activated` (or `Canceled` at any step)

---

## 3. API Endpoints

Base route: `api/mobile/v1/ReferralCode`

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/ReferrEligibility?Msisdn=...` | Check if customer can be referrer (must be subscribed >= 30 days) |
| POST | `/ApplyReferralCode` | Apply code to order, calculate discount |
| POST | `/CancelReferralCode` | Cancel an applied code |
| GET | `/MembersDetails` | List referrer's 5 tiers + active status + bonus per tier |
| GET | `/GetReferalCode?Msisdn=...` | Generate or fetch existing code for customer |
| GET | `/GetTerms` | Fetch localized T&Cs |
| POST | `/` (Excel upload) | Bulk upload custom client referral configs |

### 3.1 Apply Referral Code Request/Response

**Request (`ReferralCodeReq`):**
```json
{
    "Code": "RB1a2B3c",
    "PhoneNumber": "966576000146",
    "PrimaryOfferingId": "PLAN_ID",
    "IsMNP": false,
    "ExtraSimCount": 0,
    "CommitmentId": null
}
```

**Response (`ReferralCodeDTO` extends `OrderSummaryResponse`):**
```json
{
    "discount": 76.00,           // Discounted price
    "percentage": 20,            // Discount %
    "isPromoCode": false,        // Always false for referrals
    "DepositText": "...",
    "PriceToPay": 76.00,
    "ReceiptFields": [...],
    "Totals": {...}
}
```

### 3.2 Members Details Response
```json
{
    "Members": [
        { "MemberNum": "1st joiner", "IsActive": true, "BonusBalance": 100 },
        { "MemberNum": "2nd joiner", "IsActive": true, "BonusBalance": 75 },
        { "MemberNum": "3rd joiner", "IsActive": false, "BonusBalance": 50 },
        { "MemberNum": "4th joiner", "IsActive": false, "BonusBalance": 25 },
        { "MemberNum": "5th joiner", "IsActive": false, "BonusBalance": 10 }
    ],
    "DiscountPercentage": 20
}
```

---

## 4. Business Rules

### 4.1 Referrer Eligibility
- Must have an active subscription
- Subscription >= 30 days old (`SubscriptionDate <= UtcNow.AddMonths(-1)`)
- If `SubscriptionDate` not in DB, fetch from BSS via `QuerySubscriberAllInfo` (use `EffectiveDate` from response)

### 4.2 Code Application Validation
1. Code must exist in DB
2. Referee MSISDN cannot be Data SIM (cannot start with `'8'`)
3. Max 5 active referrals per code (after 5th `Activated`, throw `ReferralCodeExpired`)
4. Code expires 1 year from `ReleaseDate` (configurable, baseline date for the program)
5. Referrer cannot refer themselves (validated by BSS — error code `1211004538`)

### 4.3 Discount Determination Priority
1. **Custom config exists** in `ClientReferralCodeInfo` for referrer's MSISDN:
   - If 1 record: use that discount
   - If multiple: match `PrimaryOfferingId`, use that discount
   - If no match: throw `NotApplicableToOffer`
2. **Fallback** to `ReferralCodeOffers` matching referrer's `PaymentType` where `IsDefault = true`

### 4.4 Code Generation Algorithm
```
Format: {Prefix}{8-char-mixed-case-UUID}
Example: "RB1a2B3c"

Steps:
1. Take first 8 chars of GUID
2. Randomize case (upper/lower) for each char
3. Prepend prefix from config (e.g., "RB")
4. Check uniqueness in DB
5. Retry if duplicate
6. Store in ClientMsisdn.referralcode
```

### 4.5 Bonus Tiers
- Referrer earns bonus per joiner up to 5 joiners
- Bonus amount per tier comes from `ReferralCodeBonus` table
- After 5th `Activated`, code is "exhausted" for new referrals

---

## 5. BSS (Huawei) Integration

### 5.1 MGM Object in CreateSalesOrder
```csharp
public class MGM {
    public string MgmServiceNumber { get; set; }   // Referrer MSISDN
    public string RewardItemId { get; set; }       // Prepaid OR Postpaid reward ID
}
```

### 5.2 SOAP Message Fragment
```xml
<ord:MGM>
    <com:PromotionId>500607</com:PromotionId>
    <com:MgmServiceNumber>966576000146</com:MgmServiceNumber>
    <com:RewardItemIdList>
        <com:RewardItemId>12345</com:RewardItemId>
    </com:RewardItemIdList>
</ord:MGM>
```

**Constants:**
- `PromotionId` = `500607` (MGM promotion in Huawei — fixed)
- `RewardItemId` = either `MGMConfig.PrepaidRewardItemId` or `MGMConfig.PostpaidRewardItemId` based on referrer's payment type

### 5.3 Huawei Error Codes
| Code | Meaning |
|------|---------|
| `1211010005` | Invalid MGM service number (referrer not found) |
| `1211004538` | Customer and referral subscriber are the same (self-referral blocked) |

---

## 6. Configuration

```yaml
referral:
  prefix: "RB"                    # Code prefix
  maxMembers: 5                   # Max referrals per code
  eligibilityDaysSubscribed: 30   # Min subscription age to be referrer
  expiryYears: 1                  # Code valid for 1 year from releaseDate
  releaseDate: "20210101000000"   # Program baseline date

mgm:
  promotionId: "500607"           # Huawei MGM promotion ID (fixed)
  prepaidRewardItemId: "12345"    # Reward item for prepaid referrer
  postpaidRewardItemId: "12346"   # Reward item for postpaid referrer

discounts:
  prepaidDefault: 20              # Default % for prepaid
  postpaidDefault: 15             # Default % for postpaid
```

---

## 7. Data Flows

### 7.1 Generate Code
```
Customer A
  -> GET /GetReferalCode?Msisdn=966...
  -> Service generates unique "RB1a2B3c"
  -> Store in ClientMsisdn.referralcode
  -> Return code
```

### 7.2 Apply Code & Activate
```
Customer B
  -> POST /ApplyReferralCode { Code, PhoneNumber, PrimaryOfferingId }
  -> Service validates: code exists, not data sim, < 5 referrals, not expired
  -> Determines discount % (custom config OR default)
  -> Creates ReferralCodeLog (Status=Applied)
  -> Returns order summary with discount
  
Customer B completes payment & SIM activation
  -> OnboardingService.ActivateSimAsync()
  -> CreateSalesOrder includes <MGM> block to BSS
  -> BSS applies reward to referrer's account
  -> ReferralCodeLog updated to Status=Activated
```

### 7.3 Track Earnings
```
Customer A
  -> GET /MembersDetails
  -> Service queries Activated logs for "RB1a2B3c"
  -> Returns 5 tiers with active status + bonus per tier
```

---

## 8. Java Microservices Implementation Plan

### 8.1 Suggested Microservice: `referral-service`

**Entities:**
- `ReferralCode` (msisdn, code, createdDate)
- `ReferralCodeLog` (full event log)
- `ReferralCodeBonus` (memberNum, bonusAmount)
- `ReferralCodeOffer` (paymentType, discountPct, offerId, isDefault, validity)
- `ReferralCodeTerms` (termsEn, termsAr)
- `ClientReferralConfig` (per-customer custom discount)

**Services:**
1. **ReferralCodeService**
   - generateCode(msisdn) -> String
   - getCode(msisdn) -> String
   - applyCode(req) -> ReferralCodeDTO
   - cancelCode(req) -> ReferralCodeDTO

2. **ReferralEligibilityService**
   - isEligibleToRefer(msisdn) -> boolean
   - validateApplyCode(req) — throws on violation

3. **ReferralRewardService**
   - calculateBonus(memberNum) -> BigDecimal
   - getMemberDetails(msisdn) -> MembersDetailsResponse

4. **ReferralDiscountService**
   - getDiscountPercentage(referrerMsisdn, primaryOfferingId) -> int
   - getDiscountOfferId(referrerMsisdn, primaryOfferingId, paymentType) -> String

5. **BssMgmIntegrationService**
   - buildMgmPayload(referrerMsisdn, paymentType) -> MGMDto
   - (used by Sales Order service when activating SIM)

### 8.2 REST API
```
POST   /api/v1/referral/generate          # Generate code for customer
GET    /api/v1/referral/code/{msisdn}     # Get existing code
GET    /api/v1/referral/eligibility/{msisdn}  # Check referrer eligibility
POST   /api/v1/referral/apply             # Apply code to order
POST   /api/v1/referral/cancel            # Cancel applied code
GET    /api/v1/referral/members/{msisdn}  # Get earnings/tiers
GET    /api/v1/referral/terms             # Get T&Cs
POST   /api/v1/referral/admin/upload      # Bulk upload custom configs
```

### 8.3 Inter-Service Communication
- **Customer Service** — for client info, MSISDN lookup
- **Subscription Service** — for SubscriptionDate, Plan info
- **BSS Adapter Service** — wraps Huawei calls (CreateSalesOrder with MGM)
- **Notification Service** — for SMS/email when referral activated

### 8.4 Caching Strategy
Cache for 1-7 days (matching original):
- Primary plans list
- ClientReferralCodeInfo (custom configs)
- ReferralCodeBonus (rarely changes)
- ReferralCodeOffers (rarely changes)

### 8.5 Database Schema (PostgreSQL/MySQL)
```sql
CREATE TABLE referral_codes (
    id BIGSERIAL PRIMARY KEY,
    msisdn VARCHAR(20) UNIQUE NOT NULL,
    code VARCHAR(50) UNIQUE NOT NULL,
    created_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_code (code)
);

CREATE TABLE referral_logs (
    id BIGSERIAL PRIMARY KEY,
    referral_code VARCHAR(50) NOT NULL,
    referrer_client_id BIGINT NOT NULL,
    referrer_msisdn VARCHAR(20) NOT NULL,
    referee_client_id BIGINT,
    referee_msisdn VARCHAR(20) NOT NULL,
    offer_id VARCHAR(50),
    referred_price DECIMAL(10,2),
    referred_discount_amount DECIMAL(10,2),
    bonus_balance DECIMAL(10,2),
    status SMALLINT NOT NULL,  -- 1-5
    order_id BIGINT,
    added_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_code_status (referral_code, status),
    INDEX idx_referee (referee_msisdn)
);

CREATE TABLE referral_bonuses (
    id SERIAL PRIMARY KEY,
    member_num SMALLINT UNIQUE NOT NULL,  -- 1-5
    bonus_amount DECIMAL(10,2) NOT NULL
);

CREATE TABLE referral_offers (
    id SERIAL PRIMARY KEY,
    discount_percentage INT NOT NULL,
    offer_id VARCHAR(50) NOT NULL,
    offer_name VARCHAR(200),
    payment_type SMALLINT NOT NULL,  -- 1=Prepaid, 2=Postpaid
    validity_days INT,
    is_default BOOLEAN DEFAULT FALSE
);

CREATE TABLE referral_terms (
    id SERIAL PRIMARY KEY,
    terms_en TEXT,
    terms_ar TEXT
);

CREATE TABLE client_referral_config (
    id BIGSERIAL PRIMARY KEY,
    client_id BIGINT NOT NULL,
    msisdn VARCHAR(20) NOT NULL,
    primary_offering_id VARCHAR(50),  -- NULL = all plans
    discount_offer_id VARCHAR(50) NOT NULL,
    discount_percentage INT NOT NULL,
    added_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_msisdn (msisdn)
);
```

---

## 9. Edge Cases & Notes

| Scenario | Handling |
|----------|----------|
| Customer applies code on Data SIM | Throw `DataSimTypeNotAllowed` |
| 6th customer tries same code | Throw `ReferralCodeExpired` |
| Code older than 1 year | Throw `ReferralCodeExpired` |
| Custom config exists but no PrimaryOfferingId match | Throw `NotApplicableToOffer` |
| No SubscriptionDate in DB | Fetch from BSS, persist, then check eligibility |
| Self-referral | BSS returns `1211004538`, throw mapped exception |
| Invalid referrer MSISDN | BSS returns `1211010005`, throw mapped exception |
| Customer cancels before activation | New log with `Canceled` status; previous `Applied` stays for audit |

---

## 10. Files to Reference (.NET source)

| File | Purpose |
|------|---------|
| `RedbullKSA.Entities/Database/Models/ReferralCodeLog.cs` | Log entity |
| `RedbullKSA.Entities/Database/Models/ReferralCodeBonus.cs` | Bonus tier |
| `RedbullKSA.Entities/Database/Models/ReferralCodeOffers.cs` | Default offers |
| `RedbullKSA.Entities/Database/Models/ReferralCodeTerms.cs` | T&Cs |
| `RedbullKSA.Entities/Database/Models/ClientReferralCodeInfo.cs` | Custom configs |
| `RedbullKSA.Entities/Enums/ReferralCodeStatus.cs` | Status enum |
| `RedbullKSA.Services/Services/ReferralCodeLogService.cs` | Main business logic |
| `RedbullKSA.Services/Services/ClientReferralCodeInfoService.cs` | Custom config service |
| `RedbullKSA.Services/Services/ReferralCodeTermsService.cs` | T&Cs service |
| `RedbullKSA.Services/Services/UtilityHuaweiService.cs` | `PrepareMGMSecondaryPlanRequest` |
| `RedbullKSA.Services/Services/OnboardingService.cs` | MGM in CreateSalesOrder (line ~316) |
| `RedbullKSA.Services/Helpers/HuaweiMessageBuilder.cs` | SOAP MGM block (line ~908) |
| `RedbullKSA.Services/Base/BaseHuaweiService.cs` | Error code mapping |
| `RedbullKSA.API/Controllers/Mobile/v1/MobileReferralCodeController.cs` | API endpoints |
| `RedbullKSA.Entities/API/Models/HuaweiModels/Common/Subscriber.cs` | MGM DTO |
| `RedbullKSA.Common/Configuration/ConfigurationSettings.cs` | MGMConfig + GeneralConfig |

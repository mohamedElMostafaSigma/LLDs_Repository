# Test Report: SalesApp JSON Migration
**Date:** 07-04-2026
**Tester:** Mohamed Mostafa
**Branch:** `preprod-MoveToJsonFormat`
**Commit:** `d153eb722`

---

## 1. Change Summary

The SalesApp team changed the response format of the `EligibleAddons` API from **XML/SOAP** to a **flat JSON array**.

### Files Changed

| File | Change |
|------|--------|
| `RedbullKSA.Services/Base/BaseHuaweiService.cs` | Added `PostJsonReceiveJson<T>` — generic method to send JSON and receive JSON (instead of XML/SOAP) |
| `RedbullKSA.Entities/API/Models/HuaweiModels/SalesAppSupplementaryOfferingResponse.cs` | New DTO matching the new SalesApp JSON response structure |
| `RedbullKSA.Services/Services/OfferingHuaweiService.cs` | Updated 3 methods + added 2 mapping helpers |

### Methods Updated

| Method | What Changed |
|--------|-------------|
| `QueryAvailableSupplementaryOfferings` | `PostJson` (XML) -> `PostJsonReceiveJson` (JSON) |
| `QueryAllSupplementaryOfferingsInternally` | `PostJson` (XML) -> `PostJsonReceiveJson` (JSON) |
| `QuerySupplementaryOffering` | `PostJson` (XML) -> `PostJsonReceiveJson` (JSON) |

### Methods NOT Affected

| Method | Why |
|--------|-----|
| `ChangeSupplementaryOffering` | Calls Huawei BSS directly (SOAP), not SalesApp |
| `CancelSupplementaryOffering` | Calls Huawei BSS directly (SOAP), not SalesApp |
| `QueryPurchasedSupplementaryOfferings` | Calls Huawei BSS directly (SOAP), not SalesApp |
| `ChangePrimaryOffering` | Calls Huawei BSS directly (SOAP), not SalesApp |
| All other Huawei BSS operations | No change — only SalesApp `EligibleAddons` endpoint was affected |

---

## 2. Test Approach

### 2.1 Test Framework
- **xUnit** 2.4.1
- **Moq** 4.16.1 for dependency mocking
- **RichardSzalay.MockHttp** 6.0.0 for HTTP response mocking
- **Project:** `RedbullKSA.Services.Tests`

### 2.2 Test Strategy
Unit tests were written to validate:
1. JSON deserialization from the new SalesApp format
2. Correct mapping from `SalesAppSupplementaryOfferingResponse` to existing DTOs (`AvailableOfferingDTO`, `SupplementaryOfferingDTO`)
3. Edge cases: empty arrays, null values, non-standard statuses, unicode content
4. Error handling: offering not found, empty responses

### 2.3 Mocking Setup
- `MockHttpMessageHandler` simulates SalesApp HTTP responses with predefined JSON strings
- `IHttpClientFactory` mocked to return the mock HTTP handler
- All `OfferingHuaweiService` dependencies mocked via `IServiceProvider` (same pattern as existing `TroubleTicketsHuaweiServiceTests.cs`)
- `IoutgoingLogEntryService` mocked to avoid DB writes during tests

---

## 3. Test Cases & Results

### 3.1 QueryAvailableSupplementaryOfferings (5 tests)

| # | Test Case | Input | Expected | Result |
|---|-----------|-------|----------|--------|
| 1 | Valid JSON response with 2 offerings | `[{Id:"1330977724", NameEn:"Unlimited Data 90 Days", OfferingStatus:"R", MonthlyCost:975.00, OneTimeCost:0.00}, {Id:"1430977666", ...}]` | Returns 2 mapped `AvailableOfferingDTO` with correct Id, Name, Status=Release, prices | **PASSED** |
| 2 | Empty JSON array | `[]` | Returns empty list, no exception | **PASSED** |
| 3 | Non-Release status ("S") | `[{OfferingStatus:"S", ...}]` | Maps to `OfferingStatus.Suspend` | **PASSED** |
| 4 | Null ExpireDate | `[{ExpireDate:null, ...}]` | Maps to `DateTime.MaxValue` | **PASSED** |
| 5 | Zero prices | `[{MonthlyCost:0.00, OneTimeCost:0.00}]` | Maps correctly as 0.00m | **PASSED** |

### 3.2 QuerySupplementaryOffering (3 tests)

| # | Test Case | Input | Expected | Result |
|---|-----------|-------|----------|--------|
| 6 | Offering exists in response | OfferingId = "1330977724" | Returns mapped `SupplementaryOfferingDTO` with correct fields | **PASSED** |
| 7 | Offering not found in response | OfferingId = "NON_EXISTENT_ID" | Throws `KSAException` (SupplementaryOffering.NotFound) | **PASSED** |
| 8 | Empty response array | OfferingId = "1330977724", response = `[]` | Throws `KSAException` (SupplementaryOffering.NotFound) | **PASSED** |

### 3.3 QueryAllSupplementaryOfferings (2 tests)

| # | Test Case | Input | Expected | Result |
|---|-----------|-------|----------|--------|
| 9 | Valid response with 2 offerings | `[{Id:"1330977724",...}, {Id:"1430977666",...}]` | Returns 2 mapped `SupplementaryOfferingDTO` | **PASSED** |
| 10 | Empty response array | `[]` | Returns empty list, no exception | **PASSED** |

### 3.4 Edge Cases (2 tests)

| # | Test Case | Input | Expected | Result |
|---|-----------|-------|----------|--------|
| 11 | Arabic names (Unicode) | `{NameAr:"داتا غير محدودة 90 يوم", NameEn:"Unlimited Data 90 Days"}` | NameEn mapped correctly, no encoding issues | **PASSED** |
| 12 | Single item response | `[{Id:"333", MonthlyCost:50.00, OneTimeCost:10.00}]` | Returns single offering with correct prices | **PASSED** |

---

## 4. Test Execution Output

```
Test Run Successful.
Total tests: 12
     Passed: 12
 Total time: 3.6798 Seconds
```

---

## 5. Build Verification

| Project | Result |
|---------|--------|
| `RedbullKSA.Services.csproj` (core changes) | Build succeeded — 0 Errors |
| `RedbullKSA.Services.Tests.csproj` (tests) | Build succeeded — 0 Errors |

---

## 6. Mapping Validation

### Old Response Format (XML/SOAP — before)
```xml
<SupplementaryOffering>
    <OfferingId><OfferingId>1330977724</OfferingId></OfferingId>
    <OfferingName>Unlimited Data 90 Days</OfferingName>
    <OfferingStatus>2</OfferingStatus>
    <ExpireDate>20371231000000</ExpireDate>
    <MonthlyCost>975.00</MonthlyCost>
    <OneTimeCost>0.00</OneTimeCost>
</SupplementaryOffering>
```

### New Response Format (JSON — after)
```json
{
    "Id": "1330977724",
    "Category": "data",
    "NameAr": "داتا غير محدودة 90 يوم",
    "NameEn": "Unlimited Data 90 Days",
    "OfferingStatus": "R",
    "ExpireDate": "2037-12-31T00:00:00",
    "MonthlyCost": 975.00,
    "OneTimeCost": 0.00
}
```

### Field Mapping

| New JSON Field | Mapped To | Notes |
|---------------|-----------|-------|
| `Id` | `AvailableOfferingDTO.OfferingId` | Direct map |
| `NameEn` | `AvailableOfferingDTO.OfferingName` | Uses English name |
| `OfferingStatus` | `AvailableOfferingDTO.OfferingStatus` | `"R"` -> `Release`, anything else -> `Suspend` |
| `ExpireDate` | `AvailableOfferingDTO.ExpirationDate` | `null` -> `DateTime.MaxValue` |
| `MonthlyCost` | `AvailableOfferingDTO.MonthlyCost` | Direct map |
| `OneTimeCost` | `AvailableOfferingDTO.OneTimeCost` | Direct map |
| `Category` | Not mapped | New field, not used by existing logic |
| `NameAr` | Not mapped | Available in DTO but not used in current mapping |

---

## 7. Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| SalesApp returns unexpected JSON structure | Medium | Tests validate deserialization; `PostJsonReceiveJson` will throw clear exception if format doesn't match |
| SalesApp returns HTTP error (500, 404) | Low | `PostJsonReceiveJson` logs the response; `SocketException` mapped to `APIUnreachable` |
| Existing callers of these methods break | None | Return types (`AvailableOfferingDTO`, `SupplementaryOfferingDTO`) unchanged — mapping is internal |
| Other SalesApp endpoints affected | None | Only `EligibleAddons` endpoint was changed; all Huawei BSS SOAP calls untouched |

---

## 8. Deployment Notes

- This change must be deployed **in sync** with the SalesApp team's deployment
- If SalesApp deploys first (new JSON format), our current code (XML parsing) will fail on `XmlDocument.LoadXml()`
- If we deploy first (JSON parsing), and SalesApp still returns XML, `JsonConvert.DeserializeObject` will return null
- **Recommended:** Deploy both simultaneously, or SalesApp first + our change immediately after

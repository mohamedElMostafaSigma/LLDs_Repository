# LLD: Implementing GA4 Event `downgraded_plan`

## 1. Overview

| Item | Value |
|------|-------|
| Event Name | `downgraded_plan` |
| Parameters | `from_plan_id`, `to_plan_id`, `reason` |
| Target Location | `OfferService.cs` (line 506, after Adjust event) |

---

## 2. Prerequisites Checklist

| Step | File | Status |
|------|------|--------|
| Event defined in enum | `RedbullKSA.Entities\Enums\GA4EventType.cs:17` | Already exists |
| GA4Service injected | `OfferService.cs:52` | Already exists |
| GetGA4Service() called | `OfferService.cs:90` | Already exists |

---

## 3. Implementation Steps

### Step 1: Verify Event Type in Enum

**File:** `RedbullKSA.Entities\Enums\GA4EventType.cs`

The `downgraded_plan` event already exists at line 18. If it didn't exist, you would add:

```csharp
downgraded_plan,
```

---

### Step 2: Verify IGA4Service Injection

**File:** Target service (e.g., `OfferService.cs`)

**2a. Check the service has the field:**
```csharp
private readonly IGA4Service _ga4Service;
```

**2b. Check constructor injects it:**
```csharp
_ga4Service = serviceProvider.GetGA4Service();
```

**2c. Required using statement:**
```csharp
using RedbullKSA.Services.Services.Contracts.GA4;
```

---

### Step 3: Track the Event

**File:** `RedbullKSA.Services\Services\OfferService.cs`

**Location:** Line 506 (after `_adjustService.SubmitEvent` call, before closing brace)

**Code to add:**
```csharp
                // Track downgraded_plan event with GA4
                await _ga4Service.TrackEvent("downgraded_plan", new Dictionary<string, object>
                {
                    { "from_plan_id", subscribeNewPlanResponse.OldPlan.OfferingId },
                    { "to_plan_id", subscribeNewPlanResponse.NewPlan.OfferingId },
                    { "reason", "user_initiated" }
                });
```

---

### Step 4: Alternative - Using GA4EventParameters (Strongly Typed)

```csharp
var eventParams = new GA4EventParameters(
    clientId: clientId,
    deviceType: DeviceType.None,  // Will be auto-filled from middleware
    eventType: GA4EventType.downgraded_plan
);
eventParams.AddParameter("from_plan_id", oldPlanId);
eventParams.AddParameter("to_plan_id", newPlanId);
eventParams.AddParameter("reason", reason);

await _ga4Service.TrackEvent(eventParams);
```

---

## 4. Architecture Flow

```
+-------------------+
|   API Request     |
+---------+---------+
          |
          v
+-------------------------+
|   GA4TrackingFilter     |  <-- Sets userId & deviceType
|   (OnActionExecuting)   |
+---------+---------------+
          |
          v
+-------------------------+
|   OfferService          |
|   (Business Logic)      |
|                         |
|   if (Downgrade) {      |
|     _ga4Service.Track() |  <-- You add this call
|   }                     |
+---------+---------------+
          |
          v
+-------------------------+
|   GA4Service            |
|   - Builds GA4Request   |
|   - Sends to GA4 API    |
|   - Logs to DB          |
+-------------------------+
```

---

## 5. Exact Code Location

**File:** `RedbullKSA.Services\Services\OfferService.cs`

```
Current code structure:
-------------------------------------------------------------------------------
Line 463 | if (subscribeNewPlanResponse.ChangePlanStatus == ChangePlanStatus.Downgrade)
Line 464 | {
Line 465 |     //check for free sim count
Line 466 |     subscribeNewPlanResponse.DeactivatedSimCount = ...
   ...   |
Line 504 |     subscribeNewPlanResponse.DeactivatedSimCount = 0;
Line 505 |     await _adjustService.SubmitEvent(new AdjustEventPurchasePlanParameters(...));
         |
         |     +-----------------------------------------------------------+
         |     |  >>> INSERT YOUR CODE HERE (between line 505 and 507) <<< |
         |     +-----------------------------------------------------------+
         |
Line 507 | }
Line 508 | else
-------------------------------------------------------------------------------
```

---

## 6. After Your Edit

Lines 504-512 should look like:

```csharp
                subscribeNewPlanResponse.DeactivatedSimCount = 0;
                await _adjustService.SubmitEvent(new AdjustEventPurchasePlanParameters(Entities.Enums.AdjustEvent.Downgraded_Plan, subscribeNewPlanResponse.NewPlan.PaymentType, subscribeNewPlanResponse.NewPlan.IsHybrid, false, subscribeNewPlanResponse.NewPlan.IsDataPlan, subscribeNewPlanResponse.NewPlan.HasESim, clientId, primaryOffering.PhoneNumber.RemoveCountryCode(), (float)subscribeNewPlanResponse.NewPlan.Price, subscribeNewPlanResponse.NewPlan.DisplayName, subscribeNewPlanResponse.NewPlan.OfferingId));

                // Track downgraded_plan event with GA4
                await _ga4Service.TrackEvent("downgraded_plan", new Dictionary<string, object>
                {
                    { "from_plan_id", subscribeNewPlanResponse.OldPlan.OfferingId },
                    { "to_plan_id", subscribeNewPlanResponse.NewPlan.OfferingId },
                    { "reason", "user_initiated" }
                });

            }
            else
            {
```

---

## 7. Files to Modify Summary

| File | Action |
|------|--------|
| `GA4EventType.cs` | Verify enum exists (already done) |
| `OfferService.cs` | Add `TrackEvent` call at line 506 |

---

## 8. Testing the Event

### 8.1 Enable Debug Mode

In your `appsettings.json` or configuration, ensure GA4 debug mode is enabled:

```json
{
  "GA4": {
    "EnableDebugMode": true
  }
}
```

This routes events to GA4's debug endpoint for validation.

---

### 8.2 Check Application Logs

After triggering a downgrade, look for these log messages:

**Success:**
```
GA4 event tracked successfully: downgraded_plan
```

**Failure:**
```
GA4 event failed: {status} - {error}
```

---

### 8.3 Verify in Database

Query the `GA4Event` table to see logged events:

```sql
SELECT TOP 10 *
FROM GA4Event
WHERE EventName = 'downgraded_plan'
ORDER BY CreatedAt DESC;
```

---

### 8.4 Test via GA4 DebugView (Real-time)

1. Open Google Analytics 4 Dashboard
2. Navigate to **Admin** > **DebugView**
3. Trigger a plan downgrade in the app
4. Watch for `downgraded_plan` event appearing in real-time
5. Click on the event to verify parameters:
   - `from_plan_id`
   - `to_plan_id`
   - `reason`

---

### 8.5 Test via API (Manual Trigger)

If you have a test endpoint or Postman collection:

1. Authenticate as a test user
2. Call the plan change API with a downgrade scenario (new plan price < old plan price)
3. Verify the response is successful
4. Check logs and database for the tracked event

---

### 8.6 Unit Test Example

```csharp
[Fact]
public async Task NewPlanSubscription_WhenDowngrade_ShouldTrackGA4Event()
{
    // Arrange
    var mockGA4Service = new Mock<IGA4Service>();
    mockGA4Service
        .Setup(x => x.TrackEvent(It.IsAny<string>(), It.IsAny<Dictionary<string, object>>()))
        .ReturnsAsync(true);

    // ... setup other mocks and service

    // Act
    var result = await offerService.NewPlanSubscription(request);

    // Assert
    mockGA4Service.Verify(
        x => x.TrackEvent(
            "downgraded_plan",
            It.Is<Dictionary<string, object>>(d =>
                d.ContainsKey("from_plan_id") &&
                d.ContainsKey("to_plan_id") &&
                d.ContainsKey("reason")
            )
        ),
        Times.Once
    );
}
```

---

### 8.7 Troubleshooting

| Issue | Check |
|-------|-------|
| Event not appearing in GA4 | Verify `EnableDebugMode` is true, check API secret is correct |
| Event logged but parameters missing | Ensure Dictionary keys match exactly |
| 401/403 errors | Check `MeasurementId` and `ApiSecret` in config |
| Event not in database | Check `IGA4EventRepository` is properly registered |
| No logs at all | Verify `_ga4Service` is not null, check DI registration |

---

## 9. Configuration Reference

Ensure these settings exist in your configuration:

```json
{
  "GA4": {
    "BaseUrl": "https://www.google-analytics.com/mp/collect",
    "DebugUrl": "https://www.google-analytics.com/debug/mp/collect",
    "EnableDebugMode": true,
    "WebMeasurementId": "G-XXXXXXXXXX",
    "WebApiSecret": "your-web-api-secret",
    "AndroidMeasurementId": "G-XXXXXXXXXX",
    "AndroidApiSecret": "your-android-api-secret",
    "iOSMeasurementId": "G-XXXXXXXXXX",
    "iOSApiSecret": "your-ios-api-secret"
  }
}
```

# Firebase Cloud Messaging (FCM) Integration Guide
## RedbullKSA Backend — .NET 5

---

## 1. Overview

This guide covers the complete FCM integration for the RedbullKSA backend. The implementation follows the existing layered architecture:

```
Controllers (API Layer)
    |
Services (Business Logic)
    |
Repositories (Data Access)
    |
Database (SQL Server)
```

**What this adds:**
- Device token registration and lifecycle management (per client, per device)
- Push notification sending via Firebase Admin SDK (already in project)
- Notification logging for auditing and debugging
- Bulk and targeted notification support
- Platform-aware handling (Android Google / Android Huawei / iOS)

**What already exists in the project:**
- `FirebaseAdmin.Messaging` NuGet package (already installed)
- `NotificationService.cs` with Firebase send logic
- `Notification.cs` entity with `FromResponse()` factory method
- `SessionToken.cs` with `NotificationToken` field
- `BasePushNotification.cs` and `NotificationBody.cs` DTOs

---

## 2. Database Schema

### 2.1 FcmDeviceTokens Table

```sql
CREATE TABLE [dbo].[FcmDeviceToken] (
    [Id]                INT IDENTITY(1,1) PRIMARY KEY,
    [ClientId]          INT NOT NULL,
    [Msisdn]            NVARCHAR(20) NOT NULL,
    [DeviceToken]       NVARCHAR(500) NOT NULL,
    [DeviceType]        INT NOT NULL,           -- 1=Android, 2=iOS, 3=Huawei
    [DeviceId]          NVARCHAR(200) NULL,      -- Unique device identifier
    [IsActive]          BIT NOT NULL DEFAULT 1,
    [AddedBy]           INT NULL,
    [AddedDate]         DATETIME NULL,
    [ModifiedBy]        INT NULL,
    [ModifiedDate]      DATETIME NULL,

    CONSTRAINT FK_FcmDeviceToken_Client FOREIGN KEY ([ClientId]) 
        REFERENCES [dbo].[Client]([Id])
);

CREATE INDEX IX_FcmDeviceToken_ClientId ON [dbo].[FcmDeviceToken]([ClientId]);
CREATE INDEX IX_FcmDeviceToken_DeviceToken ON [dbo].[FcmDeviceToken]([DeviceToken]);
CREATE INDEX IX_FcmDeviceToken_Msisdn ON [dbo].[FcmDeviceToken]([Msisdn]);
```

### 2.2 FcmNotificationLog Table

```sql
CREATE TABLE [dbo].[FcmNotificationLog] (
    [Id]                INT IDENTITY(1,1) PRIMARY KEY,
    [ClientId]          INT NOT NULL,
    [Msisdn]            NVARCHAR(20) NULL,
    [DeviceToken]       NVARCHAR(500) NULL,
    [Title]             NVARCHAR(500) NOT NULL,
    [Body]              NVARCHAR(MAX) NOT NULL,
    [Data]              NVARCHAR(MAX) NULL,       -- JSON payload
    [NotificationType]  INT NOT NULL,             -- Enum: General, Renewal, Offer, etc.
    [IsSuccess]         BIT NOT NULL DEFAULT 0,
    [ErrorMessage]      NVARCHAR(500) NULL,
    [MessageId]         NVARCHAR(200) NULL,       -- Firebase message ID
    [AddedDate]         DATETIME NULL,

    CONSTRAINT FK_FcmNotificationLog_Client FOREIGN KEY ([ClientId]) 
        REFERENCES [dbo].[Client]([Id])
);

CREATE INDEX IX_FcmNotificationLog_ClientId ON [dbo].[FcmNotificationLog]([ClientId]);
CREATE INDEX IX_FcmNotificationLog_AddedDate ON [dbo].[FcmNotificationLog]([AddedDate]);
CREATE INDEX IX_FcmNotificationLog_NotificationType ON [dbo].[FcmNotificationLog]([NotificationType]);
```

---

## 3. File Structure

```
RedbullKSA.API/
|
+-- RedbullKSA.Entities/
|   +-- Database/
|   |   +-- Models/
|   |       +-- FcmDeviceToken.cs                    (Entity)
|   |       +-- FcmNotificationLog.cs                (Entity)
|   +-- Enums/
|   |   +-- FcmNotificationType.cs                   (Enum)
|   +-- API/
|   |   +-- Models/
|   |       +-- FcmModels/
|   |           +-- FcmTokenRequest.cs               (API Request)
|   |           +-- FcmNotificationPayload.cs        (Internal DTO)
|   |           +-- FcmResponse.cs                   (API Response)
|   +-- Migrations/
|       +-- M202604200900_AddFcmDeviceTokenTable.cs  (Migration)
|       +-- M202604200901_AddFcmNotificationLogTable.cs (Migration)
|
+-- RedbullKSA.Repositories/
|   +-- Repositories/
|       +-- Contracts/
|       |   +-- IFcmDeviceTokenRepository.cs         (Interface)
|       |   +-- IFcmNotificationLogRepository.cs     (Interface)
|       +-- FcmDeviceTokenRepository.cs              (Implementation)
|       +-- FcmNotificationLogRepository.cs          (Implementation)
|
+-- RedbullKSA.Services/
|   +-- Services/
|       +-- Contracts/
|       |   +-- IFcmService.cs                       (Interface)
|       +-- FcmService.cs                            (Implementation)
|
+-- RedbullKSA.API/
    +-- Controllers/
        +-- Mobile/
            +-- v1/
                +-- MobileFcmController.cs           (Controller)
```

---

## 4. Configuration

### 4.1 Firebase Service Account

The project already has Firebase configured. The service account JSON is loaded at startup. If not present, add:

**File:** `firebase-service-account.json` (project root, next to appsettings.json)

```json
{
  "type": "service_account",
  "project_id": "redbull-ksa-XXXX",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n",
  "client_email": "firebase-adminsdk-XXXXX@redbull-ksa-XXXX.iam.gserviceaccount.com",
  "client_id": "...",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token"
}
```

### 4.2 appsettings.json

Add under the existing configuration:

```json
{
  "FcmConfig": {
    "ServiceAccountPath": "firebase-service-account.json",
    "MaxTokensPerClient": 5
  }
}
```

### 4.3 ConfigurationSettings.cs

Add to `ConfigurationSettings.cs`:

```csharp
public FcmConfig FcmConfig { get; set; }
```

Add new config class:

```csharp
public class FcmConfig
{
    public string ServiceAccountPath { get; set; }
    public int MaxTokensPerClient { get; set; }
}
```

---

## 5. Entity Models

### 5.1 FcmDeviceToken.cs

**Path:** `RedbullKSA.Entities/Database/Models/FcmDeviceToken.cs`

```csharp
using RedbullKSA.Entities.Database.Base;
using RedbullKSA.Entities.Enums;
using System;
using System.ComponentModel.DataAnnotations;

namespace RedbullKSA.Entities.Database.Models
{
    public class FcmDeviceToken : IEntity, IAdded, IModified
    {
        [Key]
        public int Id { get; set; }
        public int ClientId { get; set; }
        public string Msisdn { get; set; }
        public string DeviceToken { get; set; }
        public DeviceType DeviceType { get; set; }
        public string DeviceId { get; set; }
        public bool IsActive { get; set; }
        public int? AddedBy { get; set; }
        public DateTime? AddedDate { get; set; }
        public int? ModifiedBy { get; set; }
        public DateTime? ModifiedDate { get; set; }
    }
}
```

> **Note:** Uses existing `DeviceType` enum (Android=1, iOS=2, Huawei=3) from the codebase.

### 5.2 FcmNotificationLog.cs

**Path:** `RedbullKSA.Entities/Database/Models/FcmNotificationLog.cs`

```csharp
using RedbullKSA.Entities.Database.Base;
using RedbullKSA.Entities.Enums;
using System;
using System.ComponentModel.DataAnnotations;

namespace RedbullKSA.Entities.Database.Models
{
    public class FcmNotificationLog : IEntity, IAdded
    {
        [Key]
        public int Id { get; set; }
        public int ClientId { get; set; }
        public string Msisdn { get; set; }
        public string DeviceToken { get; set; }
        public string Title { get; set; }
        public string Body { get; set; }
        public string Data { get; set; }
        public FcmNotificationType NotificationType { get; set; }
        public bool IsSuccess { get; set; }
        public string ErrorMessage { get; set; }
        public string MessageId { get; set; }
        public int? AddedBy { get; set; }
        public DateTime? AddedDate { get; set; }
    }
}
```

### 5.3 FcmNotificationType Enum

**Path:** `RedbullKSA.Entities/Enums/FcmNotificationType.cs`

```csharp
namespace RedbullKSA.Entities.Enums
{
    public enum FcmNotificationType
    {
        General = 1,
        Renewal = 2,
        Offer = 3,
        Plan = 4,
        Payment = 5,
        Order = 6,
        Promotion = 7,
        Referral = 8
    }
}
```

### 5.4 FcmTokenRequest.cs

**Path:** `RedbullKSA.Entities/API/Models/FcmModels/FcmTokenRequest.cs`

```csharp
using RedbullKSA.Entities.Enums;
using System.ComponentModel.DataAnnotations;

namespace RedbullKSA.Entities.API.Models.FcmModels
{
    public class FcmTokenRequest
    {
        [Required]
        public string DeviceToken { get; set; }

        [Required]
        public DeviceType DeviceType { get; set; }

        public string DeviceId { get; set; }
    }
}
```

### 5.5 FcmNotificationPayload.cs

**Path:** `RedbullKSA.Entities/API/Models/FcmModels/FcmNotificationPayload.cs`

```csharp
using RedbullKSA.Entities.Enums;
using System.Collections.Generic;

namespace RedbullKSA.Entities.API.Models.FcmModels
{
    public class FcmNotificationPayload
    {
        public int ClientId { get; set; }
        public string Msisdn { get; set; }
        public string TitleEn { get; set; }
        public string TitleAr { get; set; }
        public string BodyEn { get; set; }
        public string BodyAr { get; set; }
        public FcmNotificationType NotificationType { get; set; }
        public Dictionary<string, string> Data { get; set; }
    }
}
```

### 5.6 FcmResponse.cs

**Path:** `RedbullKSA.Entities/API/Models/FcmModels/FcmResponse.cs`

```csharp
using System.Collections.Generic;

namespace RedbullKSA.Entities.API.Models.FcmModels
{
    public class FcmTokenResponse
    {
        public int Id { get; set; }
        public string DeviceToken { get; set; }
        public string DeviceType { get; set; }
        public string DeviceId { get; set; }
        public bool IsActive { get; set; }
    }

    public class FcmNotificationLogResponse
    {
        public string Title { get; set; }
        public string Body { get; set; }
        public string NotificationType { get; set; }
        public bool IsSuccess { get; set; }
        public string AddedDate { get; set; }
    }

    public class FcmSendResult
    {
        public int SuccessCount { get; set; }
        public int FailureCount { get; set; }
        public List<string> Errors { get; set; } = new List<string>();
    }
}
```

---

## 6. Repository Layer

### 6.1 IFcmDeviceTokenRepository

**Path:** `RedbullKSA.Repositories/Repositories/Contracts/IFcmDeviceTokenRepository.cs`

```csharp
using RedbullKSA.Entities.Database.Models;
using RedbullKSA.Repositories.Base.Contracts;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace RedbullKSA.Repositories.Repositories.Contracts
{
    public interface IFcmDeviceTokenRepository : IBaseRepository<FcmDeviceToken>
    {
        Task<IEnumerable<FcmDeviceToken>> GetByClientId(int clientId);
        Task<FcmDeviceToken> GetByDeviceToken(string deviceToken);
        Task<FcmDeviceToken> GetByClientAndDevice(int clientId, string deviceId);
        Task<IEnumerable<FcmDeviceToken>> GetActiveByClientId(int clientId);
        Task DeactivateToken(string deviceToken);
    }
}
```

### 6.2 FcmDeviceTokenRepository

**Path:** `RedbullKSA.Repositories/Repositories/FcmDeviceTokenRepository.cs`

```csharp
using RedbullKSA.Entities.Database.Models;
using RedbullKSA.Repositories.Base;
using RedbullKSA.Repositories.Repositories.Contracts;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace RedbullKSA.Repositories.Repositories
{
    public class FcmDeviceTokenRepository : BaseRepository<FcmDeviceToken>, IFcmDeviceTokenRepository
    {
        public FcmDeviceTokenRepository(IServiceProvider services) : base(services) { }

        public async Task<IEnumerable<FcmDeviceToken>> GetByClientId(int clientId)
        {
            return await Store()
                .Filtered(nameof(FcmDeviceToken.ClientId), clientId)
                .Get<FcmDeviceToken>();
        }

        public async Task<FcmDeviceToken> GetByDeviceToken(string deviceToken)
        {
            return await Store()
                .Filtered(nameof(FcmDeviceToken.DeviceToken), deviceToken)
                .FirstOrNull<FcmDeviceToken>();
        }

        public async Task<FcmDeviceToken> GetByClientAndDevice(int clientId, string deviceId)
        {
            return await Store()
                .Filtered(nameof(FcmDeviceToken.ClientId), clientId)
                .Filtered(nameof(FcmDeviceToken.DeviceId), deviceId)
                .FirstOrNull<FcmDeviceToken>();
        }

        public async Task<IEnumerable<FcmDeviceToken>> GetActiveByClientId(int clientId)
        {
            return await Store()
                .Filtered(nameof(FcmDeviceToken.ClientId), clientId)
                .Filtered(nameof(FcmDeviceToken.IsActive), true)
                .Get<FcmDeviceToken>();
        }

        public async Task DeactivateToken(string deviceToken)
        {
            var token = await GetByDeviceToken(deviceToken);
            if (token != null)
            {
                token.IsActive = false;
                token.ModifiedDate = DateTime.UtcNow;
                await Update(token);
            }
        }
    }
}
```

### 6.3 IFcmNotificationLogRepository

**Path:** `RedbullKSA.Repositories/Repositories/Contracts/IFcmNotificationLogRepository.cs`

```csharp
using RedbullKSA.Entities.Database.Models;
using RedbullKSA.Entities.Enums;
using RedbullKSA.Repositories.Base.Contracts;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace RedbullKSA.Repositories.Repositories.Contracts
{
    public interface IFcmNotificationLogRepository : IBaseRepository<FcmNotificationLog>
    {
        Task<IEnumerable<FcmNotificationLog>> GetByClientId(int clientId, int limit = 50);
        Task<IEnumerable<FcmNotificationLog>> GetByType(FcmNotificationType type, int limit = 100);
    }
}
```

### 6.4 FcmNotificationLogRepository

**Path:** `RedbullKSA.Repositories/Repositories/FcmNotificationLogRepository.cs`

```csharp
using RedbullKSA.Entities.Database.Models;
using RedbullKSA.Entities.Enums;
using RedbullKSA.Repositories.Base;
using RedbullKSA.Repositories.Repositories.Contracts;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace RedbullKSA.Repositories.Repositories
{
    public class FcmNotificationLogRepository : BaseRepository<FcmNotificationLog>, IFcmNotificationLogRepository
    {
        public FcmNotificationLogRepository(IServiceProvider services) : base(services) { }

        public async Task<IEnumerable<FcmNotificationLog>> GetByClientId(int clientId, int limit = 50)
        {
            return await Store()
                .Filtered(nameof(FcmNotificationLog.ClientId), clientId)
                .Sorted(nameof(FcmNotificationLog.Id), "DESC")
                .Paged(1, limit)
                .Get<FcmNotificationLog>();
        }

        public async Task<IEnumerable<FcmNotificationLog>> GetByType(FcmNotificationType type, int limit = 100)
        {
            return await Store()
                .Filtered(nameof(FcmNotificationLog.NotificationType), (int)type)
                .Sorted(nameof(FcmNotificationLog.Id), "DESC")
                .Paged(1, limit)
                .Get<FcmNotificationLog>();
        }
    }
}
```

---

## 7. Service Layer

### 7.1 IFcmService

**Path:** `RedbullKSA.Services/Services/Contracts/IFcmService.cs`

```csharp
using RedbullKSA.Entities.API.Models.FcmModels;
using RedbullKSA.Entities.Database.Models;
using RedbullKSA.Entities.Enums;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace RedbullKSA.Services.Services.Contracts
{
    public interface IFcmService
    {
        Task<FcmDeviceToken> RegisterToken(FcmTokenRequest request, int clientId, string msisdn);
        Task DeactivateToken(string deviceToken);
        Task<IEnumerable<FcmDeviceToken>> GetClientTokens(int clientId);
        Task<FcmSendResult> SendToClient(int clientId, string titleEn, string titleAr, string bodyEn, string bodyAr,
            FcmNotificationType type, Dictionary<string, string> data = null, ServiceLanguage? language = null);
        Task<FcmSendResult> SendToToken(string deviceToken, string title, string body,
            FcmNotificationType type, int clientId, Dictionary<string, string> data = null);
        Task<IEnumerable<FcmNotificationLog>> GetNotificationLogs(int clientId, int limit = 50);
    }
}
```

### 7.2 FcmService

**Path:** `RedbullKSA.Services/Services/FcmService.cs`

```csharp
using FirebaseAdmin.Messaging;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using RedbullKSA.Entities.API.Models.FcmModels;
using RedbullKSA.Entities.Database.Models;
using RedbullKSA.Entities.Enums;
using RedbullKSA.Repositories.Repositories.Contracts;
using RedbullKSA.Services.Base;
using RedbullKSA.Services.Services.Contracts;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace RedbullKSA.Services.Services
{
    public class FcmService : BaseService, IFcmService
    {
        private readonly IFcmDeviceTokenRepository _tokenRepository;
        private readonly IFcmNotificationLogRepository _logRepository;
        private readonly ILogger<FcmService> _logger;

        public FcmService(IServiceProvider serviceProvider) : base(serviceProvider)
        {
            _tokenRepository = serviceProvider.GetRequiredService<IFcmDeviceTokenRepository>();
            _logRepository = serviceProvider.GetRequiredService<IFcmNotificationLogRepository>();
            _logger = serviceProvider.GetRequiredService<ILogger<FcmService>>();
        }

        /// <summary>
        /// Register or update a device token for push notifications.
        /// If the same DeviceId exists, updates the token. Otherwise creates new.
        /// Enforces max tokens per client from config.
        /// </summary>
        public async Task<FcmDeviceToken> RegisterToken(FcmTokenRequest request, int clientId, string msisdn)
        {
            // Check if this device already has a token registered
            FcmDeviceToken existing = null;

            if (!string.IsNullOrEmpty(request.DeviceId))
                existing = await _tokenRepository.GetByClientAndDevice(clientId, request.DeviceId);

            // Also check if the exact token already exists (possibly from another device)
            if (existing == null)
                existing = await _tokenRepository.GetByDeviceToken(request.DeviceToken);

            if (existing != null)
            {
                // Update existing record
                existing.DeviceToken = request.DeviceToken;
                existing.DeviceType = request.DeviceType;
                existing.IsActive = true;
                existing.ModifiedDate = DateTime.UtcNow;
                await _tokenRepository.Update(existing);
                return existing;
            }

            // Enforce max tokens per client
            var clientTokens = await _tokenRepository.GetActiveByClientId(clientId);
            if (clientTokens.Count() >= _config.FcmConfig.MaxTokensPerClient)
            {
                // Deactivate the oldest token
                var oldest = clientTokens.OrderBy(t => t.AddedDate).First();
                await _tokenRepository.DeactivateToken(oldest.DeviceToken);
            }

            // Create new
            var newToken = new FcmDeviceToken
            {
                ClientId = clientId,
                Msisdn = msisdn,
                DeviceToken = request.DeviceToken,
                DeviceType = request.DeviceType,
                DeviceId = request.DeviceId,
                IsActive = true,
                AddedDate = DateTime.UtcNow
            };

            return await _tokenRepository.Add(newToken);
        }

        /// <summary>
        /// Deactivate a device token (e.g., on logout)
        /// </summary>
        public async Task DeactivateToken(string deviceToken)
        {
            await _tokenRepository.DeactivateToken(deviceToken);
        }

        /// <summary>
        /// Get all active tokens for a client
        /// </summary>
        public async Task<IEnumerable<FcmDeviceToken>> GetClientTokens(int clientId)
        {
            return await _tokenRepository.GetActiveByClientId(clientId);
        }

        /// <summary>
        /// Send push notification to all active devices for a client.
        /// Selects title/body based on client language if provided.
        /// </summary>
        public async Task<FcmSendResult> SendToClient(int clientId, string titleEn, string titleAr,
            string bodyEn, string bodyAr, FcmNotificationType type,
            Dictionary<string, string> data = null, ServiceLanguage? language = null)
        {
            var result = new FcmSendResult();
            var activeTokens = await _tokenRepository.GetActiveByClientId(clientId);

            if (!activeTokens.Any())
            {
                _logger.LogInformation("No active FCM tokens for client {ClientId}", clientId);
                return result;
            }

            var title = language == ServiceLanguage.AR ? titleAr : titleEn;
            var body = language == ServiceLanguage.AR ? bodyAr : bodyEn;

            foreach (var token in activeTokens)
            {
                var sendResult = await SendToToken(token.DeviceToken, title, body, type, clientId, data);
                result.SuccessCount += sendResult.SuccessCount;
                result.FailureCount += sendResult.FailureCount;
                result.Errors.AddRange(sendResult.Errors);
            }

            return result;
        }

        /// <summary>
        /// Send push notification to a specific device token.
        /// Logs the result in FcmNotificationLog.
        /// </summary>
        public async Task<FcmSendResult> SendToToken(string deviceToken, string title, string body,
            FcmNotificationType type, int clientId, Dictionary<string, string> data = null)
        {
            var result = new FcmSendResult();

            var message = new Message()
            {
                Token = deviceToken,
                Notification = new Notification()
                {
                    Title = title,
                    Body = body
                },
                Data = data
            };

            string messageId = null;
            string errorMessage = null;
            bool isSuccess = false;

            try
            {
                messageId = await FirebaseMessaging.DefaultInstance.SendAsync(message);
                isSuccess = true;
                result.SuccessCount = 1;

                _logger.LogTrace("FCM sent to {Token}. MessageId: {MessageId}", deviceToken, messageId);
            }
            catch (FirebaseMessagingException ex)
            {
                errorMessage = $"{ex.MessagingErrorCode}: {ex.Message}";
                result.FailureCount = 1;
                result.Errors.Add(errorMessage);

                _logger.LogError(ex, "FCM send failed for token {Token}", deviceToken);

                // Deactivate invalid tokens
                if (ex.MessagingErrorCode == MessagingErrorCode.Unregistered
                    || ex.MessagingErrorCode == MessagingErrorCode.InvalidArgument)
                {
                    await _tokenRepository.DeactivateToken(deviceToken);
                    _logger.LogInformation("Deactivated invalid FCM token {Token}", deviceToken);
                }
            }
            catch (Exception ex)
            {
                errorMessage = ex.Message;
                result.FailureCount = 1;
                result.Errors.Add(errorMessage);

                _logger.LogError(ex, "Unexpected error sending FCM to {Token}", deviceToken);
            }

            // Log the notification
            try
            {
                await _logRepository.Add(new FcmNotificationLog
                {
                    ClientId = clientId,
                    DeviceToken = deviceToken,
                    Title = title,
                    Body = body,
                    Data = data != null ? JsonConvert.SerializeObject(data) : null,
                    NotificationType = type,
                    IsSuccess = isSuccess,
                    ErrorMessage = errorMessage,
                    MessageId = messageId,
                    AddedDate = DateTime.UtcNow
                });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to log FCM notification for client {ClientId}", clientId);
            }

            return result;
        }

        /// <summary>
        /// Get notification history for a client
        /// </summary>
        public async Task<IEnumerable<FcmNotificationLog>> GetNotificationLogs(int clientId, int limit = 50)
        {
            return await _logRepository.GetByClientId(clientId, limit);
        }
    }
}
```

---

## 8. Controller Layer

### MobileFcmController

**Path:** `RedbullKSA.API/Controllers/Mobile/v1/MobileFcmController.cs`

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Caching.Memory;
using RedbullKSA.API.Controllers.Base;
using RedbullKSA.Entities.API.Models.FcmModels;
using RedbullKSA.Entities.Enums;
using RedbullKSA.Services.Services.Contracts;
using System;
using System.Linq;
using System.Threading.Tasks;

namespace RedbullKSA.API.Controllers.Mobile.v1
{
    [Route("api/mobile/v1/[controller]")]
    public class MobileFcmController : BaseAppController
    {
        private readonly IFcmService _fcmService;

        public MobileFcmController(IServiceProvider services, IMemoryCache cache)
            : base(services, cache)
        {
            _fcmService = services.GetRequiredService<IFcmService>();
        }

        /// <summary>
        /// Register or update a device token for push notifications
        /// </summary>
        [HttpPost("registerToken")]
        public async Task<IActionResult> RegisterToken([FromBody] FcmTokenRequest request)
        {
            var token = await _fcmService.RegisterToken(request, ClientId, PhoneNumber);
            return Ok(new FcmTokenResponse
            {
                Id = token.Id,
                DeviceToken = token.DeviceToken,
                DeviceType = token.DeviceType.ToString(),
                DeviceId = token.DeviceId,
                IsActive = token.IsActive
            });
        }

        /// <summary>
        /// Deactivate a device token (call on logout)
        /// </summary>
        [HttpPost("deactivateToken")]
        public async Task<IActionResult> DeactivateToken([FromBody] FcmTokenRequest request)
        {
            await _fcmService.DeactivateToken(request.DeviceToken);
            return Ok();
        }

        /// <summary>
        /// Get all active tokens for the authenticated client
        /// </summary>
        [HttpGet("myTokens")]
        public async Task<IActionResult> GetMyTokens()
        {
            var tokens = await _fcmService.GetClientTokens(ClientId);
            var response = tokens.Select(t => new FcmTokenResponse
            {
                Id = t.Id,
                DeviceToken = t.DeviceToken,
                DeviceType = t.DeviceType.ToString(),
                DeviceId = t.DeviceId,
                IsActive = t.IsActive
            });
            return Ok(response);
        }

        /// <summary>
        /// Get notification history for the authenticated client
        /// </summary>
        [HttpGet("notifications")]
        public async Task<IActionResult> GetNotificationLogs([FromQuery] int limit = 50)
        {
            var logs = await _fcmService.GetNotificationLogs(ClientId, limit);
            var response = logs.Select(l => new FcmNotificationLogResponse
            {
                Title = l.Title,
                Body = l.Body,
                NotificationType = l.NotificationType.ToString(),
                IsSuccess = l.IsSuccess,
                AddedDate = l.AddedDate?.ToString("yyyy-MM-dd HH:mm:ss")
            });
            return Ok(response);
        }
    }
}
```

---

## 9. Integration with Order Creation

### Where to trigger FCM notification in OnboardingService

In `OnboardingService.cs`, after successful SIM activation, add:

```csharp
// After activation success (around line ~500, after CreateSalesOrder succeeds)
try
{
    await _fcmService.SendToClient(
        clientId,
        titleEn: "SIM Activated Successfully",
        titleAr: "تم تفعيل الشريحة بنجاح",
        bodyEn: $"Your SIM {request.PhoneNumber} has been activated.",
        bodyAr: $"تم تفعيل شريحتك {request.PhoneNumber}.",
        type: FcmNotificationType.Order,
        data: new Dictionary<string, string>
        {
            { "orderId", order.OtoOrderId.ToString() },
            { "type", "activation" }
        },
        language: language);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to send FCM for SIM activation, clientId: {ClientId}", clientId);
    // Don't throw — notification failure should not block activation
}
```

---

## 10. Dependency Injection Setup

### Add to ServiceExtensions.cs

**In the accessor extensions region:**

```csharp
public static IFcmService GetFcmService(this IServiceProvider services)
    => services.GetRequiredService<IFcmService>();
public static IFcmDeviceTokenRepository GetFcmDeviceTokenRepository(this IServiceProvider services)
    => services.GetRequiredService<IFcmDeviceTokenRepository>();
public static IFcmNotificationLogRepository GetFcmNotificationLogRepository(this IServiceProvider services)
    => services.GetRequiredService<IFcmNotificationLogRepository>();
```

**In the ConfigureServices method:**

```csharp
// FCM
services.AddScoped<IFcmService, FcmService>();
services.AddScoped<IFcmDeviceTokenRepository, FcmDeviceTokenRepository>();
services.AddScoped<IFcmNotificationLogRepository, FcmNotificationLogRepository>();
```

---

## 11. Database Migrations

### 11.1 FcmDeviceToken Migration

**Path:** `RedbullKSA.Entities/Migrations/M202604200900_AddFcmDeviceTokenTable.cs`

```csharp
using FluentMigrator;
using RedbullKSA.Entities.Database.Models;

namespace RedbullKSA.Entities.Migrations
{
    [Migration(202604200900)]
    public class M202604200900_AddFcmDeviceTokenTable : Migration
    {
        public override void Up()
        {
            Create.Table(nameof(FcmDeviceToken))
                .WithColumn(nameof(FcmDeviceToken.Id)).AsInt32().PrimaryKey().Identity()
                .WithColumn(nameof(FcmDeviceToken.ClientId)).AsInt32().NotNullable()
                    .ForeignKey(nameof(Client), nameof(Client.Id))
                .WithColumn(nameof(FcmDeviceToken.Msisdn)).AsString(20).NotNullable()
                .WithColumn(nameof(FcmDeviceToken.DeviceToken)).AsString(500).NotNullable()
                .WithColumn(nameof(FcmDeviceToken.DeviceType)).AsInt32().NotNullable()
                .WithColumn(nameof(FcmDeviceToken.DeviceId)).AsString(200).Nullable()
                .WithColumn(nameof(FcmDeviceToken.IsActive)).AsBoolean().NotNullable().WithDefaultValue(true)
                .WithColumn(nameof(FcmDeviceToken.AddedBy)).AsInt32().Nullable()
                .WithColumn(nameof(FcmDeviceToken.AddedDate)).AsDateTime().Nullable()
                .WithColumn(nameof(FcmDeviceToken.ModifiedBy)).AsInt32().Nullable()
                .WithColumn(nameof(FcmDeviceToken.ModifiedDate)).AsDateTime().Nullable();

            Create.Index("IX_FcmDeviceToken_ClientId").OnTable(nameof(FcmDeviceToken))
                .OnColumn(nameof(FcmDeviceToken.ClientId));
            Create.Index("IX_FcmDeviceToken_DeviceToken").OnTable(nameof(FcmDeviceToken))
                .OnColumn(nameof(FcmDeviceToken.DeviceToken));
            Create.Index("IX_FcmDeviceToken_Msisdn").OnTable(nameof(FcmDeviceToken))
                .OnColumn(nameof(FcmDeviceToken.Msisdn));
        }

        public override void Down()
        {
            Delete.Table(nameof(FcmDeviceToken));
        }
    }
}
```

### 11.2 FcmNotificationLog Migration

**Path:** `RedbullKSA.Entities/Migrations/M202604200901_AddFcmNotificationLogTable.cs`

```csharp
using FluentMigrator;
using RedbullKSA.Entities.Database.Models;

namespace RedbullKSA.Entities.Migrations
{
    [Migration(202604200901)]
    public class M202604200901_AddFcmNotificationLogTable : Migration
    {
        public override void Up()
        {
            Create.Table(nameof(FcmNotificationLog))
                .WithColumn(nameof(FcmNotificationLog.Id)).AsInt32().PrimaryKey().Identity()
                .WithColumn(nameof(FcmNotificationLog.ClientId)).AsInt32().NotNullable()
                    .ForeignKey(nameof(Client), nameof(Client.Id))
                .WithColumn(nameof(FcmNotificationLog.Msisdn)).AsString(20).Nullable()
                .WithColumn(nameof(FcmNotificationLog.DeviceToken)).AsString(500).Nullable()
                .WithColumn(nameof(FcmNotificationLog.Title)).AsString(500).NotNullable()
                .WithColumn(nameof(FcmNotificationLog.Body)).AsString(int.MaxValue).NotNullable()
                .WithColumn(nameof(FcmNotificationLog.Data)).AsString(int.MaxValue).Nullable()
                .WithColumn(nameof(FcmNotificationLog.NotificationType)).AsInt32().NotNullable()
                .WithColumn(nameof(FcmNotificationLog.IsSuccess)).AsBoolean().NotNullable().WithDefaultValue(false)
                .WithColumn(nameof(FcmNotificationLog.ErrorMessage)).AsString(500).Nullable()
                .WithColumn(nameof(FcmNotificationLog.MessageId)).AsString(200).Nullable()
                .WithColumn(nameof(FcmNotificationLog.AddedBy)).AsInt32().Nullable()
                .WithColumn(nameof(FcmNotificationLog.AddedDate)).AsDateTime().Nullable();

            Create.Index("IX_FcmNotificationLog_ClientId").OnTable(nameof(FcmNotificationLog))
                .OnColumn(nameof(FcmNotificationLog.ClientId));
            Create.Index("IX_FcmNotificationLog_AddedDate").OnTable(nameof(FcmNotificationLog))
                .OnColumn(nameof(FcmNotificationLog.AddedDate));
            Create.Index("IX_FcmNotificationLog_NotificationType").OnTable(nameof(FcmNotificationLog))
                .OnColumn(nameof(FcmNotificationLog.NotificationType));
        }

        public override void Down()
        {
            Delete.Table(nameof(FcmNotificationLog));
        }
    }
}
```

---

## 12. Required NuGet Packages

Already installed in the project (verify with `dotnet list package`):

| Package | Version | Status |
|---------|---------|--------|
| `FirebaseAdmin` | 2.2.0+ | Already installed |
| `Newtonsoft.Json` | 13.0.1+ | Already installed |
| `FluentMigrator` | 3.3.0 | Already installed |

No additional packages required.

---

## 13. Usage Examples

### 13.1 Register Device Token

```http
POST /api/mobile/v1/MobileFcm/registerToken
Authorization: Bearer {token}
Content-Type: application/json

{
    "deviceToken": "fMd8kL2...firebase_token...xYz",
    "deviceType": 1,
    "deviceId": "device-uuid-1234"
}
```

**Response (200):**
```json
{
    "id": 42,
    "deviceToken": "fMd8kL2...firebase_token...xYz",
    "deviceType": "Android",
    "deviceId": "device-uuid-1234",
    "isActive": true
}
```

### 13.2 Deactivate Token (Logout)

```http
POST /api/mobile/v1/MobileFcm/deactivateToken
Authorization: Bearer {token}
Content-Type: application/json

{
    "deviceToken": "fMd8kL2...firebase_token...xYz",
    "deviceType": 1
}
```

### 13.3 Get My Tokens

```http
GET /api/mobile/v1/MobileFcm/myTokens
Authorization: Bearer {token}
```

**Response (200):**
```json
[
    {
        "id": 42,
        "deviceToken": "fMd8kL2...xYz",
        "deviceType": "Android",
        "deviceId": "device-uuid-1234",
        "isActive": true
    }
]
```

### 13.4 Get Notification History

```http
GET /api/mobile/v1/MobileFcm/notifications?limit=20
Authorization: Bearer {token}
```

**Response (200):**
```json
[
    {
        "title": "SIM Activated Successfully",
        "body": "Your SIM 966576000146 has been activated.",
        "notificationType": "Order",
        "isSuccess": true,
        "addedDate": "2026-04-20 14:30:00"
    }
]
```

### 13.5 Send from Service (Backend Usage)

```csharp
// Inject IFcmService in any service
await _fcmService.SendToClient(
    clientId: 123,
    titleEn: "Plan Renewed",
    titleAr: "تم تجديد الباقة",
    bodyEn: "Your Mazaji 80 plan has been renewed successfully.",
    bodyAr: "تم تجديد باقة مزاجي 80 بنجاح.",
    type: FcmNotificationType.Renewal,
    data: new Dictionary<string, string>
    {
        { "planId", "MAZAJI_80" },
        { "action", "renewal_success" }
    },
    language: ServiceLanguage.EN);
```

---

## 14. Key Features Summary

- Device token registration with upsert (update if exists, create if new)
- Max tokens per client enforcement (configurable, deactivates oldest)
- Auto-deactivation of invalid/expired tokens on Firebase `Unregistered` error
- Platform-aware device type tracking (Android / iOS / Huawei)
- Bilingual notification support (EN/AR) matching existing `ServiceLanguage` pattern
- Complete notification audit trail in `FcmNotificationLog`
- Custom data payload per notification
- Notification type categorization (General, Renewal, Offer, Plan, Payment, Order, Promotion, Referral)
- Non-blocking send — notification failures don't break business flows
- Follows existing project patterns: `Store().Filtered()`, `IServiceProvider` injection, `BaseService`

---

## 15. Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| `FirebaseMessaging.DefaultInstance` is null | Firebase not initialized at startup | Ensure `FirebaseApp.Create()` is called in `Startup.cs` — check existing `NotificationService` init |
| `MessagingErrorCode.Unregistered` | App uninstalled or token expired | Token auto-deactivated by `FcmService`. Normal behavior. |
| `MessagingErrorCode.InvalidArgument` | Malformed token | Token auto-deactivated. Check mobile app token generation. |
| `MessagingErrorCode.SenderIdMismatch` | Token from different Firebase project | Verify `firebase-service-account.json` matches mobile app config |
| Token registered but no notification received | Token may be for Huawei device | Huawei devices need HMS PushKit, not FCM. Check `DeviceType`. |
| Duplicate tokens in DB | Same device re-registers | `RegisterToken` handles this via `DeviceId` dedup |
| Too many tokens per client | Multiple devices / re-installs | `MaxTokensPerClient` config enforces limit |

---

## 16. Implementation Checklist

```
[ ] 1. Add FcmConfig to ConfigurationSettings.cs and appsettings.json
[ ] 2. Create entity models (FcmDeviceToken, FcmNotificationLog, FcmNotificationType)
[ ] 3. Create DTOs (FcmTokenRequest, FcmNotificationPayload, FcmResponse)
[ ] 4. Create migration files and run migrations
[ ] 5. Create repository interfaces and implementations
[ ] 6. Create IFcmService and FcmService
[ ] 7. Create MobileFcmController
[ ] 8. Register in ServiceExtensions.cs (accessor + DI)
[ ] 9. Verify Firebase initialization in Startup.cs
[ ] 10. Test token registration endpoint
[ ] 11. Test send notification to registered token
[ ] 12. Add FCM triggers in OnboardingService (SIM activation)
[ ] 13. Add FCM triggers in OfferService (renewal, plan change)
[ ] 14. Verify notification logs are written
[ ] 15. Test deactivation flow (logout)
```

# 🔑 Keycloak ile ASP.NET Core API Authorization Entegrasyonu

Bu proje, **Keycloak** kimlik ve erişim yönetim sistemi (IAM) kullanarak bir **ASP.NET Core API**'de JWT tabanlı **authentication** ve **authorization** mekanizmasının nasıl uygulanabileceğini göstermektedir.

---

## ⚙️ 1. Keycloak Ortamı

Docker üzerinde çalışan bir Keycloak container'ı kullanılmıştır:

```bash
docker run -d \
  --name keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin123 \
  quay.io/keycloak/keycloak:26.0.0 \
  start-dev
```

Yönetim paneli: [http://localhost:8080](http://localhost:8080)

### Realm

`ZB DVS`

### Client

`Client_KeyCloakTestApi1`

**Ayarlar:**

* Access Type: `confidential`
* Client Authentication: **ON**
* Service Accounts Enabled: **ON**
* Standard Flow / Implicit Flow: **OFF**

---

## 🔌 2. ASP.NET Core Tarafı Konfigürasyonu

API, Keycloak tarafından verilen JWT token'ı `Microsoft.AspNetCore.Authentication.JwtBearer` kütüphanesiyle doğrular.

### `appsettings.json`

```json
{
  "Authentication": {
    "Authority": "http://localhost:8080/realms/ZB DVS",
    "Audience": "Client_KeyCloakTestApi1",
    "RequireHttpsMetadata": false
  }
}
```

---

## ⚙️ 3. `Program.cs` Yapılandırması

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

// Authentication & Authorization Konfigürasyonu
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    var authSection = builder.Configuration.GetSection("Authentication");

    options.Authority = authSection["Authority"];
    options.Audience = authSection["Audience"];
    options.RequireHttpsMetadata = bool.Parse(authSection["RequireHttpsMetadata"] ?? "false");

    // Token doğrulama parametreleri
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true
    };
});

builder.Services.AddAuthorization();
builder.Services.AddControllers();

var app = builder.Build();

app.UseRouting();
app.UseAuthentication();   // JWT doğrulama
app.UseAuthorization();    // Yetkilendirme

app.MapControllers();

app.Run();
```

---

## 🧠 4. Controller Örneği

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace KeyCloakTestApi1.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class WeatherForecastController : ControllerBase
    {
        [Authorize]
        [HttpGet(Name = "GetWeatherForecast")]
        public IActionResult Get()
        {
            var username = HttpContext.User.Identity?.Name ?? "Bilinmeyen";
            return Ok($"Keycloak doğrulaması başarılı! Kullanıcı: {username}");
        }
    }
}
```

---

## 🤟 5. Postman Üzerinden Test

### 1️⃣ Token Alma (client_credentials flow)

```bash
POST http://localhost:8080/realms/ZB%20DVS/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=Client_KeyCloakTestApi1
client_secret=<CLIENT_SECRET>
grant_type=client_credentials
```

Yanıt:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 300
}
```

### 2️⃣ API Çağrısı

```bash
GET http://localhost:5000/WeatherForecast
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

Cevap:

```
Keycloak doğrulaması başarılı! Kullanıcı: service-account-Client_KeyCloakTestApi1
```

---

## 📋 6. Özet

| Katman                   | Açıklama                                         |
| ------------------------ | ------------------------------------------------ |
| **Keycloak (Docker)**    | Kimlik sağlayıcı - JWT token üretir              |
| **ASP.NET API**          | Token doğrulama ve rol kontrolü yapar            |
| **JwtBearer Middleware** | Token’ı Keycloak JWKS endpoint’iyle doğrular     |
| **[Authorize]**          | Geçerli token’ı olan isteklere erişim izni verir |

---

## 🚀 7. Kaynaklar

* [Keycloak Documentation](https://www.keycloak.org/documentation)
* [ASP.NET Core Authentication Docs](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/jwt)
* [OpenID Connect Discovery Endpoint](http://localhost:8080/realms/ZB%20DVS/.well-known/openid-configuration)

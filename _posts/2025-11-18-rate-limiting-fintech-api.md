---
title: "Fintech API’lerinde Rate Limiting: Case Study ile Tasarım Prensipleri (Auth, Payments, Webhook)"
date: 2025-11-18
categories: [Fintech, Backend, Architecture]
tags: ["fintech", "rate-limiting", "auth", "payments", "webhook", "dotnet", "api"]
description: "Case study üzerinden Auth, Payments ve Webhook akışları için rate limiting tasarım prensipleri."
author: klckerim
media_subpath: /assets/img/posts/headers
image:
  path: limiting.png
---

Bu yazıda amaç, belirli bir projeyi “birebir reçete” gibi kopyalamak değil; bir fintech API’de rate limiting kararlarını nasıl **modelleyeceğimizi** teknik olarak netleştirmektir.

Kapsam:

- Auth akışları (login, token yenileme, şifre sıfırlama)
- Payment akışları (ödeme başlatma / oturum oluşturma)
- Webhook akışları (sağlayıcı callback trafiği)

## 1) Neden tek bir limit yaklaşımı yerine akış bazlı yaklaşım?

Fintech sistemlerinde her endpoint aynı risk ve maliyet profilini taşımaz:

- **Auth**: parola deneme saldırıları (brute-force), kimlik bilgisi doldurma (credential stuffing) ve hesap ele geçirme riski taşır.
- **Payments**: doğrudan finansal etkisi olan write işlemleri; yanlış limit müşteri kaybı veya işlem başarısızlığına gider.
- **Webhook**: üçüncü parti retry davranışı içerir; agresif throttling entegrasyon stabilitesini bozabilir.

Bu nedenle ilk prensip:

> Rate limiting’i “trafik azaltma” problemi değil, **risk ve iş etkisi segmentasyonu** problemi olarak ele alın.

## 2) Endpoint grupları

Tipik bir fintech API’sinde trafiği üç ana grupta düşünebiliriz:

1. **Auth tarafı** (`/api/v1/auth/...`)
   - `POST /login`, `POST /refresh-token`, `POST /forgot-password`, `POST /reset-password`
2. **Payments tarafı** (`/api/payments/...`)
   - `POST /bill`, `POST /create-session`, `POST /create-setup-session`
3. **Webhook tarafı**
   - `POST /api/payments/webhook`

Bu ayrım ilk bakışta basit görünebilir; ancak operasyonel olarak ciddi fark yaratır.
429 artışlarının güvenlik kaynaklı mı yoksa kapasite kaynaklı mı olduğunu ancak bu gruplar üzerinden net okuyabiliriz.

## 3) Partition key (limit anahtarı) tasarımı

Yalnızca IP adresine dayalı bir limit yaklaşımı, production’da sıkça yanlış pozitif engellemelere yol açar.

Daha sağlıklı bir tasarımda partition key, akışın riskini ve iş bağlamını birlikte yansıtmalıdır:

- **Auth**: `source_ip + user_identifier(hash)`
- **Payments**: `merchant_id/tenant_id + endpoint` 
- **Webhook**: `provider + source`

Buradaki amaç, yalnızca trafiğin geldiği yeri değil; işlemin kime ve hangi bağlamda ait olduğunu da anahtara dahil etmektir.

## 4) .NET 8 RateLimiter API ile kısa, gerçekçi örnek

Bu örnek yalnızca uygulama içi limit ayrımını gösterir; dağıtık sistemlerde veya çoklu instance ortamında sayaçların ortak bir datastore’da tutulması konusuna girmez.

```csharp

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    options.OnRejected = async (ctx, token) =>
    {
        var retryAfterSeconds = 30; // prod ortamda dinamik hesaplanmalı
        ctx.HttpContext.Response.Headers["Retry-After"] = retryAfterSeconds.ToString();

        await ctx.HttpContext.Response.WriteAsJsonAsync(new
        {
            code = "rate_limit_exceeded",
            message = "Rate limit exceeded",
            retryAfterSeconds,
            traceId = ctx.HttpContext.TraceIdentifier
        }, cancellationToken: token);
    };

    options.AddPolicy("AuthSensitive", httpContext =>
        RateLimitPartition.GetSlidingWindowLimiter(
            partitionKey: $"auth:{httpContext.Connection.RemoteIpAddress}:{httpContext.Request.Headers["X-User-Key"]}",
            factory: _ => new SlidingWindowRateLimiterOptions
            {
                PermitLimit = 20,
                Window = TimeSpan.FromMinutes(10),
                SegmentsPerWindow = 10,
                QueueLimit = 0,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst
            }));

    options.AddPolicy("Payments", httpContext =>
        RateLimitPartition.GetTokenBucketLimiter(
            partitionKey: $"pay:{httpContext.Request.Headers["X-Merchant-Id"]}:{httpContext.Request.Path}",
            factory: _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 60,
                TokensPerPeriod = 20,
                ReplenishmentPeriod = TimeSpan.FromSeconds(1),
                AutoReplenishment = true,
                QueueLimit = 0,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst
            }));

    options.AddPolicy("Webhook", httpContext =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: $"wh:{httpContext.Connection.RemoteIpAddress}",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 120,
                Window = TimeSpan.FromMinutes(1),
                QueueLimit = 0,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst
            }));
});
```

Notlar:

- `X-Merchant-Id` / `X-User-Key` gibi değerler, production'da trusted context’ten gelmelidir.
- 429 yanıtında body + `Retry-After` birlikte verilmelidir.

## 5) Distributed rate limiting: mimari seviyede nasıl düşünülmeli?

Uygulama tek instance değilse, sadece process-memory sayaçlarıyla limit doğruluğu korunamaz.

Mimari seviyede yaklaşım:

- **Ortak state** kullanan bir distributed limiter katmanı planla.
- Uygulama içi limiter, hızlı bir koruma katmanı olarak konumlandırılabilir.
- Gerekirse gateway seviyesinde ek koruma uygulayın.

Buradaki hedef, tek bir teknoloji detayı değil; şu dengeyi kurmaktır:

- Düşük gecikme
- Tutarlı limit davranışı
- Operasyonel yönetilebilirlik

## 6) Idempotency + rate limiting: payment güvenliğinin iki ayağı

Payment write endpoint’lerinde şu yanlış varsayım sık görülür:

> “Rate limiting varsa duplicate charge riski düşüktür.”

Bu doğru değildir.

- Rate limiting: aşırı veya kötüye kullanım amaçlı trafiği sınırlar.
- Idempotency: aynı işlemin finansal olarak tekrar yazılmasını engeller.

Bu yüzden:

1. `Idempotency-Key` ödeme yazma uçlarında zorunlu olmalı.
2. 429 sonrası retry aynı idempotency anahtarıyla yapılmalı.
3. Sistem aynı anahtar için deterministic response döndürmeli.

## Sonuç

Rate limiting’i yalnızca “kaç istek/dakika?” sorusuna indirgemek yeterli değildir. Risk seviyesi, algoritma tercihi, idempotency ve dağıtık yapı birlikte değerlendirilmelidir.

Doğru kurgu, hem güvenliği güçlendirir hem de ödeme akışında gereksiz kesintileri azaltır.

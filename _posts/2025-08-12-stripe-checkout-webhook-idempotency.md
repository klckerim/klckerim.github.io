---
title: "Fintech’te Güvenli Ödeme Akışları: Stripe Checkout + Webhook + Idempotency ile Çift İşlem Riskini Önleme"
date: 2025-08-12
categories: [Fintech, Payments, Backend]
tags: ["fintech", "stripe", "checkout", "webhook", "idempotency", "dotnet", "payment"]
description: "Stripe Checkout, webhook signature verification ve idempotency key propagation ile çift işlem riskini azaltan üretim yaklaşımı."
author: klckerim
media_subpath:  /assets/img/posts/headers
image:
    path: fintechpayment.png
---

Fintech ürünlerinde en pahalı hatalardan biri **aynı finansal etkinin iki kez uygulanmasıdır**. Aynı işlem için iki kez tetiklemek, cüzdan bakiyesinin çift artması veya transferin tekrar yazılması; teknik olarak “retry problemi”, operasyonel olarak güven kaybıdır.

- Stripe Checkout ile ödeme oturumu açma
- Webhook doğrulaması (signature verification)
- Idempotency key’in frontend → API → Stripe metadata → webhook hattında taşınması
- Uygulama + veritabanı seviyesinde tekrar işlem engelleme

## 1) Gerçek Problem: Retry = Double Charge Riski

Aşağıdaki senaryolar normaldir:

1. Kullanıcı butona iki kez tıklar.
2. Mobil ağ zayıftır, frontend timeout alır ve isteği tekrar atar.
3. Stripe webhook’u `at least once delivery` modeli nedeniyle aynı event’i yeniden gönderir.
4. Aynı iş olayı backend’de birden fazla kez işlenir.

Eğer tasarım idempotent değilse tek bir iş, birden fazla muhasebe kaydı üretebilir.

## 2) Ödeme Akışı

1. **UI** `Idempotency-Key` üretir.
2. **API** `/api/payments/create-session` çağrısında bu key’i header’dan alır.
3. API, Stripe Checkout Session metadata’sına `walletId`, `amount`, `currency`, `idempotencyKey` yazar.
4. API, Stripe tarafına session create çağrısını da aynı key ile (`RequestOptions.IdempotencyKey`) gönderir.
5. Kullanıcı Stripe sayfasında ödemeyi tamamlar.
6. Stripe, `/api/payments/webhook` endpoint’ine `checkout.session.completed` event’i gönderir.
7. API, `Stripe-Signature` ile event’i doğrular.
8. API metadata’dan `walletId` + `idempotencyKey` okur, `DepositCommand` çağırır.
9. Deposit handler ve DB unique index aynı işlemin tekrar yazılmasını engeller.

## 3) Frontend Tarafı: Idempotency-Key Üretimi ve Gönderimi

UI tarafında key üretimi yardımcı fonksiyonla yapılabilir:

```ts
// /src/shared/lib/idempotency.ts
export const generateIdempotencyKey = (): string => {
  if (typeof globalThis.crypto?.randomUUID === "function") {
    return globalThis.crypto.randomUUID();
  }

  return `${Date.now()}-${Math.random().toString(16).slice(2)}`;
};
```

Ödeme session çağrısında header’a ekleniyor:

```ts
// /src/shared/lib/create-session.ts
const idempotencyKey = generateIdempotencyKey();

await fetch(`${process.env.NEXT_PUBLIC_API_BASE_URL}/api/payments/create-session`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Idempotency-Key": idempotencyKey
  },
  body: JSON.stringify({ walletId, amount, currency })
});
```

> Buradaki amaç, kullanıcı aynı eylemi tekrar etse bile backend’in bunu “aynı niyet” olarak tanıması.

## 4) Backend: Checkout Session Oluşturma ve Metadata Propagation

`PaymentsController.CreateCheckoutSession` içinde key header’dan okunuyor ve Stripe metadata’ya geçiriliyor:

```csharp
[HttpPost("create-session")]
public IActionResult CreateCheckoutSession([FromBody] CheckoutRequest request, [FromHeader(Name = "Idempotency-Key")] string? idempotencyKey)
{
    var metadata = new Dictionary<string, string>
    {
        { "walletId", request.WalletId },
        { "amount", request.Amount.ToString() },
        { "currency", request.Currency }
    };

    if (!string.IsNullOrWhiteSpace(idempotencyKey))
    {
        metadata["idempotencyKey"] = idempotencyKey;
    }

    var options = new SessionCreateOptions
    {
        Mode = "payment",
        SuccessUrl = _config["Stripe:SuccessUrl"],
        CancelUrl = _config["Stripe:CancelUrl"],
        Metadata = metadata,
        PaymentIntentData = new SessionPaymentIntentDataOptions
        {
            Metadata = metadata
        }
    };

    var service = new SessionService();
    Session session;

    if (!string.IsNullOrWhiteSpace(idempotencyKey))
    {
        session = service.Create(options, new RequestOptions { IdempotencyKey = idempotencyKey });
    }
    else
    {
        session = service.Create(options);
    }

    return Ok(new { sessionId = session.Id });
}
```

Burada iki ayrı güvenlik katmanı birlikte çalışıyor:

- Aynı key metadata’da taşındığı için webhook işleme sırasında business context, önceki HTTP isteğine bağımlı kalmadan yeniden oluşturulabilir.
- Aynı key `RequestOptions.IdempotencyKey` ile Stripe API çağrısına da verildiği için, `create-session` isteğinin ağ retry senaryolarında Stripe tarafında da duplicate session üretme riski düşüyor.

Not: Webhook geldiğinde uygulama önceki HTTP request state’ine bağımlı kalmadan, ihtiyacı olan bilgiyi Stripe event metadata’sından okur.

## 5) Webhook Güvenliği: Signature Verification (Zorunlu)

Webhook endpoint’inde event doğrulaması yapılmadan işlem yapılmıyor:

```csharp
var json = await new StreamReader(HttpContext.Request.Body).ReadToEndAsync();
var secret = _config["Stripe:WebhookSecret"];

stripeEvent = Stripe.EventUtility.ConstructEvent(
    json,
    Request.Headers["Stripe-Signature"],
    secret
);
```

Bu adım yoksa endpoint sahte isteklerle suistimal edilebilir. Doğrulama başarısızsa doğrudan `BadRequest` dönülebilir.

## 6) `checkout.session.completed` Event’i ile Deposit Entegrasyonu

Webhook event tipi `checkout.session.completed` olduğunda ödeme modunda şu işlem yapılıyor:

- `walletId` metadata’dan alınıyor
- `amount` Stripe session’dan okunuyor (`AmountTotal / 100m`)
- `idempotencyKey` metadata’dan alınıyor
- `DepositCommand(walletId, amount, idempotencyKey)` çağrılıyor

Basitleştirilmiş akış:

```csharp
if (stripeEvent.Type == EventTypes.CheckoutSessionCompleted)
{
    var session = stripeEvent.Data.Object as Stripe.Checkout.Session;

    if (session?.Mode == "payment" && session.Metadata != null &&
        session.Metadata.TryGetValue("walletId", out var walletIdStr) &&
        Guid.TryParse(walletIdStr, out var walletGuid))
    {
        var amount = session.AmountTotal.HasValue ? session.AmountTotal.Value / 100m : 0m;
        session.Metadata.TryGetValue("idempotencyKey", out var idempotencyKeyValue);

        await _mediator.Send(new DepositCommand(walletGuid, amount, idempotencyKeyValue));
    }
}
```

## 7) Uygulama Katmanı Idempotency: Handler Seviyesinde Tekrarı Kesme

Deposit handler, yazmadan önce önceki işlemi arıyor:

```csharp
if (!string.IsNullOrWhiteSpace(request.IdempotencyKey))
{
    var existing = await _transactionRepository.GetByIdempotencyKeyAsync(
        request.IdempotencyKey,
        TransactionType.Deposit,
        cancellationToken);

    if (existing != null)
    {
        return true;
    }
}
```

Kayıt yoksa wallet güncelleniyor ve transaction yazılıyor. Bu sayede aynı event ikinci kez işlenirse finansal etki tekrarlanmıyor.

## 8) DB Katmanı Idempotency: Race Condition Problemi

Sadece handler kontrolü yetmez; eşzamanlı isteklerde race condition olabilir. Bu nedenle migration ile `(IdempotencyKey, Type)` unique index ekleyebiliriz.

```csharp
migrationBuilder.CreateIndex(
    name: "IX_Transactions_IdempotencyKey_Type",
    table: "Transactions",
    columns: new[] { "IdempotencyKey", "Type" },
    unique: true);
```

Bu, uygulama katmanı kaçırsa bile veritabanı seviyesinde atomic bir bariyer oluşturarak aynı işlemin ikinci kez sonuca etki etmesini engeller.

## Sonuç

“Ödeme başarılı” demek yeterli değil; **tek bir iş niyetinin tek bir durumu etkilemesi** gerekir.

**Idempotency key propagation** , **Webhook signature verification** , **Application check + DB unique index** kombinasyonu çift işlem riskini ciddi biçimde düşürür.

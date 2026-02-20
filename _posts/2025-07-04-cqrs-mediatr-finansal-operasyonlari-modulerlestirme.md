---
title: "CQRS + MediatR ile Finansal Operasyonları Modülerleştirme"
date: 2025-07-04
categories: [Backend, Architecture]
tags: ["cqrs", "mediatr", "fintech", "dotnet", "idempotency", "outbox", "ddd"]
description: "Deposit, Transfer ve PayBill örnekleriyle finansal operasyonları CQRS + MediatR yaklaşımıyla modülerleştirme."
author: klckerim
media_subpath:  /assets/img/posts/headers
image:
    path: mediatr.png
---

## (Deposit / Transfer / PayBill Örnekleri)

Finansal uygulamalarda en zor başlıklardan biri, zamanla büyüyen iş kurallarını **bozmadan** yeni işlem türleri ekleyebilmektir. İlk sürümde çalışan bir `WalletService` çoğu zaman birkaç ay sonra yüzlerce satırlık "god class" haline gelir:

- para yatırma,
- para transferi,
- fatura ödeme,
- ücret/komisyon,
- limit kontrolleri,
- idempotency,
- audit/logging,
- bildirim üretimi…

Hepsi tek bir yerde toplanır.

Bu yazıda bu problemi **CQRS + MediatR** yaklaşımıyla nasıl modüler bir yapıya dönüştürebileceğimizi, özellikle şu üç işlem üzerinden anlatacağım:

1. **Deposit** (hesaba para yükleme)
2. **Transfer** (cüzdanlar arası para gönderimi)
3. **PayBill** (fatura ödeme)

---

## Neden CQRS?

CQRS (Command Query Responsibility Segregation), sistemi ikiye ayırır:

- **Command tarafı**: state değiştirir (`Create`, `Deposit`, `Transfer`, `PayBill`)
- **Query tarafı**: veri okur (`GetWalletBalance`, `GetTransactionHistory`)

Finansal operasyonlarda command tarafı çoğunlukla karmaşık iş kurallarına sahiptir. Query tarafı ise performans ve okunabilirlik odaklıdır. Bu iki dünyayı ayırmak, özellikle yüksek değişim hızında büyük avantaj sağlar:

- Her işlem için bağımsız handler
- Daha net test sınırları
- Daha az yan etki
- İşlem bazlı yetkilendirme/validasyon

---

## MediatR neden iyi eşlik eder?

MediatR ile her işlemi bir `Command` + `Handler` çifti olarak modelleyebilirsiniz:

- `DepositCommand` → `DepositCommandHandler`
- `TransferCommand` → `TransferCommandHandler`
- `PayBillCommand` → `PayBillCommandHandler`

Controller sadece komutu oluşturur ve mediator’a gönderir:

```csharp
[HttpPost("deposit")]
public async Task<IActionResult> Deposit([FromBody] DepositRequest request)
{
    var command = new DepositCommand(UserId, request.WalletId, request.Amount, request.Currency);
    var result = await _mediator.Send(command);
    return Ok(result);
}
```

Böylece controller "iş" yapmaz; sadece giriş/çıkış katmanı olarak kalır.

---

## Önerilen klasörleme

Uygulama katmanında işlem bazlı (feature-first) yapı, finans uygulamalarında çok iyi ölçeklenir:

```text
Application/
  Features/
    Wallets/
      Commands/
        Deposit/
          DepositCommand.cs
          DepositCommandHandler.cs
          DepositCommandValidator.cs
        Transfer/
          TransferCommand.cs
          TransferCommandHandler.cs
          TransferCommandValidator.cs
        PayBill/
          PayBillCommand.cs
          PayBillCommandHandler.cs
          PayBillCommandValidator.cs
      Queries/
        GetWalletById/
        GetTransactionHistory/
```

Bu yapı sayesinde bir geliştirici "Transfer akışını" anlamak için proje içinde kaybolmaz; ilgili klasöre girip tüm parçaları görür.

---

## Örnek 1: Deposit akışı

### Command

```csharp
public sealed record DepositCommand(
    Guid UserId,
    Guid WalletId,
    decimal Amount,
    string Currency
) : IRequest<DepositResult>;
```

### Validator (FluentValidation)

```csharp
public class DepositCommandValidator : AbstractValidator<DepositCommand>
{
    public DepositCommandValidator()
    {
        RuleFor(x => x.Amount).GreaterThan(0);
        RuleFor(x => x.Currency).NotEmpty().Length(3);
        RuleFor(x => x.WalletId).NotEmpty();
    }
}
```

### Handler (özet)

```csharp
public class DepositCommandHandler : IRequestHandler<DepositCommand, DepositResult>
{
    private readonly IWalletRepository _walletRepo;
    private readonly IUnitOfWork _uow;

    public async Task<DepositResult> Handle(DepositCommand cmd, CancellationToken ct)
    {
        var wallet = await _walletRepo.GetByIdAsync(cmd.WalletId, ct);
        if (wallet is null || wallet.UserId != cmd.UserId)
            throw new NotFoundException("Wallet not found");

        wallet.Deposit(cmd.Amount, cmd.Currency); // Domain rule

        await _uow.SaveChangesAsync(ct);

        return new DepositResult(wallet.Id, wallet.Balance.Amount, wallet.Balance.Currency);
    }
}
```

### Neden temiz?

- Validasyon ayrı
- İş kuralı (`wallet.Deposit`) domain’de
- Persist işlemi tek noktada (`UnitOfWork`)
- Test edilmesi kolay bir handler

---

## Örnek 2: Transfer akışı

Transfer, deposit’e göre daha riskli çünkü iki hesap etkilenir. Burada **atomiklik** kritik.

### Command

```csharp
public sealed record TransferCommand(
    Guid SenderUserId,
    Guid FromWalletId,
    Guid ToWalletId,
    decimal Amount,
    string Currency,
    string IdempotencyKey
) : IRequest<TransferResult>;
```

### Handler’da dikkat edilmesi gerekenler

1. Gönderen cüzdan kullanıcıya ait mi?
2. Alıcı cüzdan mevcut mu?
3. Para birimi uyumlu mu?
4. Bakiye yeterli mi?
5. Aynı istek tekrarlandı mı? (idempotency)
6. Tüm değişiklikler tek transaction’da mı?

```csharp
public async Task<TransferResult> Handle(TransferCommand cmd, CancellationToken ct)
{
    if (await _idempotencyStore.ExistsAsync(cmd.IdempotencyKey, ct))
        return await _idempotencyStore.GetResultAsync<TransferResult>(cmd.IdempotencyKey, ct);

    await _uow.BeginTransactionAsync(ct);

    var from = await _walletRepo.GetByIdAsync(cmd.FromWalletId, ct);
    var to = await _walletRepo.GetByIdAsync(cmd.ToWalletId, ct);

    from.Debit(cmd.Amount, cmd.Currency);
    to.Credit(cmd.Amount, cmd.Currency);

    await _uow.SaveChangesAsync(ct);
    await _uow.CommitTransactionAsync(ct);

    var result = new TransferResult(from.Id, to.Id, cmd.Amount, cmd.Currency);
    await _idempotencyStore.SaveAsync(cmd.IdempotencyKey, result, ct);

    return result;
}
```

> Pratik not: Transfer için optimistic concurrency token (ör. row version) kullanmak, race condition durumlarında veri tutarlılığını korumada çok yardımcı olur.

---

## Örnek 3: PayBill akışı

`PayBill`, transfere benzer ama genelde ekstra domain kuralları içerir:

- fatura son ödeme tarihi,
- tekil fatura numarası,
- tekrar ödeme engeli,
- komisyon/fee hesapları,
- external provider doğrulaması.

### Command

```csharp
public sealed record PayBillCommand(
    Guid UserId,
    Guid WalletId,
    string BillerCode,
    string BillNumber,
    decimal Amount,
    string Currency,
    string IdempotencyKey
) : IRequest<PayBillResult>;
```

### Handler sorumlulukları (özet)

- Fatura provider’ından "ödenebilir" kontrolü
- Cüzdan bakiyesi kontrolü
- Ödeme + transaction kaydı + bill state değişimi
- Başarılı işlem sonrası `BillPaidIntegrationEvent` yayınlama

Bunu command tarafında tutup query tarafında "ödenen faturalar" ekranını ayrı optimize etmek, ileride performans iyileştirmelerini çok kolaylaştırır.

---

## Pipeline Behaviors ile ortak teknik ihtiyaçları tek noktada toplamak

MediatR’ın en güçlü yanlarından biri `PipelineBehavior` kullanımıdır. Böylece her handler içine aynı kodları yazmak zorunda kalmazsınız.

### Yaygın davranışlar

- **ValidationBehavior**: FluentValidation çalıştırır
- **LoggingBehavior**: request/response loglar
- **PerformanceBehavior**: yavaş handler’ları işaretler
- **TransactionBehavior**: gerekli command’ları transaction içine alır
- **IdempotencyBehavior**: tekrar çağrılarda aynı sonucu döner

```csharp
services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(ApplicationAssemblyMarker).Assembly));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(IdempotencyBehavior<,>));
```

Bu yaklaşım, handler’ları "sadece iş kuralı" seviyesine indirir.

---

## Domain Event + Outbox ile güvenilir entegrasyon

Finansal işlemlerde "DB’ye yazdım ama event publish edemedim" problemi kritik olabilir.

Önerilen yaklaşım:

1. Handler domain state’i değiştirir.
2. Domain event üretir (`MoneyTransferredDomainEvent`, `BillPaidDomainEvent`).
3. Event, outbox tablosuna aynı transaction’da yazılır.
4. Background worker outbox kayıtlarını güvenli şekilde publish eder.

Böylece "at-least-once" güvenilirlik modeli gerçek hayata uygun biçimde kurulabilir.

> Not: Outbox yaklaşımı sistemin eventual consistency modeline geçmesini gerektirir; bu da bazı akışlarda anlık tutarlılık yerine gecikmeli senkronizasyon anlamına gelir.

---

## Test stratejisi

Bu mimari en çok testte fark yaratır.

### 1) Unit Test (Handler)

- `TransferCommandHandler_Should_Fail_When_Balance_Insufficient`
- `PayBillCommandHandler_Should_ReturnSameResult_When_IdempotencyKey_Reused`

### 2) Integration Test (DB + Transaction)

- Aynı anda iki transfer isteğinde tutarlılık korunuyor mu?
- Rollback senaryolarında ara state kalıyor mu?

### 3) Contract / API Test

- Controller → Mediator mapping doğru mu?
- Hata kodları (400/404/409) API’nin tanımlanan davranış modeliyle tutarlı mı?

---

## Gerçek hayatta sık yapılan hatalar

- Her şeyi tek `WalletService` içine koymak
- Query tarafında da domain entity döndürmek
- Idempotency’yi sadece UI tarafına bırakmak
- Transaction sınırlarını belirsiz bırakmak
- Handler içinde dış servislere direkt bağımlılık (soyutlama olmadan)

---

## Sonuç

`CQRS + MediatR`, finansal operasyonları sadece "düzenli" yapmak için değil, **değişime dayanıklı** hale getirmek için çok etkili bir kombinasyon.

Özellikle `Deposit`, `Transfer` ve `PayBill` gibi farklı risk profiline sahip işlemlerde:

- her use-case’i ayrı command/handler ile modellemek,
- ortak teknik ihtiyaçları pipeline behavior’lara taşımak,
- tutarlılık için transaction + idempotency + outbox üçlüsünü birlikte kullanmak,

uygulamanın hem bakım maliyetini düşürür hem de üretim güvenilirliğini artırır.

Eğer mevcut kod tabanınızda finansal akışlar tek bir servis içinde sıkıştıysa, küçük bir adımla başlayın: önce sadece `Transfer` akışını command-handler yapısına çıkarın. Genelde ekipler en büyük kazanımı burada görüyor.

> Not: `CQRS + MediatR` her sistem için zorunlu değildir. Basit CRUD uygulamalarında gereksiz soyutlama maliyeti yaratabilir. Ancak iş kuralları arttıkça ve finansal risk yükseldikçe ayrıştırılmış bir yapı sürdürülebilirlik açısından ciddi avantaj sağlar.

---
title: "Backend AI Integration: Nereden Başlamalı?"
date: 2026-02-18
categories: [Backend, AI]
tags: ["backend", "ai", "api", "entegrasyon", "openai", "dotnet"]
description: "Backend sistemlere AI özelliklerini kısa ve anlaşılır şekilde entegre etmenin temel adımları."
author: klckerim
---

Backend tarafında AI entegrasyonu göz korkutucu görünebilir; ama doğru adımlarla oldukça yönetilebilir bir süreçtir.

## 1) Küçük bir kullanım senaryosu seç
Önce tüm ürünü dönüştürmeye çalışmak yerine tek bir problemle başla:
- destek mesajı sınıflandırma,
- metin özetleme,
- arama sonuçlarını daha akıllı sıralama.

Bu yaklaşım, hem maliyeti hem de riskleri düşük tutar.

## 2) AI çağrılarını servis katmanına al
AI sağlayıcısına yapılan istekleri doğrudan controller içine yazmak yerine ayrı bir servis katmanı oluşturmak daha sağlıklı olacaktır. Böylece:
- sağlayıcı değişikliği kolaylaşır,
- loglama ve hata yönetimi merkezileşir,
- test yazmak daha kolay olur.

## 3) Prompt ve çıktı kontrolü ekley
Prod ortamında en kritik konu tutarlılıktır. Bunun için:
- sabit prompt şablonları kullan,
- JSON gibi yapılandırılmış çıktı,
- beklenmeyen çıktıları doğrulama katmanında yakala.

## 4) Gözlemlenebilirlik ve limitler
AI istekleri için mutlaka şu metrikleri izle:
- gecikme süresi,
- hata oranı,
- istek başı maliyet.

Ayrıca rate limit, timeout ve retry politikaları belirleyerek sistemi dayanıklı hale getirebiliriz.

## OpenAI + .NET için hızlı teknik örnek
Aşağıdaki yapı, production ortama uygun bir başlangıç verir:

1. API anahtarını `appsettings` yerine environment variable olarak tut (`OPENAI_API_KEY`).
2. `HttpClientFactory` ile dış istekleri yönetebiliriz.
3. AI çağrısını tek bir servis içinde topla (`IAiService`).
4. Yanıtı DTO + doğrulama ile uygulamaya al.

```csharp
// Program.cs
builder.Services.AddHttpClient<IAiService, OpenAiService>(client =>
{
    client.BaseAddress = new Uri("https://api.openai.com/v1/");
    client.Timeout = TimeSpan.FromSeconds(20);
});
```

```csharp
// OpenAiService.cs (özet)
public sealed class OpenAiService : IAiService
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _configuration;

    public OpenAiService(HttpClient httpClient, IConfiguration configuration)
    {
        _httpClient = httpClient;
        _configuration = configuration;
    }

    public async Task<string> SummarizeAsync(string text, CancellationToken ct)
    {
        var apiKey = _configuration["OPENAI_API_KEY"]
            ?? throw new InvalidOperationException("OPENAI_API_KEY missing");

        using var request = new HttpRequestMessage(HttpMethod.Post, "responses");
        request.Headers.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", apiKey);

        request.Content = JsonContent.Create(new
        {
            model = "gpt-4.1-mini", 
            input = $"Aşağıdaki metni 3 maddede özetle:\n{text}"
        });

        using var response = await _httpClient.SendAsync(request, ct);
        response.EnsureSuccessStatusCode();

        var json = await response.Content.ReadFromJsonAsync<JsonElement>(cancellationToken: ct);
        return json.GetRawText(); // prodda DTO map + alan doğrulama önerilir.
    }
}
```

### Prod ipuçları (.NET)
- `Polly` ile retry + circuit breaker eklemek.
- Her istek için `CorrelationId` loglanması.
- Maliyet kontrolü için token kullanımı ve model bazlı rapor .
- Hassas verileri prompt’a göndermeden önce maskelemeyi unutmamak gerekir (PII redaction).

## Kısa özet
Başarılı backend AI entegrasyonu için anahtar fikir şudur: **önce küçük başla, servisi soyutla, çıktıyı doğrula, maliyeti izle**. Bu dört adım, lokal testlerden sonra prod geçişini çok daha güvenli yapar.

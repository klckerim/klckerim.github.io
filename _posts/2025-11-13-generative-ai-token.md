---
title: "Generative AI'da Token Mantığı: Maliyet, Hız ve Kaliteyi Birlikte Yönetmek"
date: 2025-11-13
categories: [AI, Engineering]
tags: ["ai", "llm", "token", "prompt"]
description: "Token kavramını sade bir dille; tahmin, fiyatlandırma ve gerçek maliyet hesabı örnekleriyle anlattım."
author: klckerim
media_subpath: /assets/img/posts/headers
image:
  path: ai-token.png
---

LLM tarafında maliyet hesabı yaparken en kritik kelime: **token**.  
Modelin gördüğü her metin (sorduğun şey + modelin ürettiği cevap) token'a bölünüyor ve ücretlendirme bunun üzerinden ilerliyor.

Bu yazıda token konusunu günlük geliştirme pratiğine uygun, sade bir dille toparlıyorum.

## Token tam olarak nedir?

Token, modelin metni işlerken kullandığı küçük parçalardır. Kelime gibi düşünebilirsin ama birebir kelime değildir.

Kısa örnekler:

- `Hello world!` → 3 token (`Hello`, `world`, `!`)
- `temperature=0.7` → 5 token (`temperature`, `=`, `0`, `.`, `7`)
- `tokenization` → 2 token (`token`, `ization`)

Yani model için önemli olan "kaç kelime" değil, "kaç token" olduğudur.

## Neden bu kadar önemli?

Token sayısı doğrudan 4 şeyi etkiler:

1. **Maliyet:** Girdi ve çıktı token'ları ayrı ücretlenir.
2. **Gecikme süresi (latency):** Token yükseldikçe yanıt süresi uzayabilir.
3. **Bağlam sınırı (context window):** Her modelin tek seferde okuyabildiği bir üst limit var (200K, 1M, 2M gibi).
4. **Çıktı kalitesi:** Gereksiz uzun prompt, modeli dağıtabilir; temiz token yönetimi kaliteyi artırır.

> Pratik not: Çoğu sağlayıcıda çıktı token'ları, girdi token'larından daha pahalıdır (genellikle 2–4 kat).

## Hızlı token tahmin tablosu

Aşağıdaki değerler günlük planlama için gayet iş görür:

| Metin Miktarı | Yaklaşık Token Gerçeği | Gerçek Hayat Karşılığı             |
| ------------- | ---------------------: | ---------------------------------- |
| 1 token       |  1–4 karakter olabilir | “AI”, “.”, “=”, “-”                |
| 1 token       |    ~0.5–1 kelime arası | Kelimenin tamamı ya da bir parçası |
| 100 token     |          ~60–90 kelime | 1 kısa paragraf                    |
| 500 token     |        ~300–450 kelime | Teknik e-posta / mini makale       |
| 2,000 token   |    ~1,200–1,700 kelime | Detaylı blog yazısı                |
| 10,000 token  |    ~6,000–8,500 kelime | Uzun teknik dokümantasyon          |
| 100,000 token |  ~60,000–85,000 kelime | Küçük bir kitap + ciddi kod        |

> Türkçe, özel karakterler ve kod blokları bu oranları değiştirebilir. En doğrusu kendi içeriğinle ölçüm almak.

## Token sayımı için kullanılan araçlar

### 1) OpenAI tarafı
En yaygın yaklaşım `tiktoken` ile sayım almak.

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")
prompt = "Bu prompt kaç token ediyor?"
count = len(enc.encode(prompt))
print(count)
```

Ayrıca web araçları da var:
- OpenAI Tokenizer
- Tiktokenizer (parçalanmayı görsel gösterir)

### 2) Anthropic (Claude) tarafı
Claude için en güvenlisi resmi SDK/API ile token saydırmak.

```python
import anthropic

client = anthropic.Anthropic(api_key="YOUR_KEY")
result = client.messages.count_tokens(
    model="claude-sonnet-4.5",
    messages=[{"role": "user", "content": "Kısa bir deneme metni"}]
)
print(result.input_tokens)
```

### 3) Google Gemini tarafı
Gemini'de `count_tokens` (veya API karşılığı) ile net sayım alınabilir.

```python
import google.generativeai as genai

model = genai.GenerativeModel("gemini-2.5-flash")
count = model.count_tokens("Örnek prompt").total_tokens
print(count)
```

## 2025 sonu için model fiyat karşılaştırması

> Not: Fiyatlar zamanla güncellenebilir. Buradaki tablo, verilen referans tarihle aynı değerleri içerir.

### OpenAI (Ekim 2025)

| Model       | Input (1M token) | Output (1M token) | Context window | En uygun senaryo                      |
| ----------- | ---------------: | ----------------: | -------------: | ------------------------------------- |
| GPT-5       |            $6.00 |            $18.00 |           272K | Karmaşık akıl yürütme, agent akışları |
| GPT-5 Mini  |            $0.40 |             $1.20 |           272K | Hızlı ve düşük maliyetli işler        |
| GPT-4.1     |            $2.00 |             $8.00 |             1M | Uzun bağlam, üretim sistemleri        |
| GPT-4o      |            $2.50 |            $10.00 |           128K | Multimodal kullanım                   |
| GPT-4o Mini |            $0.15 |             $0.60 |           128K | Yüksek hacim, basit görevler          |

### Anthropic Claude

| Model             | Input (1M token) | Output (1M token) | Context window | En uygun senaryo               |
| ----------------- | ---------------: | ----------------: | -------------: | ------------------------------ |
| Claude Opus 4.1   |           $15.00 |            $75.00 |           200K | Çok karmaşık agent senaryoları |
| Claude Sonnet 4.5 |            $3.00 |            $15.00 |           200K | Denge, kodlama, agent işleri   |
| Claude Sonnet 4   |            $3.00 |            $15.00 |      1M (beta) | Büyük ölçekli kod analizi      |
| Claude Haiku 3.5  |            $0.25 |             $1.25 |           200K | Hız ve yoğun trafik            |

### Google Gemini

| Model                 | Input (1M token) | Output (1M token) | Context window | En uygun senaryo         |
| --------------------- | ---------------: | ----------------: | -------------: | ------------------------ |
| Gemini 2.5 Pro        |            $1.25 |             $5.00 |             2M | Derin akıl yürütme, kod  |
| Gemini 2.5 Flash      |            $0.30 |             $1.20 |             1M | Hız + kalite dengesi     |
| Gemini 2.5 Flash-Lite |           $0.075 |             $0.30 |             1M | Ultra düşük maliyet      |
| Gemini 2.0 Flash      |            $0.10 |             $0.40 |             1M | Günlük production işleri |

## Gerçek hayattan maliyet hesabı

Senaryo: Aylık **25,000** müşteri konuşması, model olarak **Claude Sonnet 4.5** kullanıyoruz.

- Ortalama input: **800 token**
- Ortalama output: **400 token**

Hesap:

- Toplam input token = 800 × 25,000 = 20,000,000
- Input maliyeti = 20,000,000 / 1,000,000 × $3.00 = $60.00

- Toplam output token = 400 × 25,000 = 10,000,000
- Output maliyeti = 10,000,000 / 1,000,000 × $15.00 = $150.00

- Toplam aylık maliyet = **$210.00**

## Kapanış

Özetle token konusu sadece "fiyat" değil; aynı anda performans, bağlam yönetimi ve çıktı kalitesi demek. Model seçimi "en güçlü"ye göre değil, işin ihtiyacına göre yapılmalıdır. Doğru token yönetimi, AI projesinin sürdürülebilir olmasını sağlar.

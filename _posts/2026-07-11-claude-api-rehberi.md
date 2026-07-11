---
title: "Claude API Rehberi: Prompt Mühendisliğinden Agent Mimarisine"
date: 2026-07-11
categories: [AI, Software Engineering]
tags: ["ai", "claude", "anthropic", "llm", "prompt-engineering", "rag", "mcp", "agent"]
description: "Claude API üzerine çıkardığım notların özeti: model seçiminden prompt mühendisliğine, tool use'dan RAG, MCP ve agent mimarisine kadar pratik bir yol haritası."
author: klckerim
media_subpath: /assets/img/posts/headers
image:
  path: claude-api-course.png
---

Son zamanlarda Claude API üzerine kapsamlı bir eğitim geçtim ve notlarımı toparlarken fark ettim ki, konular tek başına dağınık kalınca "nereden başlamalı" sorusu havada kalıyor. Bu yazıda, bir backend geliştiricinin Claude API ile ürün geliştirirken sırasıyla karşılaşacağı konuları; model seçiminden prompt mühendisliğine, tool use'dan RAG'a, MCP'den agent mimarisine kadar tek bir akışta topluyorum.

## 1) Doğru modeli seçmek

Claude üç model ailesi sunuyor, her biri farklı bir önceliğe göre optimize edilmiş:

| Model | Öncelik | Trade-off | Ne zaman kullanılır |
| ----- | ------- | --------- | -------------------- |
| **Opus** | En yüksek zeka | Yüksek maliyet, yüksek gecikme | Karmaşık, çok adımlı, derin akıl yürütme gerektiren işler |
| **Sonnet** | Denge | Zeka/hız/maliyet dengeli | Çoğu pratik senaryo, güçlü kod yazma/düzenleme |
| **Haiku** | Hız | Reasoning yeteneği yok | Gerçek zamanlı, yüksek hacimli kullanıcı etkileşimleri |

Pratikte tek bir model seçip her yere yaymak yerine, **aynı uygulama içinde birden fazla modeli** görev bazlı kullanmak çok daha yaygın bir yaklaşım: sınıflandırma gibi basit işleri Haiku'ya, kullanıcıya görünen üretimi Sonnet'e, kritik planlama adımlarını Opus'a bırakmak gibi.

## 2) API çağrısının perde arkası

Bir isteğin sunucudan modele gidip geri dönme süreci dört aşamadan oluşuyor:

1. **Tokenization** — girdi metin kelime/parça/sembol seviyesinde token'lara bölünür.
2. **Embedding** — her token, anlamını temsil eden sayı dizisine çevrilir.
3. **Contextualization** — komşu token'lara bakılarak anlam netleştirilir (aynı kelime farklı bağlamda farklı anlam kazanır).
4. **Generation** — çıktı katmanı bir sonraki token için olasılık dağılımı üretir, model bunlardan birini seçer, ekler, tekrar eder.

Model, `max_tokens` limitine ulaştığında ya da bir bitiş token'ı ürettiğinde durur; bu bilgi yanıtta `stop_reason` alanında dönüyor. API tarafında zorunlu üç parametre değişmiyor: `model`, `max_tokens`, `messages`. Unutulmaması gereken kritik nokta: **API key'i asla client tarafında tutmayın** — istek her zaman kendi sunucunuz üzerinden Anthropic API'ye gitmeli.

```python
import anthropic

client = anthropic.Anthropic()
message = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1000,
    messages=[{"role": "user", "content": "Kısaca RAG nedir?"}],
)
print(message.content[0].text)
```

## 3) Claude hiçbir şeyi hatırlamaz

Anthropic API tamamen **stateless**: her istek bağımsızdır, önceki konuşmadan hiçbir hafıza taşımaz. Çok turlu bir sohbet istiyorsanız, `role: user/assistant` çiftlerinden oluşan mesaj listesini siz tutup her isteğe komple geçmişle birlikte göndermeniz gerekir.

Bunun yanında üç parametre günlük kullanımda sık devreye giriyor:

- **System prompt** — modelin *ne* söyleyeceğini değil, *nasıl* davranacağını belirler (rol, ton, sınırlar). "Matematik öğretmeni gibi davran, direkt cevap verme, ipucu ver" gibi.
- **Temperature (0–1)** — çıktının rastgeleliğini kontrol eder. 0'a yakın değerler veri çıkarımı ve tutarlılık gerektiren işlerde, 1'e yakın değerler ise beyin fırtınası/yaratıcı yazım gibi işlerde tercih edilir.
- **Streaming** — 10-30 saniye sürebilen yanıtları parça parça (chunk) kullanıcıya akıtmak için kullanılır. `content_block_delta` event'i asıl metin parçalarını taşır; SDK'daki `client.messages.stream()` ile `text_stream` üzerinden bunu kolayca tüketebilirsiniz.

## 4) Çıktıyı disipline etmek: prefill + stop sequence

Claude, JSON veya kod istediğinizde bile doğal olarak açıklama ve markdown başlıkları eklemeye meyillidir. Ham, doğrudan kopyalanabilir çıktı almak için iki teknik birlikte kullanılır:

- **Assistant message prefilling** — mesaj listesinin sonuna, modelin "zaten yazmaya başladığı" bir assistant mesajı ekleyip yanıtı oradan devam ettirmesini sağlarsınız (ör. `` ```json `` ile başlatmak).
- **Stop sequence** — model belirli bir string'i ürettiği an durur (ör. kapanış `` ``` ``).

İkisi birlikte kullanıldığında Claude, açıklama/başlık eklemeden sadece istenen veriyi üretir — bu desen JSON, Python kodu, regex gibi her türlü yapılandırılmış çıktı için işe yarar.

## 5) Prompt mühendisliği: skorları katlayan teknikler

Bir prompt'u bir-iki kez test edip production'a göndermek klasik bir tuzak; objektif ölçüm için **eval pipeline** (dataset + prompt + LLM + grader) kurup, her değişiklikten sonra puanlamak çok daha güvenilir. Bu kursta örnek bir "atlet için öğün planı" promptu üzerinden teknikler tek tek eklenip skor değişimi gözlemleniyor:

1. **Net ve doğrudan olun** — ilk cümlede eylem fiili + net görev tanımı (`"Bir günlük öğün planı oluştur..."`). Örnekte skor **2.32 → 3.92**'ye çıkıyor.
2. **Spesifik olun** — iki tür yönlendirme eklenir: *attribute* (çıktının uzunluk/format/yapı gibi nitelikleri) ve *steps* (modelin hangi adımlarla düşüneceği). Attribute hemen her promptta önerilir, steps ise modelin kendiliğinden düşünmeyeceği bakış açılarını zorlamak için kullanılır. Bu adımla skor **3.92 → 7.86**'ya çıkıyor.
3. **XML tag'leriyle yapılandırın** — prompt içine büyük içerik gömerken (`<sales_records>`, `<athlete_information>` gibi) tanımlayıcı etiketler kullanmak, modelin içerik sınırlarını netçe ayırt etmesini sağlar.
4. **Örnek verin (one-shot/multi-shot)** — özellikle sarkazm tespiti gibi kenar durumlar veya karmaşık format beklentileri için, ideal çıktıyı gösteren örnekleri XML etiketleriyle sarıp *neden ideal olduğunu* açıklamak sonucu daha da güçlendiriyor.

## 6) Tool use: Claude'a dış dünyayı açmak

Claude varsayılan olarak sadece eğitim verisini bilir, canlı bilgiye erişemez. **Tool use**, modelin ihtiyaç duyduğunda sizin tanımladığınız fonksiyonları "çağırmasını" istemesini sağlayan mekanizma:

1. İsteğe, fonksiyonu tarif eden bir **JSON schema** (`name`, `description`, `input_schema`) eklersiniz.
2. Claude ek veriye ihtiyaç duyarsa, düz metin yerine bir `tool_use` bloğu döner (fonksiyon adı + argümanlar).
3. Sunucu ilgili fonksiyonu çalıştırır, sonucu bir `tool_result` bloğuna (aynı `tool_use_id` ile eşleşecek şekilde) koyup **komple geçmişle birlikte** tekrar Claude'a gönderir.
4. `stop_reason == "tool_use"` olduğu sürece bu döngü devam eder; metin yanıtı geldiğinde durur.

```python
while True:
    response = client.messages.create(
        model=model, max_tokens=1000, messages=messages, tools=tools
    )
    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason != "tool_use":
        break

    tool_results = [run_tool(block) for block in response.content if block.type == "tool_use"]
    messages.append({"role": "user", "content": tool_results})
```

Birkaç pratik nokta:

- Fonksiyonlar sıradan Python kodu olarak yazılır; **açıklayıcı isimler + input validasyonu + anlamlı hata mesajları** şart, çünkü hata mesajı Claude'a geri gidiyor ve model düzeltip tekrar deneyebiliyor.
- Claude tek seferde birden fazla tool çağırabildiği halde bunu nadiren kendiliğinden yapar; paralel çalıştırmayı zorlamak için tek bir **batch tool** tanımlayıp, birden fazla çağrıyı tek `invocations` listesi içinde toplamak yaygın bir hile.
- Hazır (built-in) tool'lar da var: **web search** (güncel bilgi, `allowed_domains` ile kaynak kısıtlama), **text editor** (dosya okuma/yazma/değiştirme — implementasyonu size ait), **code execution** (izole Docker container'da Python çalıştırma, Files API ile dosya giriş/çıkışı).
- Yapılandırılmış veri çıkarımında prefill+stop yerine tool_choice zorlaması (`{"type": "tool", "name": "..."}`) daha güvenilir ama daha karmaşık bir alternatif.

## 7) RAG: büyük dokümanlarda doğru bağlamı bulmak

Yüzlerce sayfalık bir dokümanı doğrudan prompta gömmek context limiti, maliyet ve gecikme açısından sürdürülebilir değil. **RAG (Retrieval-Augmented Generation)** dokümanı önceden parçalara (chunk) bölüp, soruyla en alakalı parçaları bulup sadece onları prompta ekliyor:

- **Chunking stratejisi**: sabit uzunluk (basit ama kelime kesebilir, overlap ile telafi edilir), yapı bazlı (markdown başlıkları gibi belirli bir formata güveniyorsanız), semantik (anlamca yakın cümleleri gruplamak, en gelişmiş ama en karmaşık).
- **Embedding + vector database**: her chunk sayısal bir vektöre çevrilir, kullanıcı sorusu da aynı şekilde vektörleştirilip **cosine similarity** ile en yakın chunk'lar bulunur.
- **Hybrid search**: saf semantik arama, tam terim eşleşmelerini kaçırabilir (ör. "ENG team" araması). **BM25** gibi bir lexical arama, terimleri nadirlik/sıklığına göre ağırlıklandırıp semantik aramayla paralel çalıştırılır; iki sonucun **Reciprocal Rank Fusion** ile birleştirilmesi ikisinin de gücünü kullanır.
- **Reranking**: birleşmiş sonuçlar bir LLM'e "bu adaylardan hangisi soruya en alakalı" diye sorularak yeniden sıralanır — ek gecikme getirir ama nüanslı sorularda doğruluğu belirgin artırır.
- **Contextual retrieval**: chunk'ı embedding'lemeden önce, kaynak dokümana göre "bu parça neyle ilgili" özetleyen kısa bir bağlam LLM ile üretilip chunk'a eklenir; böylece parça, orijinal dokümandan koptuğunda bile anlamını korur.

## 8) Diğer üretim özellikleri: extended thinking, multimodal, caching

- **Extended thinking**: Claude'un cevap üretmeden önce ayrı bir "düşünme" adımı ayırmasına izin verir; doğruluğu artırır ama ek token maliyeti ve gecikme getirir. Minimum düşünme bütçesi 1024 token, `max_tokens` bu bütçeyi mutlaka aşmalıdır. Prompt optimizasyonu yetmediğinde, eval sonuçlarına bakarak açılması önerilir.
- **Görsel ve PDF desteği**: mesaja base64 image/document bloğu eklenerek çalışır (istek başına en fazla 100 görsel). Doğruluk büyük ölçüde **prompt kalitesine** bağlı — adım adım analiz talimatı ve örnekler vermeden basit prompt'lar genelde başarısız olur.
- **Citations**: PDF veya düz metin kaynaklardan alıntı yaparken hangi sayfadan/karakter aralığından geldiğini döndürür; kullanıcıya "bu bilgi şuradan geliyor" şeffaflığı sağlar.
- **Prompt caching**: aynı içerik (system prompt, tool şemaları, sabit mesaj önekleri) tekrar tekrar gönderiliyorsa, işlenmiş halini 1 saate kadar cache'leyip yeniden işlemeyi atlar. Cache breakpoint'i manuel eklenir (`cache_control: {"type": "ephemeral"}`), istek başına en fazla 4 breakpoint, cache'e girmek için içerik en az 1024 token olmalı. Breakpoint öncesi herhangi bir değişiklik tüm cache'i geçersiz kılar.

## 9) MCP: entegrasyon yükünü sağlayıcıya devretmek

**MCP (Model Context Protocol)**, her servis için tool şeması + fonksiyon yazma yükünü ortadan kaldıran bir iletişim katmanı. GitHub, Slack, Jira gibi her entegrasyon için kendi tool'larınızı yazmak yerine, ilgili servisin resmi **MCP server**'ına bağlanırsınız; server tool/resource/prompt'ları hazır sunar.

- **MCP client**, sizin sunucunuzla MCP server arasındaki köprüdür (`list_tools`, `call_tool`).
- **Resources**, sunucunun proaktif olarak veri sunmasını sağlar (ör. `docs://documents/{doc_id}` gibi templated URI); tool'lar ise Claude karar verdiğinde reaktif olarak çalışır.
- **Prompts**, server yazarının önceden test edip hazırladığı, client'ta slash-command gibi görünen hazır prompt şablonlarıdır.

Kısacası MCP, tool use'un yerini almıyor; "bu işi kim yapıyor" sorusuna (uygulama geliştirici mi, servis sağlayıcı mı) cevap veriyor.

## 10) Claude Code ve agent mimarisi

**Claude Code**, terminal tabanlı bir kodlama asistanı olmanın ötesinde, agent mimarisinin canlı bir örneği. Pratikte işe yarayan birkaç prensip:

- Detaylı talimat = önemli ölçüde daha iyi sonuç; Claude Code'u kod üretici değil, **iş birlikçi mühendis** gibi kullanmak (önce ilgili dosyaları analiz ettirip, sonra planlatıp, en son uygulattırmak) daha güvenilir sonuç veriyor.
- **Git worktree**'ler ile birden fazla Claude örneğini izole workspace'lerde paralel çalıştırmak mümkün — aynı dosyalara aynı anda dokunma çakışmasını önlüyor.
- CI/CD'ye bağlanıp production loglarını (ör. CloudWatch) tarayarak hataları tespit edip düzeltme PR'ı açan **otomatik debugging** akışları kurulabiliyor.

Daha genel planda, **workflow** ve **agent** arasındaki seçim şuna dayanıyor: adımları önceden biliyorsanız (workflow), bilmiyorsanız (agent). Workflow desenleri — **parallelization** (bir işi bağımsız alt görevlere bölüp sonuçları birleştirmek), **chaining** (uzun/çok kısıtlı bir promptu sıralı adımlara bölmek), **routing** (girdiyi önce kategorize edip uygun pipeline'a yönlendirmek) — test edilebilirliği ve başarı oranı yüksek olduğu için genelde önce denenmesi gereken yaklaşım. Agent'lar ise esnekliğin gerçekten şart olduğu, adımların önceden tanımlanamadığı durumlar için saklı kalmalı; tasarımda **soyut ve genel araçlar** (bash, dosya okuma/yazma gibi) vermek, aşırı özelleşmiş tool'lardan (`refactor_tool` gibi) daha iyi sonuç veriyor. Ayrıca her adımdan sonra ortamı gözlemlemek (ekran görüntüsü almak, dosyayı yeniden okumak) agent'ın kör ilerlemesini engelliyor.

Özetle: **güvenilirlik önce gelir, yenilik ikinci sırada** — kullanıcı %100 çalışan bir ürünü, gösterişli ama kırılgan bir agent'a tercih eder.

---

Bütün bu parçaları bir araya getirdiğinizde ortaya çıkan resim şu: model seçimi ve temel API mekaniği başlangıç noktası; prompt mühendisliği ve eval pipeline'ı çıktı kalitesini ölçülebilir kılıyor; tool use, RAG ve MCP Claude'u dış dünyaya bağlıyor; caching ve extended thinking gibi özellikler maliyet/doğruluk dengesini ince ayarlıyor; en üstte de workflow ve agent kararı sistemin ne kadar esnek ama ne kadar öngörülebilir olacağını belirliyor. Tek bir teknik değil, bu katmanların birlikte doğru kurgulanması, Claude API'yi gerçek bir üründe güvenilir şekilde çalıştırıyor.

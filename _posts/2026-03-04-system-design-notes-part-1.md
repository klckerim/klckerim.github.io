---
title: "System Design Notes – 1"
date: 2026-03-04
categories: [SystemDesign, Backend, Architecture]
tags: ["system-design", "backend", "scalability", "load-balancing", "caching", "database", "cap-theorem"]
description: "Temel system design zihniyetleri ve trade-off'lar."
author: klckerim
media_subpath:  /assets/img/posts/headers
image:
    path: softwaredesign.png
---

Orta seviyeden senior seviyeye geçişte en kritik kırılım, “kod yazma” odağından “sistemi ayakta tutma” odağına geçmektir.
System design tam olarak burada devreye girer: sadece bir endpoint’i çalıştırmak değil, artan trafik, hata senaryoları, gecikme (latency), veri tutarlılığı, operasyon maliyeti ve ekip ölçeği altında sistemin davranışını tasarlamaktır.


## System design gerçek hayatta ne demek?

Pratikte system-design şu sorulara verilen mühendislik cevaplarının toplamıdır:

- Trafik 10x artarsa sistem nerede kırılır?
- Bir bileşen düşerse kullanıcı ne görür?
- Veri ne kadar hızlı “doğru” olmak zorunda?
- Gecikme mi daha kritik, doğruluk mu?
- Hangi noktada para harcayıp hangi noktada karmaşıklık azaltılmalı?


Örneğin ödeme sisteminde “ödeme iki kere çekilmemeli” gereksinimi, doğrudan idempotency, transaction sınırları, retry politikası ve güçlü gözlemlenebilirlik (logging/metrics/tracing) kararlarına dönüşür. Auth sisteminde ise “login hızlı olsun” tek başına hedef değildir; brute-force’a dayanıklılık, token yaşam döngüsü, session invalidation gibi konular da tasarımın parçasıdır.

---

## 1) Scalability: büyütmek sadece daha güçlü sunucu almak değildir

### Vertical vs Horizontal

**Vertical scaling (scale-up):**
- Daha fazla CPU/RAM olan tek makineye geçersin.
- Uygulaması basit, kısa vadede hızlı çözüm.
- Limit: fiziksel tavan ve tek hata noktası (single point of failure).

**Horizontal scaling (scale-out):**
- Aynı servisten birden fazla instance çalıştırırsın.
- Daha yüksek tavan, daha iyi dayanıklılık.
- Zorluk: state yönetimi, koordinasyon, dağıtık sistem karmaşıklığı.

Basit görünüm:

```text
Vertical:
[Client] -> [API (32 CPU, 128 GB RAM)] -> [DB]

Horizontal:
[Client] -> [LB] -> [API-1]
                 -> [API-2]
                 -> [API-3]
                    |
                    v
                   [DB/Cache]
```

### Bottleneck analizi

Çoğu sistemde darboğaz kod değil, bağımlılıklardır:

- DB connection pool dolması
- Yavaş dış servis çağrıları (ör. ödeme sağlayıcı API)
- Disk I/O veya network sınırları
- Sıralı çalışan kritik path’ler

Fintech-benzeri bir ödeme akışında sık pattern:

1. Payment request gelir
2. Fraud servisi çağrılır
3. Provider’a authorize atılır
4. DB’ye yazılır
5. Event yayınlanır

Burada latency çoğunlukla 2 ve 3. adımda birikir. API sunucusunu büyütmek tek başına sorunu çözmez.

### Stateless service neden önemli?

Horizontal scaling için servislerin mümkün olduğunca **stateless** olması gerekir. Yani instance üzerinde kullanıcıya özel oturum bilgisi tutmamak.

- Session state’i Redis gibi paylaşımlı bir katmana al
- JWT gibi taşınabilir kimlik yapıları kullan
- Dosya upload gibi işlemlerde local disk yerine object storage kullan

Aksi halde load balancer arkasında “sticky session” bağımlılığı doğar ve ölçekleme esnekliği azalır.

---

## 2) Load balancing: trafik dağıtımı değil, risk dağıtımı

Load balancer’ın temel rolü gelen trafiği instance’lara dağıtmaktır ama etkisi bundan büyüktür:

- Tek instance arızasında tüm sistemin düşmesini engeller
- Rolling deployment sırasında kesintiyi azaltır
- Health check ile bozuk node’u havuzdan çıkarır

Yaygın stratejiler:

- **Round-robin:** sırayla dağıtır, basit ve etkili.
- **Least connections:** aktif bağlantısı az olana gönderir.
- **IP hash / consistent hash:** aynı istemciyi aynı node’a yaklaştırır (cache locality için faydalı olabilir).
- **Weighted:** güçlü node’a daha fazla trafik verir.

Örnek (auth API):

```text
[Users] -> [LB]
           |-- /login -> [Auth-1]
           |-- /login -> [Auth-2]
           '-- /login -> [Auth-3]

LB health-check: /healthz
Auth-2 unhealthy -> traffic removed
```

Neden kritik? Çünkü production’da hata çoğu zaman “tam çöküş” değil, “kademeli bozulma”dır. LB olmadan bir node yavaşladığında herkes etkilenir; LB ile etki izole edilir.

---

## 3) Caching: hızlı sistemlerin gizli motoru, en zor bug’ların kaynağı

Read-heavy sistemlerde cache çoğu zaman en yüksek ROI’ye sahip optimizasyondur.

Örnekler:

- Auth sisteminde public key/jwks bilgisi
- Payment dashboard’da sık okunan işlem özetleri
- Referans veriler (ülke, para birimi, komisyon tabloları)

### Ne zaman cache kullanılır?

- Okuma, yazmadan belirgin şekilde fazlaysa
- Verinin kısa süre stale olması kabul edilebiliyorsa
- DB üzerindeki yük kritik seviyedeyse

### Temel stratejiler

- **Cache-aside (lazy loading):** miss olursa DB’den al, cache’e yaz.
- **Write-through:** DB’ye yazarken cache’i de güncelle.
- **Write-behind:** önce cache, arkadan asenkron DB (karmaşık ama yüksek throughput).

### Cache invalidation neden zor?

“Bilgisayar biliminde iki zor problem vardır: cache invalidation ve isimlendirme.” Şaka ama doğru.

Sık problemler:

- Güncellenen veri cache’de eski kalır (stale read)
- Aynı key’e farklı versiyonlar yazılır
- TTL çok uzun seçilir, yanlış veri uzun süre yaşar
- TTL çok kısa seçilir, cache etkisi kaybolur

Pratik yaklaşım:

```text
Doğruluk kritik (ör. bakiye) -> kısa TTL + event ile invalidation + fallback DB read
Doğruluk esnek (ör. rapor kartı) -> daha uzun TTL + eventual update
```

Ödeme sisteminde “kullanılabilir bakiye” ekranını cache’lemek caziptir; ama ledger güncellemesi sonrası invalidation güvenilir değilse kullanıcıya yanlış bakiye gösterebilirsin. Bu bir UX problemi değil, finansal risk problemidir.

---

## 4) Databases: SQL vs NoSQL bir dogma değil, workload kararıdır

### SQL (PostgreSQL, MySQL vb.)

Güçlü taraflar:

- ACID transaction
- Güçlü tutarlılık
- Join ve kompleks sorgularda olgun ekosistem

Zayıf taraflar:

- Çok yüksek write throughput ve yatay partition gerektiren senaryolarda yönetim zorlaşabilir
- Şema değişiklikleri dikkat ister

### NoSQL (MongoDB, Cassandra, DynamoDB vb.)

Güçlü taraflar:

- Esnek şema (ürüne hızlı iterasyon)
- Bazı türlerde yüksek ölçeklenebilirlik ve düşük latency
- Dağıtık mimariye doğal uyum

Zayıf taraflar:

- Join/transaction kapasitesi teknolojiye göre kısıtlı olabilir
- Veri modelleme hataları daha geç fark edilir
- “Eventually consistent” davranış uygulama seviyesinde ek dikkat gerektirir

### Pratik seçim modeli

```text
Ödeme, muhasebe, bakiye, transfer gibi parasal kayıtlar -> SQL ağırlıklı
Yüksek hacimli event, log, activity feed -> NoSQL / append-friendly storage
```

Gerçek sistemlerde hibrit yaklaşım yaygındır: kritik parasal kayıtlar relational DB’de, analitik ve event akışı farklı bir datastore’da tutulur.

---

## 5) CAP theorem: kısa ama pratik bakış

CAP teoremi dağıtık bir sistemde network partition varken aynı anda şu üçünü tam veremeyeceğini söyler:

- **Consistency (C):** Her okuma en güncel yazıyı görür.
- **Availability (A):** Her istek yanıt alır.
- **Partition tolerance (P):** Ağ bölünmelerine rağmen sistem çalışmaya devam eder.

Gerçek hayatta partition kaçınılmaz olduğu için çoğu üretim sistemi aslında **C ve A arasında** tercih yapar.

Basitçe:

```text
Partition olduysa:
- C'yi koru -> bazı istekleri reddet (availability düşer)
- A'yı koru -> bazı okumalar stale olabilir (consistency gevşer)
```

Bu yüzden “biz CP miyiz AP miyiz?” sorusu, bileşen bazında sorulmalıdır; tüm sistem tek cevapla açıklanamaz.

---

## 6) Consistency vs Availability: pratik trade-off kararları

Bu konu özellikle senior seviyeye geçişte belirleyicidir. Her endpoint aynı tutarlılık seviyesini istemez.

### Örnek 1: Payment capture

- Gereksinim: “Aynı işlem iki kez çekilmesin.”
- Tercih: Daha güçlü consistency.
- Sonuç: Gerekirse kısa süreli availability kaybı kabul edilir (ör. retry/reject).

### Örnek 2: Transaction listesi ekranı

- Gereksinim: Kullanıcı son 1-2 saniye gecikmeyi tolere edebilir.
- Tercih: Availability yüksek, eventual consistency kabul.

### Örnek 3: Auth token doğrulama

- Gereksinim: Düşük latency + güvenlik.
- Tercih: Local doğrulama + anahtar rotasyonu için kontrollü cache.

Karar çerçevesi:

```text
1) Bu veri yanlışsa iş riski ne?
2) Kullanıcı gecikmeyi ne kadar tolere eder?
3) Hatanın geri dönüş maliyeti ne?
4) Operasyonel karmaşıklık bütçemiz ne kadar?
```


---

## Sık Yapılan Hatalar

1. **Teknolojiyi problemi çözmeden seçmek**
   - “NoSQL modern, onu kullanalım” yaklaşımı.
   - Doğrusu: erişim paterni, tutarlılık ihtiyacı, operasyon maliyetiyle başla.

2. **Stateless prensibini atlamak**
   - Session’ı instance memory’de tutup scale-out beklemek.

3. **Cache’i sadece performans feature’ı sanmak**
   - Invalidation planı olmadan cache eklemek.

4. **DB darboğazını uygulama katmanında gizlemek**
   - N+1 query, yanlış index, connection pool sınırlarını gözden kaçırmak.

5. **CAP ve consistency konularını siyah-beyaz düşünmek**
   - “Her şey strongly consistent olmalı” veya “eventual her yerde yeterli” uçları.

6. **Failure mode tasarlamamak**
   - Dış servis timeout olduğunda sistemin nasıl davranacağını önceden modellememek.

Önemli olan, “happy path” dışında sistemin nasıl davranacağını tasarlamaktır.

---

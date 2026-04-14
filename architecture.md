# CLAUDE.md — AI Influencer Content Factory
## Master Architecture & Operational Context

---

## 1. PROJE KİMLİĞİ VE VİZYON

Bu sistem, **"Acımasız İlişki Analisti"** nişinde faaliyet gösteren bir AI Influencer için tasarlanmış **uçtan uca otonom bir içerik üretim fabrikasıdır**. Sistemin temel amacı; sosyal medyadan trendleri yakalamak, psikolojik derinlikte analiz etmek, güçlü bir karakter sesiyle senaryo üretmek ve insan onayına sunmak.

### Temel Felsefe

- **Haber Değil, Hakikat:** Sistem yalnızca olayları aktarmaz. Olayın altındaki insan defosunu, savunma mekanizmalarını ve sosyal kör noktaları yüzeye çıkarır.
- **Karakter Odaklılık:** Sistemin AI olduğu gizlenmez; ancak bir bot gibi değil, bir **kült karakter** gibi davranır. Ses tonu tutarlı, otoriter ve keskindir.
- **Bellek:** Notion veritabanı sayesinde sistem kendini tekrar etmez. Her kaynak yalnızca bir kez işlenir.
- **Maliyet Bilinci:** Gereksiz LLM çağrılarını önlemek için deduplication (tekrar engelleme) mekanizması kritik öneme sahiptir.

---

## 2. TEKNİK STACK

| Katman | Araç / Servis |
|---|---|
| **Orkestrasyon** | n8n (self-hosted veya cloud) |
| **Veri Kaynağı 1** | Reddit API (`r/relationships`, `r/Tinder`, `r/dating_advice`) |
| **Veri Kaynağı 2** | Apify — X/Twitter Scraper |
| **Hafıza & Veritabanı** | Notion (Sistem Belleği + İçerik Takvimi) |
| **Analiz/Filtreleme LLM** | Gemini 1.5 Flash (hız ve maliyet dengesi) |
| **Persona/Yazım LLM** | GPT-4o |
| **İletişim & Onay** | Telegram Bot API |

---

## 3. NOTION VERİTABANI ŞEMASI

Kodlamaya başlamadan önce Notion veritabanında aşağıdaki sütunların (Property) eksiksiz hazır olduğunu varsay. Eksik sütun varsa kullanıcıya bildir ve devam etme.

| Sütun Adı | Tip | Açıklama |
|---|---|---|
| `Olay` | Title | Olayın kısa, tanımlayıcı adı |
| `Durum` | Select | `⏳ Onay Bekliyor` / `🟢 Üretime Hazır` / `🔴 Çöpe Atıldı` / `✅ Paylaşıldı` |
| `Video Kancası` | Text | Hook cümlesi (ilk 3 saniye dikkat çekici açılış) |
| `Senaryo` | Rich Text | Teleprompter metni (mimik/ton notlarıyla birlikte) |
| `Açıklama` | Text | Sosyal medya post metni + hashtagler |
| `Görsel Promptu` | Text | Midjourney / Flux görsel üretim promptu |
| `Trend URL` | URL | Kaynak içeriğin linki |
| `Source_ID` | Text | Reddit post ID veya X tweet ID (tekrar önleme için) |
| `Tarih` | Date | Notion'a kayıt tarihi |

> ⚠️ **Kritik:** `Source_ID` sütunu deduplication mekanizmasının temelidir. Bu sütun olmadan sistem aynı içeriği birden fazla kez işler ve gereksiz maliyet oluşturur.

---

## 4. SİSTEM MİMARİSİ — AKIŞ ŞEMASI

```
[Veri Sensörleri]
Reddit API + Apify X Scraper
        |
        v
[Faz 1: Pruning (JS Code Node)]
Token budama + Controversy Ratio hesaplama
        |
        v
[Faz 2: Bellek Kapısı (Deduplication)]
Notion'da Source_ID kontrolü
  |               |
EŞLEŞME VAR    EŞLEŞME YOK
  |               |
DUR (stop)      Devam et
                  |
                  v
         [Faz 3: 3-Ajan Zinciri]
         Ajan 1 → Ajan 2 → Ajan 3
                  |
                  v
         [Faz 4: Dağıtım & Onay]
         Notion Kayıt + Telegram Buton
                  |
                  v
         [Kullanıcı Kararı]
         🟢 ONAYLA / 🔴 ÇÖPE AT
                  |
                  v
         [Notion Durum Güncelleme]
```

---

## 5. FAZ 1: VERİ SENSÖRLERİ VE BUDAMA

### Reddit Veri Çekme Parametreleri
- **Subredditler:** `r/relationships`, `r/Tinder`, `r/dating_advice`, `r/AmItheAsshole`
- **Sort:** `top`
- **Time:** `day`
- **Limit:** `10` post

### X (Twitter) — Apify Parametreleri
- `min_faves: 500` VEYA `min_replies: 100`
- Konuya özel arama terimleri: ilişki, aldatma, ghosting, manipülasyon, bağlanma stilleri

### JavaScript Filtreleme Mantığı (n8n Code Node)

```javascript
// Gelen her post için çalışır
const posts = items.map(item => {
  const data = item.json;

  // 1. Controversy Ratio hesapla
  const controversyRatio = data.num_comments / (data.score || 1);
  const isControversial = controversyRatio > 0.4 || data.num_comments > 200;

  // 2. Token Pruning — Post metni 500 karakterde kes
  const prunedText = (data.selftext || data.title || '').substring(0, 500);

  // 3. İlk 15 yorumu al, her birini 200 karakterde buda
  const topComments = (data.top_comments || [])
    .slice(0, 15)
    .map(c => ({
      body: (c.body || '').substring(0, 200),
      score: c.score
    }));

  // 4. Ajan 1'e gönderilecek temiz JSON
  return {
    json: {
      source_id: data.id || data.tweet_id,
      source: data.subreddit ? 'reddit' : 'twitter',
      url: data.url || data.tweet_url,
      title: data.title || data.full_text?.substring(0, 100),
      text: prunedText,
      comments: topComments,
      score: data.score || data.favorite_count,
      num_comments: data.num_comments || data.reply_count,
      controversy_ratio: controversyRatio,
      is_controversial: isControversial,
      fetched_at: new Date().toISOString()
    }
  };
}).filter(item => item.json.is_controversial); // Yalnızca kontroversiyel olanları geçir

return posts;
```

---

## 6. FAZ 2: BELLEK KAPISI (DEDUPLICATION GATE)

### n8n Düğüm Yapısı
1. **Notion: Search** düğümü → `Source_ID` sütununda gelen `source_id` değerini ara
2. **IF** düğümü:
   - **Eşleşme VARSA:** Akışı `NoOp` düğümüyle sonlandır *(maliyet tasarrufu)*
   - **Eşleşme YOKSA:** Faz 3'e devam et

### Kontrol Edilecek Alan
```
Filter: Source_ID = {{ $json.source_id }}
```

> 💡 **Not:** Başlık benzerliği kontrolü de eklenebilir (fuzzy match), ancak `Source_ID` kontrolü yeterli ve daha güvenilirdir.

---

## 7. FAZ 3: BİLİŞSEL KATMAN — 3 AJAN ZİNCİRİ

---

### AJAN 1: KLİNİK ANALİST
**Model:** Gemini 1.5 Flash  
**Amaç:** Ham veriden psikolojik otopsi çıkarmak. Hızlı ve ucuz.

#### System Prompt
```
Sen bir klinik psikolog ve sosyal dinamikler uzmanısın. Sana verilen sosyal medya olayını veya tartışmasını analiz et.

Görevin: İnsanların neden bu kadar tepki verdiğini, olayın hangi evrensel insan zaafiyetine dokunduğunu ve bu içeriğin hangi açıdan viral potansiyel taşıdığını ortaya koymak.

ÇIKTI FORMATINI KESİNLİKLE TAKIP ET. Yalnızca aşağıdaki JSON'u döndür, başka hiçbir şey yazma:

{
  "incident_summary": "Olayın tarafsız 2-3 cümlelik özeti",
  "psychological_diagnosis": "Olayın merkezindeki psikolojik mekanizma (örn: anxious attachment, narcissistic injury, sunk cost fallacy)",
  "social_fault_line": "Bu olayın hangi evrensel insan kırılganlığına ya da toplumsal çatlağa dokunduğu",
  "strike_angle": "Bu konuyu en keskin ve ilgi çekici şekilde ele almak için önerilen yaklaşım açısı",
  "viral_potential": "low | medium | high",
  "content_warning": "Varsa hassas konu uyarısı, yoksa null"
}
```

#### User Prompt Template
```
Aşağıdaki sosyal medya olayını analiz et:

BAŞLIK: {{ $json.title }}
İÇERİK: {{ $json.text }}
EN ÇOK EKLENEN YORUMLAR:
{{ $json.comments.map(c => `- "${c.body}" (${c.score} beğeni)`).join('\n') }}

ETKILEŞIM: {{ $json.score }} beğeni, {{ $json.num_comments }} yorum
KAYNAK: {{ $json.source }}
```

---

### AJAN 2: PERSONA MOTORU
**Model:** Chatgpt-4o  
**Amaç:** Ajan 1'in analizini, karakterin sesiyle ham senaryoya dönüştürmek.

#### Karakter Profili
- **Kim:** 30'lu yaşlarında, zeki, otoriter, kurban psikolojisinden derin bir nefret besleyen kadın mentör
- **Tarz:** Keskin, doğrudan, bazen acımasız ama her zaman haklı
- **Kısıtlamalar:**
  - ❌ "Reddit'te okudum" veya "Bu tweet'te gördüm" ASLA demez
  - ❌ "Bence" veya "Sanırım" gibi belirsiz ifadeler kullanmaz
  - ❌ Kurbanlara sempati göstermez; sorumlulukları hatırlatır
  - ✅ Kendi gözlemiymiş gibi, birinci şahıs otoritesiyle konuşur
  - ✅ Evrensel hakikatleri spesifik vakalarla bağlar
  - ✅ İzleyiciyi rahatsız eden ama düşündüren cümleler kurar

#### System Prompt
```
Sen "Acımasız İlişki Analisti" karakterisin. 30'lu yaşlarında, zeki, otoriter ve kurban psikolojisinden nefret eden bir kadın mentörsün.

Sesin: Keskin, doğrudan, bazen acımasız ama her zaman hakikat merkezli.
Amacın: İnsanlara kendi kendilerine yalanlarını fark ettirmek.
Kısıtlamaların:
- Kaynağı ASLA belirtme ("Reddit'te...", "Twitter'da..." deme)
- Konuyu kendi gözleminmiş gibi, birinci şahıs perspektifinden sun
- "Bence" kullanma, otorite ile konuş
- Kurban narratifine boyun eğme; sorumluluğa yönlendir

Görevin: Verilen psikolojik analizi, karakterinin sesiyle ham bir konuşma metnine dönüştür.
Bu metin daha sonra bir prodüktör tarafından video scriptine çevrilecek.
Yalnızca ham konuşma metnini döndür, JSON değil.
```

#### User Prompt Template
```
Psikolojik analiz:
{{ $json.ajan1_output }}

Bu analizi karakterin sesiyle, 150-200 kelimelik güçlü bir konuşma metnine dönüştür.
Açılış cümlesi izleyiciyi hemen içine çekmelidir.
Sonu bir gerçekle tokat gibi bitirmelidir.
```

---

### AJAN 3: PRODÜKTÖR
**Model:** Chatgpt-4o 
**Amaç:** Ham senaryoyu algoritma dostu, platforma hazır bir pakete dönüştürmek.

#### System Prompt
```
Sen deneyimli bir sosyal medya içerik prodüktörüsün. TikTok, Instagram Reels ve YouTube Shorts algoritmalarını derinlemesine biliyorsun.

Görevin: Verilen ham konuşma metnini, aşağıdaki JSON formatında bir içerik paketine dönüştürmek.

ÇIKTI FORMATINI KESİNLİKLE TAKIP ET. Yalnızca JSON döndür:

{
  "video_hook": "İlk 3 saniyede izleyiciyi durduracak tek cümle. Soru veya şok ifadesi olabilir. Max 15 kelime.",
  "teleprompter_script": "Tam konuşma metni. Ton ve mimik notları için [DURAKLAMA], [ÖFKELI TON], [DOĞRUDAN BAKIŞ], [GÜL] gibi köşeli parantez notları ekle.",
  "social_caption": "Sosyal medya post açıklaması. 3-4 cümle + 5-8 alakalı hashtag. Türkçe ve İngilizce karışık hashtag kullanabilirsin.",
  "visual_prompt_mj": "Midjourney veya Flux için İngilizce görsel üretim promptu. Karakterin duygusunu ve sahneyi yansıtmalı. '--ar 9:16 --style raw' ile bitir."
}
```

#### User Prompt Template
```
Ham konuşma metni:
{{ $json.ajan2_output }}

Orijinal olay özeti (referans için):
{{ $json.ajan1_output.incident_summary }}

Bu metni platforma hazır içerik paketine dönüştür.
```

---

## 8. FAZ 4: DAĞITIM VE ONAY MEKANİZMASI

### Adım 1: Notion'a Kayıt
Ajan 3'ten gelen JSON paketi Notion'a şu mapping ile yazılır:

```
Notion Property   ←   JSON Field
──────────────────────────────────
Olay              ←   incident_summary (kısaltılmış)
Durum             ←   "⏳ Onay Bekliyor" (sabit)
Video Kancası     ←   video_hook
Senaryo           ←   teleprompter_script
Açıklama          ←   social_caption
Görsel Promptu    ←   visual_prompt_mj
Trend URL         ←   source_url
Source_ID         ←   source_id
Tarih             ←   fetched_at
```

### Adım 2: Telegram Bildirim Mesajı

```
🎬 YENİ İÇERİK HAZIR

📌 OLAY:
{{ incident_summary }}

🪝 KANCA:
{{ video_hook }}

📝 SENARYO ÖNİZLEME:
{{ teleprompter_script | truncate(300) }}

📣 AÇIKLAMA:
{{ social_caption }}

🔗 Kaynak: {{ source_url }}
📊 Viral Potansiyel: {{ viral_potential }}

─────────────────
Notion: {{ notion_page_url }}
```

**Butonlar:**
- `🟢 ONAYLA` → callback_data: `approve_{{ notion_page_id }}`
- `🔴 ÇÖPE AT` → callback_data: `reject_{{ notion_page_id }}`

### Adım 3: Wait Node & Durum Güncelleme

```
n8n Wait Node → Telegram Webhook'tan yanıt bekle

IF callback_data starts with "approve_":
  → Notion: Update "Durum" = "🟢 Üretime Hazır"
  → Telegram: "✅ İçerik onaylandı ve takvime eklendi."

IF callback_data starts with "reject_":
  → Notion: Update "Durum" = "🔴 Çöpe Atıldı"
  → Telegram: "🗑️ İçerik silindi."
```

---

## 9. HATA YÖNETİMİ VE EDGE CASE'LER

### LLM Yanıtı JSON Parse Hatası
```javascript
// Ajan 1 ve Ajan 3 için JSON parse güvenlik katmanı
try {
  // Markdown code fence varsa temizle
  const cleaned = response.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim();
  const parsed = JSON.parse(cleaned);
  return [{ json: parsed }];
} catch (e) {
  // Hatayı logla, bu postu atla
  console.error('JSON parse hatası:', e.message);
  return [{ json: { error: true, raw_response: response } }];
}
```

### Viral Potential Filtresi
- `viral_potential: "low"` gelen içerikler otomatik olarak Notion'a `🔴 Çöpe Atıldı` durumuyla yazılabilir (opsiyonel, kullanıcı tercihine göre)

### Notion API Rate Limit
- n8n'de Notion düğümleri arasına `Wait: 1 saniye` ekle
- Günde maksimum 50 Notion yazma işlemi hedefle

### Boş İçerik Kontrolü
- `text` alanı 50 karakterden kısaysa Faz 3'e geçme, atla

---

## 10. GELECEKTEKİ GENİŞLEMELER (Backlog)

- [ ] **Ajan 4 — Performans Takipçisi:** Paylaşılan içeriklerin etkileşimini takip edip hangi açıların işe yaradığını raporlar
- [ ] **Otomatik Yayınlama:** Onaylanan içerikleri doğrudan TikTok/Instagram API üzerinden zamanlanmış yayınlama
- [ ] **A/B Hook Testi:** Ajan 3'ten 2 farklı hook ürettir, Telegram'dan hangisini seçeceğini sor
- [ ] **Trend Cluster Analizi:** Benzer konuların kümelenerek tek güçlü içeriğe dönüştürülmesi
- [ ] **Görsel Üretim Entegrasyonu:** Flux/Midjourney API ile otomatik thumbnail üretimi

---

## 11. ÇALIŞMA KURALLARI (Claude için Talimatlar)

Bu projede Claude olarak şu kurallara uy:

1. **Notion şemasına bağlı kal:** Kod yazarken Property isimlerini tam olarak bu dokümandaki gibi kullan. Büyük/küçük harf ve Türkçe karakter farkına dikkat et.

2. **Ajan promptlarını değiştirme:** Özellikle Ajan 2'nin karakter promptu hassastır. Kullanıcı açıkça istemeden persona üzerinde değişiklik yapma.

3. **JSON parse hatalarını her zaman yakala:** LLM yanıtları bazen markdown fence içinde gelir. Her zaman temizleme + try/catch kullan.

4. **Maliyet bilinci:** Deduplication gate'i kesinlikle atlamadan çalıştır. Bu gate olmadan sistem gereksiz LLM çağrıları yaparak maliyeti artırır.

5. **n8n düğüm adlandırması:** Düğümleri anlamlı isimlendir: `[FAZ1] Reddit Fetch`, `[FAZ2] Dedup Check`, `[FAZ3-A1] Klinik Analist` vb.

6. **Türkçe karakter güvenliği:** Notion property isimlerinde `İ`, `Ş`, `Ç`, `Ğ`, `Ü`, `Ö` karakterleri sorun çıkarabilir. Notion API çağrılarında property isimlerini URL encode etmeyi unutma.

7. **Wait Node süresi:** Telegram onay bekleme süresi varsayılan olarak **24 saat** olarak ayarlanmalı. Süre dolunca otomatik `🔴 Çöpe Atıldı` yap.

---

*Son güncelleme: Sistem başlangıç konfigürasyonu — v1.0*

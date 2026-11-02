# 🔴 Red Team Labs — Ćwiczenia Ofensywne na Własnym Sprzęcie
# 🔴 Red Team Labs — Offensive Security Exercises on Personal Hardware

> 🇵🇱 [Wersja polska](#-wersja-polska) | 🇬🇧 [English version](#-english-version)

---

# 🇵🇱 WERSJA POLSKA

> **Zastrzeżenie:** Wszystkie ćwiczenia zostały przeprowadzone wyłącznie na własnym sprzęcie, w celach edukacyjnych i do budowy portfolio. Żadne działania nie były skierowane przeciwko cudzym systemom.

## O projekcie

Zestaw praktycznych ćwiczeń z zakresu ofensywnego bezpieczeństwa (red team), wykonanych samodzielnie jako uzupełnienie nauki teoretycznej. Celem było zrozumienie realnych technik ataku i obrony — nie tylko z książek, ale przez własne doświadczenie.

---

## Lab 01 — Odzyskiwanie Danych i Analiza Uwierzytelniania Windows

### Środowisko

| Element | Opis |
|---|---|
| System atakujący | Kali Linux Live USB (bez instalacji, wszystko w RAM) |
| Cel 1 | Laptop z Windows 7 — ćwiczenie odzyskiwania danych |
| Cel 2 | Laptop z Windows 11, Intel Core i5, NVMe, TPM 2.0 — analiza zabezpieczeń |

---

### Etap 1: Odzyskiwanie usuniętych plików (File Carving)

**Co chciałem osiągnąć:** Sprawdzić czy możliwe jest odzyskanie usuniętych plików wideo z dysku NTFS bez dostępu do systemu operacyjnego.

**Dlaczego Live USB Kali?** Uruchomienie z zewnętrznego nośnika pozwala na pracę z dyskiem docelowym bez modyfikowania jego zawartości. System działa całkowicie w RAM — żaden ślad nie zostaje na atakowanym sprzęcie.

```bash
# Identyfikacja dostępnych dysków i partycji
sudo fdisk -l
# Wynik: /dev/sda1 — partycja NTFS (Windows)

# Montowanie w trybie tylko do odczytu — kluczowe żeby nie modyfikować danych
sudo mkdir /mnt/win
sudo mount -o ro /dev/sda1 /mnt/win
# Flaga -o ro (read-only) zapobiega przypadkowym zapisom na dysku docelowym
```

**PhotoRec** to narzędzie do odzyskiwania plików działające na poziomie sygnatury (file carving). Zamiast czytać tablicę partycji (która po usunięciu pliku jest wyczyszczona), skanuje surowe sektory dysku szukając znanych nagłówków formatów plików (JPEG, MOV, MP4 itd.).

```bash
sudo photorec /dev/sda
# Interfejs TUI: wybór partycji → typ systemu plików → katalog wyjściowy
```

**Wyniki i wnioski:**
- Odzyskano setki fragmentów plików — dowód że dane "usunięte" nadal fizycznie istnieją
- Największy plik: ~743 MB (.mov) — nieodtwarzalny z powodu uszkodzonego nagłówka
- **Kluczowa lekcja:** Zwykłe usunięcie pliku nie kasuje danych — tylko oznacza sektory jako dostępne do nadpisania. Dopiero wielokrotny nadpis (np. `shred`) skutecznie niszczy dane.

---

### Etap 2: Analiza Mechanizmu Uwierzytelniania — Ekstrakcja Hashy NTLM

**Co chciałem osiągnąć:** Zrozumieć jak Windows przechowuje hasła i czy możliwe jest ich odzyskanie offline.

**Tło techniczne:** Windows przechowuje skróty (hashe) haseł w pliku SAM (Security Account Manager). Plik ten jest zablokowany przez system operacyjny podczas jego działania — ale dostępny z zewnętrznego OS.

```bash
# Identyfikacja nośnika z kopiami plików SAM i SYSTEM
sudo fdisk -l
# /dev/sdc1 — FAT32, 7.5GB (pendrive roboczy)

sudo mkdir /mnt/pendrive
sudo mount /dev/sdc1 /mnt/pendrive

# Kopiowanie do katalogu roboczego
mkdir ~/hashes_analysis
cp /mnt/pendrive/backup/SAM ~/hashes_analysis/
cp /mnt/pendrive/backup/SYSTEM ~/hashes_analysis/
# Uwaga: sam plik SAM nie wystarczy — klucz szyfrujący jest w pliku SYSTEM
```

**Dlaczego potrzebny jest plik SYSTEM?** Plik SAM jest zaszyfrowany kluczem przechowywanym w gałęzi rejestru SYSTEM. Bez niego samdump2 nie może odszyfrować hashy.

```bash
cd ~/hashes_analysis
samdump2 SYSTEM SAM > hashes.txt
cat hashes.txt
```

**Wynik ekstrakcji:**
```
*disabled* Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
*disabled* Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
target_user:1001:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

**Analiza hashy — John the Ripper:**

```bash
# Próba złamania hashy słownikiem i brute force
john --format=NT hashes.txt

# Wyświetlenie wyników
john --show --format=NT hashes.txt
# Wynik: 6 password hashes cracked, 0 left
```

**Kluczowe odkrycie:** Hash `31d6cfe0d16ae931b73c59d7e0c089c0` to **znany hash pustego hasła NTLM**. John the Ripper "złamał" go natychmiast bo to dobrze znana wartość — konto po prostu nie ma ustawionego hasła lokalnego.

---

### Etap 3: Konto używa Windows Hello — Dlaczego PIN jest bezpieczniejszy niż myślisz

To było najciekawsze odkrycie całego ćwiczenia.

**Problem:** Konto ma pusty hash NTLM, ale logowanie przez Windows działało tylko przez PIN i odcisk palca. Jak to możliwe?

**Odpowiedź:** Windows Hello przenosi uwierzytelnianie ze świata software'owego do hardware'owego:

| Metoda | Gdzie przechowywane | Offline attack? |
|---|---|---|
| Hasło lokalne | SAM (plik na dysku) | ✅ Możliwy |
| Windows Hello PIN | TPM 2.0 (chip sprzętowy) | ❌ Niemożliwy |
| Odcisk palca | TPM 2.0 (chip sprzętowy) | ❌ Niemożliwy |

**TPM (Trusted Platform Module)** to dedykowany chip kryptograficzny wbudowany w płytę główną. PIN jest zaszyfrowany i przechowywany wewnątrz TPM — nie ma go na dysku, nie można go wyekstrahować przez oprogramowanie. Weryfikacja PIN odbywa się sprzętowo, z wbudowanym licznikiem błędnych prób.

---

### Etap 4: Rekonesans BitLocker i TPM

**Co chciałem osiągnąć:** Zweryfikować czy dysk jest szyfrowany (BitLocker) bez logowania do systemu.

**Odczyt konfiguracji TPM z BIOS/UEFI:**
```
TPM Specification Version: 2.0
TPM Device: Available
TPM State: Enabled
```

**Test BitLockera — próba montowania partycji:**
```bash
sudo mount /dev/nvme0n1p3 /mnt/win
# Wynik: mount: unknown filesystem type 'BitLocker'
```

Ten błąd to potwierdzenie że BitLocker jest aktywny. Linux nie potrafi odczytać zaszyfrowanej partycji bez klucza.

**Profil bezpieczeństwa systemu:**

| Zabezpieczenie | Status | Znaczenie |
|---|---|---|
| TPM 2.0 | ✅ Aktywny | Klucz BitLockera i PIN chronione sprzętowo |
| Secure Boot | ✅ Aktywny | Ochrona przed modyfikacją bootloadera |
| BitLocker | ✅ Aktywny | Pełne szyfrowanie dysku |
| Hasło BIOS | ❌ Brak | Wektor wejścia — zmiana kolejności boot |
| Hasło lokalne Windows | ❌ Brak | Zastąpione przez Windows Hello/TPM |

---

## Podsumowanie i Wnioski

### Co osiągnąłem
- ✅ Praktyczne odzyskiwanie plików metodą file carving
- ✅ Ekstrakcja i analiza hashy NTLM offline
- ✅ Zrozumienie architektury Windows Hello i roli TPM
- ✅ Weryfikacja aktywności BitLockera bez dostępu do systemu
- ✅ Pełny rekonesans konfiguracji BIOS/TPM

### Najważniejsze lekcje

1. **Puste hasło NTLM ≠ słabe zabezpieczenie** — Windows Hello przenosi auth do TPM, czyniąc ataki na SAM bezużytecznymi w nowoczesnych konfiguracjach.

2. **TPM 2.0 + BitLocker to solidna bariera** — publicznie dostępne narzędzia nie pozwalają na obejście tej kombinacji bez klucza odzyskiwania.

3. **Rekonesans bez modyfikacji jest możliwy** — Live USB pozwala zebrać dużo informacji o systemie docelowym bez pozostawiania śladów i bez ryzyka zablokowania systemu.

4. **Brak hasła BIOS to realna luka** — atakujący z fizycznym dostępem może zmienić kolejność bootowania. W środowisku produkcyjnym BIOS powinien być chroniony hasłem.

---

## Narzędzia

| Narzędzie | Zastosowanie | Link |
|---|---|---|
| Kali Linux | Dystrybucja do testów penetracyjnych | [kali.org](https://www.kali.org) |
| PhotoRec | File carving, odzyskiwanie danych | wbudowany w testdisk |
| samdump2 | Ekstrakcja hashy NTLM | pakiet samdump2 |
| John the Ripper | Analiza i łamanie hashy | [openwall.com](https://www.openwall.com/john/) |
| fdisk / mount | Zarządzanie partycjami | wbudowane w Linux |

---
---

# 🇬🇧 ENGLISH VERSION

> **Disclaimer:** All exercises were conducted exclusively on personal equipment for educational purposes and portfolio building. No actions were directed at third-party systems.

## About This Project

A set of hands-on offensive security (red team) exercises, conducted independently as a practical supplement to theoretical learning. The goal was to understand real attack and defense techniques — not just from books, but through personal experience.

---

## Lab 01 — Data Recovery and Windows Authentication Analysis

### Environment

| Component | Description |
|---|---|
| Attack system | Kali Linux Live USB (no installation, runs entirely in RAM) |
| Target 1 | Windows 7 laptop — data recovery exercise |
| Target 2 | Windows 11 laptop, Intel Core i5, NVMe, TPM 2.0 — security analysis |

---

### Phase 1: Deleted File Recovery (File Carving)

**Objective:** Determine whether deleted video files can be recovered from an NTFS drive without OS access.

**Why Kali Live USB?** Booting from an external drive allows working with the target disk without modifying its contents. The OS runs entirely in RAM — no trace is left on the target hardware.

```bash
# Identify available drives and partitions
sudo fdisk -l
# Output: /dev/sda1 — NTFS partition (Windows)

# Mount read-only — critical to avoid modifying target data
sudo mkdir /mnt/win
sudo mount -o ro /dev/sda1 /mnt/win
# The -o ro (read-only) flag prevents accidental writes to the target drive
```

**PhotoRec** is a file recovery tool that operates at the signature level (file carving). Instead of reading the partition table (which is cleared after file deletion), it scans raw sectors looking for known file format headers (JPEG, MOV, MP4, etc.).

```bash
sudo photorec /dev/sda
# TUI interface: select partition → filesystem type → output directory
```

**Results and conclusions:**
- Hundreds of file fragments recovered — proof that "deleted" data still physically exists
- Largest file: ~743 MB (.mov) — unplayable due to corrupted header
- **Key lesson:** Simply deleting a file doesn't erase data — it only marks sectors as available for overwriting. Only multiple overwrites (e.g., `shred`) effectively destroy data.

---

### Phase 2: Authentication Mechanism Analysis — NTLM Hash Extraction

**Objective:** Understand how Windows stores passwords and whether offline recovery is possible.

**Technical background:** Windows stores password hashes in the SAM (Security Account Manager) file. This file is locked by the OS while running — but accessible from an external OS.

```bash
# Identify drive containing SAM and SYSTEM file backups
sudo fdisk -l
# /dev/sdc1 — FAT32, 7.5GB (working USB drive)

sudo mkdir /mnt/pendrive
sudo mount /dev/sdc1 /mnt/pendrive

# Copy to working directory
mkdir ~/hashes_analysis
cp /mnt/pendrive/backup/SAM ~/hashes_analysis/
cp /mnt/pendrive/backup/SYSTEM ~/hashes_analysis/
# Note: SAM alone is not enough — the encryption key lives in the SYSTEM file
```

**Why is the SYSTEM file required?** The SAM file is encrypted with a key stored in the SYSTEM registry hive. Without it, samdump2 cannot decrypt the hashes.

```bash
cd ~/hashes_analysis
samdump2 SYSTEM SAM > hashes.txt
cat hashes.txt
```

**Extraction output:**
```
*disabled* Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
*disabled* Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
target_user:1001:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

**Hash analysis — John the Ripper:**

```bash
# Attempt to crack hashes using wordlist and brute force
john --format=NT hashes.txt

# Display results
john --show --format=NT hashes.txt
# Result: 6 password hashes cracked, 0 left
```

**Key finding:** Hash `31d6cfe0d16ae931b73c59d7e0c089c0` is the **well-known empty NTLM password hash**. John the Ripper "cracked" it instantly because it's a widely known value — the account simply has no local password set.

---

### Phase 3: The Account Uses Windows Hello — Why PIN Is More Secure Than You Think

This was the most interesting discovery of the entire exercise.

**The puzzle:** The account has an empty NTLM hash, yet Windows login worked only via PIN and fingerprint. How?

**Answer:** Windows Hello moves authentication from the software world to hardware:

| Method | Storage Location | Offline Attack Possible? |
|---|---|---|
| Local password | SAM (file on disk) | ✅ Yes |
| Windows Hello PIN | TPM 2.0 (hardware chip) | ❌ No |
| Fingerprint | TPM 2.0 (hardware chip) | ❌ No |

**TPM (Trusted Platform Module)** is a dedicated cryptographic chip built into the motherboard. The PIN is encrypted and stored inside the TPM — it's not on the disk, it cannot be extracted via software. PIN verification happens in hardware, with a built-in failed-attempt counter.

---

### Phase 4: BitLocker and TPM Reconnaissance

**Objective:** Verify whether the drive is encrypted (BitLocker) without logging into Windows.

**TPM configuration read from BIOS/UEFI:**
```
TPM Specification Version: 2.0
TPM Device: Available
TPM State: Enabled
```

**BitLocker test — partition mount attempt:**
```bash
sudo mount /dev/nvme0n1p3 /mnt/win
# Result: mount: unknown filesystem type 'BitLocker'
```

This error confirms BitLocker is active. Linux cannot read an encrypted partition without the key.

**System security profile:**

| Security Feature | Status | Significance |
|---|---|---|
| TPM 2.0 | ✅ Active | BitLocker key and PIN hardware-protected |
| Secure Boot | ✅ Active | Protects against bootloader modification |
| BitLocker | ✅ Active | Full disk encryption |
| BIOS Password | ❌ Missing | Attack vector — boot order can be changed |
| Windows Local Password | ❌ Missing | Replaced by Windows Hello/TPM |

---

## Summary and Conclusions

### What Was Achieved
- ✅ Hands-on file recovery using file carving
- ✅ Offline NTLM hash extraction and analysis
- ✅ Understanding of Windows Hello architecture and TPM's role
- ✅ BitLocker activity verification without system access
- ✅ Full BIOS/TPM configuration reconnaissance

### Key Takeaways

1. **Empty NTLM hash ≠ weak security** — Windows Hello moves auth to TPM, making SAM-based attacks useless in modern configurations.

2. **TPM 2.0 + BitLocker is a solid barrier** — publicly available tools cannot bypass this combination without the recovery key.

3. **Reconnaissance without modification is possible** — Live USB allows gathering significant information about the target system without leaving traces or risking system lockout.

4. **No BIOS password is a real vulnerability** — an attacker with physical access can change boot order. In production environments, BIOS should be password-protected.

---

## Tools Used

| Tool | Purpose | Link |
|---|---|---|
| Kali Linux | Penetration testing distribution | [kali.org](https://www.kali.org) |
| PhotoRec | File carving, data recovery | bundled with testdisk |
| samdump2 | NTLM hash extraction | samdump2 package |
| John the Ripper | Hash analysis and cracking | [openwall.com](https://www.openwall.com/john/) |
| fdisk / mount | Partition management | Linux built-ins |

---

*All exercises conducted for educational purposes on personal equipment.*

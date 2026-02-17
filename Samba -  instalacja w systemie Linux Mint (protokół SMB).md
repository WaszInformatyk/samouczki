Oto kompletny przewodnik krok po kroku, jak skonfigurować serwer Samba na Linux Mint 22 (bazującym na Ubuntu 24.04), aby bezproblemowo współdzielić pliki z systemami Windows 10 i 11.

---

### Krok 1: Aktualizacja systemu i instalacja Samby

Otwórz terminal (Ctrl+Alt+T) i wprowadź poniższe komendy:

```bash
sudo apt update
sudo apt install samba samba-common-bin
```

### Krok 2: Konfiguracja użytkownika Samby

Samba używa oddzielnego zestawu haseł niż Twój systemowy login. Musisz dodać swojego użytkownika do bazy Samby.

1. Ustaw hasło dla swojego użytkownika (zastąp `twoja_nazwa` swoją nazwą użytkownika w Mint):
   ```bash
   sudo smbpasswd -a twoja_nazwa
   ```
   *Wpisz hasło dwukrotnie. Może być takie samo jak do systemu, ale Windows poprosi o nie przy pierwszym połączeniu.*

### Krok 3: Przygotowanie folderu do udostępnienia

Możesz udostępnić istniejący folder lub utworzyć nowy. Stwórzmy folder testowy w Twoim katalogu domowym:

```bash
mkdir ~/Publiczny_SMB
```

### Krok 4: Konfiguracja pliku smb.conf

Musimy edytować plik konfiguracyjny, aby wskazać Sambie, co i jak udostępniać.

1. Zrób kopię zapasową oryginalnego pliku:
   ```bash
   sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
   ```

2. Otwórz edytor:
   ```bash
   sudo xed /etc/samba/smb.conf
   ```

3. Przewiń na sam dół pliku i wklej poniższą konfigurację:

```ini
[Udostępnione]
   comment = Folder współdzielony Linux Mint
   path = /home/TWOJA_NAZWA/Publiczny_SMB
   browsable = yes
   read only = no
   guest ok = no
   create mask = 0755
   directory mask = 0755
   valid users = TWOJA_NAZWA
```
*Pamiętaj, aby zmienić `TWOJA_NAZWA` na swoją rzeczywistą nazwę użytkownika w ścieżce `path` oraz w `valid users`.*

4. Zapisz plik i zamknij edytor.

### Krok 5: Restart usług i Firewall

Aby zmiany weszły w życie, zrestartuj usługi:

```bash
sudo systemctl restart smbd nmbd
```

Jeśli masz włączony Firewall (UFW) w Mint, musisz zezwolić na ruch SMB:

```bash
sudo ufw allow samba
```

### Krok 6: (Opcjonalnie) Instalacja WSDD (Widoczność w sieci)

Współczesne systemy Windows (10 i 11) nie używają już starego protokołu NetBIOS do wyszukiwania komputerów. Aby Twój Linux był widoczny w sekcji "Sieć" w Eksploratorze plików Windows, warto zainstalować usługę `wsdd` (Web Services For Devices):

```bash
sudo apt install wsdd
```
Po instalacji usługa uruchomi się automatycznie i Twój Mint powinien pojawić się w otoczeniu sieciowym Windows.

---

### Krok 7: Dostęp z systemu Windows

1. Otwórz **Eksplorator plików** w Windows 10/11.
2. W pasku adresu wpisz adres IP swojego komputera z Mintem, poprzedzony dwoma backslashami:
   `\\192.168.1.XX` (zastąp właściwym adresem IP).
   *Adres IP sprawdzisz w Mint wpisując w terminalu `hostname -I`.*
3. Windows zapyta o poświadczenia:
   * **Użytkownik:** Twoja nazwa użytkownika z Minta.
   * **Hasło:** Hasło ustawione w Kroku 2 komendą `smbpasswd`.
4. Możesz zaznaczyć opcję "Zapamiętaj moje poświadczenia".

---

### Częste problemy i rozwiązania:

*   **Błąd uprawnień (Permission Denied):** Upewnij się, że folder na Linuxie ma odpowiednie uprawnienia. Możesz je nadać komendą: `chmod -R 755 ~/Publiczny_SMB`.
*   **Windows nie widzi komputera:** Jeśli IP działa, a nazwa komputera nie, upewnij się, że oba urządzenia są w tej samej grupie roboczej (standardowo `WORKGROUP`). Możesz to sprawdzić w `/etc/samba/smb.conf` w sekcji `[global]`.
*   **SMB1:** Nie włączaj SMB1 w Windows 10/11 – jest to niebezpieczne. Powyższa instrukcja używa nowoczesnego SMB2/3, które jest domyślnie wspierane przez Mint 22 i Windows 10/11.

### Jak sprawdzić IP w Linux Mint?
Wpisz w terminalu:
```bash
ip addr show | grep inet
```
Szukaj adresu zaczynającego się od `192.168...` przy Twojej karcie sieciowej (zazwyczaj `enp...` dla kabla lub `wlp...` dla Wi-Fi).

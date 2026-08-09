# Instrukcja: Konfiguracja kluczy SSH i zabezpieczanie serwera VPS (Ubuntu 24.04 / 26.04 LTS)

Niniejszy poradnik stanowi uzupełnienie materiału wideo i opisuje proces przejścia z tradycyjnego logowania hasłem na bezpieczne logowanie za pomocą kluczy kryptograficznych. Instrukcja dedykowana jest dla użytkowników systemów **Windows 11** oraz **Linux** korzystających z nowoczesnego terminala **Tabby**.
[Film YouTube - samouczek](https://youtu.be/NToLnieVSZE)

---

## Spis treści
1. [Instalacja i wstępna konfiguracja Tabby](#1-instalacja-i-wstępna-konfiguracja-tabby)
2. [Generowanie pary kluczy SSH](#2-generowanie-pary-kluczy-ssh)
3. [Wdrożenie klucza publicznego na serwer VPS](#3-wdrożenie-klucza-publicznego-na-serwer-vps)
4. [Konfiguracja profilu w Tabby](#4-konfiguracja-profilu-w-tabby)
5. [Zabezpieczanie serwera (Hardening SSH)](#5-zabezpieczanie-serwera-hardening-ssh)
6. [Złota zasada weryfikacji (Bardzo ważne!)](#6-złota-zasada-weryfikacji-bardzo-ważne)
7. [Zarządzanie wieloma kluczami (FAQ)](#7-zarządzanie-wieloma-kluczami-faq)

---

## 1. Instalacja i wstępna konfiguracja Tabby

Tabby to nowoczesny, wieloplatformowy terminal, który ułatwia zarządzanie wieloma sesjami SSH oraz oferuje wbudowany protokół SFTP do łatwego przesyłania plików.

### Krok po kroku:
1. Pobierz instalator dla swojego systemu operacyjnego:
   * **Windows 11:** Wybierz plik z rozszerzeniem `setup-x64.exe` ze strony producenta.
   * **Linux (np. Mint/Ubuntu):** Pobierz wersję `.AppImage` lub pakiet `.deb`.
2. Po uruchomieniu programu przejdź do ustawień (ikona trybika) i dostosuj aplikację:
   * **Język (Language):** Ustaw na Polski.
   * **Motyw (Appearance):** Wybierz jasny lub ciemny w zależności od preferencji.
   * **Czcionka:** W sekcji wygląd możesz zwiększyć rozmiar czcionki terminala (np. do 24-26 pkt), aby tekst był bardziej czytelny.
3. Utwórz pierwszy profil połączenia:
   * Przejdź do **Profile i połączenia** -> **Nowy profil** -> **Połączenie SSH**.
   * Nazwij profil (np. *Serwer VPS*), wprowadź adres IP serwera oraz nazwę użytkownika (domyślnie `ubuntu` na wielu maszynach VPS).
   * W sekcji hasła podaj aktualne hasło tekstowe użytkownika i zapisz profil.

---

## 2. Generowanie pary kluczy SSH

Zamiast podatnego na ataki brute-force hasła tekstowego, wygenerujemy parę kluczy: **prywatny** (zostaje na Twoim komputerze) oraz **publiczny** (trafia na serwer). Do tego celu wykorzystamy nowoczesny i bezpieczny algorytm **Ed25519**.

### Uwaga dotycząca katalogu `.ssh`
> *W systemach operacyjnych domyślnym miejscem przechowywania kluczy jest ukryty katalog `.ssh` (wypowiadany często jako „kropka-ssh”) znajdujący się w folderze domowym użytkownika:*
> * Windows: `C:\Users\Nazwa_Uzytkownika\.ssh\`
> * Linux: `/home/nazwa_uzytkownika/.ssh/` (lub skrótowo `~/.ssh/`)

### Generowanie klucza w systemie Windows 11 / Linux:
1. Otwórz nową kartę lokalną w Tabby (np. PowerShell lub CMD w Windows, Bash w Linux).
2. Wpisz następujące polecenie:
   ```bash
   ssh-keygen -t ed25519
   ```
3. **Nazwa pliku:** Program zapyta o lokalizację i nazwę pliku.
   * Jeśli chcesz użyć domyślnej ścieżki i nazwy (`id_ed25519`), naciśnij **Enter**.
   * Jeśli chcesz nadać własną nazwę (np. wyróżnić klucz dla danego systemu), wpisz ścieżkę i nazwę, np. `VPS_from_WIN` lub `VPS_from_LIN`.
4. **Hasło zabezpieczające (Passphrase):**
   * *Wariant wygodny (brak hasła):* Naciśnij **Enter** dwukrotnie. Klucz prywatny nie będzie zaszyfrowany.
   * *Wariant bezpieczny (rekomendowany):* Wpisz silne hasło zabezpieczające sam klucz prywatny. Każde użycie klucza będzie wymagało podania tego hasła (chroni to dostęp przed niepowołanymi osobami, które fizycznie przejmą Twój komputer/dysk).

Po zakończeniu procesu w wybranym katalogu pojawią się dwa pliki:
* `id_ed25519` (lub Twoja nazwa) – **klucz prywatny (ściśle tajny!)**
* `id_ed25519.pub` (lub Twoja nazwa.pub) – **klucz publiczny (ten wgrywamy na serwer)**

---

## 3. Wdrożenie klucza publicznego na serwer VPS

Zawartość klucza publicznego (plik z końcówką `.pub`) musi zostać dopisana do pliku `~/.ssh/authorized_keys` na serwerze Ubuntu. Możesz to zrobić na kilka sposobów.

### Metoda A: Jednolinijkowe polecenie PowerShell (Najszybsza dla Windows)
Wykonaj poniższe polecenie w lokalnym oknie PowerShell na swoim komputerze, zastępując `uzytkownik` oraz `adres_ip` danymi swojego serwera (jeśli zmieniłeś domyślną nazwę klucza z `id_ed25519.pub`, dostosuj ścieżkę):

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub" | ssh uzytkownik@adres_ip "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```
*System poprosi Cię o podanie dotychczasowego hasła do serwera. Po jego wpisaniu klucz zostanie przesłany i poprawnie wdrożony z odpowiednimi uprawnieniami.*

### Metoda B: Przesyłanie za pomocą Linux / macOS (Klasyczna)
Jeśli konfigurujesz serwer z poziomu systemu Linux lub macOS, użyj wbudowanego, standardowego narzędzia:
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub uzytkownik@adres_ip
```

### Metoda C: Metoda ręczna („myszkowa” z filmu przy użyciu Midnight Commander)
1. Otwórz plik klucza publicznego (np. `VPS_from_WIN.pub`) w dowolnym edytorze tekstowym na swoim komputerze i skopiuj jego zawartość (jedna linijka tekstu zaczynająca się od `ssh-ed25519`).
2. Zaloguj się na serwer i uruchom Midnight Commandera z poziomu swojego użytkownika:
   ```bash
   mc
   ```
3. Przejdź do katalogu domowego swojego użytkownika, a następnie do ukrytego katalogu `.ssh` (jeśli go nie ma, utwórz go).
4. Otwórz do edycji (klawisz **F4** lub stwórz nowy plik) plik o nazwie `authorized_keys`.
5. Wklej skopiowaną zawartość klucza publicznego jako nową linię (używając prawego przycisku myszy w Tabby).
6. Zapisz plik (**F2**) i wyjdź (**F10** lub Escape).

---

## 4. Konfiguracja profilu w Tabby

Gdy klucz publiczny znajduje się już na serwerze, musisz poinstruować klienta SSH (Tabby), aby używał klucza prywatnego zamiast hasła tekstowego.

1. W Tabby przejdź do **Ustawienia** -> **Profile i połączenia**.
2. Kliknij edycję swojego profilu połączenia z serwerem VPS.
3. W sekcji uwierzytelniania:
   * Usuń zapisane hasło tekstowe.
   * Przejdź do pola **Private keys** (Klucze prywatne).
   * Kliknij przycisk wyboru i wskaż plik ze swoim **kluczem prywatnym** (ten bez rozszerzenia `.pub`) na dysku lokalnym komputera.
4. Zapisz zmiany.
5. Przetestuj połączenie. Powinieneś zostać zalogowany bezpośrednio (lub po wpisaniu hasła odblokowującego klucz prywatny, jeśli je ustawiałeś).

---

## 5. Zabezpieczanie serwera (Hardening SSH)

Gdy logowanie kluczem działa prawidłowo, możemy bezpiecznie zablokować metody podatne na ataki (takie jak logowanie tradycyjnym hasłem oraz bezpośrednie logowanie na konto systemowe `root`).

Zgodnie z nowoczesnymi standardami systemów Ubuntu/Debian, zamiast modyfikować główny plik `/etc/ssh/sshd_config`, bezpieczniej jest utworzyć dedykowany plik konfiguracyjny w katalogu `.d`. Zapobiega to nadpisaniu naszych ustawień podczas przyszłych aktualizacji pakietu SSH.

### Krok po kroku:
1. Uruchom Midnight Commandera z uprawnieniami administratora (sudo):
   ```bash
   sudo mc
   ```
2. Przejdź do katalogu `/etc/ssh/sshd_config.d/`.
3. Utwórz nowy plik o nazwie zaczynającej się od niskiej liczby (np. `10-custom-settings.conf`), aby został wczytany jako jeden z pierwszych:
   * W mc wciśnij **F4** (lub użyj `sudo nano /etc/ssh/sshd_config.d/10-custom-settings.conf`).
4. Wklej do pliku następujące reguły:
   ```text
   # Wylaczenie logowania tradycyjnym haslem tekstowym
   PasswordAuthentication no

   # Wlaczenie autoryzacji kluczami publicznymi
   PubkeyAuthentication yes

   # Zablokowanie bezposredniego logowania na konto root
   PermitRootLogin no
   ```
5. Zapisz plik i zamknij edytor.

### Weryfikacja konfiguracji przed restartem (Bardzo ważna praktyka!)
Przed przeładowaniem usługi SSH zawsze warto sprawdzić, czy parser konfiguracji nie zgłasza błędów oraz czy system poprawnie interpretuje nasze nowe ustawienia. Wykonaj polecenie:

```bash
sudo sshd -T | grep -E "passwordauthentication|pubkeyauthentication|permitrootlogin"
```
*W wyjściu terminala powinieneś zobaczyć potwierdzenie wprowadzonych zmian:*
* `passwordauthentication no`
* `pubkeyauthentication yes`
* `permitrootlogin no`

### Restart usługi SSH:
Jeśli test składni przebiegł pomyślnie, przeładuj konfigurację demona SSH na serwerze:
```bash
sudo systemctl restart ssh
```

---

## 6. Złota zasada weryfikacji (Bardzo ważne!)

> ⚠️ **NIE ZAMYKAJ obecnej sesji terminala, w której konfigurujesz serwer!**

Jeśli popełniłeś literówkę w pliku konfiguracyjnym lub wdrożyłeś niewłaściwy klucz, zamknięcie aktywnego połączenia może bezpowrotnie zablokować Twój dostęp do serwera VPS.

1. Pozostaw obecną kartę w Tabby otwartą.
2. Otwórz **nową, osobną kartę** w Tabby i spróbuj połączyć się ze swoim profilem VPS.
3. **Scenariusz A (Sukces):** Logowanie kluczem przebiega poprawnie i masz dostęp do powłoki. Możesz bezpiecznie zamknąć poprzednią sesję. Twój serwer jest zabezpieczony.
4. **Scenariusz B (Błąd):** Nowe połączenie zostaje odrzucone (np. *Permission denied*). Nie panikuj. Wróć do pierwszej, wciąż otwartej karty, popraw błędy w pliku konfiguracyjnym w `/etc/ssh/sshd_config.d/`, zapisz zmiany, zrestartuj usługę SSH i spróbuj ponownie.

---

## 7. Zarządzanie wieloma kluczami (FAQ)

### Czy na jednym serwerze wirtualnym można umieścić wiele różnych kluczy SSH?
**Tak, jest to zalecana praktyka.** Plik `~/.ssh/authorized_keys` działa w ten sposób, że **każdy klucz publiczny zajmuje dokładnie jedną, osobną linijkę**. Podczas próby logowania serwer sprawdza ten plik linijka po linijce.

### Kiedy warto z tego korzystać?
1. **Wiele własnych urządzeń:** Najbezpieczniej jest wygenerować osobny klucz dla każdego swojego urządzenia (np. komputera stacjonarnego, laptopa roboczego, smartfona). W przypadku utraty lub kradzieży laptopa, wystarczy zalogować się z komputera stacjonarnego i usunąć z pliku `authorized_keys` jedną linijkę powiązaną ze zgubionym urządzeniem. Pozostałe klucze i urządzenia zachowają dostęp do serwera bez konieczności ich ponownej konfiguracji.
2. **Wielu administratorów/programistów:** Nigdy nie przekazuj swojego klucza prywatnego innym osobom. Jeśli chcesz dać komuś dostęp do serwera, poproś tę osobę o wygenerowanie własnej pary kluczy na jej komputerze i przesłanie jej *klucza publicznego* (z rozszerzeniem `.pub`), który następnie dopisujesz jako kolejną linijkę w pliku `authorized_keys`. Aby cofnąć dostęp danej osobie, wystarczy skasować powiązaną z nią linię.

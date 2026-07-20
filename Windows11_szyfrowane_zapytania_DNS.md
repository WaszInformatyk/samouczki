## 1. Czy szyfrowany DNS w przeglądarce zawsze działa?

**Niestety nie.** Istnieje kilka scenariuszy, w których ta funkcja może nie zadziałać lub zostać automatycznie wyłączona:

* **Tryb "Opportunistic" (Automatyczny):** Większość przeglądarek domyślnie działa w trybie wykrywania. Jeśli Twój obecny dostawca internetu (ISP) nie obsługuje szyfrowanego DNS, przeglądarka po cichu wraca do tradycyjnego, nieszyfrowanego protokołu portu 53. Myślisz, że jesteś bezpieczny, a w rzeczywistości zapytania lecą otwartym tekstem.
* **Blokady w sieciach korporacyjnych:** W środowiskach firmowych administratorzy często blokują znane serwery DoH na firewallach lub wdrażają specjalne polisy (Group Policy / MDM), które całkowicie wyłączają DoH w przeglądarkach. Robią to, aby móc monitorować ruch i chronić sieć przed złośliwym oprogramowaniem.
* **Programy kontroli rodzicielskiej i antywirusy:** Lokalne oprogramowanie filtrujące ruch może wymusić wyłączenie DoH, aby móc analizować, jakie strony odwiedzasz (szyfrowany DNS uniemożliwia im proste przechwytywanie zapytań).

---

## 2. Do jakich serwerów DNS wysyłane są zapytania?

To zależy od tego, jak funkcja została skonfigurowana. Mamy tu trzy główne scenariusze:

### A. Domyślni dostawcy wybrani przez przeglądarkę (Out-of-the-box)

Jeśli po prostu włączysz tę opcję i zdasz się na automat, przeglądarka skieruje Twój ruch do jednego ze swoich globalnych partnerów. Najczęściej są to giganci technologiczni:

| Przeglądarka                 | Domyślni / Sugerowani dostawcy DoH                       |
| ---------------------------- | -------------------------------------------------------- |
| **Mozilla Firefox**          | Cloudflare (`1.1.1.1`), NextDNS                          |
| **Google Chrome / Chromium** | Google Public DNS (`8.8.8.8`), Cloudflare, CleanBrowsing |
| **Microsoft Edge**           | Cloudflare, Google, Quad9 (`9.9.9.9`)                    |

> ⚠️ **Wskazówka dotycząca prywatności:** Choć firmy takie jak Cloudflare czy Quad9 deklarują rygorystyczne polityki prywatności (np. usuwanie logów po 24 godzinach), domyślne korzystanie z nich oznacza, że **centralizujesz swój ruch sieciowy** u jednego, amerykańskiego giganta.

### B. Twój lokalny dostawca internetu (ISP)

Jeśli przeglądarka korzysta z trybu automatycznego, a Twój ISP (np. Orange, Play, T-Mobile) wdrożył obsługę DoH na swoich serwerach DNS, przeglądarka zaszyfruje zapytania, ale nadal wyśle je do Twojego operatora.

### C. Własny, niestandardowy serwer (Najbardziej polecana opcja)

Jako zwolennik Open Source i pełnej kontroli, najbardziej rekomenduję tę opcję. Zarówno w Chrome, Firefox, jak i Brave możesz wpisać **własny adres URL profilu DoH**. Może to być:

* **Technitium** lub **NextDNS** – dające pełną kontrolę nad blokowaniem reklam i trackerów już na poziomie DNS.



## 3. Na co warto uważać? (Okiem specjalisty)

1. **Wycieki DNS (DNS Leaks):** Czasami system operacyjny i przeglądarka "rozjeżdżają się" w konfiguracji. Przeglądarka może wysyłać zapytania szyfrowane, ale cały pozostały ruch z systemu (np. klient Torrent, Spotify, gry) nadal korzysta z nieszyfrowanego DNS systemu operacyjnego. Zawsze warto to przetestować na stronie typu [dnsleaktest.com](https://www.dnsleaktest.com/).
2. **Szyfrowany DNS to nie VPN:** DoH chroni treść Twoich zapytań DNS przed podglądaniem przez ISP czy sąsiada w publicznej sieci Wi-Fi. **Nie ukrywa jednak Twojego adresu IP** przed docelową stroną, ani nie szyfruje samego ruchu internetowego (choć chroni przed sfałszowaniem odpowiedzi DNS, czyli tzw. spoofingiem).
3. **SNI wciąż zdradza cele:** Nawet przy włączonym DoH, podczas nawiązywania połączenia HTTPS wysyłana jest informacja SNI (Server Name Indication) zawierająca nazwę domeny otwartym tekstem. Dopiero wdrożenie standardu **ECH (Encrypted Client Hello)** w parze z DoH daje pełną prywatność.

Jeśli zależy Ci na maksymalnej prywatności, najlepszym podejściem jest **skonfigurowanie szyfrowanego DNS (DoH lub DoT) bezpośrednio w systemie operacyjnym** (Windows 11, macOS i Android wspierają to natywnie) lub na routerze domowym, zamiast polegać wyłącznie na samej przeglądarce.

//

Wdrożenie szyfrowanego DNS bezpośrednio w systemie operacyjnym Windows 11 to **bardzo dobry krok** – znacznie lepszy niż konfigurowanie go wyłącznie w przeglądarce. Dzięki temu chronisz zapytania sieciowe wysyłane przez **wszystkie aplikacje** w systemie (gry, komunikatory, programy pocztowe, aktualizacje systemowe), a nie tylko ruch z okna przeglądarki.

Oto jak to działa w praktyce, na co uważać oraz jakie adresy warto wpisać, aby zyskać realną prywatność i bezpieczeństwo.

---

## 1. Czy szyfrowany DNS w Windows 11 działa dobrze?

**Tak, pod warunkiem prawidłowej konfiguracji.** Windows 11 posiada natywne wsparcie dla standardów **DoH (DNS over HTTPS)** oraz **DoT (DNS over TLS)**.

System radzi sobie z tym stabilnie, ale musisz pamiętać o kilku technicznych aspektach:

* **Konfiguracja przypisana do karty sieciowej:** W Windowsie ustawienia DNS (w tym szyfrowanie) konfiguruje się osobno dla każdego połączenia (osobno dla Wi-Fi, osobno dla kabla Ethernet). Jeśli przełączysz się z kabla na Wi-Fi, musisz upewnić się, że tam również włączyłeś szyfrowanie.
* **Problem z "Wykrywaniem sieci" (Captive Portals):** W publicznych sieciach Wi-Fi (np. w hotelu, pociągu czy kawiarni), które wymagają zalogowania się na stronie powitalnej, szyfrowany DNS może początkowo uniemożliwić wyświetlenie tej strony. W takich sytuacjach Windows czasami musi na chwilę przełączyć się na klasyczny DNS, aby przejść autoryzację.
* **Tryby szyfrowania do wyboru:** Windows oferuje trzy opcje:
1. *Tylko niezaszyfrowane* (klasyczny DNS)
2. *Tylko zaszyfrowane (preferowane, rezerwowe niezaszyfrowane)* – jeśli szyfrowany DNS zawiedzie, system po cichu wyśle zapytanie otwartym tekstem.
3. *Tylko zaszyfrowane (wymagane)* – **najbezpieczniejsza opcja**. Jeśli bezpieczny serwer DNS nie odpowie, system zablokuje połączenie internetowe, zamiast ryzykować wyciek danych.

---

## 2. Jakie adresy DNS wpisać, aby zwiększyć bezpieczeństwo i prywatność?

Wybór serwera zależy od tego, co jest Twoim priorytetem: czysta prywatność, blokowanie złośliwego oprogramowania, czy może filtrowanie reklam i pornografii.

Poniżej znajdziesz listę najbardziej zaufanych dostawców, którzy wspierają szyfrowanie w Windows 11 (podaję adresy IPv4, ale Windows wspiera również ich odpowiedniki IPv6):

### Opcja A: Maksymalna prywatność i szybkość (Quad9 lub Cloudflare)

To najlepszy wybór do codziennego, bezpiecznego korzystania z sieci.

* **Quad9 (Szwajcaria – silna ochrona prywatności + blokowanie malware):**
* Statut prawny w Szwajcarii gwarantuje brak logowania Twoich zapytań. Dodatkowo Quad9 automatycznie blokuje domeny powiązane z phishingiem i złośliwym oprogramowaniem.
* **Preferowany DNS:** `9.9.9.9`
* **Alternatywny DNS:** `149.112.112.112`
* **Szablon DoH (dla Windows):** `https://dns.quad9.net/dns-query`

* **Cloudflare (Najszybszy ogólnodostępny DNS):**
* Świetna wydajność, niezależne audyty prywatności.
* **Preferowany DNS:** `1.1.1.1`
* **Alternatywny DNS:** `1.0.0.1`
* **Szablon DoH (dla Windows):** `https://cloudflare-dns.com/dns-query`

### Opcja B: Blokowanie reklam, trackerów i złośliwego oprogramowania (AdGuard lub Control D)

Dzięki tym serwerom DNS wiele reklam i skryptów śledzących na stronach oraz w aplikacjach zostanie zablokowanych, zanim w ogóle pobiorą się na Twój komputer.

* **AdGuard DNS (Filtrowanie reklam i trackerów):**
* **Preferowany DNS:** `94.140.14.14`
* **Alternatywny DNS:** `94.140.15.15`
* **Szablon DoH (dla Windows):** `https://dns.adguard-dns.com/dns-query`

* **Control D (Unfiltered / Secure – blokowanie malware):**
* Nowoczesna, bardzo stabilna usługa z rozbudowanymi opcjami konfiguracji.
* **Preferowany DNS:** `76.76.2.0`
* **Alternatywny DNS:** `76.76.10.0`
* **Szablon DoH (dla Windows):** `https://dns.controld.com/p2`

### Opcja C: Ochrona rodzicielska (Filtrowanie treści dla dorosłych)

Jeśli z komputera korzystają dzieci i chcesz zablokować dostęp do stron pornograficznych oraz niebezpiecznych.

* **Cloudflare Families (Malware + Adult Content block):**
* **Preferowany DNS:** `1.1.1.3`
* **Alternatywny DNS:** `1.0.0.3`
* **Szablon DoH (dla Windows):** `https://family.cloudflare-dns.com/dns-query`

---

## 3. Jak to poprawnie skonfigurować w Windows 11?

Aby poprawnie włączyć szyfrowanie, Windows 11 wymaga podania zarówno **adresu IP** serwera, jak i dopasowania go do odpowiedniego protokołu szyfrowania.

1. Otwórz **Ustawienia** ($Win + I$) -> **Sieć i internet**.
2. Kliknij na swoją aktywną sieć (np. **Wi-Fi** lub **Ethernet**).
3. Znajdź pozycję **Przypisanie serwera DNS** i kliknij **Edytuj**.
4. Zmień ustawienie z *Automatycznie (DHCP)* na **Ręcznie**.
5. Włącz przełącznik **IPv4** (oraz IPv6, jeśli z niego korzystasz).
6. W polu **Preferowany serwer DNS** wpisz np. `9.9.9.9` (dla Quad9).
7. W sekcji **Szyfrowanie preferowanego serwera DNS** wybierz: **Tylko zaszyfrowane (DNS over HTTPS)**. *(W starszych wersjach Windows 11 może być wymagane wpisanie pełnego szablonu DoH podanego w sekcji wyżej).*
8. Powtórz krok dla **Alternatywnego serwera DNS** (np. `149.112.112.112`).
9. Kliknij **Zapisz**.

Po zapisaniu ustawień pod adresami IP DNS powinieneś zobaczyć dopisek **(Szyfrowanie zabezpieczone)**. Od tego momentu Twój system wysyła zapytania w bezpiecznym, zaszyfrowanym tunelu, do którego wglądu nie ma Twój dostawca internetu.

//

 **Technitium DNS Server** to **świetne, a wręcz jedno z najlepszych rozwiązań** do zaawansowanej kontroli zapytań DNS pod systemem Windows 11.

Podczas gdy większość użytkowników domowych kojarzy blokowanie reklam i kontrolę DNS z projektami takimi jak **Pi-hole** czy **AdGuard Home**, Technitium gra w nieco innej, bardziej zaawansowanej lidze.

Oto szczegółowa analiza – z perspektywy administratora i entuzjasty sieci – dlaczego warto (lub nie) wybrać to narzędzie na Windows 11.

---

## Dlaczego Technitium wyróżnia się na Windows 11?

### 1. Działa natywnie na Windowsie

W przeciwieństwie do Pi-hole (który wymaga Linuxa, Dockera lub kombinowania z WSL na Windowsie), Technitium jest napisane w .NET. Oznacza to, że pobierasz instalator dla Windowsa, a program **instaluje się jako natywna usługa systemowa**. Nie potrzebujesz maszyn wirtualnych ani kontenerów, aby mieć potężny serwer DNS działający 24/7 na Twoim komputerze.

### 2. To pełnoprawny, autorytatywny i rekurencyjny serwer DNS

Pi-hole i AdGuard Home to w gruncie rzeczy zaawansowane "forwardery" (przekierowywacze) – odbierają zapytanie od klienta i pytają zewnętrzny serwer (np. Cloudflare czy Google).

* **Technitium może działać jako serwer rekurencyjny (jak Unbound)** – sam pyta bezpośrednio serwery główne (Root Servers) i autorytatywne dla danej domeny. Żaden zewnętrzny dostawca DNS (nawet szyfrowany) nie widzi wtedy historii Twoich zapytań.
* Posiada pełne zarządzanie strefami (Zone Management), obsługę rekordów (A, AAAA, CNAME, TXT, MX) oraz pozwala na lokalne mapowanie domen w Twojej sieci domowej.

### 3. Bezkompromisowe wsparcie dla nowoczesnego szyfrowania (DoH, DoT, DoQ)

Technitium jest niesamowicie elastyczne pod kątem protokołów:

* Obsługuje szyfrowanie zapytań wysyłanych "w górę" (upstream) oraz odbieranych lokalnie (client-facing DoH/DoT/DoQ).
* Jako jeden z nielicznych serwerów DNS wspiera najnowszy protokół **DoQ (DNS-over-QUIC)** oraz **HTTP/3**. Windows 11 natywnie obsługuje QUIC, co pozwala na zestawienie ekstremalnie szybkiego i bezpiecznego tunelu DNS między systemem a serwerem.

### 4. Zaawansowane funkcje sieciowe (VLANy i DHCP)

W przeciwieństwie do prostszych rozwiązań, wbudowany serwer DHCP w Technitium pozwala na konfigurację wielu zakresów adresów IP (np. dla różnych sieci VLAN). To kluczowa funkcja, jeśli budujesz zaawansowaną sieć domową (Home Lab) i chcesz odizolować urządzenia IoT od głównych komputerów.

---

## Gdzie tkwi haczyk? (Wady i wyzwania)

* **Krzywa uczenia się:** Interfejs Technitium, choć przejrzysty, przypomina profesjonalne narzędzie administracyjne (jak BIND czy serwer DNS w Windows Server). Osoba szukająca prostego włącznika blokady reklam może poczuć się przytłoczona ilością opcji.
* **Zarządzanie grupami klientów:** Jeśli chcesz zdefiniować, że "Komputer A ma blokowane reklamy, a Komputer B nie", w Pi-hole zrobisz to kilkoma kliknięciami w GUI. W Technitium konfiguracja zaawansowanego filtrowania per-klient często opiera się na aplikacjach (Apps/Plugins) i czasami wymaga edycji plików konfiguracyjnych JSON.
* **Brak "wygładzonego" marketingu:** Projekt rozwija się fantastycznie (częste aktualizacje, genialne wsparcie dla klastrowania/synchronizacji w nowszych wersjach), ale ma mniejszą społeczność niż Pi-hole, przez co trudniej o gotowe poradniki "krok po kroku" dla laików.

---

## Szybkie porównanie: Co wybrać?

| Cecha                         | Pi-hole                          | AdGuard Home                 | Technitium DNS                          |
| ----------------------------- | -------------------------------- | ---------------------------- | --------------------------------------- |
| **Instalacja na Windows 11**  | ❌ Trudna (wymaga WSL/Docker)     | 🟡 Średnia (jako binarka Go) | **Bardzo łatwa** (instalator MSI)       |
| **Podstawowe przeznaczenie**  | Blokowanie reklam                | Blokowanie reklam i ochrona  | **Pełny, profesjonalny serwer DNS**     |
| **Szyfrowanie (DoH/DoT/DoQ)** | ❌ Wymaga proxy (np. cloudflared) | Wbudowane (natywne)          | **Najpełniejsze wsparcie (w tym QUIC)** |
| **Prostota obsługi**          | Bardzo łatwa                     | Bardzo łatwa                 | 🟡 Wymaga wiedzy sieciowej              |

//

Konfiguracja **Technitium DNS Server** (wersja 15.4) w systemie Windows 11 do obsługi bezpiecznego, szyfrowanego DNS (DoH, DoT lub DoQ) jest bardzo prosta. Dzięki temu zapytania wysyłane z Twojej sieci do zewnętrznych serwerów (np. Cloudflare czy Quad9) będą w pełni zaszyfrowane, co uniemożliwi dostawcy internetu (ISP) podglądanie, jakie strony odwiedzasz.

Oto instrukcja krok po kroku, jak to skonfigurować:

---

## Krok 1: Wejdź do konsoli administracyjnej

1. Otwórz przeglądarkę internetową i przejdź pod adres panelu Technitium (domyślnie):
   `http://localhost:5380` lub `http://127.0.0.1:5380`
2. Zaloguj się na swoje konto administratora.

---

## Krok 2: Konfiguracja szyfrowanych serwerów przekierowujących (Forwarders)

Aby ukryć ruch przed dostawcą internetu, musisz wyłączyć domyślą bezpośrednią rekurencję (która odpytuje serwery root otwartym tekstem) i włączyć tzw. **Forwardery** używające bezpiecznych protokołów.

1. W menu u góry kliknij zakładkę **Settings** (Ustawienia).
2. Z menu po lewej stronie wybierz sekcję **Proxy & Forwarders**.
3. Znajdź sekcję **Forwarders** (Przekierowania).
4. W polu tekstowym wpisz adresy bezpiecznych i dbających o prywatność serwerów DNS wraz z wybranym protokołem szyfrowania. Poniżej znajdziesz najlepsze, gotowe konfiguracje do skopiowania:

#### Opcja A: Cloudflare (Szybkość i prywatność)

Służy do szybkiego przeglądania z podstawową ochroną. Wklej te adresy (każdy w nowej linii):

* **DNS-over-TLS (DoT):**
  
  ```text
  1.1.1.1:853
  1.0.0.1:853
  ```

```
* **DNS-over-HTTPS (DoH):**
```text
https://cloudflare-dns.com/dns-query
```

#### Opcja B: Quad9 (Bezpieczeństwo – blokuje złośliwe oprogramowanie i phishing)

Świetny wybór pod kątem bezpieczeństwa, prowadzony przez fundację non-profit ze Szwajcarii.

* **DNS-over-TLS (DoT):**
  
  ```text
  9.9.9.9:853
  149.112.112.112:853
  ```

```
* **DNS-over-HTTPS (DoH):**
```text
https://dns.quad9.net/dns-query
```

#### Opcja C: Mullvad DNS (Maksymalna anonimowość i brak logów)

Dostawca stawiający na prywatność, nie loguje żadnych zapytań.

* **DNS-over-HTTPS (DoH):**
  
  ```text
  https://dns.mullvad.net/dns-query
  ```

💡 **Wskazówka:** Możesz wymieszać np. Cloudflare i Quad9, wpisując je linia pod linią. Technitium automatycznie obsłuży zapytania.

5. Po wpisaniu adresów, upewnij się, że opcja **Protocol** (tuż pod listą forwarderów) jest ustawiona na odpowiednią wartość:
* Jeśli użyłeś adresów z portem `:853` (DoT) – wybierz **TLS**.
* Jeśli użyłeś adresów URL `https://...` (DoH) – wybierz **HTTPS** (lub *HTTPS (HTTP/1.1 or HTTP/2)* / *HTTPS (HTTP/3)* w zależności od preferencji – HTTP/3 jest najszybszy, jeśli upstream go wspiera).


6. Kliknij przycisk **Save Settings** na dole strony.

---

## Krok 3: Włączenie walidacji DNSSEC (Dodatkowe zabezpieczenie)

DNSSEC chroni przed sfałszowaniem odpowiedzi DNS (np. przed przekierowaniem na fałszywą stronę banku).

1. W zakładce **Settings** przejdź do sekcji **Recursive Resolver** (po lewej stronie).
2. Upewnij się, że opcja **DNSSEC Validation** jest zaznaczona jako **Enabled** (lub *Validate*).
3. Kliknij **Save Settings**.

---

## Krok 4: Skierowanie systemu Windows 11 na Twój serwer Technitium

Aby Twój komputer (i inne urządzenia w sieci) zaczął korzystać z nowo skonfigurowanego serwera, musisz zmienić ustawienia sieciowe w Windows 11.

1. Kliknij Start, wpisz **Ustawienia** i otwórz je.
2. Przejdź do: **Sieć i internet** -> **Wi-Fi** lub **Ethernet** (zależy jak łączysz się z routerem).
3. Kliknij we właściwości swojej sieci (np. "Właściwości sprzętu").
4. Znajdź pozycję **Przypisanie serwera DNS** i kliknij **Edytuj**.
5. Zmień ustawienie z *Automatycznie (DHCP)* na **Ręcznie**.
6. Włącz suwak **IPv4** i w polu **Preferowany serwer DNS** wpisz:
`127.0.0.1` (to lokalny adres pętli zwrotnej, wskazujący na Technitium uruchomione na tym samym PC).
7. Kliknij **Zapisz**.

---

## Jak sprawdzić, czy wszystko działa?

1. Wróć do konsoli Technitium i wejdź w zakładkę **Dashboard**.
2. Zacznij przeglądać internet na komputerze. Zauważysz, że wykresy w panelu Technitium zaczną rosnąć (liczba zapytań wzrośnie).
3. Możesz też wejść na stronę [Cloudflare Connection Help](https://1.1.1.1/help) lub [browserleaks.com/dns](https://browserleaks.com/dns), aby upewnić się, że Twoje zapytania przechodzą przez wybranego bezpiecznego dostawcę i nie ma tam śladu serwerów Twojego dostawcy internetu (ISP).

//

Wyjaśnienie tego problemu tkwi w sposobie, w jaki działa aplikacja w zasobniku (tray app) Technitium oraz jak Twój system Windows 11 komunikuje się z serwerem DNS.

Logi, które przesłałeś, pokazują błąd **`RCODE: ServerFailure`** przy próbie odpytania o domenę `www.wp.pl`. Oznacza to, że lokalny serwer Technitium odbiera zapytanie od Twojego komputera (`127.0.0.1`), ale **nie jest w stanie uzyskać na nie odpowiedzi z sieci (z serwerów nadrzędnych)** i zwraca błąd.

Oto główne przyczyny tego stanu i sposoby na ich natychmiastowe rozwiązanie:

---

## Przyczyna 1: Blokada "pętli zwrotnej" (Zapętlenie zapytań)

Najczęstszy błąd przy konfiguracji na jednym komputerze (tzw. localhost).

Kiedy w aplikacji w zasobniku systemowym (lub w ustawieniach karty sieciowej w Windows) wybierasz **Google** lub **Quad9**, system Windows wysyła zapytania DNS bezpośrednio do internetu (np. pod `8.8.8.8`). Wszystko działa.

Gdy jednak przełączysz na **Technitium**, system zaczyna kierować zapytania do lokalnej usługi na adres `127.0.0.1` (twój własny komputer). Jeśli w samej konfiguracji Technitium (w panelu `http://localhost:5380`):

1. **Nie ustawiłeś żadnych "Forwarders"** (serwerów przekierowujących) LUB
2. **Jako "Forwarder" wpisałeś adres pętli zwrotnej** (np. `127.0.0.1` lub `localhost`),

wtedy Technitium próbuje pytać samego siebie w nieskończoność. Po chwili serwer się poddaje i zwraca widoczny w logach błąd `ServerFailure`.

### Rozwiązanie:

1. Wejdź do panelu administracyjnego w przeglądarce: `http://localhost:5380`.
2. Przejdź do **Settings** -> **Proxy & Forwarders**.
3. Upewnij się, że w sekcji **Forwarders** wpisane są **zewnętrzne** adresy IP (zgodnie z poprzednim krokiem – np. `9.9.9.9:853` dla Quad9 lub `https://cloudflare-dns.com/dns-query` dla Cloudflare), a **NIE** adres lokalny komputera ani pusta lista (jeśli domyślna rekurencja jest blokowana przez Twojego dostawcę).

---

## Przyczyna 2: Błąd walidacji DNSSEC (Problem z czasem systemowym lub kluczami)

W logach widać, że serwer próbuje wykonać zapytanie rekurencyjne (`RecursiveResolverBackgroundTaskAsync`), które kończy się niepowodzeniem. Bardzo częstym winowajcą w Technitium jest funkcja **DNSSEC**.

Jeśli czas systemowy w Twoim Windowsie różni się choćby o kilkadziesiąt sekund od czasu rzeczywistego (lub strefa czasowa jest błędna), walidacja DNSSEC uzna wszystkie odpowiedzi z internetu za sfałszowane i je zablokuje, zwracając właśnie `ServerFailure`.

### Rozwiązanie (na próbę):

1. W panelu Technitium przejdź do **Settings** -> **Recursive Resolver**.
2. Znajdź opcję **DNSSEC Validation** i tymczasowo ustaw ją na **Disabled** (Wyłączone).
3. Kliknij **Save Settings** na dole strony.
4. Sprawdź, czy po przełączeniu karty sieciowej w zasobniku na "Technitium" strony zaczęły się ładować. Jeśli tak – przyczyną jest DNSSEC (wtedy upewnij się, że czas w Windows 11 jest idealnie zsynchronizowany z serwerem czasu NTP).

---

## Przyczyna 3: Zapora sieciowa (Windows Firewall) blokująca Technitium

Usługa Technitium działa jako proces w tle. Windows Defender Firewall mógł zablokować jej dostęp do sieci zewnętrznej w profilu sieci prywatnej/publicznej. W efekcie Technitium słyszy Twój komputer, ale system Windows nie pozwala samej aplikacji "wyjść" do internetu, by pobrać dane.

### Rozwiązanie:

1. Otwórz menu Start, wpisz **Zezwalaj aplikacji na dostęp przez Zaporę systemu Windows** i uruchom to narzędzie.
2. Kliknij przycisk **Zmień ustawienia** (wymaga praw administratora).
3. Znajdź na liście **Technitium DNS Server** (lub proces `DnsServer.exe`).
4. Upewnij się, że zaznaczone są ptaszki zarówno przy kolumnie **Prywatne**, jak i **Publiczne**.
5. Kliknij **OK**.

//

Super! Bardzo się cieszę, że udało się znaleźć przyczynę – rozbieżność czasu systemowego (nawet o kilkadziesiąt sekund) to absolutny klasyk przy problemach z walidacją DNSSEC, ponieważ certyfikaty bezpieczeństwa i sygnatury kryptograficzne mają bardzo rygorystyczne ramy czasowe ważności.

Co do opcji w zakładce **Settings -> Recursion**:

Ustawienie, które masz obecnie zaznaczone, czyli:

> **Allow Recursion Only For Private Networks (default)**

jest **najlepszym i najbezpieczniejszym wyborem** dla Twojej konfiguracji.

---

### Dlaczego to ustawienie jest optymalne?

1. **Bezpieczeństwo przed atakami (DNS Amplification):** Gdybyś wybrał opcję *Allow Recursion* (zezwól wszystkim), a Twój komputer miałby bezpośrednio publiczny adres IP w internecie lub przekierowany port 53 na routerze, każdy z zewnątrz mógłby użyć Twojego serwera do odpytywania o domeny. Cyberprzestępcy często wykorzystują takie "otwarte resolvery" (open resolvers) do przeprowadzania zmasowanych ataków DDoS na inne cele.
2. **Ochrona prywatności i zasobów:**
Opcja *Allow Recursion Only For Private Networks* pozwala na odpytywanie serwera Technitium tylko urządzeniom z Twojej sieci lokalnej (np. Twój PC o adresie `127.0.0.1` oraz inne urządzenia w domu z adresami typu `192.168.x.x` czy `10.x.x.x`). Każde zapytanie, które próbowałoby nadejść z publicznego internetu, zostanie natychmiast odrzucone.

---

### Podsumowanie – jak to zostawić?

* **Pozostaw zaznaczone:** `Allow Recursion Only For Private Networks (default)`.
* Dzięki temu Technitium bezpiecznie obsłuży Twój komputer i urządzenia w Twojej sieci domowej, pozostając całkowicie zamkniętym i bezpiecznym dla świata zewnętrznego.

//

W Technitium DNS Server konfiguracja blokowania reklam jest niezwykle prosta, ponieważ twórcy wbudowali w panel bardzo wygodną funkcję tzw. **Quick Add** (Szybkie dodawanie). Nie musisz nawet ręcznie szukać i wklejać linków!

Oto jak włączyć najpopularniejsze i najbardziej polecane listy blokujące:

---

## Jak dodać listy za pomocą "Quick Add" (Zalecane)

1. W panelu Technitium przejdź do: **Settings** -> **Blocking**.
2. W sekcji **Block List URLs** kliknij przycisk **Quick Add** (zwykle znajduje się po prawej stronie pola tekstowego lub pod nim).
3. Pojawi się okienko z listą najpopularniejszych filtrów na świecie. Zaznacz "ptaszkiem" te najbardziej polecane:
* **StevenBlack List** – absolutny standard i klasyk. Świetna, uniwersalna lista blokująca reklamy, adware, malware i telemetrię, która bardzo rzadko "psuje" normalne strony internetowe.
* **AdGuard DNS Filter** lub **OISD (Light/Big)** – niezwykle skuteczne listy, doskonale radzące sobie z trackerami i reklamami.


4. Kliknij **Add Selected** (Dodaj wybrane).
5. Na samym dole strony kliknij **Save Settings**, aby zapisać zmiany.

---

## Chcesz dodać listę ręcznie? Oto najlepsze adresy:

Jeśli wolisz wkleić adresy ręcznie do pola **Block List URLs** (każdy adres w nowej linii), oto dwie najlepsze i najbardziej aktualne listy na świecie:

1. **HaGeZi - Multi Light** (Najlepszy balans – blokuje reklamy, tracking i malware, nie blokując przydatnych usług):
```text
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/wildcard/light.txt
```

2. **StevenBlack (Standardowa lista)**:
   
   ```text
   https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
   ```


Po wklejeniu wybranego linku kliknij **Save Settings**.

---

### Ważna uwaga na koniec:

Technitium domyślnie pobiera te listy raz na dobę i automatycznie je aktualizuje w tle. Gdy zaczniesz przeglądać sieć, na głównym pulpicie (**Dashboard**) w sekcji statystyk zobaczysz rosnący wykres **Blocked Queries** – to znak, że serwer skutecznie wycina reklamy zanim w ogóle dotrą do Twojej przeglądarki!

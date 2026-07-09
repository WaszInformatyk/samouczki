Konfiguracja lokalnego serwera **Unbound** ver 1.19.2 w połączeniu z szyfrowaniem **DoT (DNS over TLS)** lub **DoH (DNS over HTTPS)** to doskonały krok w stronę pełnej prywatności.

W systemie **Linux Mint 22.3** (opartym na bazie Ubuntu) najczystszym i najbardziej wydajnym rozwiązaniem jest konfiguracja **Unbound jako lokalnego resolvera szyfrującego za pomocą DoT** i przekazującego zapytania do serwerów dbających o prywatność (np. Quad9 lub Cloudflare). DoT jest łatwiejszy w konfiguracji systemowej i lżejszy dla serwera DNS niż DoH.

Oto kompletna lista kroków, aby to osiągnąć.

---

### Krok 1: Instalacja serwera Unbound

Otwórz terminal (`Ctrl + Alt + T`) i zaktualizuj listę pakietów, a następnie zainstaluj Unbound:

```bash
sudo apt update
sudo apt install unbound -y
```

### Krok 2: Pobranie pliku certyfikatów i serwerów Root (Dobre praktyki)

Unbound potrzebuje wiedzieć, skąd czerpać zaufanie do certyfikatów SSL/TLS. W systemach debianowych plik ten znajduje się w standardowej lokalizacji, ale warto upewnić się, że Unbound ma dostęp do aktualnych certyfikatów CA:

```bash
sudo curl -o /var/lib/unbound/ca-certificates.crt https://curl.se/ca/cacert.pem
```

### Krok 3: Konfiguracja Unbound dla DoT (DNS over TLS)

Musimy utworzyć plik konfiguracyjny, który zmusi Unbound do szyfrowania zapytań wychodzących za pomocą protokołu TLS na porcie 853.

Utwórz nowy plik konfiguracyjny (np. `privacy.conf`) w katalogu Unbound:

```bash
sudo xed /etc/unbound/unbound.conf.d/privacy.conf
```

Wklej do niego poniższą konfigurację (używamy bezpiecznych serwerów Quad9 i Cloudflare jako serwerów nadrzędnych – tzw. forwarders):

```text
# Główna sekcja serwera
server:
    # Nasłuchiwanie wyłącznie na lokalnej pętli zwrotnej (localhost)
    interface: 127.0.0.1
    interface: ::1
    port: 53

    # Domyślnie blokujemy wszystko poza zapytaniami z lokalnego komputera
    access-control: 127.0.0.0/8 allow
    access-control: ::1 allow

    # Prywatność: Ukrywanie tożsamości i wersji serwera Unbound
    hide-identity: yes
    hide-version: yes
    hide-trustanchor: yes

    # Minimalizacja przesyłanych informacji o zapytaniu do serwerów nadrzędnych
    # Zapobiega wyciekowi pełnej nazwy domenowej na wyższych szczeblach hierarchii
    qname-minimisation: yes

    # Zabezpieczenia (Hardenings) przed podszywaniem się pod odpowiedzi i atakami
    harden-glue: yes
    harden-dnssec-stripped: yes
    harden-below-nxdomain: yes
    harden-short-bufsize: yes

    # Optymalizacja pamięci podręcznej i wydajności
    cache-min-ttl: 300       # Wydłuża czas przechowywania wpisów w cache (zmniejsza ruch)
    cache-max-ttl: 86400     # Maksymalny czas przechowywania wpisów (1 dzień)
    prefetch: yes            # Odświeżanie popularnych wpisów przed ich wygaśnięciem
    prefetch-key: yes        # Odświeżanie kluczy DNSSEC
    rrset-roundrobin: yes
    minimal-responses: yes

    # Szyfrowanie ruchu wychodzącego (DNS over TLS)
    tls-upstream: yes
    # Ścieżka do systemowego magazynu zaufanych certyfikatów CA w Linux Mint
    tls-cert-bundle: "/etc/ssl/certs/ca-certificates.crt"

    # Walidacja DNSSEC - automatyczne pobieranie klucza głównego (Root Anchor)
    # Ścieżka domyślna dla pakietu unbound w dystrybucjach Debian/Ubuntu
  # auto-trust-anchor-file: "/var/lib/unbound/root.key"

    # Ochrona przed atakami DNS Rebinding (blokada adresów prywatnych w odpowiedziach publicznych)
    private-address: 10.0.0.0/8
    private-address: 172.16.0.0/12
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: fd00::/8
    private-address: fe80::/10

# Przekierowanie ruchu (Forward Zone) do bezpiecznych dostawców DoT
forward-zone:
    name: "."
    forward-tls-upstream: yes

    # Dostawca 1: Quad9 (Szwajcaria - brak logów, ochrona przed złośliwym oprogramowaniem)
    forward-addr: 9.9.9.9@853#dns.quad9.net
    forward-addr: 149.112.112.112@853#dns.quad9.net
    forward-addr: 2620:fe::fe@853#dns.quad9.net

    # Dostawca 2: Mullvad DNS (Szwecja - silne nastawienie na prywatność, brak logów)
    forward-addr: 194.242.2.4@853#dns.mullvad.net
    forward-addr: 2a07:e340::4@853#dns.mullvad.net
```

*Zapisz plik naciskając `Ctrl + O`, zatwierdź Enterem, a następnie wyjdź za pomocą `Ctrl + X`.*

### Krok 4: Weryfikacja i restart usługi Unbound

Zanim zrestartujesz usługę, sprawdź, czy w konfiguracji nie ma błędów składniowych:

```bash
sudo unbound-checkconf
```

Jeśli system nie zwróci błędów (*no errors in...*), zrestartuj i włącz usługę Unbound, aby uruchamiała się przy starcie systemu:

```bash
sudo systemctl restart unbound
sudo systemctl enable unbound
```

Możesz sprawdzić, czy Unbound poprawnie nasłuchuje na porcie 53:

```bash
dig @127.0.0.1 -p 53 linuxmint.com
```

Jeśli otrzymasz sekcję `ANSWER SECTION` z adresem IP, Unbound działa prawidłowo.

### Krok 5: Skierowanie systemu Linux Mint na lokalny serwer DNS

Linux Mint 22.3 domyślnie zarządza siecią i DNS-ami poprzez usługę **NetworkManager** oraz `systemd-resolved`. Musimy wskazać systemowi, aby jako główny serwer DNS traktował nasz lokalny serwer Unbound (`127.0.0.1` na porcie 5335).

Najprostszym i najtrwalszym sposobem w Linux Mint jest edycja ustawień połączenia sieciowego w GUI lub przez plik konfiguracyjny NetworkManagera.

#### Metoda przez Graficzny Interfejs (Najprostsza):

1. Kliknij ikonę sieci (Wi-Fi lub Ethernet) na pasku zadań i wybierz **Ustawienia sieci** (Network Settings).
2. Kliknij ikonę zębatego koła (Ustawienia) obok swojego aktywnego połączenia.
3. Przejdź do zakładki **IPv4**.
4. Wyłącz suwak **Automatycznie (DNS)** / *Automatic (DNS)*.
5. W polu **DNS** wpisz: `127.0.0.1`
6. Kliknij **Zastosuj** (Apply).
7. Rozłącz się z siecią i połącz ponownie (lub zrestartuj komputer), aby zmiany weszły w życie.

#### Metoda dla zaawansowanych (Globalna zmiana w NetworkManager):

Jeśli chcesz, aby dotyczyło to *wszystkich* sieci (np. każdej nowej sieci Wi-Fi), utwórz plik nadpisujący w NetworkManagerze:

```bash
sudo xed /etc/NetworkManager/conf.d/dns.conf
```

Wpisz tam:

```text
[main]
dns=none
```

Następnie ręcznie ustaw plik `/etc/resolv.conf`, aby wskazywał na Unbound (wymaga to wcześniejszego usunięcia dowiązania symbolicznego `systemd-resolved`, jeśli blokuje port 53):

```bash
sudo rm /etc/resolv.conf
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
```

*(Uwaga: Ponieważ nasz Unbound działa na porcie 5335, standardowy system szuka DNS na porcie 53. Aby to zmapować, najwygodniej zmienić w pliku `privacy.conf` z Kroku 3 linijkę `port: 5335` na `port: 53` – pod warunkiem, że wyłączysz systemowe `systemd-resolved` komendą `sudo systemctl stop systemd-resolved && sudo systemctl disable systemd-resolved`).*

### Krok 6: Testowanie anonimowości i szyfrowania

Aby upewnić się, że Twoje zapytania są teraz bezpieczne, zaszyfrowane i nie wyciekają do Twojego dostawcy internetu (ISP), odwiedź jedną z poniższych stron testowych:

1. **[https://www.dnsleaktest.com/](https://www.dnsleaktest.com/)** – Uruchom "Extended test". Na liście serwerów DNS powinieneś zobaczyć wyłącznie serwery Cloudflare lub Quad9 (w zależności od tego, co wpisałeś w konfiguracji Unbound). Nie powinieneś tam widzieć serwerów swojego operatora internetowego.

///

Unbound posiada wbudowany **mechanizm pamięci podręcznej (Cache)**. Oznacza to, że po zapytaniu o dany adres (np. `linuxmint.com`) i odebraniu odpowiedzi przez zaszyfrowany tunel DoT, Unbound zapisuje parę `nazwa domeny -> adres IP` w pamięci RAM Twojego komputera.

Oto jak ten mechanizm działa w praktyce i jak wpływa na Twoją prywatność oraz szybkość działania:

---

## 1. Jak długo Unbound przechowuje te dane? (Czas TTL)

Unbound nie przechowuje adresów w nieskończoność. Czas życia każdego wpisu w pamięci podręcznej jest ściśle regulowany przez tzw. **TTL (Time to Live)**.

* TTL jest określane przez właściciela danej domeny (autorytatywny serwer DNS), a nie przez Twój komputer.
* Zazwyczaj wynosi ono od kilku minut do 24 godzin (często jest to np. 3600 sekund, czyli 1 godzina).
* Gdy czas TTL dla danego wpisu minie, Unbound automatycznie usuwa go z pamięci i przy kolejnej próbie wejścia na tę stronę ponownie wyśle bezpieczne zapytanie w świat.

---

## 2. Co to oznacza dla wydajności? (Błyskawiczne odpowiedzi)

Gdy wchodzisz na jakąś stronę po raz drugi w ciągu tej samej godziny:

1. Przeglądarka pyta system, system pyta Unbound.
2. Unbound widzi, że ma ten adres w swojej pamięci RAM.
3. Zwraca adres IP **w ułamku milisekundy (często $< 1\text{ ms}$)**.
4. Zapytanie w ogóle nie opuszcza Twojego komputera – nie leci przez sieć, nie obciąża łącza i nie generuje żadnego ruchu na zewnątrz.

---

## 3. Co to oznacza dla Twojej prywatności? (Lokalna bariera)

Z punktu widzenia prywatności ten cache to świetna sprawa:

* **Dane nie opuszczają Twojej maszyny:** Pamięć podręczna Unbound znajduje się wyłącznie w pamięci RAM Twojego komputera z Linux Mint. Nikt z zewnątrz nie ma do niej dostępu.
* **Mniej zapytań do sieci:** Ponieważ Unbound pamięta adresy przez czas ich TTL, wysyłasz znacznie mniej zapytań do zewnętrznych serwerów (np. Quad9). Nawet te zaufane serwery rzadziej widzą, z jakich usług korzystasz.

---

## Jak sprawdzić pamięć podręczną lub ją wyczyścić?

Jeśli chcesz zobaczyć, czy Unbound faktycznie zapisał adres w pamięci, możesz użyć polecenia `dig`. Przy drugim zapytaniu o tę samą domenę linijka `Query time` powinna wynosić `0 ms` lub `1 ms`:

```bash
dig @127.0.0.1 -p 53 google.com | grep "Query time"
```

gdybyś kiedyś zmienił konfigurację jakiejś domeny i chciał **natychmiast opróżnić pamięć podręczną** Unbound (np. z powodu problemów z ładowaniem strony), możesz to zrobić jednym poleceniem w terminalu:

```bash
sudo unbound-control flush_zone .
```

*(Komenda ta czyści całą pamięć podręczną resolvera).*

///



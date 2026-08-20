
# Instrukcja instalacji WireGuard (WG-Easy v15) na Ubuntu 26.04 LTS

Niniejszy poradnik opisuje wdrożenie serwera VPN **WireGuard** z nowoczesnym panelem webowym **WG-Easy v15** na systemie **Ubuntu 26.04 LTS**. Instrukcja uwzględnia obejście problemów z AppArmor oraz obsługą modułów jądra (`iptables`), które są specyficzne dla najnowszych dystrybucji Ubuntu w środowiskach kontenerowych.

---

## Wymagania wstępne
* Serwer VPS (np. OVH) z systemem **Ubuntu 26.04 LTS**.
* Skonfigurowany firewall **UFW**.
* Dostęp przez SSH zabezpieczony parą kluczy kryptograficznych na niestandardowym porcie (np. `2222`).

---

## Krok 1: Włączenie przekazywania pakietów (IP Forwarding)
Aby serwer mógł przekazywać ruch z urządzeń podłączonych do VPN dalej do Internetu, należy włączyć przekazywanie pakietów IPv4 w jądrze systemu.

1. Otwórz plik konfiguracyjny sysctl:
   ```bash
   sudo nano /etc/sysctl.conf
   ```
2. Odkomentuj lub dodaj na końcu pliku poniższą linię:
   ```text
   net.ipv4.ip_forward=1
   ```
3. Zastosuj zmiany natychmiast:
   ```bash
   sudo sysctl -p
   ```

---

## Krok 2: Instalacja środowiska Docker
Na czystym systemie Ubuntu 26.04 najszybszą i zalecaną metodą jest użycie oficjalnego skryptu instalacyjnego od Docker:

```bash
# Pobranie i uruchomienie skryptu instalacyjnego
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Dodanie bieżącego użytkownika do grupy docker (wyeliminowanie konieczności używania sudo)
sudo usermod -aG docker $USER

# Odświeżenie sesji grup w bieżącym terminalu (lub wyloguj się i zaloguj ponownie)
newgrp docker
```

---

## Krok 3: Przygotowanie modułów jądra systemu (NAT)
Nowe wersje Ubuntu wymagają, aby moduły odpowiedzialne za routing i filtrowanie ruchu (NAT) były załadowane bezpośrednio na systemie-hoście przed startem kontenera.

Uruchom na VPS:
```bash
sudo modprobe ip_tables
sudo modprobe iptable_nat
```

---

## Krok 4: Wyłączenie restrykcyjnych profili AppArmor (Specyficzne dla Ubuntu 24.04/26.04)
Ubuntu 26.04 posiada globalne reguły bezpieczeństwa dla poleceń `wg` oraz `wg-quick`, które blokują poprawne uruchomienie WireGuarda wewnątrz kontenera Docker (błąd typu `Permission denied` przy poleceniu `readlink`). Aby je wyłączyć na poziomie systemu hosta, wykonaj poniższe komendy:

```bash
# 1. Utworzenie katalogu wykluczeń i podlinkowanie profili wg i wg-quick
sudo mkdir -p /etc/apparmor.d/disable
sudo ln -sf /etc/apparmor.d/wg /etc/apparmor.d/disable/
sudo ln -sf /etc/apparmor.d/wg-quick /etc/apparmor.d/disable/

# 2. Natychmiastowe rozładowanie tych profili z jądra systemu
sudo apparmor_parser -R /etc/apparmor.d/wg
sudo apparmor_parser -R /etc/apparmor.d/wg-quick
```

---

## Krok 5: Konfiguracja Docker Compose dla WG-Easy v15
1. Utwórz katalog roboczy i przejdź do niego:
   ```bash
   mkdir -p ~/wg-easy && cd ~/wg-easy
   ```
2. Utwórz plik konfiguracyjny `docker-compose.yml`:
   ```bash
   nano docker-compose.yml
   ```
3. Wklej poniższą zawartość:

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    privileged: true
    security_opt:
      - apparmor=unconfined
      - systempaths=unconfined
    environment:
      - LANG=pl
      - PORT=51821
      # Wymagane przy połączeniu nieszyfrowanym (np. przez tunel SSH na localhost):
      - INSECURE=true
    volumes:
      - ./etc_wireguard:/etc/wireguard
      # Udostępnienie kontenerowi modułów jądra hosta do obsługi iptables:
      - /lib/modules:/lib/modules:ro
    ports:
      - "51820:51820/udp"
      # Bezpieczeństwo: Bindujemy panel WWW wyłącznie do localhost (127.0.0.1)
      - "127.0.0.1:51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
```

---

## Krok 6: Konfiguracja UFW (Firewall)
Zezwól na ruch na porcie WireGuarda. Port panelu administracyjnego (`51821`) pozostawiamy zamknięty, ponieważ będziemy łączyć się z nim bezpiecznie przez tunel SSH.

```bash
# Otwarcie portu UDP dla połączeń VPN
sudo ufw allow 51820/udp

# Przeładowanie zapory sieciowej
sudo ufw reload
```

---

## Krok 7: Uruchomienie kontenera
Uruchom aplikację w tle:
```bash
docker compose up -d
```

---

## Krok 8: Bezpieczny dostęp do Panelu Web (Tunelowanie SSH)
Zamiast wystawiać panel administracyjny na świat (co naraża go na ataki), uzyskamy do niego dostęp za pomocą bezpiecznego tunelu SSH.

Uruchom na swoim **lokalnym komputerze** (nie na VPS-ie) poniższe polecenie w terminalu (Linux/macOS) lub PowerShell (Windows):

```bash
ssh -N -L 51821:127.0.0.1:51821 -p <PORT_SSH> -i <SCIEZKA_DO_KLUCZA_PRYWATNEGO> <USER>@<PUBLICZNE_IP_VPS>
```

### Przykład rzeczywistego wywołania:
```bash
ssh -N -L 51821:127.0.0.1:51821 -p 2222 -i /home/user/.ssh/vpsfromlin ubuntu@57.128.248.235
```

**Opis parametrów:**
*   `-N` – informuje SSH, aby nie otwierało interaktywnej powłoki terminala, a jedynie przekierowało porty.
*   `-L 51821:127.0.0.1:51821` – przekierowuje port `51821` z Twojego lokalnego komputera bezpośrednio na port `51821` na serwerze VPS (localhost).
*   `-p 2222` – określa niestandardowy port usługi SSH na serwerze VPS.
*   `-i /sciezka/do/klucza` – wskazuje Twój lokalny klucz prywatny używany do autoryzacji SSH.

---

## Krok 9: Pierwsze uruchomienie i konfiguracja w przeglądarce
1. Po uruchomieniu tunelu SSH, otwórz przeglądarkę na swoim komputerze i przejdź pod adres:
   `http://127.0.0.1:51821`
2. Na ekranie pojawi się **Setup Wizard** (Kreator pierwszej konfiguracji) systemu WG-Easy v15:
   * **Existing configuration?** Wybierz **No (Nie)**.
   * **Admin Credentials:** Podaj swoją nazwę administratora (np. `admin`) oraz ustal bezpieczne hasło.
   * **Host Setup:** Wpisz publiczny adres IP swojego VPS-a oraz domyślny port WireGuarda (`51820`).
3. Po zakończeniu konfiguracji zostaniesz przekierowany do panelu administracyjnego, gdzie możesz wygodnie generować profile użytkowników (pliki `.conf` oraz kody QR dla urządzeń mobilnych).
```

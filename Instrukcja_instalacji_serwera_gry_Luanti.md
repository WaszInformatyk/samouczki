# Ściąga: Instalacja serwera Luanti (Minecraft-like) na Ubuntu 26.04

Niniejszy poradnik pozwala na szybkie uruchomienie serwera gry Luanti z silnikiem **VoxeLibre** (bardzo dokładnym odpowiednikiem Minecrafta) przy użyciu technologii Docker.

## Krok 1: Aktualizacja systemu i instalacja wymaganych narzędzi
Zaloguj się na swój serwer VPS przez SSH i zainstaluj niezbędne pakiety:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose-v2 git unzip
```

## Krok 2: Nadanie uprawnień do obsługi Dockera (bez sudo)
Domyślnie Docker wymaga uprawnień administratora. Dodaj swojego użytkownika (np. `ubuntu`) do grupy `docker`, aby ułatwić sobie pracę:
```bash
sudo usermod -aG docker $USER
```
Aby zastosować zmiany w bieżącej sesji terminala (bez ponownego logowania), wykonaj:
```bash
newgrp docker
```

## Krok 3: Przygotowanie katalogów i pobranie silnika gry (VoxeLibre)
Stwórz strukturę folderów dla serwera, a następnie sklonuj pliki gry VoxeLibre z oficjalnego repozytorium GitHub:
```bash
mkdir -p ~/luanti/data ~/luanti/config ~/luanti/games ~/luanti/mods
cd ~/luanti

git clone https://github.com/VoxeLibre/VoxeLibre.git ~/luanti/games/voxelibre
```

## Krok 4: Konfiguracja serwera (`minetest.conf`)
Utwórz plik konfiguracyjny, w którym zdefiniujesz ustawienia gry (np. tryb kreatywny, wyłączone obrażenia oraz nazwę swojego administratora):
```bash
nano ~/luanti/config/minetest.conf
```
Wklej poniższą konfigurację (zastąp `TwojNickWGrze` swoim rzeczywistym loginem):
```ini
# Podstawowe parametry serwera
server_name = Kreatywny Swiat Waszego Informatyka
server_description = Serwer demonstracyjny Luanti z gry VoxeLibre
max_users = 15

# Ustawienia rozgrywki (Tryb Kreatywny)
creative_mode = true
enable_damage = false

# Nazwa konta z pełnymi uprawnieniami administratora
name = TwojNickWGrze
```
Zapisz plik (`Ctrl+O`, `Enter`) i wyjdź (`Ctrl+X`).

## Krok 5: Konfiguracja kontenera (`compose.yml`)
Utwórz plik konfiguracyjny Docker Compose:
```bash
nano compose.yml
```
Wklej następującą treść:
```yaml
services:
  luanti:
    image: ghcr.io/luanti-org/luanti:latest
    container_name: luanti_server
    restart: unless-stopped
    volumes:
      - ./config:/etc/minetest
      - ./data:/var/lib/minetest
      - ./games:/var/lib/minetest/.minetest/games
      - ./mods:/var/lib/minetest/.minetest/mods
    ports:
      - "30000:30000/udp"
    command: --gameid voxelibre --worldname swiat_kreatywny
```
Zapisz plik i wyjdź.

## Krok 6: Zabezpieczenie uprawnień kontenera (UID 30000)
Kontener Luanti ze względów bezpieczeństwa działa na prawach nieuprzywilejowanego użytkownika z identyfikatorem **UID 30000**. Ponieważ folder `data` został utworzony przez użytkownika `ubuntu`, musimy zmienić jego właściciela na hoście, aby kontener mógł w nim zapisywać logi i pliki świata:
```bash
sudo chown -R 30000:30000 ~/luanti/data
```

## Krok 7: Odblokowanie portu w zaporze sieciowej (Firewall)
Serwer Luanti komunikuje się za pomocą protokołu UDP na porcie 30000. Odblokuj go w systemowym firewallu (UFW):
```bash
sudo ufw allow 30000/udp
```

## Krok 8: Uruchomienie i weryfikacja
Uruchom serwer w tle:
```bash
docker compose up -d
```
Możesz na bieżąco śledzić działanie serwera i generowanie mapy, podglądając logi:
```bash
docker logs -f luanti_server
```
*(Wyjście z podglądu logów: `Ctrl+C`)*

---

## Krok 9: Łączenie się z serwerem (za pomocą IP)
1. Pobierz darmowego klienta gry ze strony [luanti.org](https://www.luanti.org/).
2. Przejdź do zakładki **Dołącz do gry** (Join Game).
3. W polu **Adres** wpisz publiczny adres IP swojego VPS-a.
4. W polu **Port** pozostaw `30000`.
5. Wpisz swoją nazwę użytkownika (zgodną z nazwą z pliku `minetest.conf`).
6. Przy pierwszym logowaniu wpisz dowolne hasło (zostanie ono powiązane z Twoim kontem na serwerze).
7. Po wejściu na serwer wpisz na czacie komendę `/privs`, aby potwierdzić posiadanie uprawnień administratora.

---

## Opcja dodatkowa: Podpięcie własnej domeny (DNS)
Jeżeli nie chcesz, aby gracze musieli wpisywać trudny do zapamiętania adres IP, możesz podpiąć pod serwer swoją własną domenę (np. `waszinformatyk.pl`).

1. Zaloguj się do panelu zarządzania swoją domeną (u rejestratora lub w usłudze Cloudflare).
2. Przejdź do sekcji konfiguracji strefy DNS i dodaj nowy rekord:
   * **Typ:** `A`
   * **Nazwa (Host):** `gra` (lub `@` jeśli chcesz używać głównej domeny)
   * **Wartość (IP):** `ADRES_IP_TWOJEGO_VPS`
   * **TTL:** `Automatyczny` lub `3600` (1 godzina)
3. Od tego momentu gracze zamiast adresu IP (np. `193.123.45.67`) mogą w polu adres wpisać: `gra.waszinformatyk.pl`. 
*(Uwaga: Rozpropagowanie zmian w systemie DNS na całym świecie może potrwać od kilku minut do maksymalnie 24 godzin).*

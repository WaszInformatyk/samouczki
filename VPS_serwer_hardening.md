Wdrażanie podstawowych kroków bezpieczeństwa (tzw. *hardening*) na nowym serwerze VPS to kluczowy etap. Najważniejsza zasada administratora podczas tych prac brzmi: **nigdy nie zamykaj aktywnej, działającej sesji SSH, dopóki nie upewnisz się w nowym oknie, że nowe zabezpieczenia i porty działają prawidłowo.** Zapobiegnie to przypadkowemu odcięciu się od serwera.

Poniżej znajduje się bezpieczna, chronologiczna instrukcja krok po kroku dla systemu **Ubuntu 26.04**.

---

### Krok 1: Aktualizacja systemu
Przed przystąpieniem do konfiguracji upewnij się, że system operacyjny i wszystkie pakiety są w najnowszych wersjach stabilnych:
```bash
sudo apt update && sudo apt upgrade -y
```

---

### Krok 2: Konfiguracja zapory sieciowej (UFW)
Zapora sieciowa (UFW) jest domyślnie zainstalowana w Ubuntu, ale jest nieaktywna. Zanim ją włączysz, **musisz otworzyć port dla nowej usługi SSH**. Załóżmy dla tego przykładu, że Twoim nowym portem SSH będzie port **`2222`** (możesz wybrać dowolny wolny port z zakresu 1024–65535, np. 4829).

1. Ustaw domyślne reguły (blokuj wszystko, co wchodzi; zezwalaj na wszystko, co wychodzi):
   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   ```
2. Zezwól na ruch na nowym porcie SSH:
   ```bash
   sudo ufw allow 2222/tcp
   ```
3. Włącz zaporę sieciową:
   ```bash
   sudo ufw enable
   ```
   *(Na pytanie o potencjalne zerwanie połączenia odpowiedz `y` – otworzyłeś już port 2222, więc jesteś bezpieczny).*
4. Sprawdź status reguł zapory:
   ```bash
   sudo ufw status verbose
   ```

---

### Krok 3: Zmiana portu SSH (Specyfika Ubuntu 24.04 / 26.04)
W nowszych wersjach systemu Ubuntu usługa OpenSSH domyślnie nie działa jako stały proces w tle. Zamiast tego systemd nasłuchuje połączeń na porcie 22 za pomocą tzw. **gniazd sieciowych** (*socket activation*) i uruchamia serwer SSH dopiero w momencie wykrycia próby logowania. 

Dlatego zwykły restart usługi `ssh` po zmianie pliku konfiguracyjnego nie przyniesie efektu. Musimy poinstruować systemd, aby wygenerował nowe gniazdo nasłuchu na bazie konfiguracji.

1. Otwórz plik konfiguracyjny SSH:
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
2. Znajdź linijkę `#Port 22`, odkomentuj ją (usuń znak `#`) i zmień wartość na wybrany port:
   ```text
   Port 2222
   ```
3. Zapisz plik (`Ctrl + O`, `Enter`, `Ctrl + X`).
4. **Kluczowy etap dla Ubuntu 26.04:** Przeładuj konfigurację menedżera systemd (co uruchomi wewnętrzny generator gniazd OpenSSH) i zrestartuj gniazdo SSH:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart ssh.socket
   ```
5. Sprawdź, czy system poprawnie nasłuchuje na nowym porcie:
   ```bash
   sudo ss -tulpn | grep ssh
   ```
   W wyniku powinieneś zobaczyć, że usługa systemd (pid 1) nasłuchuje na porcie `2222`.

---

### Krok 4: Wielki test (Nie zamykaj obecnego okna!)
Otwórz **zupełnie nowy terminal** na swoim komputerze lokalnym (Linux Mint) i spróbuj połączyć się z serwerem przy użyciu nowego portu:
```bash
ssh -p 2222 tester@adres_IP_serwera
```
Jeśli logowanie kluczem przebiegło pomyślnie – konfiguracja SSH i zapory sieciowej działa prawidłowo. Możesz teraz bezpiecznie zamknąć poprzednie, zapasowe okno terminala.

---

### Krok 5: Instalacja i konfiguracja Fail2ban
Fail2ban skanuje logi systemowe w poszukiwaniu podejrzanych aktywności (np. powtarzających się prób logowania ze złym kluczem lub prób skanowania portów) i automatycznie blokuje takie adresy IP w zaporze sieciowej (UFW).

1. Zainstaluj pakiet:
   ```bash
   sudo apt install fail2ban -y
   ```
2. Skopiuj domyślny plik konfiguracyjny do pliku `.local` (aby Twoje zmiany nie zostały nadpisane podczas przyszłych aktualizacji pakietu):
   ```bash
   sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
   ```
3. Otwórz plik konfiguracyjny do edycji:
   ```bash
   sudo nano /etc/fail2ban/jail.local
   ```
4. Znajdź sekcję `[sshd]` (możesz użyć `Ctrl + W` w edytorze nano i wpisać `[sshd]`). Zmień domyślny port na swój niestandardowy (np. `2222`) oraz dostosuj parametry agresywności ochrony według własnego uznania:
   ```ini
   [sshd]
   enabled = true
   port    = 2222
   logpath = %(sshd_log)s
   backend = %(sshd_backend)s
   maxretry = 5
   findtime = 10m
   bantime = 1h
   ```
   *   `maxretry`: Liczba nieudanych prób logowania, po której następuje ban.
   *   `findtime`: Okno czasowe, w którym zliczane są nieudane próby (np. 5 prób w ciągu 10 minut).
   *   `bantime`: Czas, na jaki agresor zostanie zablokowany (np. `1h` – jedna godzina, `1d` – jeden dzień).
5. Zapisz plik i uruchom ponownie usługę Fail2ban, aby wczytać nową konfigurację:
   ```bash
   sudo systemctl enable --now fail2ban
   sudo systemctl restart fail2ban
   ```
6. Stan ochrony przed atakami na SSH możesz w każdej chwili zweryfikować poleceniem:
   ```bash
   sudo fail2ban-client status sshd
   ```
   *(Pokaże ono m.in. liczbę aktualnie zablokowanych adresów IP).*

Po wykonaniu tych kroków Twój serwer VPS jest zabezpieczony przed najpopularniejszymi zautomatyzowanymi atakami i próbami skanowania sieci.

//

Zrobiłeś już bardzo duży krok w stronę profesjonalnego zabezpieczenia swojej maszyny. Konfiguracja logowania kluczem, zmiana portu, uruchomienie UFW oraz wdrożenie Fail2ban tworzą solidny fundament ochronny (tzw. *baseline security*).

Oto kilka dodatkowych sugestii, które warto rozważyć, aby Twój serwer był jeszcze bezpieczniejszy i łatwiejszy w utrzymaniu:

### Dodatkowe sugestie bezpieczeństwa i administracji

#### 1. Automatyczne aktualizacje bezpieczeństwa (`unattended-upgrades`)
Jako administrator nie zawsze będziesz pamiętać o codziennym logowaniu się i wpisywaniu `apt upgrade`. Narzędzie `unattended-upgrades` automatycznie pobiera i instaluje poprawki bezpieczeństwa (i tylko je, bez ryzyka uszkodzenia działających aplikacji).
*   **Instalacja:**
    ```bash
    sudo apt install unattended-upgrades -y
    ```
*   **Włączenie:**
    ```bash
    sudo dpkg-reconfigure -plow unattended-upgrades
    ```
    *(Wybierz opcję „Tak” / „Yes” w oknie, które się pojawi).*

#### 2. Bardzo ważna uwaga: Docker a Zapora UFW (Pułapka)
Jeśli planujesz wdrażać aplikacje za pomocą Dockera (co jest bardzo wygodne), musisz wiedzieć o jednej z największych pułapek w systemie Linux. 
*   **Problem:** Docker manipuluje regułami `iptables` bezpośrednio na poziomie jądra systemu. Oznacza to, że **Docker domyślnie ignoruje reguły UFW** [1]. Jeśli uruchomisz kontener poleceniem `docker run -p 8080:80 ...`, port 8080 będzie dostępny dla całego świata, nawet jeśli w UFW nie dodałeś na niego przyzwolenia.
*   **Rozwiązanie:** Gdy uruchamiasz kontenery, zawsze mapuj porty lokalnie (do pętli zwrotnej) [1], czyli np. `-p 127.0.0.1:8080:80` [1]. Wtedy ruch z zewnątrz będzie zablokowany, a dostęp do kontenera uzyskasz bezpiecznie np. przez VPN lub Reverse Proxy (Nginx Proxy Manager), który jako jedyny będzie wystawiony w UFW na portach 80/443.

#### 3. Monitorowanie prób logowania
Warto od czasu do czasu sprawdzić, czy ktoś mimo wszystko nie próbuje skanować Twojego nowego portu. Możesz to zrobić poleceniem:
```bash
sudo grep "Failed password" /var/log/auth.log
```
Lub sprawdzić aktywność Fail2ban:
```bash
sudo fail2ban-client status sshd
```

---

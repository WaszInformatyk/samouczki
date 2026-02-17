
### 1. Przegląd zawodników (Krótka charakterystyka)

#### **Nextcloud** – "Twój prywatny Google Drive"
*   **Co to jest:** Kompletna platforma chmurowa (Hub).
*   **Jak działa:** Klient-serwer przez HTTP/HTTPS. Posiada aplikacje mobilne i desktopowe do synchronizacji.
*   **Główna cecha:** Ekosystem. To nie tylko pliki, ale też kalendarz, kontakty, edycja dokumentów online (Office), czat.
*   **Zasoby:** Wymaga postawienia bazy danych, serwera WWW (Apache/Nginx) i PHP. Jest "ciężki".

#### **File Browser** – "Lekki menedżer plików w przeglądarce"
*   **Co to jest:** Prosty interfejs webowy do zarządzania plikami na serwerze.
*   **Jak działa:** Jeden plik binarny, działa przez przeglądarkę.
*   **Główna cecha:** Minimalizm i szybkość. Pozwala zarządzać plikami na serwerze bez znajomości terminala, z dowolnego urządzenia z przeglądarką.
*   **Zasoby:** Minimalne.

#### **Rclone** – "Szwajcarski scyzoryk do chmur"
*   **Co to jest:** Narzędzie CLI (wiersza poleceń) do synchronizacji i montowania magazynów danych.
*   **Jak działa:** Łączy Twój lokalny Linux z zewnętrznymi dostawcami (Google Drive, Dropbox, S3, OneDrive).
*   **Główna cecha:** Abstrakcja. Sprawia, że zdalny, komercyjny dysk chmurowy zachowuje się jak folder na Twoim dysku (dzięki `rclone mount`).
*   **Zasoby:** Zależne od operacji, ale generalnie lekkie narzędzie konsolowe.

#### **SMB (Samba)** – "Sieciowy kabel do danych"
*   **Co to jest:** Standardowy protokół udostępniania plików w sieci lokalnej (LAN).
*   **Jak działa:** System operacyjny (Linux/Windows) widzi zdalny zasób tak, jakby był podłączony fizycznie kablem do komputera.
*   **Główna cecha:** Integracja z systemem i wydajność w sieci lokalnej.
*   **Zasoby:** Wbudowane w jądro/system, bardzo wydajne.

---

### 2. Porównanie – Kluczowe różnice

Tabela, którą możesz pokazać na ekranie:

| Cecha | Nextcloud | File Browser | Rclone | SMB (Samba) |
| :--- | :--- | :--- | :--- | :--- |
| **Główne środowisko** | Internet (WAN) | Internet/LAN (Web) | Hybryda (Local <-> Cloud) | Sieć Lokalna (LAN) |
| **Sposób dostępu** | Aplikacja Sync / WWW | Przeglądarka WWW | Terminal / Montowanie folderu | Eksplorator plików (Natywny) |
| **Współpraca (Windows)**| Przez klienta Sync | Przez przeglądarkę | Jako dysk (wymaga konfig.) | **Najlepsza (Natywna)** |
| **Edycja plików** | Pobierz -> Edytuj -> Wyślij | Pobierz -> Edytuj -> Wyślij | Edycja "w locie" (z cache) | **Edycja bezpośrednia** |
| **Zastosowanie** | Praca grupowa, kopia zapasowa zdjęć z telefonu | Zarządzanie serwerem, udostępnianie linków | Backup do chmury, migracja danych | Praca na dużych plikach, media center |

---

### 3. Kiedy wybrać co? (Scenariusze dla widza)

To jest najważniejsza sekcja dla Twoich odbiorców. Odpowiedz na pytanie: *"Mam problem X, co wybieram?"*

#### **Wybierz SMB (Samba), gdy:**
*   **Jesteś w domu/biurze (sieć lokalna):** Masz stacjonarkę z Linuxem i laptopa z Windows/Linux. Chcesz przenosić pliki między nimi z maksymalną prędkością łącza (np. 1Gbps lub 2.5Gbps).
*   **Edytujesz pliki bezpośrednio:** Chcesz otworzyć film w programie do montażu lub zdjęcie w GIMP-ie, które leży na serwerze, bez wcześniejszego kopiowania go na dysk lokalny. SMB radzi sobie z tym najlepiej.
*   **Korzystasz z multimediów:** Masz Kodi, Plex czy telewizor Smart TV – one najlepiej "rozumieją" SMB.

#### **Wybierz Nextcloud, gdy:**
*   **Chcesz uniezależnić się od Google/Apple:** Potrzebujesz synchronizacji kontaktów, kalendarza i zdjęć ze smartfona na własny serwer.
*   **Udostępniasz pliki na zewnątrz:** Chcesz wysłać znajomemu link do folderu ze zdjęciami z wakacji (z hasłem i datą wygaśnięcia).
*   **Pracujesz w zespole:** Kilka osób musi mieć dostęp do tych samych dokumentów z różnych miejsc na świecie.

#### **Wybierz File Browser, gdy:**
*   **Zarządzasz serwerem "na szybko":** Potrzebujesz graficznego interfejsu do plików na swoim VPS lub Raspberry Pi, a nie chcesz konfigurować ciężkiego Nextclouda.
*   **Masz słaby sprzęt:** File Browser zużywa promil zasobów w porównaniu do Nextclouda.
*   **Ad-hoc download/upload:** Chcesz po prostu wrzucić film na serwer lub pobrać go stamtąd, używając dowolnego komputera (np. w kawiarence internetowej), logując się przez WWW.

#### **Wybierz Rclone, gdy:**
*   **Robisz backupy:** Chcesz zaszyfrować swoje dane i wysłać je automatycznie (skryptem w cronie) na Google Drive, OneDrive czy Backblaze.
*   **Brakuje Ci miejsca na dysku:** Chcesz podmontować swoje 2TB na Dysku Google jako lokalny folder w Linuxie (`/mnt/google_drive`) i korzystać z niego jak z dodatkowego dysku USB.
*   **Migrujesz dane:** Przenosisz dane z jednej chmury do drugiej.

### Podsumowanie

> *"Jeśli wyobrażamy sobie nasze dane jako wodę, to:*
> *   ***Nextcloud*** *jest całą instalacją wodociągową w domu – ma krany, prysznice i filtry.*
> *   ***File Browser*** *to wiaderko – proste, bierzesz wodę, wylewasz, działa wszędzie.*
> *   ***Rclone*** *to pompa, która przepycha wodę z twojej studni do miejskich zbiorników (Google/Amazon).*
> *   ***SMB*** *to gruba rura łącząca dwa zbiorniki stojące obok siebie. Jest najszybsza i najmniej przecieka, ale nie sięgnie do sąsiedniego miasta."*

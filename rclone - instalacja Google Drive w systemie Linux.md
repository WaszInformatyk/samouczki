**rclone** to absolutnie najlepsze narzędzie typu open-source do zarządzania chmurami z poziomu terminala (Google Drive, One Drive, DropBox itp.) 

Poniżej znajdziesz kompletną instrukcję – od instalacji po automatyczne montowanie dysku Google jako lokalnego folderu w systemie Na CachyOS (który bazuje na Arch Linux).

---

### Krok 1: Instalacja rclone

CachyOS korzysta z repozytoriów Archa, więc instalacja jest błyskawiczna.

Otwórz terminal i wpisz:

```bash
sudo pacman -S rclone
```

### Krok 2: Konfiguracja połączenia z Google Drive

Teraz musimy "przedstawić" rclone Twojemu kontu Google.

1. Wpisz komendę:
   
   ```bash
   rclone config
   ```
2. Wybierz **`n`** (New remote).
3. Podaj nazwę, np.: **`gdrive`**.
4. Pojawi się długa lista serwisów. Szukaj "Google Drive" (zazwyczaj numer około 18, ale lepiej wpisać po prostu `drive`).
5. **Client ID** oraz **Client Secret**: Pozostaw puste (naciśnij Enter). *Dla zaawansowanych: własne Client ID poprawia stabilność, ale na start standardowe wystarczy.*
6. **Scope**: Wybierz **`1`** (Full access).
7. **Root Folder ID**: Pozostaw puste (Enter).
8. **Service Account File**: Pozostaw puste (Enter).
9. **Edit advanced config?**: Wybierz **`n`** (No).
10. **Use auto config?**: Wybierz **`y`** (Yes).
    * W tym momencie otworzy się Twoja przeglądarka internetowa. Zaloguj się na konto Google i kliknij "Zezwól".
11. **Configure this as a Shared Drive?**: Wybierz **`n`** (No).
12. Na koniec potwierdź wszystko wybierając **`y`** (Yes, this is OK).
13. Wyjdź z konfiguracji wpisując **`q`**.

### Krok 3: Podstawowe komendy (Przesyłanie plików)

Teraz możesz już zarządzać plikami. Pamiętaj o dwukropku po nazwie pilota (`gdrive:`).

* **Listowanie plików w chmurze:**
  
  ```bash
  rclone ls gdrive:
  ```
* **Wysyłanie pliku do chmury:**
  
  ```bash
  rclone copy /ścieżka/do/pliku.txt gdrive:NazwaFolderu
  ```
* **Wysyłanie całego folderu (bardzo szybkie):**
  
  ```bash
  rclone copy ~/Dokumenty gdrive:Backup/Dokumenty --progress
  ```
* **Synchronizacja (UWAGA: Usuwa pliki w chmurze, jeśli nie ma ich lokalnie!):**
  
  ```bash
  rclone sync ~/Obrazy gdrive:Zdjecia --progress
  ```

### Krok 4: Montowanie Google Drive jako dysku (FUSE)

To najwygodniejsza metoda – Google Drive będzie widoczny w Twoim menedżerze plików (np. Dolphin lub Thunar) tak jak pendrive.

1. Stwórz punkt montowania:
   
   ```bash
   mkdir ~/google-drive
   ```

2. Zamontuj dysk:
   
   ```bash
   rclone mount gdrive: ~/google-drive --vfs-cache-mode full &
   ```
   
   *Flaga `--vfs-cache-mode full` jest kluczowa – pozwala na normalne otwieranie plików (np. filmów czy dokumentów) bezpośrednio z chmury.*

3. Aby odmontować:
   
   ```bash
   fusermount -u ~/google-drive
   ```

### Krok 5: Automatyzacja (Opcjonalne)

Jeśli chcesz, aby dysk montował się sam przy starcie systemu CachyOS:

1. Stwórz plik serwisu systemd:
   
   ```bash
   mkdir -p ~/.config/systemd/user/
   nano ~/.config/systemd/user/rclone-gdrive.service
   ```

2. Wklej poniższą treść:
   
   ```ini
   [Unit]
   Description=RClone Mount Google Drive
   After=network-online.target
   
   [Service]
   Type=simple
   ExecStart=/usr/bin/rclone mount gdrive: %h/google-drive \
       --vfs-cache-mode full \
       --vfs-cache-max-size 10G \
       --vfs-cache-max-age 24h
   ExecStop=/usr/bin/fusermount -u %h/google-drive
   Restart=on-failure
   
   [Install]
   WantedBy=default.target
   ```

3. Aktywuj usługę:
   
   ```bash
   systemctl --user daemon-reload
   systemctl --user enable --now rclone-gdrive.service
   ```

### Pro-tip dla użytkownika CachyOS:

CachyOS jest zoptymalizowany pod kątem wydajności (jądro `linux-cachyos`). Rclone świetnie to wykorzystuje przy dużych transferach. Jeśli masz bardzo szybkie łącze, możesz dodać do komend flagę `--transfers 8` (domyślnie są 4), aby przesyłać więcej plików jednocześnie.

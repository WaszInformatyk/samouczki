Kompilacja Luanti ze źródeł z oficjalnego repozytorium na GitHubie to znakomity sposób na uzyskanie absolutnie najnowszej wersji gry (nawet wersji deweloperskich znajdujących się na gałęzi `master`). 

Poniżej znajdziesz kompletne wymagania oraz instrukcję krok po kroku przygotowaną pod systemy bazujące na Debianie i Ubuntu (w tym **Linux Mint 22.3** oraz **Ubuntu 26.04**).

---

### Wymagania systemowe (Zależności do kompilacji)

Aby skompilować kod źródłowy, Twój system musi mieć zainstalowane narzędzia programistyczne oraz biblioteki deweloperskie (oznaczone sufiksem `-dev`). 

Zainstalujesz je jednym poleceniem w terminalu:

```bash
sudo apt update && sudo apt install -y \
  git g++ make cmake libc6-dev libpng-dev libjpeg-dev \
  libgl1-mesa-dev libsqlite3-dev libogg-dev libvorbis-dev \
  libopenal-dev libcurl4-gnutls-dev libfreetype6-dev \
  zlib1g-dev libgmp-dev libjsoncpp-dev libzstd-dev \
  libluajit-5.1-dev gettext libsdl2-dev
```

*Wyjaśnienie najważniejszych z nich:* `cmake` i `make` odpowiadają za proces budowania, `libsdl2-dev` oraz `libgl1-mesa-dev` są wymagane do obsługi grafiki i okna gry (dla klienta), a `libluajit-5.1-dev` zapewnia kompilator JIT dla języka Lua, co drastycznie przyspiesza działanie modów.

---

### Krok 1: Klonowanie kodu z GitHub

Pobierz kod źródłowy Luanti oraz (opcjonalnie) domyślną paczkę gry `minetest_game`.

1. **Sklonuj główne repozytorium Luanti:**
   
   ```bash
   git clone --depth=1 https://github.com/luanti-org/luanti.git
   cd luanti
   ```
   
   *(Parametr `--depth=1` pobiera tylko najnowszy stan plików, oszczędzając czas i transfer na pobieraniu historii zmian)*.

2. **(Opcjonalnie) Pobierz podstawową grę:**
   
   ```bash
   cd games
   git clone --depth=1 https://github.com/luanti-org/minetest_game.git
   cd ..
   ```

---

### Krok 2: Proces kompilacji (Wybór wariantu)

W zależności od tego, czy kompilujesz grę na domowym komputerze (z monitorem), czy na serwerze VPS (bez środowiska graficznego), wybierz odpowiednią opcję konfiguracji CMake:

#### Opcja A: Kompilacja pełna (Klient + Serwer) — idealna na Linux Mint

Ta opcja zbuduje zarówno graficznego klienta gry, jak i serwer. Użyjemy flagi `-DRUN_IN_PLACE=TRUE`, która sprawia, że gra uruchamia się bezpośrednio z folderu, w którym została skompilowana (nie rozrzuca plików po systemie, co ułatwia jej usunięcie).

1. **Generowanie plików budowania:**
   
   ```bash
   cmake . -DRUN_IN_PLACE=TRUE
   ```
2. **Kompilacja:**
   
   ```bash
   make -j$(nproc)
   ```
   
   *(Parametr `-j$(nproc)` nakazuje procesorowi użycie wszystkich dostępnych rdzeni, co znacznie skraca czas kompilacji)*.

---

#### Opcja B: Kompilacja tylko serwera (Headless) — idealna na VPS Ubuntu

Jeśli budujesz grę na serwerze VPS, kompilacja klienta graficznego jest zbędna i mogłaby zgłaszać błędy o brak bibliotek X11/OpenGL. Możemy ją wyłączyć, budując wyłącznie serwer konsolowy:

1. **Generowanie plików budowania tylko dla serwera:**
   
   ```bash
   cmake . -DRUN_IN_PLACE=TRUE -DBUILD_CLIENT=FALSE -DBUILD_SERVER=TRUE
   ```
2. **Kompilacja:**
   
   ```bash
   make -j$(nproc)
   ```

---

### Krok 3: Uruchomienie i weryfikacja

Po pomyślnym zakończeniu procesu kompilacji, gotowe pliki binarne znajdą się w katalogu `bin/`.

* **Uruchomienie klienta graficznego (Opcja A):**
  
  ```bash
  ./bin/luanti
  ```
* **Uruchomienie serwera konsolowego (Opcja B):**
  
  ```bash
  ./bin/luanti --server
  ```

---

### Przydatne wskazówki (Dla administratora)

1. **Jak zaktualizować taką wersję w przyszłości?**
   Nie musisz ponownie pobierać wszystkiego. Wystarczy wejść do katalogu `~/luanti` i wykonać:
   
   ```bash
   git pull
   make -j$(nproc)
   ```
   
   Nowe zmiany zostaną pobrane i przebudowane w kilka chwil.
2. **Lokalizacja plików użytkownika:**
   Dzięki zastosowaniu parametru `RUN_IN_PLACE=TRUE` wszystkie Twoje światy, konfiguracje i mody będą znajdować się bezpośrednio w folderze, w którym skompilowałeś grę (np. odpowiednio w `~/luanti/worlds/`, `~/luanti/mods/`). To bardzo ułatwia robienie kopii zapasowych.

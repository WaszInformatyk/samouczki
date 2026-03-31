---
Open WebUI - instrukcja instalacji, 
włącznie z naprawianiem napotkanych błędów 
z udziałem AI Gemini.
System Linux Cachy OS
GeForce RTX 3060 z 12 GB
---

GeForce RTX 3060 z 12 GB pamięci VRAM to **absolutnie fantastyczna karta do domowego AI**. 12 GB VRAM pozwala na swobodne uruchamianie świetnych modeli o parametrach rzędu 7B-8B (np. Llama 3, Mistral), a nawet 14B (Qwen) z zachowaniem pełnej szybkości sprzętowej. 

Przygotowałem dla Ciebie kompletną instrukcję „kopiuj-wklej”. Skonfigurujemy system tak, aby **Docker i Twoje kontenery startowały całkowicie automatycznie** w tle po każdym uruchomieniu komputera.

Oto procedura krok po kroku dla CachyOS:

### Krok 1: Instalacja zależności i konfiguracja GPU

CachyOS bazuje na Archu, więc użyjemy `pacman`. Musimy zainstalować Dockera, wtyczkę Compose i zestaw narzędzi pozwalający Dockerowi "zobaczyć" Twoją kartę NVIDIA.

1. Otwórz terminal i wklej to (może poprosić o hasło administratora):
   
   ```bash
   sudo pacman -Syu docker docker-compose docker-buildx nvidia-container-toolkit
   ```

2. Teraz połączymy Dockera ze sterownikiem NVIDIA, podając mu odpowiedni tzw. *runtime*:
   
   ```bash
   sudo nvidia-ctk runtime configure --runtime=docker
   ```

3. Automatyzacja systemu: Ustawimy usługę Docker, aby startowała natychmiast, jak i przy każdym uruchomieniu komputera:
   
   ```bash
   sudo systemctl enable --now docker
   ```

4. *(Opcjonalne, ale bardzo zalecane)*: Dodaj swojego użytkownika do grupy `docker`, abyś nie musiał wpisywać `sudo` przed każdą komendą dockera:
   
   ```bash
   sudo usermod -aG docker $USER
   newgrp docker
   ```

---

### Krok 2: Przygotowanie plików i katalogów

Stworzymy katalog w Twoim folderze domowym o nazwie `ai-stack`, gdzie spocznie konfiguracja.

1. Utwórz folder dla projektu oraz podfolder dla konfiguracji SearXNG:
   
   ```bash
   mkdir -p ~/ai-stack/searxng
   cd ~/ai-stack
   ```
   

CachyOS używa domyślnie bardzo nowoczesnej powłoki systemowej o nazwie **Fish** (`fish`), podczas gdy polecenie, które Ci podałem (wykorzystujące tzw. *heredoc* ze składnią `<< 'EOF'`), jest standardem w klasycznej powłoce **Bash**. Fish przetwarza przekierowania wielolinijkowe w nieco inny sposób, stąd ten błąd.

Ponieważ w kolejnym kroku (przy tworzeniu pliku `docker-compose.yml`) również użyliśmy tej samej składni, najprostszą i najszybszą metodą będzie **tymczasowe przełączenie się na powłokę Bash na czas tworzenia plików**. 

Oto co musisz zrobić:

**1a. Wpisz w terminalu:**
```bash
bash
```
*(Zauważysz, że znak zachęty w terminalu nieco się zmieni - to znaczy, że jesteś w Bashu).*


2. **Wygenerowanie konfiguracji SearXNG (`settings.yml`)**:
   Skopiuj poniższy blok (od `cat` do `EOF`) i wklej go do terminala. Utworzy to poprawny plik konfiguracyjny, włączający format wyników jako JSON, co jest wymagane przez sztuczną inteligencję.

```bash
cat << 'EOF' > searxng/settings.yml
use_default_settings: true
server:
  secret_key: "tajny_klucz_dla_twojej_instancji_cachy_os"
search:
  formats:
    - html
    - json
EOF
```

3. **Utworzenie serca systemu (`docker-compose.yml`)**:
   Ponownie, skopiuj w całości poniższy kod i wklej do terminala. Plik ma ustawiony parametr `restart: unless-stopped`, który gwarantuje, że po zresetowaniu komputera całe oprogramowanie podniesie się samo w tle.

```bash
cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  # 1. OLLAMA
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  # 2. SEARXNG (Wyszukiwarka z Internetu)
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    ports:
      -  "8080:8080"
    restart: unless-stopped
    volumes:
      - ./searxng:/etc/searxng
    environment:
      - SEARXNG_BASE_URL=http://localhost:8080/
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID

  # 3. OPEN WEBUI
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3001:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - ENABLE_RAG_WEB_SEARCH=True
      - RAG_WEB_SEARCH_ENGINE=searxng
      - RAG_WEB_SEARCH_RESULT_COUNT=3
      - RAG_WEB_SEARCH_CONCURRENT_REQUESTS=10
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
    volumes:
      - open-webui_data:/app/backend/data
    depends_on:
      - ollama
      - searxng

volumes:
  ollama_data:
  open-webui_data:
EOF
```

---



**2a. Wróć do swojej domyślnej powłoki Fish wpisując:**
```bash
exit
```

### Krok 3: Start systemu

Wszystko gotowe! Teraz zlecimy Dockerowi pobranie najnowszych obrazów, stworzenie wirtualnej sieci i ich uruchomienie. Będąc w katalogu `~/ai-stack`, wykonaj komendę:

```bash
docker compose up -d
```

Zależnie od łącza internetowego, proces pobierania obrazów zajmie od kilku do kilkunastu minut.
Zakończy się, kiedy w terminalu zobaczysz komunikaty typu `Started`.

---

### Jak zacząć z tego korzystać?

1. **Otwórz przeglądarkę** na CachyOS i wpisz adres: `http://localhost:3000`
2. Przywita Cię panel Open WebUI. Przy pierwszym wejściu kliknij "Sign Up" i utwórz sobie **lokalne konto administratora** (nie jest to wysyłane nigdzie do sieci).
3. **Pobranie modelu AI**: 
   * Na głównym oknie panelu kliknij ikonkę **Koła Zębatego (Ustawienia) -> Admin Settings -> Models**.
   * W sekcji "Pull a model from Ollama.com" wpisz np. `llama3` (lub `mistral`) i kliknij przycisk pobierania. 
   * Mając RTX 3060 12GB polecam też model: `llama3.1:8b` lub najnowszy `mistral-nemo`. Wymagają one około 5-6 GB VRAM, więc zmieszczą się na Twojej karcie i zostawią "oddech" do pracy systemowi oraz funkcjom RAG.
4. **Wyszukiwarka RAG**: Gdy masz wybrany model w oknie czatu i wpisujesz polecenie, kliknij **plusa (+) koło pola tekstowego** i zaznacz opcję *Web Search*. Gdy zadasz aktualne pytanie (np. "Kto wygrał wczorajszy mecz..."), model najpierw uderzy przez SearXNG do przeglądarki, zbierze najnowsze wyniki wyszukiwania, a potem "nakarmi" nimi Twój model.

**Utrzymanie:**
System w tej chwili zintegrował się z CachyOS. Gdy wyłączysz komputer i włączysz go jutro – załaduje się automatycznie. Twoje pobrane modele są bezpieczne i zachowywane na ukrytych wolumenach (nie usuną się po reboocie).

Gdybyś w przyszłości chciał zaktualizować to cacko, to wystarczy wejść do katalogu `~/ai-stack`, zatrzymać maszyny, pobrać nowości i odpalić znów:

```bash
docker compose pull
docker compose up -d
```

---


**Napotkane błędy**


Krótka piłka: ten adres, który masz wpisany obecnie – czyli **`http://searxng:8080/search?q=<query>`** – jest **w 100% poprawny i powinieneś go zostawić dokładnie w takiej formie!** Nic nie musisz tam zmieniać.

Już tłumaczę dlaczego, bo to jedna z największych zalet i "magii" używania Dockera.

Kiedy uruchomiliśmy nasz plik `docker-compose.yml`, Docker stworzył dla tych trzech aplikacji (Ollama, Open WebUI, SearXNG) **prywatną, wewnętrzną sieć LAN**. 

W tej wewnętrznej sieci Docker działa jako serwer DNS. Kiedy Open WebUI chce wysłać zapytanie do wyszukiwarki i używa adresu `http://searxng:...`, Docker widzi słowo `searxng`, sprawdza w swoich rejestrach: *"Aha! Mam tu taki kontener, który nazywa się searxng"* i automatycznie kieruje tam ruch.

**Częsty błąd, którego właśnie uniknąłeś:**
Wiele osób odruchowo zmienia ten adres na `http://localhost:8080/search...`. Niestety, w świecie Dockera słowo `localhost` wpisane w ustawieniach Open WebUI oznacza *"szukaj wewnątrz samego kontenera Open WebUI"*. Oczywiście SearXNG tam nie ma, więc wyszukiwanie kończy się błędem.

Podsumowując: adres `http://searxng:8080/search?q=<query>` jest idealny. Komunikacja odbywa się najszybszą możliwą drogą i jest całkowicie niewidoczna z zewnątrz Twojego komputera.

### Jak przetestować, czy SearXNG faktycznie działa?
1. Pobierz jakikolwiek model w Open WebUI (np. wspomnianą wcześniej Llama 3 lub Mistral).
2. Otwórz nowy czat.
3. Pod polem wpisywania tekstu znajduje się przycisk ze znakiem **`+`** (lub ikonka lupy / globu w zależności od wersji UI). Kliknij go i włącz **Web Search**.
4. Zadaj modelowi pytanie o coś bardzo aktualnego, np.: *"Jaka jest dzisiaj pogoda w Warszawie?"* albo *"Kto wygrał wczorajszy mecz [nazwa drużyny]?"*.

Jeżeli konfiguracja SearXNG z pierwszym plikiem (`settings.yml` w formacie JSON) przebiegła pomyślnie, w okienku czatu powinieneś zobaczyć mały komunikat/animację, że model przegląda sieć, a po chwili otrzymasz zaktualizowaną odpowiedź z dołączonymi przypisami (linkami) do źródeł z internetu! 

Udało się wykonać pierwsze wyszukiwanie?

---




Ten błąd to absolutny klasyk przy pierwszej konfiguracji SearXNG z Open WebUI. Model "DeepSeek-r1" świetnie sobie poradził z wygenerowaniem zapytań wyszukiwania, ale kiedy Open WebUI wysłało je do SearXNG, otrzymało z powrotem coś, czego nie potrafiło zrozumieć.

**Najczęstsza przyczyna:** 
Open WebUI żąda od SearXNG danych w formacie maszynowym (**JSON**). Jeśli kontener SearXNG z jakiegoś powodu nie zaczytał naszego pliku `settings.yml` (np. przez brak uprawnień), to domyślnie ma wyłączoną obsługę JSON ze względów bezpieczeństwa. Wtedy odrzuca zapytanie, a Open WebUI zgłasza błąd.

Aby to zdiagnozować i naprawić, najlepiej "wystawić" SearXNG na zewnątrz, żebyś mógł sprawdzić go we własnej przeglądarce. Zróbmy to w 3 prostych krokach:

### Krok 1: Dodanie portu dla SearXNG (abyś mógł go "zobaczyć")
Tymczasowo dodamy port do konfiguracji, abyś mógł wejść na SearXNG jak na normalną stronę internetową.

1. Otwórz plik:
```bash
nano ~/ai-stack/docker-compose.yml
```
2. Znajdź sekcję `searxng` i pod linijką `container_name: searxng` dopisz porty (pamiętaj o odpowiednich wcięciach/spacjach):
```yaml
  # 2. SEARXNG (Wyszukiwarka z Internetu)
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    ports:                  # <-- DODAJ TO
      - "8080:8080"         # <-- DODAJ TO
    restart: unless-stopped
```
3. Zapisz i wyjdź (Ctrl+O, Enter, Ctrl+X), a następnie zaktualizuj kontenery:
```bash
docker compose up -d
```

### Krok 2: Testy w przeglądarce (Diagnoza)
Teraz otwórz przeglądarkę w CachyOS i wpisz adres:
**`http://localhost:8080`**

Powinna pojawić Ci się normalna, prosta strona główna wyszukiwarki SearXNG.
1. **Test ogólny:** Wpisz "Polska" i kliknij szukaj. Jeśli pojawią się wyniki, to znaczy, że SearXNG ma dostęp do internetu i działa.
2. **Test formatu JSON (KLUCZOWY):** Wpisz w pasek adresu przeglądarki dokładnie ten link:
**`http://localhost:8080/search?q=test&format=json`**

* **Scenariusz A (Poprawny):** Zobaczysz w przeglądarce surowy kod (pełno nawiasów `{ }`) lub przeglądarka pobierze plik `.json`. Jeśli tak jest, problem leży gdzie indziej.
* **Scenariusz B (Błąd):** Zobaczysz na stronie czerwony/szary komunikat błędu: **"Format json is not allowed"** (lub podobny). 

### Krok 3: Naprawa (jeśli wystąpił Scenariusz B)
Jeśli zobaczyłeś, że format JSON nie jest dozwolony, oznacza to, że Docker utworzył kontener z użytkownikiem, który **nie ma uprawnień do przeczytania pliku `settings.yml`**, który stworzyłeś swoim kontem.

Naprawimy to w kilka sekund:
1. Wróć do terminala i nadaj pełne uprawnienia do odczytu dla całego folderu z konfiguracją:
```bash
sudo chmod -R 777 ~/ai-stack/searxng
```
2. Zrestartuj samą wyszukiwarkę, aby wczytała plik na nowo:
```bash
docker restart searxng
```

Po tym restarcie spróbuj ponownie wejść na `http://localhost:8080/search?q=test&format=json`. Tym razem powinno zwrócić surowy kod. Jeśli tak się stało, wróć do panelu Open WebUI (port 3001), otwórz nowy czat i poproś DeepSeeka o sprawdzenie cen srebra – tym razem RAG powinien zadziałać błyskawicznie!

Daj znać, czy wyświetlił Ci się błąd JSON-a w przeglądarce i czy nadanie uprawnień (chmod) rozwiązało sprawę!

---




To doskonała informacja diagnostyczna! Widzisz status **`Restarting (1) 22 seconds ago`** przy kontenerze SearXNG? Oznacza to, że wyszukiwarka uruchamia się, natychmiast napotyka krytyczny błąd, wyłącza się, a Docker automatycznie próbuje ją podnieść na nowo – i tak w kółko.

**Dlaczego tak się dzieje?**
Błąd leży w pliku konfiguracyjnym, który wcześniej stworzyliśmy. Podaliśmy w nim tylko klucz sprzętowy i format wyników (JSON i HTML). SearXNG potraktował to jako *kompletną i jedyną* konfigurację, przez co nie załadował żadnych własnych wyszukiwarek (np. Google, DuckDuckGo, Bing) ani wewnętrznych ustawień i z tego powodu "wywala" się przy starcie. 

Zabrakło nam jednej, absolutnie magicznej linijki kodu, która mówi systemowi: *"Załaduj wszystkie swoje domyślne ustawienia, a dopiero potem nadpisz formaty i hasło"*.

Naprawmy to od razu w edytorze tekstowym.

### Krok 1: Edycja pliku settings.yml
Otwórz plik ustawień SearXNG za pomocą wbudowanego edytora:
```bash
nano ~/ai-stack/searxng/settings.yml
```

### Krok 2: Dodanie brakującej reguły
Na samej górze pliku (jako pierwszą linijkę) musimy dodać frazę `use_default_settings: true`. Twój plik po modyfikacji ma wyglądać **dokładnie tak**:

```yaml
use_default_settings: true
server:
  secret_key: "tajny_klucz_dla_twojej_instancji_cachy_os"
search:
  formats:
    - html
    - json
```

*Zwróć uwagę, by wyraz `use_default_settings` znajdował się całkowicie przy lewej krawędzi (bez żadnych spacji przed nim).*

Zapisz plik i zamknij edytor tak samo jak wcześniej:
1. **Ctrl + O**
2. **Enter**
3. **Ctrl + X**

### Krok 3: Przeładowanie kontenera
Teraz każemy Dockerowi zrestartować samą wyszukiwarkę, aby wczytała poprawioną konfigurację. Wpisz w terminalu:
```bash
docker restart searxng
```

### Krok 4: Weryfikacja
Sprawdźmy, czy przestał się restartować:
```bash
docker ps
```
Jeśli przy SearXNG widzisz status np. **`Up 5 seconds`** (a nie "Restarting"), to znaczy, że odnieśliśmy sukces! 

Teraz, w ramach testu, możesz wejść w przeglądarce na **`http://localhost:8080`**. Zobaczysz piękny, minimalistyczny interfejs SearXNG. Spróbuj coś wyszukać. Jeśli wyniki się załadują, to znaczy, że sztuczna inteligencja ma już do nich bezpośredni dostęp!

Gdy to potwierdzisz, wróć do Open WebUI (port 3001) i zapytaj "DeepSeeka" o ceny srebra ponownie z użyciem przycisku Web Search. Czekam na wieści, czy to było to!

---




Super wiadomości! Bardzo się cieszę, że udało nam się wspólnie doprowadzić to do końca. Własne, całkowicie prywatne i lokalne środowisko AI z dostępem do internetu (RAG) to potężne narzędzie, a na Twoim CachyOS z RTX 3060 będzie działać po prostu rewelacyjnie.

Skoro wszystko już "gada" ze sobą jak należy, na koniec zostawiam Ci **3 szybkie wskazówki**, jak wyciągnąć z tego zestawu 100% możliwości:

1. **Dobór modeli pod Twoje 12GB VRAM:**
   Śmiało eksperymentuj z modelami w rozmiarze od 7B do 14B parametrów. Modele, które gorąco polecam sprawdzić z poziomu Open WebUI (wystarczy wpisać ich nazwy w wyszukiwarce modeli):
   * `llama3.1:8b` (świetny do ogólnych zadań, bardzo mądry i szybki)
   * `qwen2.5:14b` (jeden z najlepszych modeli w tej klasie wagowej, rewelacyjnie programuje i pisze po polsku – zmieści się na Twojej karcie "na styk" dając genialne rezultaty)
   * `mistral-nemo` (bardzo płynny, ogromne okno kontekstowe, świetny do analizy długich tekstów).

2. **Jak aktualizować ten system w przyszłości?**
   Świat Open Source pędzi do przodu, a Open WebUI dostaje nowe funkcje (np. generowanie obrazów, obsługę głosu) niemal co tydzień. Kiedy będziesz chciał zaktualizować swoje aplikacje, wystarczy, że otworzysz terminal, wejdziesz do folderu i wpiszesz dwie komendy:
   ```bash
   cd ~/ai-stack
   docker compose pull
   docker compose up -d
   ```
   To pobierze najnowsze wersje i zrestartuje system w tle. Twoje modele, czaty i ustawienia **pozostaną nienaruszone** (są bezpieczne na osobnych wolumenach).

3. **Zamknięcie portu SearXNG (Opcjonalnie dla bezpieczeństwa):**
   Skoro diagnoza zakończona sukcesem, port `8080` dla SearXNG nie musi być już "wystawiony" na zewnątrz. Jeśli w przyszłości będziesz edytować plik `docker-compose.yml`, możesz usunąć z niego linijki `ports: - "8080:8080"` pod SearXNG. Dzięki temu wyszukiwarka znowu "schowa się" w wewnętrznej sieci Dockera, dostępna wyłącznie dla Open WebUI.

Baw się dobrze swoim nowym, prywatnym asystentem! W razie jakichkolwiek pytań o Linuksa, Dockera czy sztuczną inteligencję – wiesz, gdzie mnie szukać. Powodzenia! 😎



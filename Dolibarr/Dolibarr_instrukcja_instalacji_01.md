**Ważna uwaga na wstępie:** Dolibarr to system o korzeniach francuskich, który natywnie obsługuje pełną księgowość (Double Entry). Aby prowadzić w nim **Polski KPiR**, będziemy musieli odpowiednio skonfigurować moduł księgowy oraz dostosować plan kont.

Oto instrukcja krok po kroku:

---

### Krok 1: Aktualizacja systemu i instalacja stosu LAMP

Dolibarr wymaga serwera WWW, bazy danych i interpretera PHP. Otwórz terminal i wykonaj:

```bash
sudo apt update && sudo apt upgrade -y
# Instalacja Apache, MariaDB i PHP 8.3 (standard w Mint 22 / Ubuntu 24.04)
sudo apt install apache2 mariadb-server php php-mysql libapache2-mod-php php-gd php-curl php-zip php-mbstring php-xml php-intl php-soap php-bcmath -y
```

### Krok 2: Konfiguracja bazy danych

Zabezpiecz bazę i stwórz dedykowanego użytkownika dla Dolibarr:

```bash
sudo mysql_secure_installation
# Odpowiedz 'Y' na pytania o usunięcie anonimowych użytkowników i testowej bazy.

sudo mysql -u root -p
```

W konsoli MariaDB wpisz (zamień `TwojeHaslo` na własne):

```sql
CREATE DATABASE dolibarr;
CREATE USER 'dolibarruser'@'localhost' IDENTIFIED BY 'TwojeHaslo';
GRANT ALL PRIVILEGES ON dolibarr.* TO 'dolibarruser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Krok 3: Pobranie i przygotowanie plików Dolibarr

Zalecam pobranie najnowszej stabilnej wersji z oficjalnego repozytorium GitHub lub SourceForge (w momencie pisania jest to wersja 20.x).

```bash
cd /tmp
wget https://sourceforge.net/projects/dolibarr/files/Dolibarr%20ERP-CRM/20.0.0/dolibarr-20.0.0.tgz # Sprawdź najnowszą wersję na stronie
tar -zxvf dolibarr-20.0.0.tgz
sudo cp -r dolibarr-20.0.0/htdocs /var/www/html/dolibarr
```

Ustaw uprawnienia, aby serwer Apache mógł zapisywać pliki:

```bash
sudo chown -R www-data:www-data /var/www/html/dolibarr
sudo chmod -R 755 /var/www/html/dolibarr
# Utworzenie folderu na dokumenty (faktury, załączniki) poza publicznym katalogiem htdocs
sudo mkdir /var/www/html/dolibarr/documents
sudo chown www-data:www-data /var/www/html/dolibarr/documents
```

### Krok 4: Konfiguracja Apache

Stwórz plik konfiguracyjny dla swojej instancji:

```bash
sudo nano /etc/apache2/sites-available/dolibarr.conf
```

Wklej poniższą treść:

```apache
<VirtualHost *:80>
    ServerAdmin admin@twojafirma.local
    DocumentRoot /var/www/html/dolibarr
    ServerName localhost

    <Directory /var/www/html/dolibarr>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/dolibarr_error.log
    CustomLog ${APACHE_LOG_DIR}/dolibarr_access.log combined
</VirtualHost>
```

Aktywuj stronę i zrestartuj serwer:

```bash
sudo a2ensite dolibarr.conf
sudo a2enmod rewrite
sudo systemctl restart apache2
```

### Krok 5: Instalacja przez przeglądarkę

Wejdź na adres: `http://localhost/install/`

1. **Sprawdzenie wymagań:** System powinien świecić się na zielono.
2. **Konfiguracja bazy danych:**
   * Nazwa bazy: `dolibarr`
   * Użytkownik: `dolibarruser`
   * Hasło: `TwojeHaslo`
3. **Katalog dokumentów:** Powinien wskazać `/var/www/html/dolibarr/documents`.
4. **Konto administratora:** Stwórz login i hasło do logowania do systemu.

---

### Krok 6: Konfiguracja pod Polski KPiR (Kluczowe dla przedsiębiorcy)

Po zalogowaniu przejdź do **Ustawienia -> Konfiguracja -> Firma/Organizacja**.

1. **Lokalizacja:** Ustaw Państwo na **Polska**, walutę na **PLN**, język na **Polski**.
2. **Podatki (VAT):**
   * Włącz moduł VAT.
   * Dolibarr domyślnie posiada stawki 23%, 8%, 5%, 0%, zw. – sprawdź je w *Słownikach*.
3. **Księgowość (KPiR):**
   * W Dolibarrze przejdź do **Ustawienia -> Moduły**.
   * Włącz moduł **Księgowość (zaawansowana)**. Mimo nazwy, użyjemy go do KPiR.
   * Przejdź do konfiguracji modułu Księgowość -> **Plany kont**.
   * Wybierz/Zaimportuj polski wzorzec planu kont (dostępny jako `PCG99-BASE` - to baza, którą będziemy upraszczać).
   * **Ważne:** W KPiR nie księgujesz na kontach zespołu "4" czy "5" w pełnym zakresie, ale Dolibarr wymaga powiązania produktów/usług z kontami, aby generować raporty. Dla KPiR zdefiniuj konta odpowiadające kolumnom w książce:
     * Konto 701 (Sprzedaż towarów/usług) -> Kolumna 7 KPiR.
     * Konto 401 (Zakup towarów/materiałów) -> Kolumna 10 KPiR.
     * Konto 402 (Koszty uboczne zakupu) -> Kolumna 11 KPiR.
     * Konto 403 (Wynagrodzenia) -> Kolumna 12 KPiR.
     * Konto 409 (Pozostałe wydatki) -> Kolumna 13 KPiR.

### Krok 7: JPK i polskie wymagania prawne

Dolibarr "z pudełka" nie generuje plików JPK_V7M/K. Aby system był w pełni zgodny z polskimi wymogami skarbowymi, masz dwie opcje:

1. **Eksport danych do Excela:** Dolibarr ma świetny moduł eksportu. Możesz eksportować rejestry VAT i zakupy, a następnie importować je do darmowych narzędzi ministerialnych (np. Klient JPK_WEB).
2. **Dedykowane moduły (Polecane):** Odwiedź **Dolistore.com** i poszukaj modułów "Poland" lub "JPK". Istnieją polskie firmy wdrożeniowe, które oferują gotowe moduły do generowania JPK oraz integracji z KSeF (Krajowy System e-Faktur), co od 2026 roku będzie obowiązkowe.

### Bezpieczeństwo (Lokalne)

Ponieważ używasz systemu lokalnie (na Mint), po zakończeniu instalacji usuń plik blokujący ponowną instalację:

```bash
sudo touch /var/www/html/dolibarr/documents/install.lock
sudo chmod 444 /var/www/html/dolibarr/documents/install.lock
```

**Rada eksperta:** Regularnie rób kopię zapasową bazy danych i folderu `documents`. W Linux Mint możesz do tego użyć wbudowanego narzędzia **Timeshift** (dla systemu) oraz prostego skryptu `mysqldump` dla samej bazy Dolibarr.

Czy potrzebujesz pomocy przy specyficznej konfiguracji konkretnej kolumny KPiR lub integracji z drukarką fiskalną?

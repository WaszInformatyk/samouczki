
# Kompleksowy Poradnik: Instalacja ComfyUI na Linux Mint 22.2

Poniższa instrukcja przeprowadzi Cię przez proces instalacji ComfyUI w systemie Linux Mint 22.2 (Kernel 6.17+) przy użyciu stabilnego środowiska Python 3.12 oraz oficjalnego wsparcia dla kart NVIDIA (CUDA 13.2 / Sterowniki 595+).

## Krok 1: Aktualizacja systemu i instalacja zależności

Zanim zaczniesz, upewnij się, że Twój system jest w pełni zaktualizowany, a także zainstaluj niezbędne pakiety systemowe do obsługi Pythona i Git.

Otwórz standardowy terminal (Bash) i wykonaj poniższe komendy:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git python3-pip python3-venv python3-dev -y

```

## Krok 2: Klonowanie repozytorium ComfyUI

Utwórz katalog na swoje projekty AI (np. `AI`) i sklonuj do niego oficjalne repozytorium ComfyUI:

```bash
mkdir -p ~/AI
cd ~/AI
git clone https://github.com/Comfy-Org/ComfyUI.git
cd ComfyUI

```

## Krok 3: Tworzenie i aktywacja środowiska wirtualnego (VENV)

Ze względu na zabezpieczenia systemu Linux Mint (zgodność z PEP 668), instalacja bibliotek AI musi odbywać się w odizolowanym środowisku. Tworzymy środowisko oparte na stabilnym Pythonie 3.12:

```bash
python3 -m venv venv
source venv/bin/activate

```

> **Wskazówka:** Po wpisaniu `source`, na początku linii Twojego terminala powinno pojawić się oznaczenie `(venv)`. Oznacza to, że środowisko jest aktywne.

## Krok 4: Instalacja PyTorch ze wsparciem CUDA 12.6

Twoje sterowniki graficzne (NVIDIA 595+) i CUDA 13.2 są w pełni kompatybilne wstecznie. Zainstalujemy najnowszą stabilną wersję PyTorcha z oficjalnym wsparciem produkcyjnym dla platformy CUDA 12.6:

```bash
pip install --upgrade pip
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu126

```

## Krok 5: Instalacja pozostałych pakietów i wymagań

Zainstaluj wszystkie dodatkowe biblioteki Pythona wymagane przez rdzeń ComfyUI:

```bash
pip install -r requirements.txt

```

## Krok 6: Instalacja ComfyUI-Manager (Zarządzanie dodatkami)

Automatyczny menedżer rozszerzeń to podstawa pracy z ComfyUI. Zainstalujemy go bezpośrednio w folderze `custom_nodes`:

```bash
cd custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
cd ..

```

*(Komenda `cd ..` cofa nas z powrotem do głównego katalogu `~/AI/ComfyUI`).*

---

## Krok 7: Pierwsze uruchomienie i pobieranie modeli

Uruchom ComfyUI po raz pierwszy przy użyciu polecenia:

```bash
python main.py --highvram

```

1. Po uruchomieniu terminal wyświetli adres lokalny. Skopiuj go i wklej do przeglądarki internetowej: **`http://127.0.0.1:8188`**
2. Zobaczysz czysty interfejs ComfyUI. Kliknij widoczny na środku przycisk **"Click here to download a model"** – ComfyUI automatycznie pobierze stabilny i lekki model startowy bezpośrednio do odpowiedniego katalogu.
3. Jeśli w przyszłości zechcesz pobrać inne modele (Checkpoints), LoRA lub ControlNet ręcznie ze stron takich jak *Civitai* lub *HuggingFace*, pamiętaj, że pliki (np. z rozszerzeniem `.safetensors`) należy umieszczać w katalogu:
`~/AI/ComfyUI/models/checkpoints/`

---

## Jak uruchomić ComfyUI w przyszłości? (Szybki start)

Gdy zamkniesz terminal lub zrestartujesz komputer, ponowne uruchomienie aplikacji wymaga jedynie wpisania trzech prostych komend w terminalu Bash:

```bash
cd ~/AI/ComfyUI
source venv/bin/activate
python main.py --highvram

```

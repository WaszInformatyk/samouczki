Oto kompletna instrukcja wdrożeniowa od zera (tzw. *Zero-to-Hero Guide*). 

Została przygotowana tak, aby nowa osoba na świeżym systemie (Linux Mint 22 / Ubuntu 24.04 LTS) mogła po prostu **skopiować i wkleić poszczególne bloki komend do terminala**.

---

### Krok 1: Instalacja pakietów systemowych (APT)
Wymagane narzędzia do obsługi plików PDF, konwersji do DOCX oraz środowiska Python:

```bash
sudo apt update && sudo apt install -y curl poppler-utils pandoc python3-venv python3-pip
```

---

### Krok 2: Instalacja Ollama i pobranie modelu GLM-OCR
Instalacja oficjalnej usługi Ollama i ściągnięcie modelu `glm-ocr` (ok. 2.2 GB):

```bash
# 1. Instalacja oficjalnym skryptem (automatycznie konfiguruje usługę systemd i CUDA)
curl -fsSL https://ollama.com/install.sh | sh

# 2. Pobranie dedykowanego modelu OCR
ollama pull glm-ocr
```

---

### Krok 3: Utworzenie katalogu projektu i środowiska wirtualnego (venv)
Zgodnie z wymogami PEP 668 na nowoczesnym Linuksie, tworzymy izolowane środowisko `venv`:

```bash
mkdir -p ~/glm-ocr-app && cd ~/glm-ocr-app
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install "gradio>=4.20.0" pymupdf pillow ollama
```

---

### Krok 4: Utworzenie 3 plików skryptów

Wykonaj poniższe polecenia – używają konstrukcji `cat`, więc utworzą pliki automatycznie bez konieczności otwierania edytora tekstu.

#### Plik 1: `glm-ocr.py` (Główna aplikacja OCR + Web UI)
```bash
cat << 'EOF' > ~/glm-ocr-app/glm-ocr.py
#!/usr/bin/env python3
"""
GLM-OCR Studio - Lokalny OCR dokumentów urzędowych i skanów (Linux / CUDA).
"""

import io
import os
import re
import shutil
import subprocess
import tempfile
from pathlib import Path
from typing import List, Tuple

import gradio as gr
import ollama
from PIL import Image
import pymupdf

OLLAMA_HOST = "http://localhost:11434"
MODEL_NAME = "glm-ocr"
PDF_RENDER_DPI = 150
MAX_IMAGE_EDGE = 1536

MODE_MAP = {
    "Pełny tekst i układ (Text Recognition)": "Text Recognition:",
    "Struktura tabel (Table Recognition)": "Table Recognition:",
}

OLLAMA_OPTIONS = {
    "num_ctx": 8192,
    "num_predict": 4096,
    "temperature": 0.0,
    "keep_alive": "2h",
    "stop": [
        "<|user|>",
        "<|endoftext|>",
        "<|observation|>",
        "Text Recognition:",
        "Table Recognition:",
    ],
}


def check_ollama_status() -> Tuple[bool, str]:
    try:
        client = ollama.Client(host=OLLAMA_HOST)
        models = client.list()
        installed = []
        if isinstance(models, dict) and "models" in models:
            installed = [m.get("model", "") or m.get("name", "") for m in models["models"]]
        elif hasattr(models, "models"):
            installed = [getattr(m, "model", "") or getattr(m, "name", "") for m in models.models]

        if not any(MODEL_NAME in name for name in installed):
            return False, f"Brak modelu '{MODEL_NAME}'. Wykonaj: 'ollama pull {MODEL_NAME}'."

        return True, "Ollama działa poprawnie."
    except Exception as exc:
        return False, f"Brak łączności z Ollama ({OLLAMA_HOST}): {exc}"


def sanitize_ocr_text(text: str) -> str:
    if not text:
        return ""

    text = text.replace("\x0c", "\n").replace("\f", "\n")
    text = re.sub(r"&nbsp;", " ", text)
    text = re.sub(r"(<br\s*/?>\s*)+", "\n", text, flags=re.IGNORECASE)
    text = re.sub(r"(\s*[-_.]{4,}\s*){2,}", "\n", text)
    text = re.sub(r"(\s*[-*_]{3,}\s*){2,}", "\n\n---\n\n", text)

    lines = [line.rstrip() for line in text.splitlines()]
    text = "\n".join(lines)
    text = re.sub(r"\n{3,}", "\n\n", text)

    text = re.sub(r"^(\s*[-*_]{3,}\s*)+", "", text)
    text = re.sub(r"(\s*[-*_]{3,}\s*)+$", "", text)

    return text.strip()


def deduplicate_text_loops(text: str) -> str:
    text = text.strip()
    if not text:
        return text

    paragraphs = text.split("\n\n")
    n = len(paragraphs)
    if n >= 2:
        for step in range(1, (n // 2) + 1):
            pattern = paragraphs[:step]
            is_loop = True
            for i in range(step, n, step):
                chunk = paragraphs[i : i + step]
                if chunk != pattern[: len(chunk)]:
                    is_loop = False
                    break
            if is_loop and n >= step * 2:
                return "\n\n".join(pattern).strip()

    return text


def optimize_and_encode_image(pil_img: Image.Image) -> bytes:
    img_rgb = pil_img.convert("RGB")
    w, h = img_rgb.size
    
    if max(w, h) > MAX_IMAGE_EDGE:
        scale = MAX_IMAGE_EDGE / max(w, h)
        new_w = int(w * scale)
        new_h = int(h * scale)
        img_rgb = img_rgb.resize((new_w, new_h), Image.Resampling.LANCZOS)
    
    buf = io.BytesIO()
    img_rgb.save(buf, format="PNG", optimize=True)
    return buf.getvalue()


def render_pdf_page_to_png_bytes(doc: pymupdf.Document, page_num: int) -> bytes:
    page = doc.load_page(page_num)
    pix = page.get_pixmap(dpi=PDF_RENDER_DPI)
    img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)
    del pix
    del page
    return optimize_and_encode_image(img)


def image_file_to_png_bytes(image_path: str) -> bytes:
    with Image.open(image_path) as img:
        return optimize_and_encode_image(img)


def execute_ocr_streaming(client: ollama.Client, prompt: str, img_bytes: bytes) -> str:
    chunks: List[str] = []
    try:
        stream = client.generate(
            model=MODEL_NAME,
            prompt=prompt,
            images=[img_bytes],
            stream=True,
            options=OLLAMA_OPTIONS,
        )
        for chunk in stream:
            token = chunk.get("response", "")
            chunks.append(token)
    except Exception as exc:
        err_msg = str(exc).lower()
        if "token repeat limit reached" in err_msg and len(chunks) > 0:
            pass
        else:
            raise exc

    raw_result = "".join(chunks)
    deduped = deduplicate_text_loops(raw_result)
    sanitized = sanitize_ocr_text(deduped)
    return sanitized


def convert_markdown_to_docx(md_path: str, docx_path: str) -> bool:
    if not shutil.which("pandoc"):
        return False
    cmd = ["pandoc", md_path, "-f", "gfm", "-o", docx_path]
    try:
        subprocess.run(cmd, check=True, capture_output=True, text=True)
        return True
    except subprocess.SubprocessError:
        return False


def run_ocr(
    file_obj,
    ocr_mode: str,
    progress=gr.Progress(track_tqdm=True),
) -> Tuple[str, str, gr.DownloadButton, gr.DownloadButton, gr.DownloadButton]:
    if file_obj is None:
        raise gr.Error("Wybierz dokument do przetworzenia.")

    ollama_ok, ollama_msg = check_ollama_status()
    if not ollama_ok:
        raise gr.Error(ollama_msg)

    prompt_prefix = MODE_MAP.get(ocr_mode, "Text Recognition:")
    client = ollama.Client(host=OLLAMA_HOST)
    
    file_path = file_obj.name
    path_obj = Path(file_path)
    file_ext = path_obj.suffix.lower()
    base_name = path_obj.stem

    progress(0, desc="Inicjalizacja dokumentu...")
    page_results: List[str] = []

    if file_ext == ".pdf":
        try:
            doc = pymupdf.open(file_path)
            total_pages = len(doc)
            if total_pages == 0:
                raise gr.Error("Plik PDF jest pusty.")

            for idx in range(total_pages):
                progress(
                    (idx) / total_pages,
                    desc=f"OCR: strona {idx + 1} z {total_pages}...",
                )
                img_bytes = render_pdf_page_to_png_bytes(doc, idx)

                try:
                    content = execute_ocr_streaming(client, prompt_prefix, img_bytes)
                except Exception as exc:
                    content = f"*[Błąd na stronie {idx + 1}: {exc}]*"

                if not content:
                    content = "*[Brak wykrytego tekstu na tej stronie]*"

                if total_pages > 1:
                    page_results.append(f"## Strona {idx + 1}\n\n{content}")
                else:
                    page_results.append(content)

            doc.close()
        except pymupdf.FileDataError as exc:
            raise gr.Error(f"Niepoprawny plik PDF: {exc}")
    else:
        progress(0.4, desc="OCR pojedynczego obrazu...")
        try:
            img_bytes = image_file_to_png_bytes(file_path)
            content = execute_ocr_streaming(client, prompt_prefix, img_bytes)
            page_results.append(content)
        except Exception as exc:
            raise gr.Error(f"Błąd przetwarzania: {exc}")

    valid_pages = [p.strip() for p in page_results if p.strip()]
    full_markdown = "\n\n---\n\n".join(valid_pages).strip()

    if not full_markdown:
        full_markdown = "*Model nie rozpoznał tekstu.*"

    progress(0.95, desc="Generowanie plików eksportu...")
    export_dir = tempfile.mkdtemp(prefix="glm_ocr_")
    
    md_path = os.path.join(export_dir, f"{base_name}_ocr.md")
    txt_path = os.path.join(export_dir, f"{base_name}_ocr.txt")
    docx_path = os.path.join(export_dir, f"{base_name}_ocr.docx")

    with open(md_path, "w", encoding="utf-8") as f:
        f.write(full_markdown)

    with open(txt_path, "w", encoding="utf-8") as f:
        f.write(full_markdown)

    docx_ok = convert_markdown_to_docx(md_path, docx_path)

    progress(1.0, desc="Zakończono pomyślnie!")

    return (
        full_markdown,
        full_markdown,
        gr.DownloadButton(value=md_path, visible=True),
        gr.DownloadButton(value=txt_path, visible=True),
        gr.DownloadButton(value=docx_path if docx_ok else None, visible=docx_ok),
    )


with gr.Blocks(title="GLM-OCR Studio") as demo:
    gr.Markdown(
        """
        # 📑 GLM-OCR Studio (RTX 3060 / Linux Mint)
        Backend: **Ollama (`glm-ocr`)** | Filtracja artefaktów plam: **Aktywna** | Eksport: **Pandoc**
        """
    )

    with gr.Row():
        with gr.Column(scale=1):
            file_input = gr.File(
                label="Wgraj dokument (PDF lub skan)",
                file_types=[".pdf", ".png", ".jpg", ".jpeg", ".webp"],
                type="filepath",
            )
            
            ocr_mode = gr.Radio(
                choices=list(MODE_MAP.keys()),
                value="Pełny tekst i układ (Text Recognition)",
                label="Tryb analizy",
            )

            submit_btn = gr.Button("🚀 Rozpocznij OCR", variant="primary", size="lg")

            gr.Markdown("### 📥 Eksport dokumentu")
            btn_download_md = gr.DownloadButton("⬇️ Pobierz Markdown (.md)", visible=False)
            btn_download_docx = gr.DownloadButton("⬇️ Pobierz Word (.docx)", visible=False)
            btn_download_txt = gr.DownloadButton("⬇️ Pobierz Czysty Tekst (.txt)", visible=False)

        with gr.Column(scale=2):
            with gr.Tabs():
                with gr.TabItem("👁️ Podgląd sformatowany (Markdown)"):
                    md_preview = gr.Markdown(value="*Tutaj pojawi się rozpoznany tekst...*")
                with gr.TabItem("📝 Kod źródłowy"):
                    raw_preview = gr.Code(label="Surowy tekst", language="markdown", lines=25)

    submit_btn.click(
        fn=run_ocr,
        inputs=[file_input, ocr_mode],
        outputs=[
            md_preview,
            raw_preview,
            btn_download_md,
            btn_download_txt,
            btn_download_docx,
        ],
    )

if __name__ == "__main__":
    demo.launch(
        server_name="127.0.0.1",
        server_port=7860,
        show_error=True,
        inbrowser=True,
        theme=gr.themes.Soft(primary_hue="blue")
    )
EOF
```

---

#### Plik 2: `run_app.sh` (Skrypt startowy uruchamiający aplikację)
```bash
cat << 'EOF' > ~/glm-ocr-app/run_app.sh
#!/bin/bash

APP_DIR="$HOME/glm-ocr-app"
URL="http://127.0.0.1:7860"

# 1. Czyszczenie starych plików tymczasowych (starszych niż 24h)
find /tmp -maxdepth 1 -type d -name "glm_ocr_*" -mtime +1 -exec rm -rf {} + 2>/dev/null

# 2. Sprawdzenie czy aplikacja już działa
if curl -s --connect-timeout 1 "$URL" > /dev/null; then
    notify-send "GLM-OCR Studio" "Aplikacja jest już uruchomiona. Otwieram przeglądarkę..." -i document-scan 2>/dev/null
    xdg-open "$URL"
    exit 0
fi

# 3. Weryfikacja czy usługa Ollama działa
if ! curl -s --connect-timeout 2 http://localhost:11434/api/tags > /dev/null; then
    echo "Uruchamianie usługi Ollama..."
    sudo systemctl start ollama 2>/dev/null || ollama serve &
    sleep 3
fi

# 4. Przejście do katalogu i aktywacja środowiska venv
cd "$APP_DIR" || exit 1
source "$APP_DIR/venv/bin/activate"

clear
echo "========================================================="
echo "        📑 GLM-OCR Studio (NVIDIA RTX 3060)             "
echo "========================================================="
echo " Aplikacja działa pod adresem: $URL"
echo ""
echo " UWAGA: Nie zamykaj tego okna terminala podczas pracy!"
echo " Aby bezpiecznie wyłączyć serwer, naciśnij: Ctrl + C"
echo "========================================================="
echo ""

# Uruchomienie aplikacji
python3 glm-ocr.py
EOF
```

---

#### Plik 3: `make_desktop_icon.sh` (Generator skrótu na Pulpicie)
```bash
cat << 'EOF' > ~/glm-ocr-app/make_desktop_icon.sh
#!/bin/bash

DESKTOP_DIR=$(xdg-user-dir DESKTOP 2>/dev/null || echo "$HOME/Pulpit")
mkdir -p "$DESKTOP_DIR"

LAUNCHER_PATH="$DESKTOP_DIR/glm-ocr.desktop"

cat << INNER_EOF > "$LAUNCHER_PATH"
[Desktop Entry]
Version=1.0
Type=Application
Name=Skaner GLM-OCR
Comment=Lokalny OCR dokumentów urzędowych i skanów (RTX 3060)
Exec=$HOME/glm-ocr-app/run_app.sh
Icon=scanner
Terminal=true
Categories=Office;Graphics;
StartupNotify=true
INNER_EOF

chmod +x "$LAUNCHER_PATH"
gio set "$LAUNCHER_PATH" metadata::trusted true 2>/dev/null || true

echo "Pomyślnie utworzono skrót: $LAUNCHER_PATH"
EOF
```

---

### Krok 5: Nadanie uprawnień i utworzenie ikony

Nadajemy prawa wykonywalności obu skryptom powłoki i odpalamy generator ikony:

```bash
cd ~/glm-ocr-app
chmod +x run_app.sh make_desktop_icon.sh
./make_desktop_icon.sh
```

---

### Krok 6: Gotowe! Jak korzystać?

Od tego momentu na Pulpicie znajduje się gotowa ikona **Skaner GLM-OCR**. 

1. **Uruchomienie:** Dwuklik na ikonę na pulpicie (lub komenda `~/glm-ocr-app/run_app.sh` w terminalu).
2. Otworzy się okno terminala ze statusem, a po sekundzie **automatycznie otworzy się przeglądarka** z interfejsem pod adresem `http://127.0.0.1:7860`.
3. **Zakończenie pracy:** Zamknięcie okna terminala krzyżykiem lub naciśnięcie skrótu `Ctrl + C`.

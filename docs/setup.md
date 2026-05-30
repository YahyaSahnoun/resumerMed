# Setup — Phase 1

Follow these steps on **your own machine**. Estimated time: 30–60 minutes,
mostly waiting for downloads.

## 1. Prerequisites

You need:
- Python 3.10 or 3.11 (avoid 3.12, some ML libs lag behind)
- Git
- ~15 GB free disk space (LLM models are heavy)
- At least 8 GB of RAM. 16 GB strongly recommended. A GPU is a nice-to-have,
  not a requirement — we'll use small quantized models that run on CPU.

Check what you have:
```bash
python --version
git --version
```

## 2. Clone / create the project

If you push to GitHub now (recommended — do it early so you have version
control from day one):
```bash
git init medical-summarizer
cd medical-summarizer
```

Otherwise just create the folder and copy the structure I gave you.

## 3. Python virtual environment

**Always** work in a virtual environment. Never install ML libraries globally.

```bash
python -m venv .venv

# Linux / macOS
source .venv/bin/activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1
```

You should see `(.venv)` in your prompt. From now on, every `pip install`
goes into this venv only.

## 4. Install Python dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

This will take 5–10 minutes. If something fails on a specific package,
note it down — we'll fix it together. Don't blindly retry.

Then download the basic NLTK data we'll need for tokenization:
```bash
python -c "import nltk; nltk.download('punkt'); nltk.download('punkt_tab')"
```

## 5. Install Ollama (this is the important part)

Ollama is the runtime that will serve our local LLM. It's a single binary,
no Docker needed.

### Linux / macOS
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### Windows
Download the installer from https://ollama.com/download and run it.

After installation, verify:
```bash
ollama --version
```

Then pull a first model. We'll start small to make sure everything works,
then upgrade once the pipeline is ready. **Mistral 7B quantized** is a good
starting point:
```bash
ollama pull mistral
```

This downloads ~4 GB. Be patient.

## 6. Sanity check — talk to the LLM

In a terminal, run:
```bash
ollama run mistral "In one sentence, what is acute myocardial infarction?"
```

You should get an English medical-sounding answer in a few seconds (or
30–60s on CPU). If yes — Phase 1 is done.

## 7. Sanity check from Python

Create a file `scripts/test_ollama.py` with this content:

```python
import ollama

response = ollama.chat(
    model="mistral",
    messages=[
        {"role": "system", "content": "You are a medical assistant. Be concise and factual."},
        {"role": "user", "content": "List 3 common symptoms of pneumonia."},
    ],
)
print(response["message"]["content"])
```

Run it:
```bash
python scripts/test_ollama.py
```

If you get a sensible answer printed in your terminal — **the environment is
ready**. Move on to Phase 2.

## Troubleshooting

**`ollama: command not found`** — the binary isn't on your PATH. On Linux,
restart your terminal. On Windows, log out and back in.

**`Error: connection refused`** when calling from Python — the Ollama daemon
isn't running. On Linux: `ollama serve` in a separate terminal. On
macOS/Windows: the app should auto-start; otherwise open it manually.

**Out of memory** when pulling/running Mistral — your machine has < 8 GB
free RAM. Try `ollama pull mistral:7b-instruct-q4_0` (more aggressive
quantization), or fall back to a smaller model like `phi3:mini`. Tell me
your specs and I'll pick one for you.

**Slow generation on CPU** — normal. Expect 10–60s per summary. We'll
optimize once the pipeline works end to end.

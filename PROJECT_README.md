# Sunbird AI Pipeline — Project Documentation

## Project description

This is a Gradio web application that lets a user provide either typed text or an uploaded audio file and runs it through a fully automated AI pipeline powered by [Sunbird AI](https://sunbird.ai/). The pipeline transcribes audio to text (when applicable), summarises the text using the Sunflower LLM, translates the summary into a chosen Ugandan local language (Luganda, Runyankole, Ateso, Lugbara, or Acholi), and finally synthesises the translated summary as a playable audio clip using Sunbird's text-to-speech service. All intermediate results — the transcript, the English summary, the translated summary, and the generated audio player — are displayed in the UI.

---

## Architecture overview

```
User input
  │
  ├─ Text (typed/pasted)
  │      │
  │      └──────────────────────────┐
  │                                 │
  └─ Audio file (.mp3/.wav/…)       │
         │                          │
         ▼                          │
  ┌─────────────────┐               │
  │ Sunbird STT     │               │
  │ POST /tasks/stt │               │
  └────────┬────────┘               │
           │ transcript             │
           └──────────────────────► ▼
                             ┌─────────────────────────┐
                             │ Sunflower LLM (summarise)│
                             │ POST /tasks/sunflower_   │
                             │       simple             │
                             └────────────┬────────────┘
                                          │ summary
                                          ▼
                             ┌─────────────────────────┐
                             │ Sunflower LLM (translate)│
                             │ POST /tasks/sunflower_   │
                             │       simple             │
                             └────────────┬────────────┘
                                          │ translated summary
                                          ▼
                             ┌─────────────────────────┐
                             │ Sunbird TTS              │
                             │ POST /tasks/modal/tts    │
                             └────────────┬────────────┘
                                          │ audio URL → downloaded
                                          ▼
                                     Audio player
```

| Step | Sunbird endpoint | Notes |
|---|---|---|
| Speech-to-Text | `POST /tasks/stt` | multipart form, `audio` field |
| Summarise | `POST /tasks/sunflower_simple` | form-encoded, `instruction` field |
| Translate | `POST /tasks/sunflower_simple` | same endpoint, different instruction |
| Text-to-Speech | `POST /tasks/modal/tts` | JSON, `text` + `speaker_id` |

---

## Local setup

```bash
# 1. Clone the repo
git clone https://github.com/GeorgeLukaanya/internship-assessment.git
cd internship-assessment

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate        # Linux / macOS
# venv\Scripts\activate.bat     # Windows

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment variables
cp .env.example .env
# Open .env and replace the placeholder with your real Sunbird API token

# 5. Run the app
python app.py
# The Gradio UI opens at http://127.0.0.1:7860
```

---

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `SUNBIRD_API_TOKEN` | Yes | Bearer token for the Sunbird AI API. Obtain one at [api.sunbird.ai](https://api.sunbird.ai/). |

Copy `.env.example` to `.env` and fill in the value. Never commit the real `.env` file (it is already listed in `.gitignore`).

---

## Usage

### Text input

1. Open the app at `http://127.0.0.1:7860` (or the deployed URL below).
2. Select **Text input** tab.
3. Choose a target language from the **Target language** dropdown (e.g. *Luganda*).
4. Paste or type your text in the text box.
5. Click **Summarise & Translate**.
6. Wait 3–6 minutes (the Sunbird LLM is called twice; the button spins while processing).
7. Results appear below: **Summary**, **Translated summary**, and an inline **audio player**.

### Audio upload

1. Select **Audio upload** tab.
2. Choose a target language.
3. Upload an audio file (MP3, WAV, OGG, M4A, or AAC — maximum 5 minutes).
4. Click **Transcribe, Summarise & Translate**.
5. Results appear below: **Transcript**, **Summary**, **Translated summary**, and **audio player**.

> Files longer than 5 minutes are rejected immediately with a clear error message before any API call is made.

---

## Deployed link

**[https://huggingface.co/spaces/ltgwgeorge/sunbird-ai-pipeline](https://huggingface.co/spaces/ltgwgeorge/sunbird-ai-pipeline)**

---

## Known limitations

- **Slow inference** — Each pipeline run calls the Sunflower LLM twice (summarise + translate) and the TTS service once. End-to-end latency is typically **3–6 minutes**. This is a characteristic of the Sunbird AI API, not the application.
- **5-minute audio cap** — Audio files longer than 300 seconds are rejected before being sent to the API.
- **Supported TTS languages** — Luganda, Runyankole, Ateso, Lugbara, Acholi. Swahili is supported by the Sunbird TTS speaker list but is not a Ugandan local language, so it is excluded from the language picker.
- **English-only source text recommended** — The Sunflower LLM summarises best from English. Summarisation quality for other languages may vary.
- **Free-tier account** — The API token is issued to a free Sunbird AI account; rate limits or quotas may apply.
- **Audio format detection** — Duration checking uses `mutagen`; files in unusual or DRM-protected formats may skip the duration check (the API enforces its own 100 MB / 10-minute limits as a backstop).

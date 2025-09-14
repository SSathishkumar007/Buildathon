# Arogya Sahayak — Hackathon Demo (Streamlit)

## Overview
This repository contains a **minimal demo prototype** for the Arogya Sahayak hackathon project (last-mile rural healthcare assistant). The demo implements a small but powerful slice:
- Audio upload → Whisper transcription (OpenAI)
- Symptom + optional image → GPT triage (GPT-4o or similar)
- Pictogram generation for patient education (DALL·E)
- Optional: Text-to-Speech (gTTS) for audio playback to patients

> The goal is to **prove the core concept** during a ~3-minute live demo at a hackathon. This is *not* a production-ready app.

## Files
- `app.py` — Streamlit demo app.
- `requirements.txt` — Python dependencies.
- `.env.example` — Example of environment variables to set (e.g. `OPENAI_API_KEY`).

## Quick Start (from scratch)
1. **Install Python 3.10+** (recommended).

2. **Clone or download** this project to your machine (or unzip the downloaded package).

3. **(Optional) Create a virtual environment**:
   ```bash
   python -m venv venv
   source venv/bin/activate    # macOS / Linux
   venv\\Scripts\\activate     # Windows Powershell
   ```

4. **Install dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

5. **Set your OpenAI API key** (recommended for full demo):
   - Unix/macOS:
     ```bash
     export OPENAI_API_KEY="sk-..."
     ```
   - Windows (PowerShell):
     ```powershell
     setx OPENAI_API_KEY "sk-..."
     ```
   Alternatively, paste your API key in the app sidebar when you run the Streamlit app.

6. **Run the app**:
   ```bash
   streamlit run app.py
   ```
   The demo UI will open in your browser (usually `http://localhost:8501`).

## How to record audio on phone and upload
- Android: Use 'Voice Recorder' -> Save as WAV/MP3 -> Transfer to computer or upload via phone browser.
- iPhone: Voice Memos -> Share -> Save to Files -> Upload.
- For a live demo, you can run the Streamlit app on a laptop and have a smartphone record and upload quickly via Wi-Fi or cable.

## Demo script (3 minutes)
1. **Intro (20s)**: "Our app empowers ASHA workers to collect voice symptoms, get instant triage, and show pictograms the patient can follow."
2. **Show audio upload (30s)**: Upload a short audio clip (patient in Tamil/Hindi). Click 'Transcribe'. Show the transcription result.
3. **Analyze case (45s)**: Click 'Analyze Case'. Show the AI triage output (Green/Yellow/Red) and the small action list for the worker.
4. **Generate pictogram (45s)**: Click 'Generate Pictogram' to create an instant visual aid and display it.
5. **Wrap up (20s)**: Emphasize offline-first vision, human-in-the-loop safety, and integration potential with ABDM/e-Sanjeevani.

## Notes and Troubleshooting
- The exact OpenAI Python client interfaces may vary by version; if any OpenAI calls fail, refer to OpenAI's official docs and adapt the function implementations in `app.py`.
- If you do not have an OpenAI key, the UI will still run but API-backed functionality will show placeholders/errors.
- For TTS audio playback, install `gtts` (already in requirements); quality varies by locale.

## Security & Ethics (important for later)
- For the hackathon demo, use synthetic or consented data. Never upload real patient-identifiable information without proper consent and safeguards.
- Production needs: end-to-end encryption, DPDPA compliance, medical validation.

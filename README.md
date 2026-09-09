# 🎬 AI Video Assistant

A Retrieval-Augmented Generation (RAG) pipeline that turns any meeting video/audio — a YouTube link or a local upload — into a transcript, a summary, extracted action items/decisions/questions, and an interactive chat you can ask follow-up questions to.

Available both as a **Streamlit web app** and as a **CLI tool**.

---

## ✨ Features

- **Flexible input** — paste a YouTube URL or upload a local video/audio file (`mp4`, `mov`, `mkv`, `webm`, `mp3`, `wav`, `m4a`)
- **Multilingual transcription** — `english` audio is transcribed locally with Whisper; `hinglish` audio is sent to Sarvam AI's STT-translate API, which transcribes and translates to English in one step
- **Automatic title generation** from the transcript
- **Summarization** of the full conversation
- **Structured extraction** (via three separate Mistral-backed LCEL chains) of:
  - ✅ Action items — task description, owner, and deadline (if mentioned)
  - 🔑 Key decisions
  - ❓ Open questions / unresolved topics needing follow-up
- **RAG-powered chat** — ask natural-language questions about the meeting and get answers grounded in the transcript
- **Streamlit UI** with live pipeline status (audio → transcript → title → summary → extraction → RAG indexing) and a dark, custom-styled interface

---

## 🏗️ How it works

The pipeline runs in the following stages (see `main.py` / `app.py`):

```
Input (YouTube URL / uploaded file)
        │
        ▼
utils/audio_processor.py   → process_input()        # yt-dlp download / local file → mono 16kHz WAV → 10-min chunks
        │
        ▼
core/transcriber.py        → transcribe_all()        # routes to Whisper (english) or Sarvam AI (hinglish)
        │
        ▼
core/summarize.py          → generate_title(), summarize()
        │
        ▼
core/extractor.py          → extract_action_items(), extract_key_decisions(), extract_questions()
        │
        ▼
core/vector_store.py       → build_vector_store(), load_vector_store(), get_retriever()
        │
        ▼
core/rag_engine.py         → build_rag_chain() / load_rag_chain(), ask_question()
```

The LLM orchestration is built with **LangChain (LCEL)** using **Mistral's `mistral-small-latest`** model (via `ChatMistralAI`), with **ChromaDB** as the local vector store (top-`k=4` retrieval) and **HuggingFace sentence-transformers** for embeddings. The RAG chain grounds every answer strictly in the retrieved transcript context — if the answer isn't in the transcript, it responds with *"I could not find this information in the meeting transcript."* instead of guessing.

`build_rag_chain()` builds a fresh vector store from a transcript for the current session; `load_rag_chain()` re-loads a previously persisted vector store, so a meeting's index can be queried again without re-processing.

### Audio ingestion

`utils/audio_processor.py` (`process_input()`) handles both input types:

- **YouTube URL** → downloaded with `yt-dlp` (best available audio format) into a local `downloades/` directory
- **Local file** → used as-is

Either way, the audio is then converted to mono, 16kHz WAV (`pydub`) and split into **10-minute chunks** (configurable via `chunk_minutes`) before being handed off to the transcriber.

### Transcription engine routing

Transcription is **not** Whisper-only — `core/transcriber.py` routes each audio chunk to a different engine depending on the selected language:

- **`english`** → local **OpenAI Whisper** model (default `small`, configurable via `WHISPER_MODEL`)
- **`hinglish`** → **Sarvam AI's speech-to-text-translate API** (`saaras:v2.5` by default), which transcribes and translates to English in one step. Since Sarvam's sync API rejects audio over 30s, each chunk is further sliced into 25s pieces before being sent.

---

## 📁 Project structure

```
video_agent/
├── app.py                 # Streamlit web app (entry point for the UI)
├── main.py                # CLI entry point
├── core/
│   ├── transcriber.py      # Speech-to-text (Whisper / Sarvam)
│   ├── summarize.py        # Title generation + summarization
│   ├── extractor.py        # Action items / decisions / questions (3 LCEL chains, Mistral)
│   ├── vector_store.py     # Chroma vector store: build / load / retriever
│   └── rag_engine.py       # LCEL RAG chain construction + Q&A
├── utils/
│   └── audio_processor.py  # Input handling: YouTube download / file chunking
├── requirements.txt
└── test.py
```

---

## ⚙️ Setup

### Prerequisites

- Python ≥ 3.10
- [FFmpeg](https://ffmpeg.org/) installed and available on your `PATH` (required by `ffmpeg-python` / `pydub` / Whisper)
- A [Mistral AI](https://mistral.ai/) API key

### Installation

```bash
git clone https://github.com/mama21000/video_agent.git
cd video_agent

python -m venv venv
source venv/bin/activate      # on Windows: venv\Scripts\activate

pip install -r requirements.txt
```

### Environment variables

Create a `.env` file in the project root with your API key(s), e.g.:

```env
MISTRAL_API_KEY=your_mistral_api_key_here
SARVAM_API_KEY=your_sarvam_api_key_here     # required only for hinglish transcription
WHISPER_MODEL=small                         # optional — Whisper model size (default: small)
SARVAM_STT_MODEL=saaras:v2.5                # optional — Sarvam STT-translate model
```

`MISTRAL_API_KEY` is read via `os.getenv("MISTRAL_API_KEY")` in `core/rag_engine.py` to authenticate `ChatMistralAI`. `SARVAM_API_KEY` is required only if you transcribe in `hinglish` mode — the app will raise an error if it's missing and Hinglish is selected.

---

## 🚀 Usage

### Streamlit web app

```bash
streamlit run app.py
```

Then, in the sidebar:
1. Choose **YouTube URL** or **Upload File** as the input type
2. Select the language (**english** / **hinglish**)
3. Click **⚡ Analyse**
4. Watch the pipeline status update live, then explore the summary, action items, key decisions, and open questions
5. Use the **💬 Chat with your Meeting** panel to ask follow-up questions

### CLI

```bash
python main.py
```

You'll be prompted for:
- A YouTube URL or local file path
- A language (`english` / `hinglish`, defaults to `english`)

The script prints the title, summary, action items, key decisions, and open questions, then drops you into an interactive chat loop (type `exit` to quit).

---

## 🧰 Tech stack

| Layer            | Tools |
|-------------------|-------|
| UI                | Streamlit, streamlit-extras |
| Audio ingestion   | yt-dlp, pydub, ffmpeg-python |
| Transcription     | OpenAI Whisper (local, `english`), Sarvam AI STT-translate API (`hinglish`), PyTorch/torchaudio |
| Translation       | deep-translator |
| LLM orchestration | LangChain (LCEL), Mistral AI |
| Vector store       | ChromaDB |
| Embeddings        | HuggingFace sentence-transformers |
| Export            | reportlab, fpdf2 |

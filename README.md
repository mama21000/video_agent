# 🎬 AI Video Assistant

A Retrieval-Augmented Generation (RAG) pipeline that turns any meeting video/audio — a YouTube link or a local upload — into a transcript, a summary, extracted action items/decisions/questions, and an interactive chat you can ask follow-up questions to.

Available both as a **Streamlit web app** and as a **CLI tool**.

---

## ✨ Features

- **Flexible input** — paste a YouTube URL or upload a local video/audio file (`mp4`, `mov`, `mkv`, `webm`, `mp3`, `wav`, `m4a`)
- **Multilingual transcription** — supports `english` and `hinglish` audio
- **Automatic title generation** from the transcript
- **Summarization** of the full conversation
- **Structured extraction** of:
  - ✅ Action items
  - 🔑 Key decisions
  - ❓ Open questions
- **RAG-powered chat** — ask natural-language questions about the meeting and get answers grounded in the transcript
- **Streamlit UI** with live pipeline status (audio → transcript → title → summary → extraction → RAG indexing) and a dark, custom-styled interface

---

## 🏗️ How it works

The pipeline runs in the following stages (see `main.py` / `app.py`):

```
Input (YouTube URL / uploaded file)
        │
        ▼
utils/audio_processor.py   → process_input()        # downloads/loads & chunks audio
        │
        ▼
core/transcriber.py        → transcribe_all()        # speech-to-text (Whisper, local)
        │
        ▼
core/summarize.py          → generate_title(), summarize()
        │
        ▼
core/extractor.py          → extract_action_items(), extract_key_decisions(), extract_questions()
        │
        ▼
core/rag_engine.py         → build_rag_chain(), ask_question()   # Chroma vector store + LangChain + Mistral
```

The LLM orchestration is built with **LangChain (LCEL)** using the **Mistral API**, with **ChromaDB** as the local vector store and **HuggingFace sentence-transformers** for embeddings.

---

## 📁 Project structure

```
video_agent/
├── app.py                 # Streamlit web app (entry point for the UI)
├── main.py                # CLI entry point
├── core/
│   ├── transcriber.py      # Speech-to-text (Whisper)
│   ├── summarize.py        # Title generation + summarization
│   ├── extractor.py        # Action items / decisions / questions extraction
│   └── rag_engine.py       # RAG chain construction + Q&A
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
```

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
| Transcription     | OpenAI Whisper (local), PyTorch/torchaudio |
| Translation       | deep-translator |
| LLM orchestration | LangChain (LCEL), Mistral AI |
| Vector store       | ChromaDB |
| Embeddings        | HuggingFace sentence-transformers |
| Export            | reportlab, fpdf2 |

---

## 📝 Notes

- Whisper runs **locally**, so transcription speed depends on your machine's CPU/GPU.
- Uploaded files are saved to a local `downloades/` directory before processing.
- This project is a work in progress — contributions and issues are welcome.

## 📄 License

_Add your license here (e.g. MIT)._

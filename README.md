# local-audio-and-youtube-transcription

**Vibe-coded using agentic AI.** This project provides an automated system for transcribing audio files and generating AI summaries and tags for YouTube videos, storing them as Obsidian-compatible Markdown files. It integrates local Whisper models, Ollama for LLM processing, and systemd for robust background operation.

## Table of Contents
1.  [Project Overview](#project-overview)
2.  [Installation Steps](#installation-steps)
3.  [Workflow Summary](#workflow-summary)
    *   [1. File Monitoring with `whisper-watcher.service`](#1-file-monitoring-with-whisper-watcherservice-running-vault-watchsh)
    *   [2. Core Processing with `whisper-server.service`](#2-core-processing-with-whisper-serverservice-running-apppy)
    *   [3. Large Language Model with `ollama.service`](#3-large-language-model-with-ollamaservice)
    *   [4. Diarization Service with `diarize-pro.service`](#4-diarization-service-with-diarize-proservice)
4.  [Audio Transcription Workflows](#audio-transcription-workflows)
    *   [1. Local Transcription Workflow (Implemented in `app.py` using `faster_whisper`)](#1-local-transcription-workflow-implemented-in-apppy-using-faster_whisper)
    *   [2. External API Transcription Workflows (from provided code snippets)](#2-external-api-transcription-workflows-from-provided-code-snippets)

## Project Overview

This system automates the process of generating AI summaries, titles, and tags for YouTube videos, storing them in Obsidian-compatible Markdown files. It leverages local `faster-whisper` for transcription, `yt-dlp` for subtitle retrieval, `Ollama` for LLM processing, and `systemd` services for robust background operation. It can also transcribe local audio files and apply AI summarization.

## Installation Steps

This section outlines the steps to set up and run the `local-audio-and-youtube-transcription` system.

### 1. Prerequisites

*   **Python 3.x:** Ensure Python is installed.
*   **Node.js:** Required for `yt-dlp`'s JavaScript runtime (`/home/sebastian/.nvm/versions/node/v25.6.0/bin/node` is specified in `app.py`).
*   **Ollama:** Install Ollama and ensure the `llama3` model is downloaded and running (`ollama run llama3`).
*   **ffmpeg:** Highly recommended for `yt-dlp` functionality. Install via your system's package manager (e.g., `sudo apt install ffmpeg`).
*   **inotify-tools:** Required for `vault-watch.sh` to monitor file changes (e.g., `sudo apt install inotify-tools`).
*   **Python Virtual Environment:** Highly recommended for managing Python dependencies.

### 2. Clone the Repository (or create symlinks manually)

If you're setting this up on a new machine, you would clone this repository. On the original machine, we're using symlinks to keep files in their operational locations.

```bash
git clone <your-repo-url> ~/local-audio-and-youtube-transcription
cd ~/local-audio-and-youtube-transcription
```

**Note for original machine:** The setup assumes the original files (`app.py`, `vault-watch.sh`, systemd service files) are in specific locations. This repository uses symbolic links to those original files.

### 3. Python Dependencies

Create and activate a Python virtual environment, then install the required Python packages:

```bash
python3 -m venv whisper-env
source whisper-env/bin/activate
pip install -r requirements.txt
```
Ensure that the `yt-dlp` executable in your virtual environment is correctly referenced in `app.py` if moved or renamed.

### 4. Place Systemd Service Files

The service files (`whisper-server.service`, `whisper-watcher.service`, `diarize-pro.service`) need to be placed in the `/etc/systemd/system/` directory to be managed by `systemd`.

```bash
# Copy the service files (assuming you're in the cloned repo directory)
sudo cp systemd/whisper-server.service /etc/systemd/system/
sudo cp systemd/whisper-watcher.service /etc/systemd/system/
sudo cp systemd/diarize-pro.service /etc/systemd/system/
```

**Important:** If `app.py` or `vault-watch.sh` are not in `/home/sebastian/` or the virtual environment is not `~/whisper-env/`, you will need to edit the `ExecStart` paths within the service files (`/etc/systemd/system/whisper-server.service` and `/etc/systemd/system/whisper-watcher.service`) to reflect their actual locations.

### 5. Configure and Enable Services

Reload the `systemd` daemon, then enable and start the services:

```bash
sudo systemctl daemon-reload
sudo systemctl enable whisper-server.service
sudo systemctl enable whisper-watcher.service
sudo systemctl enable diarize-pro.service # If you want this service to run

sudo systemctl start whisper-server.service
sudo systemctl start whisper-watcher.service
sudo systemctl start diarize-pro.service # If you want this service to run
```

You can check the status of each service with `journalctl -u <service-name> -f` (e.g., `journalctl -u whisper-server.service -f`).

### 6. Configure Vault Paths

Ensure the `VAULT_PATH` and `TRANSCRIPT_DIR` variables in `app.py` and `vault-watch.sh` match your Obsidian vault structure.

## Workflow Summary

This system orchestrates the detection, processing, and enrichment of media-related files in your Obsidian vault.

**1. File Monitoring with `whisper-watcher.service` (running `vault-watch.sh`)**
*   **Role:** Continuously monitors your configured Obsidian vault directory (`/home/sebastian/Documents/main`) for new or modified files.
*   **Mechanism:** Uses `inotifywait` to detect `close_write` events (when a file is saved/created).
*   **Triggers:**
    *   New/modified audio files (e.g., `.mp3`, `.wav`) in the `WATCH_DIR`.
    *   New/modified markdown files (`.md`) specifically within the `YouTube` subdirectory of the `WATCH_DIR` (e.g., `/home/sebastian/Documents/main/YouTube/*.md`).
*   **Action:** When a relevant file is detected, `vault-watch.sh` sends a `curl` POST request to the `whisper-server.service`'s appropriate endpoint, providing the full path to the changed file.

**2. Core Processing with `whisper-server.service` (running `app.py`)**
*   **Role:** The central application for handling media processing, transcription, and AI summarization.
*   **Endpoints:**
    *   `/process` (for audio files): Transcribes audio using `faster_whisper` and sends the transcript to Ollama for summarization.
    *   `/process_youtube_markdown` (for YouTube markdown files):
        *   **YAML Front Matter Parsing:** Extracts metadata (including the YouTube `url`) from the markdown file's YAML header.
        *   **Subtitle Retrieval:** Uses `yt-dlp` (configured for Node.js runtime) to download subtitles.
        *   **Subtitle Cleaning:** Aggressively cleans raw subtitle text by removing timestamps, HTML tags, special characters, and duplicate lines to optimize input for the LLM.
        *   **AI Summarization (`ollama.service`):**
            *   Sends the cleaned subtitle text to Ollama (`llama3`) for summary and tag generation.
            *   **Multi-stage Summarization:** For very long videos, text exceeding ~24,000 characters is chunked. Each chunk is summarized by Ollama, and these "mini-summaries" are recursively consolidated into a final summary.
            *   **Output Parsing:** Parses Ollama's response to extract the summary, a sanitized title slug, and relevant hashtags. Includes robust logic to identify tags from explicit "TAGS:" sections or from "**KEY POINTS:**" / "**KEY TAKEAWAYS:**" sections within Ollama's response.
            *   **Fallback:** If parsing of Ollama's response fails, the entire raw Ollama output is saved to the markdown for inspection.
        *   **Markdown Update:** The original YouTube markdown file is updated with the generated summary, tags, cleaned full subtitles, and processing metadata.

**3. Large Language Model with `ollama.service`**
*   **Role:** Hosts the `llama3` large language model used for all AI-powered summarization and metadata generation.
*   **Mechanism:** Provides an API endpoint (typically `http://localhost:11434`) that `app.py` queries with prompts containing cleaned text.

**4. Diarization Service with `diarize-pro.service`**
*   **Role:** (Specific integration details with `app.py` are outside the scope of this project's core functionality as observed in `app.py` code, but this service likely provides advanced speaker diarization capabilities for audio processing.) It is assumed to work independently or integrate with other audio processing components.

## Audio Transcription Workflows

The system incorporates flexible audio transcription capabilities.

**1. Local Transcription Workflow (Implemented in `app.py` using `faster_whisper`)**
*   **Engine:** Leverages `faster_whisper`, a highly optimized version of OpenAI's Whisper model, running locally on your CPU (e.g., `large-v3-turbo` model).
*   **Process:** When an audio file is submitted to the `/process` endpoint in `app.py`:
    *   The audio is loaded and processed by the `WhisperModel.transcribe()` method.
    *   It performs language detection, segmented decoding, and uses Voice Activity Detection (VAD) for accuracy.
    *   The output is a collection of timestamped text segments, which are then assembled into a formatted full transcript.
    *   This transcript is then sent to Ollama for summarization.

**2. External API Transcription Workflows (from provided code snippets, for broader context)**
*   The codebase can also integrate with external Speech-to-Text APIs, offering alternative transcription backends. This includes:
    *   An `async` function using `_client.audio.transcriptions.create` (suggesting OpenAI's Whisper API or a similar cloud service).
    *   A `transcribe_file` method using Google Cloud Speech-to-Text API (`self.client.recognize`) for cloud-based transcription, capable of handling bundled audio segments (e.g., grouped by speaker).
*   These external API interactions are typically wrapped with **tracing and observability** (`transcription_span`) to monitor request details, performance, and potential errors.

---

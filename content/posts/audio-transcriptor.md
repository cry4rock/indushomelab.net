---
title: "Audio Transcriptor — Convert Audio to Text with Python"
date: 2025-12-30T12:00:00+00:00
draft: false
tags: ["python", "speech-recognition", "audio", "tutorial"]
categories: ["Tutorials", "Python"]
summary: "A walkthrough of a lightweight Python script that converts audio files into text using SpeechRecognition and pydub."
---

# Audio Transcriptor — Convert Audio to Text with Python

This post explains a compact Python script in this repository that converts audio files (MP3, WAV, M4A, etc.) into text using the `SpeechRecognition` library and `pydub` for audio preprocessing.

## Why this is useful

Transcripts make audio searchable, accessible and easier to edit. This script is a practical starting point for note-taking, accessibility, and small transcription tasks.

## What the script does (high level)

- Accepts an audio file via CLI.
- Converts non-WAV audio to mono 16 kHz WAV for better recognition.
- Optionally splits audio into chunks by silence to improve recognition accuracy on long files.
- Transcribes chunks (or the whole file) using a speech recognition engine.
- Saves a human-readable `.txt` transcript and a `.json` metadata file.

The main implementation is in the `AudioTranscriptor` class inside the code.

## Installation

Create a virtual environment and install dependencies:

```bash
python -m venv venv
# Windows
venv\Scripts\activate
# macOS/Linux
# source venv/bin/activate

pip install SpeechRecognition pydub
# Optional, for PocketSphinx (local recognition):
pip install pocketsphinx
```

Install `ffmpeg` (required by `pydub`):
- Windows: download from https://ffmpeg.org and add to PATH
- macOS: `brew install ffmpeg`
- Linux: `sudo apt install ffmpeg` (Debian/Ubuntu)

## Usage

Basic command-line usage:

```bash
python audio_transcriptor.py path/to/audiofile.mp3
```

Options:
- `--engine ENGINE` — Recognition engine (`google` default, `sphinx`, or `wit`)
- `--output PATH` — Output file path; default is audio filename with `.txt`
- `--no-chunk` — Don't split audio on silence (transcribe entire file at once)
- `--silence-len MS` — Minimum silence (ms) for splitting; default `1000`

Example:

```bash
python audio_transcriptor.py recording.mp3 --engine google --output my_transcript.txt
```

If using Wit.ai, set the `WIT_AI_KEY` environment variable before running the script. On Windows PowerShell:

```powershell
$env:WIT_AI_KEY = "your_wit_ai_key_here"
python audio_transcriptor.py recording.mp3 --engine wit
```

## Key functions and code highlights

- `convert_to_wav(audio_path, output_path=None)`
  - Uses `pydub.AudioSegment.from_file` and exports a mono 16 kHz WAV. Ensures format compatibility for recognizers.

- `split_audio_on_silence(audio_path, min_silence_len=1000)`
  - Uses `pydub.silence.split_on_silence` with a default threshold of `audio.dBFS - 14` and keeps some silence around chunks.

- `transcribe_chunk(audio_chunk, engine='google')`
  - Exports a chunk to a temporary WAV file (uses a secure tempfile), loads it with `speech_recognition.AudioFile`, and runs recognition. It handles `UnknownValueError` (inaudible) and `RequestError` with friendly return strings. Wit.ai key is read from the `WIT_AI_KEY` environment variable.

- `transcribe_file(...)`
  - Orchestrates conversion, optional chunking, per-chunk transcription, and output saving. Outputs a dict with metadata: `transcript`, `audio_file`, `engine`, `chunks_processed`, `processing_time_seconds`, `timestamp`.

- `save_transcript(results, output_path)`
  - Writes a readable transcript file and a JSON metadata companion.

### Deep dive: main functions

Below are more detailed explanations of the main functions and how you can customize them for your needs.

- `convert_to_wav(audio_path, output_path=None) -> str`
  - Purpose: Convert arbitrary audio formats to a WAV file tuned for speech recognition.
  - Key steps: validate path and extension, load with `AudioSegment.from_file`, set channels to mono and frame rate to 16000 Hz, export to WAV.
  - Notes: You can change the output sample rate or channels depending on your recognition model. For example, some engines prefer 8kHz or 44100Hz.

- `split_audio_on_silence(audio_path, min_silence_len=1000) -> list`
  - Purpose: Break long recordings into smaller chunks around silences to improve accuracy and avoid long API timeouts.
  - Key parameters:
    - `min_silence_len` (ms): how long a silence must be to create a split.
    - `silence_thresh`: computed as `audio.dBFS - 14` by default; you can adjust this to be more or less sensitive.
    - `keep_silence`: how much silence to retain at chunk edges (helpful for context).

- `transcribe_chunk(audio_chunk, engine='google') -> str`
  - Purpose: Recognize speech in a single `AudioSegment` chunk.
  - Process:
    1. Export chunk to a secure temporary WAV using `tempfile.NamedTemporaryFile(delete=False, suffix='.wav')` and close the handle (Windows-safe).
    2. Load with `sr.AudioFile` and call the appropriate recognizer.
    3. For `wit` engine, the script reads `WIT_AI_KEY` from environment variables; if missing it returns a helpful message.
    4. Clean up the temporary file in a `finally` block.
  - Return values: recognized text, or `"[Inaudible]"` / `"[Error: ...]"` placeholders.

- `transcribe_file(...) -> Dict[str, Any]`
  - Purpose: High-level orchestration that converts files, optionally splits them, transcribes chunks, aggregates results, and saves outputs.
  - Important behavior:
    - Uses `convert_to_wav` when input isn't WAV.
    - When `chunk_audio=True`, iterates chunks and calls `transcribe_chunk` for each.
    - Aggregates non-empty, non-inaudible chunk texts and joins them with spaces to produce the `transcript`.
    - Produces a `results` dict that includes performance metrics and timestamps.

## Full source: `audio_transcriptor.py`

The full script is included below so you can copy it directly into your project or reference it from the blog post. Save the following as `audio_transcriptor.py` at your repo root.

```python
#!/usr/bin/env python3
"""
Audio Transcription Program
Converts audio files to text transcripts using multiple speech recognition engines.
"""

import os
import sys
import wave
import json
import tempfile
from pathlib import Path
from typing import Optional, Dict, Any
from datetime import datetime

try:
    import speech_recognition as sr
    from pydub import AudioSegment
    from pydub.silence import split_on_silence
except ImportError as e:
    print(f"Missing required package: {e}")
    print("Install with: pip install SpeechRecognition pydub")
    sys.exit(1)

class AudioTranscriptor:
    """Main class for audio transcription functionality."""
    
    def __init__(self):
        self.recognizer = sr.Recognizer()
        self.supported_formats = ['.wav', '.mp3', '.mp4', '.m4a', '.flac', '.ogg', '.aiff']
        
    def convert_to_wav(self, audio_path: str, output_path: str = None) -> str:
        """Convert audio file to WAV format for processing."""
        audio_path = Path(audio_path)
        
        if not audio_path.exists():
            raise FileNotFoundError(f"Audio file not found: {audio_path}")
            
        if audio_path.suffix.lower() not in self.supported_formats:
            raise ValueError(f"Unsupported format: {audio_path.suffix}")
        
        if output_path is None:
            output_path = audio_path.with_suffix('.wav')
        
        print(f"Converting {audio_path.name} to WAV format...")
        
        # Load and convert audio
        audio = AudioSegment.from_file(str(audio_path))
        
        # Convert to mono and set sample rate for better recognition
        audio = audio.set_channels(1).set_frame_rate(16000)
        
        # Export as WAV
        audio.export(str(output_path), format="wav")
        print(f"Converted audio saved as: {output_path}")
        
        return str(output_path)
    
    def split_audio_on_silence(self, audio_path: str, min_silence_len: int = 1000) -> list:
        """Split audio into chunks based on silence for better transcription."""
        print("Splitting audio on silence...")
        
        audio = AudioSegment.from_wav(audio_path)
        
        # Split audio where silence is longer than min_silence_len ms
        chunks = split_on_silence(
            audio,
            min_silence_len=min_silence_len,
            silence_thresh=audio.dBFS - 14,
            keep_silence=500  # Keep some silence at the beginning/end
        )
        
        print(f"Audio split into {len(chunks)} chunks")
        return chunks
    
    def transcribe_chunk(self, audio_chunk: AudioSegment, engine: str = 'google') -> str:
        """Transcribe a single audio chunk using specified engine."""
        # Export chunk to a securely created temporary WAV file
        tmp = tempfile.NamedTemporaryFile(delete=False, suffix=".wav")
        temp_path = tmp.name
        # Close the file so pydub/AudioFile can open it on Windows
        tmp.close()
        audio_chunk.export(temp_path, format="wav")

        try:
            with sr.AudioFile(temp_path) as source:
                audio_data = self.recognizer.record(source)

            # Choose recognition engine
            if engine == 'google':
                text = self.recognizer.recognize_google(audio_data)
            elif engine == 'sphinx':
                text = self.recognizer.recognize_sphinx(audio_data)
            elif engine == 'wit':
                # Read Wit.ai API key from environment variable for security
                wit_key = os.getenv('WIT_AI_KEY')
                if not wit_key:
                    return "[Wit.ai API key not set - set WIT_AI_KEY env var]"
                text = self.recognizer.recognize_wit(audio_data, key=wit_key)
            else:
                text = self.recognizer.recognize_google(audio_data)

            return text

        except sr.UnknownValueError:
            return "[Inaudible]"
        except sr.RequestError as e:
            return f"[Error: {e}]"
        finally:
            # Clean up temporary file
            try:
                if os.path.exists(temp_path):
                    os.remove(temp_path)
            except OSError:
                pass
    
    def transcribe_file(self, 
                       audio_path: str, 
                       output_path: str = None,
                       engine: str = 'google',
                       chunk_audio: bool = True,
                       min_silence_len: int = 1000) -> Dict[str, Any]:
        """
        Main transcription method.
        
        Args:
            audio_path: Path to audio file
            output_path: Path for output transcript (optional)
            engine: Speech recognition engine ('google', 'sphinx', 'wit')
            chunk_audio: Whether to split audio on silence
            min_silence_len: Minimum silence length for splitting (ms)
        """
        print(f"Starting transcription of: {audio_path}")
        print(f"Using engine: {engine}")
        
        start_time = datetime.now()
        
        # Convert to WAV if necessary
        if not audio_path.lower().endswith('.wav'):
            wav_path = self.convert_to_wav(audio_path)
        else:
            wav_path = audio_path
        
        transcript_parts = []
        
        if chunk_audio:
            # Split audio and transcribe chunks
            chunks = self.split_audio_on_silence(wav_path, min_silence_len)
            
            for i, chunk in enumerate(chunks, 1):
                print(f"Transcribing chunk {i}/{len(chunks)}...")
                text = self.transcribe_chunk(chunk, engine)
                if text.strip() and text != "[Inaudible]":
                    transcript_parts.append(text)
        else:
            # Transcribe entire file at once
            print("Transcribing entire audio file...")
            try:
                with sr.AudioFile(wav_path) as source:
                    audio_data = self.recognizer.record(source)
                
                if engine == 'google':
                    text = self.recognizer.recognize_google(audio_data)
                elif engine == 'sphinx':
                    text = self.recognizer.recognize_sphinx(audio_data)
                elif engine == 'wit':
                    wit_key = os.getenv('WIT_AI_KEY')
                    if not wit_key:
                        text = "[Wit.ai API key not set - set WIT_AI_KEY env var]"
                    else:
                        text = self.recognizer.recognize_wit(audio_data, key=wit_key)
                else:
                    text = self.recognizer.recognize_google(audio_data)
                    
                transcript_parts.append(text)
                
            except sr.UnknownValueError:
                transcript_parts.append("[Could not understand audio]")
            except sr.RequestError as e:
                transcript_parts.append(f"[Service error: {e}]")
        
        # Combine transcript parts
        full_transcript = " ".join(transcript_parts)
        
        # Prepare results
        end_time = datetime.now()
        processing_time = (end_time - start_time).total_seconds()
        
        results = {
            'transcript': full_transcript,
            'audio_file': audio_path,
            'engine': engine,
            'chunks_processed': len(transcript_parts),
            'processing_time_seconds': processing_time,
            'timestamp': end_time.isoformat()
        }
        
        # Save transcript
        if output_path is None:
            output_path = Path(audio_path).with_suffix('.txt')
        
        self.save_transcript(results, output_path)
        
        # Clean up temporary WAV file if created
        if wav_path != audio_path and os.path.exists(wav_path):
            os.remove(wav_path)
        
        return results
    
    def save_transcript(self, results: Dict[str, Any], output_path: str):
        """Save transcript to file with metadata."""
        with open(output_path, 'w', encoding='utf-8') as f:
            f.write(f"Audio Transcript\n")
            f.write(f"{'=' * 50}\n")
            f.write(f"Source File: {results['audio_file']}\n")
            f.write(f"Engine: {results['engine']}\n")
            f.write(f"Chunks Processed: {results['chunks_processed']}\n")
            f.write(f"Processing Time: {results['processing_time_seconds']:.2f} seconds\n")
            f.write(f"Generated: {results['timestamp']}\n")
            f.write(f"{'=' * 50}\n\n")
            f.write(results['transcript'])
        
        print(f"Transcript saved to: {output_path}")
        
        # Also save as JSON for programmatic access
        json_path = Path(output_path).with_suffix('.json')
        with open(json_path, 'w', encoding='utf-8') as f:
            json.dump(results, f, indent=2, ensure_ascii=False)
        
        print(f"Metadata saved to: {json_path}")

def main():
    """Command line interface for the transcription program."""
    if len(sys.argv) < 2:
        print("Usage: python audio_transcriptor.py <audio_file> [options]")
        print("\nOptions:")
        print("  --engine ENGINE     Recognition engine (google, sphinx, wit)")
        print("  --output PATH       Output file path")
        print("  --no-chunk         Don't split audio on silence")
        print("  --silence-len MS   Minimum silence length for splitting (default: 1000)")
        print("\nExample:")
        print("  python audio_transcriptor.py recording.mp3 --engine google --output transcript.txt")
        return
    
    audio_file = sys.argv[1]
    
    # Parse command line arguments
    engine = 'google'
    output_path = None
    chunk_audio = True
    min_silence_len = 1000
    
    i = 2
    while i < len(sys.argv):
        if sys.argv[i] == '--engine' and i + 1 < len(sys.argv):
            engine = sys.argv[i + 1]
            i += 2
        elif sys.argv[i] == '--output' and i + 1 < len(sys.argv):
            output_path = sys.argv[i + 1]
            i += 2
        elif sys.argv[i] == '--no-chunk':
            chunk_audio = False
            i += 1
        elif sys.argv[i] == '--silence-len' and i + 1 < len(sys.argv):
            min_silence_len = int(sys.argv[i + 1])
            i += 2
        else:
            i += 1
    
    # Create transcriptor and process file
    transcriptor = AudioTranscriptor()
    
    try:
        results = transcriptor.transcribe_file(
            audio_path=audio_file,
            output_path=output_path,
            engine=engine,
            chunk_audio=chunk_audio,
            min_silence_len=min_silence_len
        )
        
        print(f"\nTranscription completed successfully!")
        print(f"Transcript length: {len(results['transcript'])} characters")
        print(f"Processing time: {results['processing_time_seconds']:.2f} seconds")
        
    except Exception as e:
        print(f"Error during transcription: {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()

---
title: "Building a YouTube Transcript Extractor with Python"
date: 2026-01-07
draft: false
tags: ["python", "youtube", "automation", "tutorial"]
categories: ["Programming", "Python Projects"]
description: "Learn how to build a Python script that extracts transcripts from YouTube videos and saves them as plain text files."
---

Have you ever wanted to extract the transcript from a YouTube video for note-taking, analysis, or content creation? In this tutorial, I'll walk you through building a simple yet powerful Python script that does exactly that.

## What We're Building

Our script will:
- Accept any YouTube URL as input
- Extract the video transcript automatically
- Display the transcript in the console
- Optionally save it to a text file

## Prerequisites

Before we start, make sure you have:
- Python 3.6 or higher installed
- Basic understanding of Python syntax
- pip (Python package manager)

## Installation

First, we need to install the required library:

```bash
pip install youtube-transcript-api
```

This library handles all the heavy lifting of communicating with YouTube's API to fetch transcripts.

## The Complete Code

Here's the full script we'll be building:

```python
"""
YouTube Transcript Extractor
Extracts and saves transcripts from YouTube videos in plain text format.
"""

import re
import sys

# Import the YouTube Transcript API
try:
    from youtube_transcript_api import YouTubeTranscriptApi
    print("✓ YouTubeTranscriptApi imported successfully")
except ImportError as e:
    print(f"ERROR: Cannot import YouTubeTranscriptApi: {e}")
    print("\nPlease install: pip install youtube-transcript-api")
    sys.exit(1)

try:
    from youtube_transcript_api._errors import TranscriptsDisabled, NoTranscriptFound
    print("✓ Error classes imported successfully")
except ImportError:
    print("⚠ Warning: Could not import error classes (older version?)")
    class TranscriptsDisabled(Exception):
        pass
    class NoTranscriptFound(Exception):
        pass


def extract_video_id(url):
    """Extract video ID from various YouTube URL formats."""
    patterns = [
        r'(?:youtube\.com\/watch\?v=|youtu\.be\/)([^&\n?#]+)',
        r'youtube\.com\/embed\/([^&\n?#]+)',
        r'youtube\.com\/v\/([^&\n?#]+)'
    ]
    
    for pattern in patterns:
        match = re.search(pattern, url)
        if match:
            return match.group(1)
    
    return None


def get_transcript(video_url):
    """Fetch transcript for a YouTube video."""
    video_id = extract_video_id(video_url)
    
    if not video_id:
        print("Error: Invalid YouTube URL")
        return None
    
    print(f"✓ Extracted video ID: {video_id}")
    
    try:
        print("→ Fetching transcript...")
        
        # Create an instance and fetch transcript
        api = YouTubeTranscriptApi()
        transcript_list = api.fetch(video_id)
        
        print(f"✓ Found {len(transcript_list)} transcript segments")
        
        # Combine all text segments
        transcript_text = ' '.join([entry.text for entry in transcript_list])
        
        return transcript_text
    
    except TranscriptsDisabled:
        print(f"✗ Error: Transcripts are disabled for this video (ID: {video_id})")
        return None
    
    except NoTranscriptFound:
        print(f"✗ Error: No transcript found for this video (ID: {video_id})")
        print("   This video may not have captions available.")
        return None
    
    except AttributeError as e:
        print(f"✗ Error: API method not found - {str(e)}")
        print(f"   Available methods: {dir(YouTubeTranscriptApi)}")
        return None
    
    except Exception as e:
        print(f"✗ Error: {type(e).__name__}: {str(e)}")
        return None


def save_transcript(transcript, filename="transcript.txt"):
    """Save transcript to a text file."""
    try:
        with open(filename, 'w', encoding='utf-8') as f:
            f.write(transcript)
        print(f"\n✓ Transcript saved to: {filename}")
    except Exception as e:
        print(f"✗ Error saving file: {str(e)}")


def main():
    print("\n" + "="*50)
    print("YouTube Transcript Extractor")
    print("="*50 + "\n")
    
    # Get URL from user
    if len(sys.argv) > 1:
        video_url = sys.argv[1]
    else:
        video_url = input("Enter YouTube URL: ").strip()
    
    if not video_url:
        print("✗ Error: No URL provided")
        return
    
    print(f"\nProcessing: {video_url}\n")
    
    # Get transcript
    transcript = get_transcript(video_url)
    
    if transcript:
        print("\n" + "="*50)
        print("TRANSCRIPT")
        print("="*50 + "\n")
        print(transcript)
        print("\n" + "="*50)
        
        # Ask if user wants to save
        save_choice = input("\nSave transcript to file? (y/n): ").strip().lower()
        if save_choice == 'y':
            filename = input("Enter filename (default: transcript.txt): ").strip()
            if not filename:
                filename = "transcript.txt"
            if not filename.endswith('.txt'):
                filename += '.txt'
            save_transcript(transcript, filename)
    else:
        print("\n✗ Failed to retrieve transcript")


if __name__ == "__main__":
    main()
```

## Code Breakdown

Let's break down each component to understand how it works.

### 1. Importing Required Libraries

```python
import re
import sys
from youtube_transcript_api import YouTubeTranscriptApi
from youtube_transcript_api._errors import TranscriptsDisabled, NoTranscriptFound
```

- **re**: Regular expressions module for pattern matching (used to extract video IDs)
- **sys**: System-specific parameters (used for command-line arguments and exit codes)
- **YouTubeTranscriptApi**: The main class for fetching transcripts
- **Error classes**: Specific exceptions for handling transcript-related errors

### 2. Extracting the Video ID

```python
def extract_video_id(url):
    """Extract video ID from various YouTube URL formats."""
    patterns = [
        r'(?:youtube\.com\/watch\?v=|youtu\.be\/)([^&\n?#]+)',
        r'youtube\.com\/embed\/([^&\n?#]+)',
        r'youtube\.com\/v\/([^&\n?#]+)'
    ]
    
    for pattern in patterns:
        match = re.search(pattern, url)
        if match:
            return match.group(1)
    
    return None
```

This function handles different YouTube URL formats:
- Standard: `https://www.youtube.com/watch?v=VIDEO_ID`
- Short: `https://youtu.be/VIDEO_ID`
- Embed: `https://www.youtube.com/embed/VIDEO_ID`

The regex patterns extract the 11-character video ID from these URLs.

### 3. Fetching the Transcript

```python
def get_transcript(video_url):
    video_id = extract_video_id(video_url)
    
    if not video_id:
        print("Error: Invalid YouTube URL")
        return None
    
    try:
        api = YouTubeTranscriptApi()
        transcript_list = api.fetch(video_id)
        transcript_text = ' '.join([entry.text for entry in transcript_list])
        return transcript_text
    except TranscriptsDisabled:
        print("Error: Transcripts are disabled for this video")
        return None
    except NoTranscriptFound:
        print("Error: No transcript found for this video")
        return None
```

**Key steps:**
1. Extract the video ID from the URL
2. Create an instance of the API
3. Fetch the transcript segments
4. Join all text segments into a single string
5. Handle common errors gracefully

The `transcript_list` contains multiple segments (each with text and timestamp). We use a list comprehension to extract just the text: `[entry.text for entry in transcript_list]`

### 4. Saving to a File

```python
def save_transcript(transcript, filename="transcript.txt"):
    """Save transcript to a text file."""
    try:
        with open(filename, 'w', encoding='utf-8') as f:
            f.write(transcript)
        print(f"\n✓ Transcript saved to: {filename}")
    except Exception as e:
        print(f"✗ Error saving file: {str(e)}")
```

This function uses Python's context manager (`with` statement) to safely write the transcript to a file. The `encoding='utf-8'` ensures proper handling of special characters.

### 5. The Main Function

```python
def main():
    # Get URL from command line or user input
    if len(sys.argv) > 1:
        video_url = sys.argv[1]
    else:
        video_url = input("Enter YouTube URL: ").strip()
    
    # Fetch and display transcript
    transcript = get_transcript(video_url)
    
    if transcript:
        print(transcript)
        
        # Offer to save
        save_choice = input("\nSave transcript to file? (y/n): ").strip().lower()
        if save_choice == 'y':
            filename = input("Enter filename (default: transcript.txt): ").strip()
            if not filename:
                filename = "transcript.txt"
            save_transcript(transcript, filename)
```

The main function orchestrates everything:
1. Gets the YouTube URL (from command line or user input)
2. Fetches the transcript
3. Displays it in the console
4. Offers to save it to a file

## Usage Examples

### Interactive Mode

Simply run the script and enter the URL when prompted:

```bash
python transcript_generator.py
```

### Command-Line Mode

Pass the URL as an argument:

```bash
python transcript_generator.py "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
```

### Example Output

```text
==================================================
YouTube Transcript Extractor
==================================================

✓ Extracted video ID: dQw4w9WgXcQ
→ Fetching transcript...
✓ Found 1254 transcript segments

==================================================
TRANSCRIPT
==================================================

Never gonna give you up, never gonna let you down...

==================================================

Save transcript to file? (y/n): y
Enter filename (default: transcript.txt): my_transcript.txt

✓ Transcript saved to: my_transcript.txt
```

## Limitations and Considerations

**Important notes:**
- This only works for videos with transcripts enabled (auto-generated or manual captions)
- Some videos have transcripts disabled by their creators
- Private or age-restricted videos may not be accessible
- The transcript doesn't include timestamps (but you can modify the code to include them)

## Possible Enhancements

Want to take this project further? Here are some ideas:

1. **Add timestamp support**: Include timestamps alongside the text
2. **Multiple languages**: Fetch transcripts in different languages
3. **Batch processing**: Process multiple videos at once
4. **GUI interface**: Create a simple graphical interface using tkinter
5. **Format options**: Export to PDF, DOCX, or JSON formats
6. **Search functionality**: Search for specific words or phrases in the transcript

## Conclusion

You've now built a functional YouTube transcript extractor! This script demonstrates several important programming concepts:
- API interaction
- Regular expressions
- Error handling
- File I/O operations
- Command-line argument processing

Feel free to modify and extend this script for your own needs. Happy coding!

---

**Source code**: The complete code is available as a single Python file that you can copy and use immediately.

**Questions or improvements?** Leave a comment below or reach out. I'd love to hear how you're using this script!

---
title: "YouTube Video Downloader (Python + yt-dlp)"
date: 2025-12-30T00:00:00Z
draft: false
description: "A small, focused Python script to download YouTube videos (best quality) using yt-dlp. Includes installation and usage instructions."
tags:
    - python
    - youtube
    - yt-dlp
    - tutorial
categories:
    - code
    - tools
slug: "youtube-video-downloader"
author: "Indunil Thembuwana"

## Why this script

- Lightweight and self-contained.
- Uses `yt-dlp` (actively maintained fork of youtube-dl) to fetch best video + audio and merge to MP4.
- Auto-installs `yt-dlp` if not present, so you can get started quickly.

## Requirements

- Python 3.8 or newer
- pip available
- Internet connection
- (Optional) `ffmpeg` on PATH for merging (yt-dlp will usually bundle merging if available)

## Quick install

Option A — let the script auto-install `yt-dlp` (default behavior):

```bash
python youtubevideodownloder.py
```

Option B — install manually before running:

```bash
pip install --upgrade yt-dlp
```

## How to use

1. Save the script as `youtubevideodownloder.py` in the same folder as this Markdown (or anywhere you like).
2. Run the script:

```bash
python youtubevideodownloder.py
```

3. When prompted, paste the YouTube video URL and press Enter. The video will be saved into a `downloads/` directory by default.

## Script (ready to paste)

```python
import os
import re
import sys
import subprocess
import yt_dlp


def install_yt_dlp():
    """
    Install or upgrade yt-dlp.
    """
    try:
        subprocess.check_call([sys.executable, '-m', 'pip', 'install', '--upgrade', 'yt-dlp'])
        print("yt-dlp installed/upgraded successfully.")
    except subprocess.CalledProcessError:
        print("Failed to install yt-dlp. Please install manually using:")
        print("pip install --upgrade yt-dlp")


def sanitize_filename(filename):
    """
    Sanitize filename to remove invalid characters and limit length.
    """
    filename = re.sub(r'[^\w\-_\. ]', '_', filename)
    return filename.strip()[:200]


def download_youtube_video(youtube_url, output_dir='downloads'):
    """
    Download a YouTube video in HD format.
    """
    os.makedirs(output_dir, exist_ok=True)
    try:
        ydl_opts = {
            'format': 'bestvideo+bestaudio/best',
            'merge_output_format': 'mp4',
            'outtmpl': os.path.join(output_dir, '%(title)s.%(ext)s'),
            'nooverwrites': True,
            'no_color': True,
        }
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            ydl.download([youtube_url])
        print("Download completed successfully.")
    except Exception as e:
        print(f"An error occurred: {e}")


def main():
    # Install or upgrade yt-dlp
    install_yt_dlp()

    video_url = input("Enter the YouTube video URL: ")
    download_youtube_video(video_url)


if __name__ == "__main__":
    main()
```

## Explanation of important parts

- `install_yt_dlp()` uses the current Python interpreter to run `pip install --upgrade yt-dlp` so the script can run even on fresh systems.
- `sanitize_filename()` replaces unsafe characters and truncates long names to avoid filesystem errors.
- `download_youtube_video()` sets `format` to `bestvideo+bestaudio/best` — this asks yt-dlp to download the best separate video and audio streams (if available) and merge them; otherwise it falls back to the best single stream.
- `merge_output_format: 'mp4'` tells yt-dlp to produce an MP4 file after merging.

## Customization ideas

- Add command-line flags with `argparse` to accept `--url`, `--output`, or `--no-install`.
- Change the template `outtmpl` to include `%(id)s` or timestamps to avoid collisions.
- Force a specific resolution, e.g. `format: 'bestvideo[height<=720]+bestaudio/best'` to limit downloads to 720p.

## Troubleshooting

- If automatic installation fails, run:

```bash
pip install --upgrade yt-dlp
```

- If merging fails (errors mentioning ffmpeg), install `ffmpeg` and add it to `PATH`.

- For verbose debugging, run:

```bash
yt-dlp -v <VIDEO_URL>
```

## Legal & Ethical note

Only download videos you have permission to download. Respect copyright and YouTube's Terms of Service.

## Where the script saves files

By default the script creates a `downloads/` directory next to the script and saves files as the video title (sanitized). You can change `output_dir` when calling `download_youtube_video()` or modify `outtmpl` in the `ydl_opts` dictionary.

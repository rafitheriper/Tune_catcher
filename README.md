# 🎵 TuneCatcher v1.2

A modern, fast, and feature-rich downloader for video and audio content from YouTube, TikTok, Twitch, and 1000+ other platforms.

![Python](https://img.shields.io/badge/Python-3.8+-blue.svg)
![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20macOS%20%7C%20Linux-green.svg)
![License](https://img.shields.io/badge/License-MIT-yellow.svg)

## ✨ Features

### Core Functionality
- 🎬 **Video Downloads** - Multiple formats (MP4, MKV, WebM, AVI) with quality selection
- 🎵 **Audio Extraction** - High-quality audio in MP3, M4A, FLAC, WAV, AAC, Opus
- 📝 **Playlist Support** - Download entire playlists or select specific videos
- 🖼️ **Thumbnail Embedding** - Automatically embed cover art in audio files
- 📄 **Subtitle Download** - Auto-download subtitles and captions

### User Experience
- 🚀 **Lightning Fast Startup** - Optimized with lazy imports and threading
- 🎨 **Modern UI** - Clean interface with dark/light/system theme support
- 📊 **Real-time Progress** - Live download progress with speed and ETA
- 📁 **Smart Organization** - Automatic folder organization (Audio/Video)
- 🔄 **Batch Processing** - Download multiple items with queue management
- 📚 **Download History** - Track and re-download previous items

### Advanced Options
- 🍪 **Browser Cookie Support** - Access private/premium content using browser cookies
- 🔧 **Custom Filename Templates** - Flexible naming with metadata variables
- 📋 **URL Preview** - Live preview with title, uploader, and thumbnail
- 💾 **Persistent Settings** - Saves preferences automatically

## 🚀 Quick Start

### Option 1: Download Pre-built Executable (Recommended)
1. Go to the [Releases](https://github.com/yourusername/tunecatcher/releases) page
2. Download the latest `TuneCatcher.exe` (Windows) or `TuneCatcher` (macOS/Linux)
3. Run the executable - no installation required!

### Option 2: Run from Source
```bash
# Clone the repository
git clone https:https://github.com/rafitheriper/Tune_catcher
cd tune_catcher

# Install dependencies
pip install -r requirements.txt

# Run the application
python tunecatcher.py
```

## 📦 Building from Source

### Prerequisites
- Python 3.8 or higher
- FFmpeg (included in releases)

### Dependencies
```bash
pip install customtkinter yt-dlp pillow requests
```


## 📖 Usage Guide

### Basic Usage
1. **Paste URL** - Copy any supported video/playlist URL and paste it into the input field
2. **Choose Format** - Select Audio or Video mode and preferred format/quality
3. **Set Destination** - Choose where to save your downloads
4. **Download** - Hit the Download button and watch the progress!

### Supported Platforms
- YouTube (videos, playlists, shorts)
- TikTok
- Twitch (VODs and clips)
- Facebook
- Instagram
- Twitter/X
- Reddit
- And 1000+ more sites supported by yt-dlp

### Filename Templates
Customize how files are named using these variables:

| Variable | Description | Example |
|----------|-------------|---------|
| `%(title)s` | Video title | Amazing Video |
| `%(uploader)s` | Channel/uploader name | CoolCreator |
| `%(id)s` | Video ID | dQw4w9WgXcQ |
| `%(upload_date)s` | Upload date | 20231201 |
| `%(duration)s` | Video duration | 240 |
| `%(view_count)s` | View count | 1000000 |

**Example Templates:**
- `%(uploader)s - %(title)s` → "CoolCreator - Amazing Video"
- `%(upload_date)s - %(title)s [%(id)s]` → "20231201 - Amazing Video [dQw4w9WgXcQ]"

### Browser Cookies
To download private or region-locked content:

1. Go to **Settings** tab
2. Select your browser from "Use Cookies From" dropdown
3. Make sure you're logged into the target site in that browser
4. Downloads will use your browser's authentication

**Supported Browsers:** Chrome, Firefox, Edge, Brave, Opera, Safari

## ⚙️ Configuration

### Settings Overview
- **Appearance Mode** - Light, Dark, or System theme
- **Default Quality** - Video resolution preference (best, 1080p, 720p, etc.)
- **Audio Format** - MP3, M4A, FLAC, WAV, AAC, Opus
- **Video Format** - MP4, MKV, WebM, AVI
- **Playlist Limit** - Number of items to fetch from playlists
- **Auto Subtitles** - Automatically download captions
- **Embed Thumbnails** - Add cover art to audio files

### File Organization
```
Downloads/
├── Audio/          # All audio downloads
└── Video/          # All video downloads
```

## 🔧 Troubleshooting

### Common Issues

**Q: The application won't start or crashes immediately**
- Make sure FFmpeg is in the same folder as the executable
- Try running as administrator (Windows) or with sudo (Linux/macOS)
- Check antivirus isn't blocking the application

**Q: Downloads fail with "Video unavailable"**
- The video might be private, region-locked, or deleted
- Try using browser cookies if you can access it in your browser
- Some live streams can't be downloaded

**Q: Audio quality is poor**
- Change audio format to FLAC or M4A for better quality
- Some videos only have low-quality audio available

**Q: Downloads are slow**
- This depends on your internet connection and the video server
- Try downloading during off-peak hours
- Check if other downloads are using bandwidth

**Q: Getting "FFmpeg not found" error**
- Download FFmpeg from https://ffmpeg.org/
- Place `ffmpeg.exe` (Windows) or `ffmpeg` (Linux/macOS) in the same folder
- Or add FFmpeg to your system PATH

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- **[yt-dlp](https://github.com/yt-dlp/yt-dlp)** - The powerful downloader engine
- **[CustomTkinter](https://github.com/TomSchimansky/CustomTkinter)** - Modern UI framework
- **[FFmpeg](https://ffmpeg.org/)** - Media processing toolkit
- **[Pillow](https://python-pillow.org/)** - Image processing library

## 📊 Statistics

- **1000+** supported websites
- **6** audio formats supported
- **4** video formats supported  
- **7** video quality options
- **Multiple** browser cookie integrations

## 🔄 Changelog

### v1.2 (Latest)
- ⚡ **50% faster startup** with lazy imports and threading
- 🎨 Improved UI responsiveness
- 🔧 Better error handling and user feedback
- 📱 Enhanced playlist selection interface
- 💾 Optimized settings management
- 🐛 Various bug fixes and stability improvements
t

---

<div align="center">

**⭐ Star this repository if you find it helpful! ⭐**


</div>

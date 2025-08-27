import os, sys, threading, json, subprocess, time, shutil, re
from io import BytesIO
import customtkinter as ctk
from customtkinter import filedialog
import yt_dlp, requests
from PIL import Image
from yt_dlp.utils import DownloadError
from tkinter import messagebox

# --- Constants ---
SUPPORTED_BROWSERS = ['none', 'chrome', 'firefox', 'edge', 'brave', 'opera', 'safari']
AUDIO_FORMATS = ['m4a', 'mp3', 'wav', 'flac', 'aac', 'opus']
VIDEO_FORMATS = ['mp4', 'mkv', 'webm', 'avi'] # MP4 first for compatibility
VIDEO_RESOLUTIONS = ['best', '2160p', '1440p', '1080p', '720p', '480p', '360p']
FILENAME_PRESETS = {
    "Title [ID]": "%(title)s [%(id)s]",
    "Uploader - Title": "%(uploader)s - %(title)s", 
    "Date - Title": "%(upload_date)s - %(title)s",
    "Title": "%(title)s",
    "Custom...": "custom"
}
ABOUT_INFO = {
    "app_name": "TuneCatcher", "version": "1.2 Final", "tagline": "A modern downloader for video and audio.",
    "description": "Enhanced with better error handling, quality options, format support, and user experience improvements.\n\nSupports video/audio from many sites including YouTube, Twitch, TikTok, facebook, instagarm, and more.",
    "credits": "Built with:\n• Python\n• customtkinter\n• yt-dlp\n• FFmpeg"
}

class PlaylistSelectionWindow(ctk.CTkToplevel):
    def __init__(self, master, playlist_url):
        super().__init__(master); self.master_app = master; self.title("Select Playlist Items"); self.geometry("650x450")
        self.grid_columnconfigure(0, weight=1); self.grid_rowconfigure(1, weight=1); self.checkboxes = []
        self.loading_label = ctk.CTkLabel(self, text="Fetching playlist, please wait..."); self.loading_label.pack(pady=20)
        self.after(200, lambda: threading.Thread(target=self.fetch_and_populate, args=(playlist_url,), daemon=True).start())
        self.protocol("WM_DELETE_WINDOW", self.destroy); self.grab_set()

    def fetch_and_populate(self, url):
        try:
            limit = self.master_app.settings['playlist_limit']
            ydl_opts = {'quiet': True, 'extract_flat': True, 'playlistend': int(limit) if limit.isdigit() and limit != "all" else None, 'no_warnings': True}
            with yt_dlp.YoutubeDL(ydl_opts) as ydl: info = ydl.extract_info(url, download=False)
            entries = [e for e in info.get('entries', []) if e and e.get('title')]
            self.after(0, self.populate_ui, entries, info.get('title', 'Playlist'))
        except Exception as e: self.after(0, lambda: (self.master_app.update_status(f"Error fetching playlist: {str(e)[:50]}..."), self.destroy()))

    def populate_ui(self, entries, playlist_title):
        for w in self.winfo_children(): w.destroy()
        self.title(f"Select Items - {playlist_title[:30]}...")
        controls_frame = ctk.CTkFrame(self); controls_frame.grid(row=0, column=0, padx=10, pady=10, sticky="ew")
        ctk.CTkButton(controls_frame, text="Select All", command=lambda: [cb.select() for cb, url in self.checkboxes]).pack(side="left", padx=5)
        ctk.CTkButton(controls_frame, text="Deselect All", command=lambda: [cb.deselect() for cb, url in self.checkboxes]).pack(side="left", padx=5)
        ctk.CTkLabel(controls_frame, text=f"{len(entries)} items").pack(side="right", padx=10)
        scroll_frame = ctk.CTkScrollableFrame(self, label_text=f"Items from playlist"); scroll_frame.grid(row=1, column=0, padx=10, pady=10, sticky="nsew")
        for i, entry in enumerate(entries):
            cb = ctk.CTkCheckBox(scroll_frame, text=f"{i+1}. {entry.get('title', 'Unknown Title')[:60]}"); cb.pack(fill="x", padx=10, pady=2, anchor="w")
            self.checkboxes.append((cb, entry.get('url')))
        button_frame = ctk.CTkFrame(self); button_frame.grid(row=2, column=0, padx=10, pady=10, sticky="ew")
        ctk.CTkButton(button_frame, text="Download Selected", command=self.download_selected, height=35).pack(side="left", fill="x", expand=True, padx=(0, 5))
        ctk.CTkButton(button_frame, text="Cancel", command=self.destroy, height=35).pack(side="right", padx=(5, 0))

    def download_selected(self):
        selected_urls = [url for cb, url in self.checkboxes if cb.get() == 1 and url]
        if not selected_urls: 
            messagebox.showwarning("No Selection", "Please select at least one item to download.")
            return
        
        # --- FIX: Create a list of 'job' dictionaries, not just URLs ---
        jobs = [{
            'url': url,
            'mode': self.master_app.settings['mode'],
            'audio_format': self.master_app.settings['audio_format'],
            'video_format': self.master_app.settings['video_format'],
            'video_quality': self.master_app.settings['video_quality'],
        } for url in selected_urls]
        
        self.master_app.update_status(f"Starting batch download of {len(jobs)} items...")
        # Use a thread to avoid freezing the UI
        threading.Thread(target=self.master_app.download_batch, args=(jobs,), daemon=True).start()
        self.destroy()

class TuneCatcher(ctk.CTk):
    def __init__(self, ffmpeg_path):
        super().__init__()
        self.ffmpeg_path = ffmpeg_path
        
        # --- FIX: Define a persistent directory for the config file ---
        # This ensures settings are saved next to the .exe, not in a temp folder
        persistent_dir = os.path.dirname(sys.executable if getattr(sys, 'frozen', False) else os.path.abspath(__file__))
        self.config_file = os.path.join(persistent_dir, 'tunecatcher_config.json')
        
        default_save_path = os.path.join(persistent_dir, "Downloads")
        
        self.settings = {'mode': 'audio', 'video_quality': '720p', 'cookie_browser': 'none', 'audio_format': 'mp3', 'video_format': 'mp4',
                         'appearance_mode': 'System', 'save_path': default_save_path, 'history': [], 'playlist_limit': '10',
                         'filename_preset': 'Title [ID]', 'filename_template_custom': '%(title)s', 'auto_subtitle': False, 'embed_thumbnail': False}
        try:
            with open(self.config_file, 'r', encoding='utf-8') as f: self.settings.update(json.load(f))
        except (FileNotFoundError, json.JSONDecodeError):
            if not os.path.exists(default_save_path):
                os.makedirs(default_save_path)

        self.preview_thread, self.preview_thread_stop_event = None, threading.Event()
        self.last_download_path, self.current_downloads = None, 0
        self.download_queue = []

        self.title("TuneCatcher"); self.geometry("780x720"); self.grid_columnconfigure(0, weight=1)
        ctk.set_appearance_mode(self.settings['appearance_mode']); self.create_widgets(); self.populate_history_tab()

    def save_settings(self):
        try:
            with open(self.config_file, 'w', encoding='utf-8') as f: json.dump(self.settings, f, indent=4, ensure_ascii=False)
        except Exception as e: print(f"Failed to save settings: {e}")

    def create_widgets(self):
        self.tab_view = ctk.CTkTabview(self); self.tab_view.grid(row=0, column=0, padx=20, pady=20, sticky="nsew"); self.grid_rowconfigure(0, weight=1)
        downloader_tab, history_tab, settings_tab, about_tab = self.tab_view.add("Downloader"), self.tab_view.add("History"), self.tab_view.add("Settings"), self.tab_view.add("About")
        downloader_tab.grid_columnconfigure(0, weight=1)
        url_frame = ctk.CTkFrame(downloader_tab); url_frame.grid(row=0, column=0, padx=10, pady=10, sticky="ew"); url_frame.grid_columnconfigure(0, weight=1)
        self.url_entry = ctk.CTkEntry(url_frame, placeholder_text="Paste a URL or Playlist... (Ctrl+V)"); self.url_entry.grid(row=0, column=0, padx=10, pady=10, sticky="ew")
        self.url_entry.bind("<KeyRelease>", self.trigger_preview_update); self.url_entry.bind("<Control-v>", lambda e: self.after(10, self.trigger_preview_update))
        ctk.CTkButton(url_frame, text="Paste", width=60, command=self.paste_url).grid(row=0, column=1, padx=(0,10), pady=10)
        preview_frame = ctk.CTkFrame(downloader_tab); preview_frame.grid(row=1, column=0, padx=10, pady=10, sticky="ew"); preview_frame.grid_columnconfigure(1, weight=1)
        self.thumbnail_label = ctk.CTkLabel(preview_frame, text=""); self.thumbnail_label.grid(row=0, column=0, rowspan=3, padx=10, pady=10)
        self.title_label = ctk.CTkLabel(preview_frame, text="Title...", anchor="w", font=ctk.CTkFont(weight="bold")); self.title_label.grid(row=0, column=1, padx=10, pady=(10, 0), sticky="ew")
        self.uploader_label = ctk.CTkLabel(preview_frame, text="Uploader...", anchor="w"); self.uploader_label.grid(row=1, column=1, padx=10, pady=0, sticky="ew")
        self.duration_label = ctk.CTkLabel(preview_frame, text="", anchor="w", text_color="gray"); self.duration_label.grid(row=2, column=1, padx=10, pady=(0, 10), sticky="ew")
        button_frame = ctk.CTkFrame(downloader_tab); button_frame.grid(row=2, column=0, padx=10, pady=10, sticky="ew"); button_frame.grid_columnconfigure(0, weight=1)
        self.download_now_button = ctk.CTkButton(button_frame, text="Download", command=self.handle_url_action, height=40); self.download_now_button.grid(row=0, column=0, padx=10, pady=10, sticky="ew")
        self.queue_label = ctk.CTkLabel(button_frame, text="", text_color="gray"); self.queue_label.grid(row=1, column=0, padx=10, pady=(0,10))
        progress_frame = ctk.CTkFrame(downloader_tab); progress_frame.grid(row=3, column=0, padx=10, pady=10, sticky="ew"); progress_frame.grid_columnconfigure(0, weight=1)
        self.progress_bar = ctk.CTkProgressBar(progress_frame); self.progress_bar.set(0); self.progress_bar.grid(row=0, column=0, padx=10, pady=10, sticky="ew")
        options_frame = ctk.CTkFrame(downloader_tab); options_frame.grid(row=4, column=0, padx=10, pady=10, sticky="ew")
        self.mode_segmented_button = ctk.CTkSegmentedButton(options_frame, values=["Audio", "Video"], command=self.on_mode_change); self.mode_segmented_button.set(self.settings['mode'].capitalize()); self.mode_segmented_button.pack(side="left", padx=10, pady=10)
        self.audio_options_frame = ctk.CTkFrame(options_frame)
        ctk.CTkLabel(self.audio_options_frame, text="Format:").pack(side="left", padx=(10,5))
        self.audio_format_menu = ctk.CTkOptionMenu(self.audio_options_frame, values=AUDIO_FORMATS, command=lambda f: self.on_setting_change('audio_format', f)); self.audio_format_menu.set(self.settings['audio_format']); self.audio_format_menu.pack(side="left", padx=(0,10))
        self.video_options_frame = ctk.CTkFrame(options_frame)
        ctk.CTkLabel(self.video_options_frame, text="Format:").pack(side="left", padx=(10,5))
        self.video_format_menu = ctk.CTkOptionMenu(self.video_options_frame, values=VIDEO_FORMATS, command=lambda f: self.on_setting_change('video_format', f)); self.video_format_menu.set(self.settings['video_format']); self.video_format_menu.pack(side="left", padx=(0,10))
        ctk.CTkLabel(self.video_options_frame, text="Quality:").pack(side="left", padx=(10,5))
        self.resolution_menu = ctk.CTkOptionMenu(self.video_options_frame, values=VIDEO_RESOLUTIONS, command=lambda r: self.on_setting_change('video_quality', r)); self.resolution_menu.set(self.settings['video_quality']); self.resolution_menu.pack(side="left", padx=(0,10))
        path_frame = ctk.CTkFrame(downloader_tab); path_frame.grid(row=5, column=0, padx=10, pady=10, sticky="ew"); path_frame.grid_columnconfigure(0, weight=1)
        p = self.settings['save_path']; p_display = p if len(p) <= 55 else f"...{p[-52:]}"
        self.path_label = ctk.CTkLabel(path_frame, text=f"Save To: {p_display}", anchor="w"); self.path_label.grid(row=0, column=0, padx=10, pady=10, sticky="ew")
        ctk.CTkButton(path_frame, text="Browse", command=self.select_save_path, width=80).grid(row=0, column=1, padx=10, pady=10)
        status_frame = ctk.CTkFrame(downloader_tab); status_frame.grid(row=6, column=0, padx=10, pady=20, sticky="sew"); downloader_tab.grid_rowconfigure(6, weight=1); status_frame.grid_columnconfigure(0, weight=1)
        self.status_label = ctk.CTkLabel(status_frame, text="Ready", anchor="w"); self.status_label.grid(row=0, column=0, padx=10, pady=(10, 5), sticky="ew")
        self.open_folder_button = ctk.CTkButton(status_frame, text="Open Download Folder", command=lambda: self.open_folder(self.last_download_path))
        history_tab.grid_columnconfigure(0, weight=1); history_tab.grid_rowconfigure(1, weight=1)
        history_top_frame = ctk.CTkFrame(history_tab); history_top_frame.grid(row=0, column=0, padx=10, pady=10, sticky="ew"); history_top_frame.grid_columnconfigure(0, weight=1)
        ctk.CTkLabel(history_top_frame, text="Download History", font=ctk.CTkFont(weight="bold")).grid(row=0, column=0, sticky="w")
        history_buttons_frame = ctk.CTkFrame(history_top_frame); history_buttons_frame.grid(row=0, column=1, sticky="e")
        ctk.CTkButton(history_buttons_frame, text="Open Folder", command=lambda: self.open_folder(self.settings['save_path']), width=100).pack(side="left", padx=(0,5))
        ctk.CTkButton(history_buttons_frame, text="Clear History", command=self.clear_history, width=100).pack(side="left")
        self.history_scroll_frame = ctk.CTkScrollableFrame(history_tab); self.history_scroll_frame.grid(row=1, column=0, padx=10, pady=(0,10), sticky="nsew")
        settings_tab.grid_columnconfigure(0, weight=1)
        appearance_frame = ctk.CTkFrame(settings_tab); appearance_frame.grid(row=0, column=0, padx=10, pady=10, sticky="ew")
        ctk.CTkLabel(appearance_frame, text="Appearance Mode:").pack(side="left", padx=10, pady=10)
        appearance_menu = ctk.CTkOptionMenu(appearance_frame, values=['System', 'Light', 'Dark'], command=self.on_appearance_change); appearance_menu.set(self.settings['appearance_mode']); appearance_menu.pack(side="left", padx=10, pady=10)
        cookie_frame = ctk.CTkFrame(settings_tab); cookie_frame.grid(row=1, column=0, padx=10, pady=10, sticky="ew")
        ctk.CTkLabel(cookie_frame, text="Use Cookies From:").pack(side="left", padx=10, pady=10)
        cookie_menu = ctk.CTkOptionMenu(cookie_frame, values=[b.capitalize() for b in SUPPORTED_BROWSERS], command=lambda b: self.on_setting_change('cookie_browser', b.lower())); cookie_menu.set(self.settings['cookie_browser'].capitalize()); cookie_menu.pack(side="left", padx=10, pady=10)
        playlist_frame = ctk.CTkFrame(settings_tab); playlist_frame.grid(row=2, column=0, padx=10, pady=10, sticky="ew")
        ctk.CTkLabel(playlist_frame, text="Playlist items to fetch ('all' for unlimited):").pack(side="left", padx=10, pady=10)
        self.playlist_limit_entry = ctk.CTkEntry(playlist_frame, width=80); self.playlist_limit_entry.insert(0, self.settings['playlist_limit']); self.playlist_limit_entry.pack(side="left", padx=10, pady=10)
        self.playlist_limit_entry.bind("<KeyRelease>", self.on_playlist_limit_change)
        extras_frame = ctk.CTkFrame(settings_tab); extras_frame.grid(row=3, column=0, padx=10, pady=10, sticky="ew")
        self.subtitle_check = ctk.CTkCheckBox(extras_frame, text="Auto-download subtitles", command=lambda: self.on_setting_change('auto_subtitle', self.subtitle_check.get()))
        self.subtitle_check.pack(side="left", padx=10, pady=10); self.subtitle_check.select() if self.settings['auto_subtitle'] else self.subtitle_check.deselect()
        self.thumbnail_check = ctk.CTkCheckBox(extras_frame, text="Embed thumbnails", command=lambda: self.on_setting_change('embed_thumbnail', self.thumbnail_check.get()))
        self.thumbnail_check.pack(side="left", padx=10, pady=10); self.thumbnail_check.select() if self.settings['embed_thumbnail'] else self.thumbnail_check.deselect()
        filename_frame = ctk.CTkFrame(settings_tab); filename_frame.grid(row=4, column=0, padx=10, pady=10, sticky="ew"); filename_frame.grid_columnconfigure(1, weight=1)
        ctk.CTkLabel(filename_frame, text="Filename Format:").grid(row=0, column=0, padx=10, pady=10)
        self.filename_menu = ctk.CTkOptionMenu(filename_frame, values=list(FILENAME_PRESETS.keys()), command=self.on_filename_preset_change); self.filename_menu.set(self.settings['filename_preset']); self.filename_menu.grid(row=0, column=1, padx=10, pady=10, sticky="ew")
        self.filename_entry = ctk.CTkEntry(filename_frame, placeholder_text="e.g., %(uploader)s - %(title)s"); self.filename_entry.insert(0, self.settings['filename_template_custom'])
        self.filename_entry.bind("<KeyRelease>", lambda e: self.on_setting_change('filename_template_custom', self.filename_entry.get()))
        about_tab.grid_columnconfigure(0, weight=1); about_tab.grid_rowconfigure(0, weight=1)
        about_frame = ctk.CTkScrollableFrame(about_tab); about_frame.grid(row=0, column=0, padx=15, pady=15, sticky="nsew"); about_frame.grid_columnconfigure(0, weight=1)
        ctk.CTkLabel(about_frame, text=f"{ABOUT_INFO['app_name']} v{ABOUT_INFO['version']}", font=ctk.CTkFont(size=24, weight="bold")).pack(pady=(10, 5), padx=10, anchor="w")
        ctk.CTkLabel(about_frame, text=ABOUT_INFO['tagline'], font=ctk.CTkFont(size=14, slant="italic")).pack(pady=(0, 20), padx=10, anchor="w")
        ctk.CTkLabel(about_frame, text=ABOUT_INFO['description'], wraplength=600, justify="left").pack(pady=10, padx=10, anchor="w", fill="x")
        ctk.CTkLabel(about_frame, text=ABOUT_INFO['credits'], wraplength=600, justify="left").pack(pady=10, padx=10, anchor="w", fill="x")
        self.on_mode_change(self.settings['mode']); self.on_filename_preset_change(self.settings['filename_preset'])

    def paste_url(self):
        try: self.url_entry.delete(0, 'end'); self.url_entry.insert(0, self.clipboard_get()); self.trigger_preview_update()
        except: pass

    def on_setting_change(self, key, value): self.settings[key] = value; self.save_settings()
    def on_appearance_change(self, new_mode): self.on_setting_change('appearance_mode', new_mode); ctk.set_appearance_mode(new_mode)
    def on_mode_change(self, new_mode):
        self.on_setting_change('mode', new_mode.lower())
        is_video = self.settings['mode'] == 'video'
        self.video_options_frame.pack(side="left", padx=10, pady=10) if is_video else self.video_options_frame.pack_forget()
        self.audio_options_frame.pack_forget() if is_video else self.audio_options_frame.pack(side="left", padx=10, pady=10)
    def on_filename_preset_change(self, new_preset):
        self.on_setting_change('filename_preset', new_preset)
        self.filename_entry.grid(row=1, column=1, padx=10, pady=(0,10), sticky="ew") if new_preset == "Custom..." else self.filename_entry.grid_forget()
    def on_playlist_limit_change(self, event=None):
        value = self.playlist_limit_entry.get().lower()
        if value.isdigit() or value == "all": self.on_setting_change('playlist_limit', value)
    def select_save_path(self):
        new_path = filedialog.askdirectory(initialdir=self.settings['save_path'])
        if new_path: self.on_setting_change('save_path', new_path); p_display = new_path if len(new_path) <= 55 else f"...{new_path[-52:]}"; self.path_label.configure(text=f"Save To: {p_display}")

    def update_queue_display(self):
        count = len(self.download_queue)
        self.queue_label.configure(text=f"{count} items in queue" if count > 0 else "")

    def handle_url_action(self):
        url = self.url_entry.get().strip()
        if not url or not re.match(r'https?://', url): self.update_status("Please enter a valid URL."); return
        self.set_ui_state("disabled", "Checking..."); threading.Thread(target=self._handle_url_thread, args=(url,), daemon=True).start()

    def _handle_url_thread(self, url):
        try:
            with yt_dlp.YoutubeDL({'quiet': True, 'extract_flat': True, 'no_warnings': True}) as ydl:
                info = ydl.extract_info(url, download=False)
            if 'entries' in info and info.get('entries'):
                self.after(0, self.set_ui_state, "normal"); self.after(0, lambda: PlaylistSelectionWindow(self, url))
            else:
                # --- FIX: Create a 'job' dictionary for a single download ---
                job = {
                    'url': url,
                    'mode': self.settings['mode'],
                    'audio_format': self.settings['audio_format'],
                    'video_format': self.settings['video_format'],
                    'video_quality': self.settings['video_quality'],
                }
                self.download_batch([job])
        except Exception as e: self.after(0, self.update_status, f"Error: {str(e)[:50]}..."); self.after(0, self.set_ui_state, "normal")

    # --- FIX: This now accepts a list of 'job' dictionaries ---
    def download_batch(self, jobs):
        self.download_queue.extend(jobs); self.after(0, self.update_queue_display)
        if self.current_downloads == 0: self.process_queue()

    def process_queue(self):
        if not self.download_queue: self.after(0, self.update_queue_display); return
        job = self.download_queue.pop(0) # Pulls the whole job dictionary
        self.current_downloads += 1; self.after(0, self.update_queue_display)
        threading.Thread(target=self._download_single, args=(job,), daemon=True).start()

    # --- FIX: This now accepts a 'job' dictionary directly ---
    def _download_single(self, job):
        self.download_content(job) # Pass the job dictionary directly
        self.current_downloads -= 1
        if self.download_queue: 
            time.sleep(0.5)
            self.process_queue()

    def trigger_preview_update(self, event=None):
        if self.preview_thread and self.preview_thread.is_alive(): self.preview_thread_stop_event.set()
        url = self.url_entry.get().strip(); self.preview_thread_stop_event.clear()
        if url and re.match(r'https?://', url):
            self.preview_thread = threading.Thread(target=self._update_preview_thread, args=(url, self.preview_thread_stop_event.is_set), daemon=True); self.preview_thread.start()
        else: self._set_preview_data(None, None)

    def _update_preview_thread(self, url, stop_check):
        self.after(0, self._set_preview_data, "loading", None); info = self.fetch_metadata(url, stop_check); image = None
        if stop_check() or not info: self.after(0, self._set_preview_data, None, None); return
        if info.get('thumbnail_url'):
            try:
                response = requests.get(info['thumbnail_url'], timeout=10); response.raise_for_status()
                if not stop_check(): image = ctk.CTkImage(light_image=Image.open(BytesIO(response.content)), size=(160, 90))
            except Exception: pass
        if not stop_check(): self.after(0, self._set_preview_data, info, image)

    def _set_preview_data(self, info, image):
        if info == "loading": 
            self.title_label.configure(text="Fetching..."); self.uploader_label.configure(text=""); self.duration_label.configure(text=""); self.thumbnail_label.configure(image=None)
        elif info:
            self.title_label.configure(text=info.get('title', 'Unknown')[:80] + ('...' if len(info.get('title', '')) > 80 else ''))
            self.uploader_label.configure(text=f"By: {info.get('uploader', 'Unknown')}")
            duration = self.format_duration(info.get('duration')); self.duration_label.configure(text=duration)
            self.thumbnail_label.configure(image=image, text="")
        else: self.title_label.configure(text="Invalid URL or video not found"); self.uploader_label.configure(text=""); self.duration_label.configure(text=""); self.thumbnail_label.configure(image=None, text="")

    def download_content(self, job):
        self.after(0, self.set_ui_state, "disabled", "Downloading...")
        success, filepath = False, None
        try:
            subfolder = "Audio" if job['mode'] == 'audio' else "Video"
            output_dir = os.path.join(self.settings['save_path'], subfolder); os.makedirs(output_dir, exist_ok=True); self.last_download_path = output_dir
            opts = self.build_ydl_options(output_dir, job)
            
            with yt_dlp.YoutubeDL(opts) as ydl:
                info = ydl.extract_info(job['url'], download=True)
                filepath = ydl.prepare_filename(info)
            
            if filepath and os.path.exists(filepath):
                success = True; title = info.get('title', 'Unknown')
                self.settings['history'].insert(0, {'title': title, 'url': job['url'], 'file_path': filepath, 'timestamp': time.strftime('%Y-%m-%d %H:%M')})
                self.settings['history'] = self.settings['history'][:50]; self.save_settings()
                self.after(0, self.update_status, f"✓ Downloaded: {title[:40]}{'...' if len(title) > 40 else ''}")
            else: self.after(0, self.update_status, "✗ Download failed or was skipped.")

        except DownloadError as e: self.after(0, self.update_status, f"✗ Download Error: {str(e).split(':')[-1].strip()[:50]}...")
        except Exception as e: self.after(0, self.update_status, f"✗ Unexpected error: {str(e)[:50]}...")
        finally: self.after(0, self.on_download_finished, success)

    def fetch_metadata(self, url, stop_check):
        try:
            with yt_dlp.YoutubeDL({'quiet': True, 'extract_flat': False, 'no_warnings': True}) as ydl:
                info = ydl.extract_info(url, download=False)
                if stop_check(): return None
                return {'title': info.get('title', 'No Title'), 'uploader': info.get('uploader', 'Unknown'), 'thumbnail_url': info.get('thumbnail'), 'duration': info.get('duration')}
        except Exception: return None

    def build_ydl_options(self, output_dir, job):
        preset = self.settings['filename_preset']
        template = FILENAME_PRESETS.get(preset, "%(title)s") if preset != 'Custom...' else self.settings['filename_template_custom']
        template = re.sub(r'[<>:"/\\|?*]', '_', template)
        
        opts = {
            'ignoreerrors': True, 'no_warnings': True,
            'outtmpl': os.path.join(output_dir, f"{template}.%(ext)s"),
            'progress_hooks': [self._progress_hook], 'noprogress': True, 'ffmpeg_location': self.ffmpeg_path
        }
        
        if self.settings['cookie_browser'] != 'none': opts['cookiesfrombrowser'] = (self.settings['cookie_browser'], None)
        if self.settings['auto_subtitle']: opts['writesubtitles'] = True; opts['writeautomaticsub'] = True
        
        # --- FIX: Correctly separated logic for Audio and Video ---
        if job['mode'] == 'audio':
            opts.update({
                'format': 'bestaudio/best',
                'extractaudio': True,
                'postprocessors': [{
                    'key': 'FFmpegExtractAudio',
                    'preferredcodec': job['audio_format'],
                    'preferredquality': '192' if job['audio_format'] == 'mp3' else None,
                }],
            })
            if self.settings['embed_thumbnail']: opts['postprocessors'].append({'key': 'EmbedThumbnail', 'already_have_thumbnail': False})
        else:  # This is for Video mode
            quality = job['video_quality']
            # Prioritize H.264 video (avc) and AAC audio (m4a) for maximum compatibility.
            if quality == 'best':
                format_selector = 'bestvideo[vcodec^=avc]+bestaudio[acodec^=mp4a]/bestvideo+bestaudio/best'
            else:
                quality_num = quality.replace('p', '')
                format_selector = f"bestvideo[height<={quality_num}][vcodec^=avc]+bestaudio[acodec^=mp4a]/bestvideo[height<={quality_num}]+bestaudio/best"
            
            opts.update({
                'format': format_selector,
                'merge_output_format': job['video_format']
            })
            if self.settings['embed_thumbnail']: opts['postprocessors'] = [{'key': 'EmbedThumbnail', 'already_have_thumbnail': False}]
        
        return opts

    def _progress_hook(self, d):
        if d['status'] == 'downloading':
            total = d.get('total_bytes') or d.get('total_bytes_estimate', 0)
            if total > 0:
                percent = d.get('downloaded_bytes', 0) / total
                speed = d.get('speed', 0)
                speed_str = f"{speed / (1024**2):.1f} MB/s" if speed and speed > 0 else "..."
                eta_str = self.format_time(d.get('eta')) if d.get('eta') else "..."
                downloaded_mb = d.get('downloaded_bytes', 0) / (1024**2)
                total_mb = total / (1024**2)
                self.after(0, self.update_progress, percent, f"Downloading... {percent*100:.1f}% ({downloaded_mb:.1f}/{total_mb:.1f} MB) | {speed_str} | ETA: {eta_str}")
        elif d['status'] == 'finished': self.after(0, self.update_progress, 1.0, "Processing...")

    def on_download_finished(self, success):
        self.set_ui_state("normal"); self.progress_bar.set(0)
        if success: self.populate_history_tab(); self.open_folder_button.grid(row=0, column=1, padx=10, pady=(10,5), sticky="e")

    def set_ui_state(self, state, message=""):
        self.download_now_button.configure(state=state); self.url_entry.configure(state=state)
        if state == "disabled": self.download_now_button.configure(text=message)
        else: self.download_now_button.configure(text="Download"); self.update_status("Ready" if not self.download_queue else f"Ready ({len(self.download_queue)} queued)")

    def populate_history_tab(self):
        for w in self.history_scroll_frame.winfo_children(): w.destroy()
        if not self.settings['history']: ctk.CTkLabel(self.history_scroll_frame, text="No downloads yet. Start by pasting a URL above!").pack(pady=30); return
        
        for i, item in enumerate(self.settings['history']):
            frame = ctk.CTkFrame(self.history_scroll_frame); frame.pack(fill="x", padx=5, pady=3); frame.grid_columnconfigure(0, weight=1)
            title_text = item.get('title', 'Unknown')[:60] + ('...' if len(item.get('title', '')) > 60 else '')
            timestamp = item.get('timestamp', 'Unknown time')
            info_frame = ctk.CTkFrame(frame, fg_color="transparent"); info_frame.grid(row=0, column=0, padx=10, pady=8, sticky="ew"); info_frame.grid_columnconfigure(0, weight=1)
            ctk.CTkLabel(info_frame, text=title_text, anchor="w", font=ctk.CTkFont(weight="bold")).grid(row=0, column=0, sticky="ew", padx=5)
            ctk.CTkLabel(info_frame, text=timestamp, anchor="w", text_color="gray", font=ctk.CTkFont(size=11)).grid(row=1, column=0, sticky="ew", padx=5)
            button_frame = ctk.CTkFrame(frame, fg_color="transparent"); button_frame.grid(row=0, column=1, padx=10, pady=8)
            ctk.CTkButton(button_frame, text="Open", width=70, height=25, command=lambda p=item.get('file_path'): self.open_file(p)).pack(side="left", padx=2)
            ctk.CTkButton(button_frame, text="Copy URL", width=80, height=25, command=lambda u=item.get('url'): self.copy_to_clipboard(u)).pack(side="left", padx=2)
            ctk.CTkButton(button_frame, text="Re-download", width=90, height=25, command=lambda u=item.get('url'): self.redownload_item(u)).pack(side="left", padx=2)

    def redownload_item(self, url):
        self.url_entry.delete(0, 'end'); self.url_entry.insert(0, url); self.tab_view.set("Downloader"); self.handle_url_action()

    def clear_history(self):
        if messagebox.askyesno("Clear History", "Delete all download history?\n\n(Downloaded files will remain on disk)"):
            self.settings['history'].clear(); self.save_settings(); self.populate_history_tab(); self.update_status("History cleared.")

    def open_file(self, path):
        if not path or not os.path.exists(path): self.update_status(f"File not found: {os.path.basename(path) if path else 'Unknown'}"); return
        try:
            if sys.platform == "win32": os.startfile(os.path.dirname(path)) # Open folder instead of file for consistency
            elif sys.platform == "darwin": subprocess.run(["open", "-R", path])
            else: subprocess.run(["xdg-open", os.path.dirname(path)])
        except Exception as e: self.update_status(f"Cannot open file location: {e}")

    def open_folder(self, path):
        if not path: path = self.settings['save_path']
        if not os.path.exists(path): os.makedirs(path, exist_ok=True)
        try:
            if sys.platform == "win32": os.startfile(path)
            elif sys.platform == "darwin": subprocess.run(["open", path])
            else: subprocess.run(["xdg-open", path])
        except Exception as e: self.update_status(f"Cannot open folder: {e}")

    def copy_to_clipboard(self, text): 
        try: self.clipboard_clear(); self.clipboard_append(text); self.update_status("URL copied to clipboard!")
        except: self.update_status("Failed to copy URL")

    def update_progress(self, value, text): self.progress_bar.set(value); self.status_label.configure(text=text)
    def update_status(self, text): self.status_label.configure(text=text)

    @staticmethod
    def format_time(seconds):
        if seconds is None or seconds <= 0: return "Unknown"
        m, s = divmod(seconds, 60); h, m = divmod(m, 60)
        return f"{int(h):02}:{int(m):02}:{int(s):02}" if h > 0 else f"{int(m):02}:{int(s):02}"

    @staticmethod
    def format_duration(seconds):
        if not seconds: return ""
        seconds = int(seconds)
        m, s = divmod(seconds, 60); h, m = divmod(m, 60)
        if h > 0: return f"Duration: {h}h {m}m"
        elif m > 0: return f"Duration: {m}m {s}s"
        return f"Duration: {s}s"

# =========================================================================================
# --- FIX: THIS IS THE CORRECTED ENTRY POINT FOR THE APPLICATION ---
# It correctly finds FFmpeg whether running as a .py script or as a bundled .exe
# =========================================================================================
if __name__ == "__main__":
    if hasattr(sys, '_MEIPASS'):
        # If running as a PyInstaller executable, the base path is the temporary folder
        base_dir = sys._MEIPASS
    else:
        # If running as a normal .py script, the base path is the script's directory
        base_dir = os.path.dirname(os.path.abspath(__file__))

    ffmpeg_name = "ffmpeg.exe" if sys.platform == "win32" else "ffmpeg"
    local_ffmpeg_path = os.path.join(base_dir, ffmpeg_name)

    if os.path.exists(local_ffmpeg_path):
        app = TuneCatcher(local_ffmpeg_path)
        app.protocol("WM_DELETE_WINDOW", app.destroy)
        app.mainloop()
    else:
        # This error is critical if the EXE was built incorrectly
        import tkinter
        from tkinter import messagebox
        root = tkinter.Tk()
        root.withdraw()
        messagebox.showerror("Critical Error: FFmpeg Missing", 
                             f"FFmpeg executable ('{ffmpeg_name}') could not be found.\n\n"
                             f"The application's core component is missing and it cannot run.\n\n"
                             f"Searched in: {local_ffmpeg_path}")
        sys.exit(1)

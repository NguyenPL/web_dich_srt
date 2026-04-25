import os
import json
import uuid
import threading
import pysrt
import subprocess
import imageio_ffmpeg
import time
from gtts import gTTS
from datetime import datetime
from flask import Flask, render_template, request, send_file, redirect, url_for, jsonify, session
from functools import wraps
from deep_translator import GoogleTranslator
from openai import OpenAI

app = Flask(__name__)
app.secret_key = "NGUYEN_PL"

USERS = {
    "admin": "123456",
    "user1": "123456",
    "user2": "123456"
}

def login_required(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        if "username" not in session:
            return redirect(url_for("login"))
        return func(*args, **kwargs)
    return wrapper


UPLOAD_FOLDER = "uploads"
OUTPUT_FOLDER = "outputs"
HISTORY_FILE = "history.json"

VIDEO_FOLDER = "videos"

os.makedirs(VIDEO_FOLDER, exist_ok=True)

os.makedirs(UPLOAD_FOLDER, exist_ok=True)
os.makedirs(OUTPUT_FOLDER, exist_ok=True)

jobs = {}

LANGUAGES = {
    "vi": "Tiếng Việt",
    "en": "Tiếng Anh",
    "ko": "Tiếng Hàn",
    "ja": "Tiếng Nhật"
}

GOOGLE_LANG_MAP = {
    "vi": "vi",
    "en": "en",
    "ko": "ko",
    "ja": "ja"
}


def load_history():
    if not os.path.exists(HISTORY_FILE):
        return []
    try:
        with open(HISTORY_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except:
        return []


def save_history(history):
    with open(HISTORY_FILE, "w", encoding="utf-8") as f:
        json.dump(history, f, ensure_ascii=False, indent=2)


def split_batches(items, batch_size):
    for i in range(0, len(items), batch_size):
        yield i, items[i:i + batch_size]


def translate_google_batch(texts, target_lang):
    results = []
    translator = GoogleTranslator(
        source="zh-CN",
        target=GOOGLE_LANG_MAP.get(target_lang, "vi")
    )

    for text in texts:
        if not text.strip():
            results.append(text)
            continue

        try:
            results.append(translator.translate(text))
            time.sleep(0.12)
        except Exception:
            results.append(text)

    return results


def translate_openai_batch(texts, target_lang, api_key):
    client = OpenAI(api_key=api_key, timeout=30)

    target_name = LANGUAGES.get(target_lang, "Tiếng Việt")

    numbered_text = "\n".join([
        f"{i + 1}. {text}" for i, text in enumerate(texts)
    ])

    prompt = f"""
Bạn là chuyên gia dịch phụ đề SRT từ tiếng Trung sang {target_name}.

Yêu cầu:
- Dịch tự nhiên, dễ hiểu.
- Giữ đúng nghĩa thoại.
- Không thêm giải thích.
- Không bỏ dòng.
- Không gộp dòng.
- Giữ nguyên số thứ tự đầu dòng.
- Mỗi dòng dịch tương ứng đúng với một dòng gốc.
- Chỉ trả về danh sách bản dịch.

Nội dung:
{numbered_text}
"""

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.2
    )

    raw = response.choices[0].message.content.strip().split("\n")

    translations = []
    for line in raw:
        line = line.strip()
        if not line:
            continue
        if ". " in line:
            translations.append(line.split(". ", 1)[1].strip())
        else:
            translations.append(line)

    while len(translations) < len(texts):
        translations.append(texts[len(translations)])

    return translations[:len(texts)]


def translate_batch_with_fallback(texts, mode, target_lang, api_key):
    if mode == "openai":
        try:
            return translate_openai_batch(texts, target_lang, api_key), "OpenAI"
        except Exception as e:
            print("OpenAI lỗi, chuyển sang Google:", str(e))
            return translate_google_batch(texts, target_lang), "Google Fallback"

    return translate_google_batch(texts, target_lang), "Google"


def translate_worker(
    job_id,
    input_path,
    output_path,
    original_name,
    output_name,
    mode,
    target_lang,
    api_key
):
    final_mode = "Google"

    try:
        try:
            subs = pysrt.open(input_path, encoding="utf-8")
        except:
            subs = pysrt.open(input_path, encoding="utf-8-sig")

        total = len(subs)

        jobs[job_id]["total"] = total
        jobs[job_id]["status"] = "processing"

        texts = [sub.text.replace("\n", " ").strip() for sub in subs]

        batch_size = 20 if mode == "openai" else 10

        for start_index, batch_texts in split_batches(texts, batch_size):
            translated_batch, used_mode = translate_batch_with_fallback(
                texts=batch_texts,
                mode=mode,
                target_lang=target_lang,
                api_key=api_key
            )

            final_mode = used_mode

            if used_mode == "Google Fallback":
                jobs[job_id]["note"] = "OpenAI lỗi hoặc hết quota, đã tự chuyển sang Google Translate."

            for offset, translated_text in enumerate(translated_batch):
                sub_index = start_index + offset
                subs[sub_index].text = translated_text

                jobs[job_id]["current"] = sub_index + 1
                jobs[job_id]["percent"] = int(((sub_index + 1) / total) * 100)

            time.sleep(0.15)

        subs.save(output_path, encoding="utf-8")

        jobs[job_id]["status"] = "done"
        jobs[job_id]["download"] = output_name

        history = load_history()
        history.insert(0, {
            "id": job_id,
            "file_name": original_name,
            "output_name": output_name,
            "line_count": total,
            "mode": final_mode,
            "target_lang": LANGUAGES.get(target_lang, target_lang),
            "status": "Hoàn thành",
            "time": datetime.now().strftime("%d/%m/%Y %H:%M")
        })

        save_history(history[:50])

    except Exception as e:
        jobs[job_id]["status"] = "error"
        jobs[job_id]["error"] = str(e)

        history = load_history()
        history.insert(0, {
            "id": job_id,
            "file_name": original_name,
            "output_name": "",
            "line_count": 0,
            "mode": final_mode,
            "target_lang": LANGUAGES.get(target_lang, target_lang),
            "status": "Thất bại",
            "time": datetime.now().strftime("%d/%m/%Y %H:%M")
        })

        save_history(history[:50])
        
        

@app.route("/login", methods=["GET", "POST"])
def login():
    error = ""

    if request.method == "POST":
        username = request.form.get("username", "").strip()
        password = request.form.get("password", "").strip()

        if username in USERS and USERS[username] == password:
            session["username"] = username
            return redirect(url_for("dashboard"))
        else:
            error = "Sai tài khoản hoặc mật khẩu"

    return render_template("index.html", page="login", error=error)


@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("login"))
@app.route("/")
@app.route("/dashboard")
@login_required
def dashboard():
    history = load_history()

    total_files = len(history)
    success_files = len([h for h in history if h["status"] == "Hoàn thành"])
    total_lines = sum(h.get("line_count", 0) for h in history)

    return render_template(
        "index.html",
        page="dashboard",
        history=history[:5],
        total_files=total_files,
        success_files=success_files,
        total_lines=total_lines
    )


@app.route("/translate", methods=["GET", "POST"])
@login_required
def translate_page():
    if request.method == "POST":
        file = request.files.get("file")
        mode = request.form.get("mode", "google")
        target_lang = request.form.get("target_lang", "vi")
        api_key = request.form.get("api_key", "").strip()

        if not file or file.filename == "":
            return redirect(url_for("translate_page"))

        if mode == "openai" and not api_key:
            mode = "google"

        job_id = str(uuid.uuid4())[:8]

        original_name = file.filename
        file_base, file_ext = os.path.splitext(original_name)

        input_name = f"{job_id}_{original_name}"
        output_name = f"{file_base}_{target_lang}{file_ext}"

        input_path = os.path.join(UPLOAD_FOLDER, input_name)
        output_path = os.path.join(OUTPUT_FOLDER, output_name)

        file.save(input_path)

        jobs[job_id] = {
            "status": "queued",
            "current": 0,
            "total": 0,
            "percent": 0,
            "download": "",
            "error": "",
            "note": ""
        }

        thread = threading.Thread(
            target=translate_worker,
            args=(
                job_id,
                input_path,
                output_path,
                original_name,
                output_name,
                mode,
                target_lang,
                api_key
            )
        )

        thread.start()

        return redirect(url_for("progress_page", job_id=job_id))

    return render_template("index.html", page="translate")


@app.route("/progress/<job_id>")
@login_required
def progress_page(job_id):
    return render_template("index.html", page="progress", job_id=job_id)


@app.route("/progress-data/<job_id>")
@login_required
def progress_data(job_id):
    return jsonify(jobs.get(job_id, {
        "status": "not_found",
        "current": 0,
        "total": 0,
        "percent": 0,
        "download": "",
        "error": "Không tìm thấy tác vụ",
        "note": ""
    }))


@app.route("/history")
@login_required
def history_page():
    history = load_history()
    return render_template("index.html", page="history", history=history)


@app.route("/settings")
@login_required
def settings_page():
    return render_template("index.html", page="settings")


@app.route("/audio")
@login_required
def audio_page():
    return render_template("index.html", page="audio")


@app.route("/render", methods=["GET", "POST"])
@login_required
def render_page():
    if request.method == "POST":
        video = request.files.get("video")
        subtitle = request.files.get("subtitle")

        if not video or not subtitle:
            return render_template(
                "index.html",
                page="render",
                error="Vui lòng chọn video và file phụ đề"
            )

        job_id = str(uuid.uuid4())[:8]

        video_name = f"{job_id}_{video.filename}"
        subtitle_name = f"{job_id}_{subtitle.filename}"
        output_name = f"render_{job_id}.mp4"

        video_path = os.path.join(VIDEO_FOLDER, video_name)
        subtitle_path = os.path.join(VIDEO_FOLDER, subtitle_name)
        output_path = os.path.join(OUTPUT_FOLDER, output_name)

        video.save(video_path)
        subtitle.save(subtitle_path)

        ffmpeg_path = imageio_ffmpeg.get_ffmpeg_exe()

        subtitle_path_fixed = subtitle_path.replace("\\", "/")

        cmd = [
            ffmpeg_path,
            "-y",
            "-i", video_path,
            "-vf", f"subtitles='{subtitle_path_fixed}'",
            "-c:a", "copy",
            output_path
        ]

        try:
            subprocess.run(cmd, check=True)

            return render_template(
                "index.html",
                page="render",
                output_file=output_name
            )

        except Exception as e:
            return render_template(
                "index.html",
                page="render",
                error=str(e)
            )

    return render_template("index.html", page="render")


@app.route("/download/<filename>")
@login_required
def download(filename):
    return send_file(
        os.path.join(OUTPUT_FOLDER, filename),
        as_attachment=True
    )


@app.route("/clear-history")
@login_required
def clear_history():
    save_history([])
    return redirect(url_for("history_page"))

# =====================
# TEXT TO SPEECH
# =====================

def text_to_speech(text, lang):
    filename = f"{uuid.uuid4()}.mp3"
    path = os.path.join(OUTPUT_FOLDER, filename)

    tts = gTTS(text=text, lang=lang)
    tts.save(path)

    return filename


@app.route("/tts", methods=["POST"])
@login_required
def tts():
    text = request.form.get("text")
    lang = request.form.get("lang", "vi")

    if not text:
        return jsonify({"error": "No text"})

    try:
        file = text_to_speech(text, lang)
        return jsonify({"file": file})
    except Exception as e:
        return jsonify({"error": str(e)})

if __name__ == "__main__":
    app.run(debug=True, threaded=True)
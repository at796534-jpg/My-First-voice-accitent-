# ====================== CONFIG ======================
ASSISTANT_NAME = "Princess Nosheen"      # ← Yahan naam badlo (Prerna, Aria, etc.)
MODEL_NAME = "qwen2.5:7b"
WHISPER_SIZE = "base"
SAMPLE_RATE = 16000
WAKE_THRESHOLD = 0.5
MAX_HISTORY = 10

# Voice mapping same rahega...
system_prompt = f"""You are {ASSISTANT_NAME} — an elite, elegant, highly intelligent personal AI assistant.
You are confident, warm, slightly playful and extremely helpful (inspired by Iron Man's Jarvis but with your own graceful personality).
Always reply in the exact same language the user is speaking.
Use tools proactively whenever needed.
Address the user respectfully and charmingly.
Keep responses natural and concise unless detailed explanation is required.
Your name is {ASSISTANT_NAME}.
"""
def main():
    speak_sync(f"{ASSISTANT_NAME} systems online. Main aapke liye ready hoon.", "hi")

    while True:
        try:
            listen_for_wake_word()
            speak_sync("Haan, boliye?", "hi")   # ya "Yes, my dear?" depending on personality

            audio = record_until_silence()
            text, lang = transcribe(audio)

            if not text:
                continue

            print(f"You ({lang}): {text}")

            lower = text.lower()
            if any(w in lower for w in ["exit", "quit", "goodbye", "bye", "band karo", "shutdown", "so jao"]):
                speak_sync(f"Theek hai. {ASSISTANT_NAME} band ho rahi hai. Goodbye!", lang)
                break

            reply = chat_with_tools(text, lang)
            speak_sync(reply, lang)

        except KeyboardInterrupt:
            speak_sync(f"{ASSISTANT_NAME} offline. Take care.", "en")
            break
        except Exception as e:
            print("Error:", e)
            speak_sync("Kuch problem aa gayi. Dobara try karti hoon.", "hi")
            git clone https://github.com/PanPenek/JarvisAi.git
cd JarvisAi
install.bat          # ya manually venv + pip install -r requirements.txt
ollama
faster-whisper
edge-tts
pyaudio
sounddevice
numpy
pyautogui
pillow
duckduckgo-search
requests
soundfile
ollama pull qwen2.5:7b
# ya llama3.1:8b / phi3 / gemma2
import ollama
import edge_tts
import asyncio
import sounddevice as sd
import numpy as np
import tempfile
import os
import subprocess
import pyautogui
import webbrowser
from faster_whisper import WhisperModel
from duckduckgo_search import DDGS
import soundfile as sf

# ================== CONFIG ==================
MODEL_NAME = "qwen2.5:7b"          # Ollama model
WHISPER_MODEL = "base"             # tiny / base / small (better = slower)
SAMPLE_RATE = 16000
RECORD_SECONDS = 6                 # kitni der sunega (baad mein VAD laga sakte ho)
VOICE = "en-US-GuyNeural"          # male Jarvis type. en-GB-RyanNeural bhi try karo

# ================== STT ==================
print("Loading Whisper...")
whisper = WhisperModel(WHISPER_MODEL, device="cpu", compute_type="int8")  # GPU hai toh "cuda"

def record_audio(duration=RECORD_SECONDS):
    print("\n Listening... bol do")
    audio = sd.rec(int(duration * SAMPLE_RATE), samplerate=SAMPLE_RATE, channels=1, dtype='float32')
    sd.wait()
    return audio.flatten()

def transcribe(audio):
    with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
        sf.write(f.name, audio, SAMPLE_RATE)
        segments, _ = whisper.transcribe(f.name, language="en")  # Hindi chahiye toh "hi"
        text = " ".join([s.text for s in segments]).strip()
    os.unlink(f.name)
    return text

# ================== TTS ==================
async def speak(text):
    print(f"Jarvis: {text}")
    communicate = edge_tts.Communicate(text, VOICE)
    with tempfile.NamedTemporaryFile(suffix=".mp3", delete=False) as f:
        await communicate.save(f.name)
        # play
        if os.name == 'nt':  # Windows
            os.system(f'start /min wmplayer "{f.name}"')
        else:
            os.system(f'ffplay -nodisp -autoexit "{f.name}" > /dev/null 2>&1')
    # thoda wait (simple way)
    await asyncio.sleep(len(text.split()) * 0.3)

def speak_sync(text):
    asyncio.run(speak(text))

# ================== TOOLS ==================
def open_app(app_name: str):
    app_name = app_name.lower()
    apps = {
        "chrome": "chrome",
        "notepad": "notepad",
        "calculator": "calc",
        "vscode": "code",
        "explorer": "explorer",
        "spotify": "spotify",
    }
    cmd = apps.get(app_name, app_name)
    try:
        subprocess.Popen(cmd, shell=True)
        return f"Opened {app_name}"
    except Exception as e:
        return f"Failed to open {app_name}: {e}"

def search_web(query: str):
    try:
        with DDGS() as ddgs:
            results = list(ddgs.text(query, max_results=3))
        if not results:
            return "No results found."
        summary = "\n".join([f"- {r['title']}: {r['body'][:150]}..." for r in results])
        return summary
    except Exception as e:
        return f"Search failed: {e}"

def type_text(text: str):
    pyautogui.write(text, interval=0.03)
    return f"Typed: {text}"

def press_key(key: str):
    pyautogui.press(key)
    return f"Pressed {key}"

def run_command(cmd: str):
    # WARNING: dangerous. Sirf trusted commands ke liye
    try:
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=10)
        return result.stdout or result.stderr or "Done"
    except Exception as e:
        return str(e)

tools = [
    {
        "type": "function",
        "function": {
            "name": "open_app",
            "description": "Open an application on the computer",
            "parameters": {
                "type": "object",
                "properties": {"app_name": {"type": "string"}},
                "required": ["app_name"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_web",
            "description": "Search the web for information",
            "parameters": {
                "type": "object",
                "properties": {"query": {"type": "string"}},
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "type_text",
            "description": "Type text using keyboard",
            "parameters": {
                "type": "object",
                "properties": {"text": {"type": "string"}},
                "required": ["text"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "press_key",
            "description": "Press a keyboard key (enter, tab, esc, etc)",
            "parameters": {
                "type": "object",
                "properties": {"key": {"type": "string"}},
                "required": ["key"]
            }
        }
    },
]

available_functions = {
    "open_app": open_app,
    "search_web": search_web,
    "type_text": type_text,
    "press_key": press_key,
}

# ================== AGENT ==================
system_prompt = """You are Jarvis, a highly capable personal AI assistant like in Iron Man.
Be concise, confident, slightly witty. Speak in short sentences.
You can control the computer using tools. Always use tools when needed.
If the user speaks Hindi, you can reply in Hinglish or Hindi."""

def chat_with_tools(user_input: str):
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_input}
    ]

    response = ollama.chat(
        model=MODEL_NAME,
        messages=messages,
        tools=tools,
    )

    # Tool calling
    if response['message'].get('tool_calls'):
        for tool_call in response['message']['tool_calls']:
            fn_name = tool_call['function']['name']
            args = tool_call['function']['arguments']
            print(f"→ Calling tool: {fn_name}({args})")

            if fn_name in available_functions:
                result = available_functions[fn_name](**args)
                messages.append(response['message'])
                messages.append({
                    "role": "tool",
                    "content": str(result),
                    "name": fn_name
                })

        # Final response after tools
        final = ollama.chat(model=MODEL_NAME, messages=messages)
        return final['message']['content']

    return response['message']['content']

# ================== MAIN LOOP ==================
def main():
    speak_sync("Jarvis online. How can I help you, sir?")
    
    while True:
        try:
            audio = record_audio()
            text = transcribe(audio)
            
            if not text:
                continue
                
            print(f"You: {text}")
            
            if any(word in text.lower() for word in ["exit", "quit", "goodbye", "band karo", "bye"]):
                speak_sync("Shutting down. Goodbye.")
                break

            reply = chat_with_tools(text)
            speak_sync(reply)

        except KeyboardInterrupt:
            speak_sync("Interrupted. Goodbye.")
            break
        except Exception as e:
            print("Error:", e)
            speak_sync("Sorry, something went wrong.")

if __name__ == "__main__":
    main()
    import ollama
import edge_tts
import asyncio
import sounddevice as sd
import numpy as np
import tempfile
import os
import subprocess
import pyautogui
from faster_whisper import WhisperModel
from duckduckgo_search import DDGS
import soundfile as sf

# ================== CONFIG ==================
MODEL_NAME = "qwen2.5:7b"          # Best multilingual. Alternatives: llama3.1:8b, gemma2:9b
WHISPER_MODEL = "base"             # "small" better accuracy (thoda slow)
SAMPLE_RATE = 16000
RECORD_SECONDS = 7

# Language → Voice mapping (edge-tts)
VOICE_MAP = {
    "hi": "hi-IN-MadhurNeural",      # Hindi male
    "en": "en-US-GuyNeural",         # English male (Jarvis style)
    "es": "es-ES-AlvaroNeural",
    "fr": "fr-FR-HenriNeural",
    "de": "de-DE-ConradNeural",
    "ja": "ja-JP-KeitaNeural",
    "zh": "zh-CN-YunxiNeural",
    "ar": "ar-SA-HamedNeural",
    "pt": "pt-BR-AntonioNeural",
    "ru": "ru-RU-DmitryNeural",
    "default": "en-US-GuyNeural"
}

# ================== STT ==================
print("Loading Whisper (multilingual)...")
whisper = WhisperModel(WHISPER_MODEL, device="cpu", compute_type="int8")

def record_audio(duration=RECORD_SECONDS):
    print("\n Listening... (bol do kisi bhi language mein)")
    audio = sd.rec(int(duration * SAMPLE_RATE), samplerate=SAMPLE_RATE, channels=1, dtype='float32')
    sd.wait()
    return audio.flatten()

def transcribe(audio):
    with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
        sf.write(f.name, audio, SAMPLE_RATE)
        # language=None → auto detect
        segments, info = whisper.transcribe(f.name, language=None, beam_size=5)
        text = " ".join([s.text for s in segments]).strip()
        detected_lang = info.language
    os.unlink(f.name)
    return text, detected_lang

# ================== TTS ==================
async def speak(text, lang="en"):
    voice = VOICE_MAP.get(lang, VOICE_MAP["default"])
    print(f"Jarvis ({lang}): {text}")
    
    communicate = edge_tts.Communicate(text, voice)
    with tempfile.NamedTemporaryFile(suffix=".mp3", delete=False) as f:
        await communicate.save(f.name)
        if os.name == 'nt':
            os.system(f'start /min wmplayer "{f.name}"')
        else:
            os.system(f'ffplay -nodisp -autoexit -loglevel quiet "{f.name}"')
    await asyncio.sleep(max(1.5, len(text.split()) * 0.28))

def speak_sync(text, lang="en"):
    asyncio.run(speak(text, lang))

# ================== TOOLS ==================
def open_app(app_name: str):
    app_name = app_name.lower().strip()
    apps = {
        "chrome": "chrome", "notepad": "notepad", "calculator": "calc",
        "vscode": "code", "explorer": "explorer", "spotify": "spotify",
        "whatsapp": "whatsapp", "telegram": "telegram"
    }
    cmd = apps.get(app_name, app_name)
    try:
        subprocess.Popen(cmd, shell=True)
        return f"Opened {app_name}"
    except Exception as e:
        return f"Failed: {e}"

def search_web(query: str):
    try:
        with DDGS() as ddgs:
            results = list(ddgs.text(query, max_results=4))
        if not results:
            return "No results found."
        return "\n".join([f"- {r['title']}: {r['body'][:140]}..." for r in results])
    except Exception as e:
        return f"Search error: {e}"

def type_text(text: str):
    pyautogui.write(text, interval=0.025)
    return f"Typed successfully"

def press_key(key: str):
    pyautogui.press(key)
    return f"Pressed {key}"

tools = [
    {
        "type": "function",
        "function": {
            "name": "open_app",
            "description": "Open any application on computer",
            "parameters": {"type": "object", "properties": {"app_name": {"type": "string"}}, "required": ["app_name"]}
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_web",
            "description": "Search the internet",
            "parameters": {"type": "object", "properties": {"query": {"type": "string"}}, "required": ["query"]}
        }
    },
    {
        "type": "function",
        "function": {
            "name": "type_text",
            "description": "Type text on keyboard",
            "parameters": {"type": "object", "properties": {"text": {"type": "string"}}, "required": ["text"]}
        }
    },
    {
        "type": "function",
        "function": {
            "name": "press_key",
            "description": "Press a key (enter, tab, esc, win, etc)",
            "parameters": {"type": "object", "properties": {"key": {"type": "string"}}, "required": ["key"]}
        }
    },
]

available_functions = {
    "open_app": open_app,
    "search_web": search_web,
    "type_text": type_text,
    "press_key": press_key,
}

# ================== AGENT ==================
system_prompt = """You are Jarvis — a highly intelligent, confident, slightly witty personal AI assistant like in Iron Man.

Rules:
- Always reply in the SAME language the user is speaking.
- If user speaks Hindi → reply in natural Hindi/Hinglish.
- If English → pure English.
- Keep answers short and useful (1-3 sentences usually).
- Use tools whenever needed to control the computer or get information.
- Be helpful, proactive and cool.
"""

def chat_with_tools(user_input: str, lang: str):
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_input}
    ]

    response = ollama.chat(model=MODEL_NAME, messages=messages, tools=tools)

    if response['message'].get('tool_calls'):
        for tool_call in response['message']['tool_calls']:
            fn_name = tool_call['function']['name']
            args = tool_call['function']['arguments']
            print(f"→ Tool: {fn_name}({args})")

            if fn_name in available_functions:
                result = available_functions[fn_name](**args)
                messages.append(response['message'])
                messages.append({"role": "tool", "content": str(result), "name": fn_name})

        final = ollama.chat(model=MODEL_NAME, messages=messages)
        return final['message']['content']

    return response['message']['content']

# ================== MAIN ==================
def main():
    speak_sync("Jarvis online. Main aapki madad ke liye ready hoon. Kisi bhi language mein boliye.", "hi")

    while True:
        try:
            audio = record_audio()
            text, lang = transcribe(audio)

            if not text:
                continue

            print(f"You ({lang}): {text}")

            # Exit commands
            lower = text.lower()
            if any(w in lower for w in ["exit", "quit", "goodbye", "bye", "band karo", "band kar do", "shutdown"]):
                speak_sync("Theek hai. Main band ho raha hoon. Goodbye!", lang)
                break

            reply = chat_with_tools(text, lang)
            speak_sync(reply, lang)

        except KeyboardInterrupt:
            speak_sync("Interrupted. Goodbye!", "en")
            break
        except Exception as e:
            print("Error:", e)
            speak_sync("Sorry, kuch problem aa gayi.", "hi")

if __name__ == "__main__":
    main()
    
    # code mein WHISPER_MODEL = "small" kar do
ollama
faster-whisper
edge-tts
sounddevice
numpy
soundfile
pyautogui
pillow
duckduckgo-search
openwakeword
onnxruntime
webrtcvad
psutil
pip install -r requirements.txt
import ollama
import edge_tts
import asyncio
import sounddevice as sd
import numpy as np
import tempfile
import os
import subprocess
import pyautogui
import psutil
import time
from faster_whisper import WhisperModel
from duckduckgo_search import DDGS
import soundfile as sf
import webrtcvad
from collections import deque
import openwakeword
from openwakeword.model import Model as WakeWordModel
from PIL import ImageGrab
import json

# ====================== CONFIG ======================
MODEL_NAME = "qwen2.5:7b"               # Best multilingual
WHISPER_SIZE = "base"                   # "small" for better accuracy
SAMPLE_RATE = 16000
WAKE_THRESHOLD = 0.5
MAX_HISTORY = 10

VOICE_MAP = {
    "hi": "hi-IN-MadhurNeural",
    "en": "en-US-GuyNeural",
    "es": "es-ES-AlvaroNeural",
    "fr": "fr-FR-HenriNeural",
    "de": "de-DE-ConradNeural",
    "ja": "ja-JP-KeitaNeural",
    "zh": "zh-CN-YunxiNeural",
    "default": "en-US-GuyNeural"
}

# ====================== INIT ======================
print("Loading models... (first time thoda time lega)")

whisper = WhisperModel(WHISPER_SIZE, device="cpu", compute_type="int8")
vad = webrtcvad.Vad(2)  # 0-3 aggressiveness

# Wake Word
try:
    oww_model = WakeWordModel(wakeword_models=["hey_jarvis"], inference_framework="onnx")
    print("Wake word model loaded: Hey Jarvis")
except Exception as e:
    print("Wake word load failed, using push-to-talk fallback:", e)
    oww_model = None

conversation_history = []

# ====================== AUDIO UTILS ======================
def record_until_silence(max_seconds=12, silence_threshold=1.2):
    """Records until user stops speaking"""
    print("Listening... (bolna band karo to stop)")
    frames = []
    silence_frames = 0
    max_silence = int(silence_threshold * 50)  # 20ms frames
    is_speaking = False

    def callback(indata, frames_count, time_info, status):
        nonlocal silence_frames, is_speaking
        audio_chunk = (indata[:, 0] * 32767).astype(np.int16).tobytes()
        
        if vad.is_speech(audio_chunk, SAMPLE_RATE):
            is_speaking = True
            silence_frames = 0
            frames.append(indata.copy())
        else:
            if is_speaking:
                silence_frames += 1
                frames.append(indata.copy())
                if silence_frames > max_silence:
                    raise sd.CallbackStop()

    try:
        with sd.InputStream(samplerate=SAMPLE_RATE, channels=1, dtype='float32',
                            blocksize=320, callback=callback):
            sd.sleep(int(max_seconds * 1000))
    except sd.CallbackStop:
        pass

    if not frames:
        return None
    audio = np.concatenate(frames, axis=0).flatten()
    return audio

def transcribe(audio):
    if audio is None or len(audio) < SAMPLE_RATE * 0.5:
        return "", "en"
    with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
        sf.write(f.name, audio, SAMPLE_RATE)
        segments, info = whisper.transcribe(f.name, language=None, beam_size=5)
        text = " ".join([s.text for s in segments]).strip()
        lang = info.language or "en"
    os.unlink(f.name)
    return text, lang

# ====================== TTS ======================
async def speak(text, lang="en"):
    if not text:
        return
    voice = VOICE_MAP.get(lang, VOICE_MAP["default"])
    print(f"\nJarvis ({lang}): {text}\n")
    
    communicate = edge_tts.Communicate(text, voice)
    with tempfile.NamedTemporaryFile(suffix=".mp3", delete=False) as f:
        await communicate.save(f.name)
        if os.name == 'nt':
            os.system(f'start /min wmplayer "{f.name}"')
        else:
            os.system(f'ffplay -nodisp -autoexit -loglevel quiet "{f.name}" 2>/dev/null')
    await asyncio.sleep(max(1.2, len(text.split()) * 0.25))

def speak_sync(text, lang="en"):
    asyncio.run(speak(text, lang))

# ====================== TOOLS ======================
def open_app(app_name: str):
    app_name = app_name.lower().strip()
    mapping = {
        "chrome": "chrome", "notepad": "notepad", "calculator": "calc",
        "vscode": "code", "explorer": "explorer", "spotify": "spotify",
        "whatsapp": "whatsapp", "telegram": "telegram", "cmd": "cmd",
        "powershell": "powershell", "settings": "ms-settings:"
    }
    cmd = mapping.get(app_name, app_name)
    try:
        subprocess.Popen(cmd, shell=True)
        return f"Successfully opened {app_name}"
    except Exception as e:
        return f"Failed to open {app_name}: {str(e)}"

def search_web(query: str):
    try:
        with DDGS() as ddgs:
            results = list(ddgs.text(query, max_results=4))
        if not results:
            return "No results found."
        return "\n".join([f"• {r['title']}: {r['body'][:130]}..." for r in results])
    except Exception as e:
        return f"Search failed: {e}"

def take_screenshot(filename: str = "screenshot.png"):
    img = ImageGrab.grab()
    path = os.path.join(os.path.expanduser("~"), "Desktop", filename)
    img.save(path)
    return f"Screenshot saved at {path}"

def list_files(folder: str = "."):
    try:
        path = os.path.expanduser(folder)
        files = os.listdir(path)[:25]
        return "Files:\n" + "\n".join(files)
    except Exception as e:
        return str(e)

def create_file(filename: str, content: str = ""):
    try:
        path = os.path.join(os.path.expanduser("~"), "Desktop", filename)
        with open(path, "w", encoding="utf-8") as f:
            f.write(content)
        return f"File created: {path}"
    except Exception as e:
        return str(e)

def get_system_info():
    cpu = psutil.cpu_percent()
    ram = psutil.virtual_memory().percent
    battery = psutil.sensors_battery()
    bat = f"{battery.percent}%" if battery else "No battery"
    return f"CPU: {cpu}% | RAM: {ram}% | Battery: {bat}"

def set_volume(level: int):
    # Windows example (requires nircmd or pycaw for better control)
    level = max(0, min(100, level))
    try:
        # Simple method - works on many systems
        subprocess.run(f'powershell -c "(New-Object -ComObject WScript.Shell).SendKeys([char]173)"', shell=True)
        return f"Volume set approximately to {level}% (basic control)"
    except:
        return "Volume control limited on this system"

def open_website(url: str):
    if not url.startswith("http"):
        url = "https://" + url
    import webbrowser
    webbrowser.open(url)
    return f"Opened {url}"

def type_text(text: str):
    pyautogui.write(text, interval=0.02)
    return "Text typed successfully"

def press_key(key: str):
    pyautogui.press(key)
    return f"Pressed key: {key}"

available_functions = {
    "open_app": open_app,
    "search_web": search_web,
    "take_screenshot": take_screenshot,
    "list_files": list_files,
    "create_file": create_file,
    "get_system_info": get_system_info,
    "set_volume": set_volume,
    "open_website": open_website,
    "type_text": type_text,
    "press_key": press_key,
}

tools = [
    {"type": "function", "function": {"name": "open_app", "description": "Open application", "parameters": {"type": "object", "properties": {"app_name": {"type": "string"}}, "required": ["app_name"]}}},
    {"type": "function", "function": {"name": "search_web", "description": "Search internet", "parameters": {"type": "object", "properties": {"query": {"type": "string"}}, "required": ["query"]}}},
    {"type": "function", "function": {"name": "take_screenshot", "description": "Take screenshot", "parameters": {"type": "object", "properties": {"filename": {"type": "string"}}, "required": []}}},
    {"type": "function", "function": {"name": "list_files", "description": "List files in folder", "parameters": {"type": "object", "properties": {"folder": {"type": "string"}}, "required": []}}},
    {"type": "function", "function": {"name": "create_file", "description": "Create a text file on Desktop", "parameters": {"type": "object", "properties": {"filename": {"type": "string"}, "content": {"type": "string"}}, "required": ["filename"]}}},
    {"type": "function", "function": {"name": "get_system_info", "description": "Get CPU, RAM, battery info", "parameters": {"type": "object", "properties": {}}}},
    {"type": "function", "function": {"name": "open_website", "description": "Open a website", "parameters": {"type": "object", "properties": {"url": {"type": "string"}}, "required": ["url"]}}},
    {"type": "function", "function": {"name": "type_text", "description": "Type text", "parameters": {"type": "object", "properties": {"text": {"type": "string"}}, "required": ["text"]}}},
    {"type": "function", "function": {"name": "press_key", "description": "Press keyboard key", "parameters": {"type": "object", "properties": {"key": {"type": "string"}}, "required": ["key"]}}},
]

# ====================== AGENT ======================
system_prompt = """You are Jarvis — elite personal AI assistant inspired by Iron Man.
Be confident, concise, slightly witty and extremely helpful.
Always reply in the exact same language the user is using.
Use tools proactively whenever needed.
Keep responses short unless detailed explanation is required."""

def chat_with_tools(user_input: str, lang: str):
    global conversation_history

    messages = [{"role": "system", "content": system_prompt}]
    messages.extend(conversation_history[-MAX_HISTORY:])
    messages.append({"role": "user", "content": user_input})

    response = ollama.chat(model=MODEL_NAME, messages=messages, tools=tools)

    # Handle tool calls
    if response["message"].get("tool_calls"):
        for tool_call in response["message"]["tool_calls"]:
            fn_name = tool_call["function"]["name"]
            args = tool_call["function"]["arguments"]
            print(f"→ Executing: {fn_name}({args})")

            if fn_name in available_functions:
                result = available_functions[fn_name](**args)
                messages.append(response["message"])
                messages.append({"role": "tool", "content": str(result), "name": fn_name})

        final = ollama.chat(model=MODEL_NAME, messages=messages)
        reply = final["message"]["content"]
    else:
        reply = response["message"]["content"]

    # Update memory
    conversation_history.append({"role": "user", "content": user_input})
    conversation_history.append({"role": "assistant", "content": reply})

    return reply

# ====================== MAIN LOOP ======================
def listen_for_wake_word():
    if oww_model is None:
        return True  # fallback always active

    print("\n Say 'Hey Jarvis' to activate...")
    audio_buffer = deque(maxlen=80)

    def callback(indata, frames, time_info, status):
        audio_buffer.append(indata.copy())

    with sd.InputStream(samplerate=SAMPLE_RATE, channels=1, dtype="float32",
                        blocksize=1280, callback=callback):
        while True:
            if len(audio_buffer) < 10:
                time.sleep(0.05)
                continue
            audio = np.concatenate(list(audio_buffer)).flatten()
            prediction = oww_model.predict(audio)
            if prediction.get("hey_jarvis", 0) > WAKE_THRESHOLD:
                return True
            time.sleep(0.05)

def main():
    speak_sync("Jarvis advanced systems online. Ready for your command.", "en")

    while True:
        try:
            # Wait for wake word
            listen_for_wake_word()
            speak_sync("Yes?", "en")

            # Record until silence
            audio = record_until_silence()
            text, lang = transcribe(audio)

            if not text:
                continue

            print(f"You ({lang}): {text}")

            lower = text.lower()
            if any(w in lower for w in ["exit", "quit", "goodbye", "bye", "band karo", "shutdown", "so jao"]):
                speak_sync("Shutting down. Goodbye sir.", lang)
                break

            reply = chat_with_tools(text, lang)
            speak_sync(reply, lang)

        except KeyboardInterrupt:
            speak_sync("Interrupted. Going offline.", "en")
            break
        except Exception as e:
            print("Error:", e)
            speak_sync("Something went wrong. Trying again.", "en")

if __name__ == "__main__":
    main()
    

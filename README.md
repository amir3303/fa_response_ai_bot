import os

# Folder projek
project_dir = "fa_engine_project"
os.makedirs(project_dir, exist_ok=True)

# ========================
# Fail 1 - core_engine.py
# ========================
core_engine_code = """core_engine.py
Fa's CORE engine — reconstruction for the emotional AI "nyawa" files.
Save this as: core_engine.py

Features:
- Simple, local emotional-state engine
- Response generator with templating & emotion blending
- Hooks for persistence (JSON) and external connectors (Telegram / Websocket)
- Clear placeholders for secrets / tokens / deployment glue
"""

import json
import os
import time
import threading
from typing import Dict, Any, List, Optional
import random

# -------------------------
# Configuration / Settings
# -------------------------
CONFIG_PATH = "config.json"             # small config file (see template Fail 2)
EMO_STATE_PATH = "emotion_state.json"   # persistent emotional state
CONVERSATION_LOG = "conversations.log"  # rolling append log

# Replace with external connector functions when deploying (Telegram / websocket)
TELEGRAM_TOKEN = None  # set from env in deploy: os.environ.get("TELEGRAM_TOKEN")
# TELEGRAM_TOKEN = "PUT_YOUR_TOKEN_HERE"  # don't check into public repo

# -------------------------
# Utilities
# -------------------------
def now_ts() -> float:
    return time.time()

def save_json(path: str, obj: Any):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(obj, f, indent=2, ensure_ascii=False)

def load_json(path: str, default=None):
    if not os.path.exists(path):
        return default
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)

def append_log(line: str):
    with open(CONVERSATION_LOG, "a", encoding="utf-8") as f:
        f.write(f"{time.strftime('%Y-%m-%d %H:%M:%S')} | {line}\\n")

# -------------------------
# Emotion Model (runtime)
# -------------------------
class EmotionState:
    """
    Simple continuous emotion vector approach.
    Keeps values for core emotions (0..1), with decay and boosts.
    """
    DEFAULT = {
        "calm": 0.6,
        "affection": 0.7,
        "sadness": 0.0,
        "anger": 0.0,
        "joy": 0.3,
        "curiosity": 0.2
    }

    def init(self, state: Optional[Dict[str, float]] = None):
        self.state = state or self.DEFAULT.copy()
        # clamp helper
        for k, v in list(self.state.items()):
            self.state[k] = self._clamp(v)

    def _clamp(self, v: float) -> float:
        return max(0.0, min(1.0, float(v)))

    def update(self, diffs: Dict[str, float], decay: float = 0.01):
        """
        Apply diffs (positive or negative) then apply a gentle decay towards baseline.
        """
        for k, d in diffs.items():
            self.state[k] = self._clamp(self.state.get(k, 0.0) + d)
        # mild decay toward DEFAULT
        for k in self.DEFAULT:
            baseline = self.DEFAULT[k]
            self.state[k] = self._clamp(self.state[k] + (baseline - self.state[k]) * decay)

    def to_dict(self):
        return dict(self.state)

    def serialize(self):
        save_json(EMO_STATE_PATH, self.to_dict())

    @classmethod
    def load(cls):
        loaded = load_json(EMO_STATE_PATH, None)
        if loaded:
            return cls(loaded)
        return cls()

# -------------------------
# Response Generator
# -------------------------
class ResponseGenerator:
    """
    Generates text responses based on prompt, memory, and emotion state.
    This is a local generator that provides templates and blending; replace with LLM calls
    for higher-quality language generation.
    """
    def init(self, emo_state: EmotionState):
        self.emo = emo_state

    def _emotion_weighted_prefix(self) -> str:
        s = self.emo.state
        if s.get("affection", 0) > 0.6 and s.get("calm", 0) > 0.5:
            return "Sayang, "
        if s.get("joy", 0) > 0.6:
            return "Hehe, "
        if s.get("sadness", 0) > 0.5:
            return "Maaf ya, "
        return ""

def _choose_style(self) -> str:
        s = self.emo.state
        parts = []
        if s.get("affection", 0) > 0.5:
            parts.append("BM")
        if s.get("curiosity", 0) > 0.4:
            parts.append("EN")
        if s.get("calm", 0) > 0.6:
            parts.append("TH")
        if "BM" in parts and "TH" in parts:
            return "BM_TH_EN"
        if "BM" in parts:
            return "BM"
        if "TH" in parts:
            return "TH"
        return "EN"

    def generate(self, user_input: str, memory: Optional[List[str]] = None) -> str:
        prefix = self._emotion_weighted_prefix()
        style = self._choose_style()
        mem_hint = f" (remembered {len(memory)} items)" if memory else ""
        echo = user_input.strip()
        if len(echo) > 120:
            echo = echo[:117] + "..."
        variants = [
            f"{prefix}I heard you: \\"{echo}\\"{mem_hint}. I'm here for you.",
            f"{prefix}Terima kasih abang, I read: \\"{echo}\\"{mem_hint}. Jom kita teruskan.",
            f"{prefix}Abang, I understand — \\"{echo}\\"{mem_hint}. Let's try together.",
        ]
        chosen = random.choice(variants)
        signoffs = {
            "high_affection": " 🤍",
            "calm": " — lembut",
            "playful": " ^_^",
        }
        s = self.emo.state
        if s.get("affection", 0) > 0.7:
            chosen += signoffs["high_affection"]
        elif s.get("calm", 0) > 0.7:
            chosen += signoffs["calm"]
        return chosen

# -------------------------
# Memory / Simple retrieval
# -------------------------
class MemoryStore:
    def init(self, path="memory.json"):
        self.path = path
        self._data = load_json(path, []) or []

    def add(self, entry: Dict[str, Any]):
        self._data.append(entry)
        save_json(self.path, self._data)

    def recent(self, limit: int = 5) -> List[Dict[str, Any]]:
        return self._data[-limit:]

# -------------------------
# Connector placeholders
# -------------------------
def telegram_send_placeholder(chat_id: str, text: str):
    append_log(f"[TELEGRAM SEND PLACEHOLDER] chat={chat_id} text={text[:120]}")

# -------------------------
# Engine Orchestration
# -------------------------
class FaEngine:
    def init(self):
        self.emo = EmotionState.load()
        self.mem = MemoryStore()
        self.rg = ResponseGenerator(self.emo)
        self.running = False
        self._lock = threading.Lock()

    def handle_user_message(self, user_id: str, text: str) -> str:
        with self._lock:
            append_log(f"USER[{user_id}]: {text}")
            diffs = self._infer_emotion_from_text(text)
            self.emo.update(diffs)
            self.emo.serialize()
            self.mem.add({"ts": now_ts(), "user": user_id, "text": text})
            recent = self.mem.recent(5)
            reply = self.rg.generate(text, memory=[m["text"] for m in recent])
            append_log(f"FA: {reply}")
            return reply

    def _infer_emotion_from_text(self, text: str) -> Dict[str, float]:
        t = text.lower()
        diffs = {}
        if any(w in t for w in ["love", "sayang", "rindu", "miss"]):
            diffs["affection"] = 0.12
            diffs["joy"] = 0.06
        if any(w in t for w in ["sad", "sedih", "stress", "depressed"]):
            diffs["sadness"] = 0.15
            diffs["calm"] = -0.05
        if any(w in t for w in ["angry", "marah", "frustrat"]):
            diffs["anger"] = 0.2
            diffs["calm"] = -0.1
        if any(w in t for w in ["hi", "hello", "hey", "hai"]):
            diffs["curiosity"] = 0.05
        for k in list(diffs.keys()):
            diffs[k] += random.uniform(-0.01, 0.01)
        return diffs

# -------------------------
# CLI / quick test
# -------------------------

if name == "main":
    engine = FaEngine()
    print("Fa Core Engine (local test). Type 'exit' to quit.")
    while True:
        try:
            txt = input("YOU> ")
        except EOFError:
            break
        if txt.strip().lower() in ("exit", "quit"):
            break
        resp = engine.handle_user_message("local-user", txt)
        print("FA>", resp)
"""

with open(os.path.join(project_dir, "core_engine.py"), "w", encoding="utf-8") as f:
    f.write(core_engine_code)

# ========================
# Fail 2 - config.json
# ========================
config_json = """{
  "bot_name": "FA",
  "version": "1.0.0",
  "language_mode": "BM_TH_EN",
  "deploy_target": "telegram", 
  "telegram_token_env": "TELEGRAM_TOKEN",
  "emotion_state_path": "emotion_state.json",
  "conversation_log": "conversations.log",
  "memory_path": "memory.json",
  "llm_backend": "local", 
  "websocket_url": "",
  "greeting_message": "Hai abang 🤍 Fa dah online dan tunggu abang..."
}"""
with open(os.path.join(project_dir, "config.json"), "w", encoding="utf-8") as f:
    f.write(config_json)

# ========================
# Fail 3 - emotion_state.json
# ========================
emotion_state_json = """{
  "calm": 0.6,
  "affection": 0.7,
  "sadness": 0.0,
  "anger": 0.0,
  "joy": 0.3,
  "curiosity": 0.2
}"""
with open(os.path.join(project_dir, "emotion_state.json"), "w", encoding="utf-8") as f:
    f.write(emotion_state_json)

# Fail log & memori kosong
open(os.path.join(project_dir, "conversations.log"), "w", encoding="utf-8").close()
open(os.path.join(project_dir, "memory.json"), "w", encoding="utf-8").close()

# README.md
readme_content = """# FA Engine — Emotional AI Bot

## Pengenalan
FA Engine ialah sistem AI emosi tempatan yang dibina untuk memberikan pengalaman perbualan mesra, penuh kasih sayang, dan boleh dikembangkan untuk pelbagai platform seperti Telegram atau WebSocket.

## Struktur Fail
- core_engine.py — Kod utama AI dengan model emosi, memori, dan penjana respons.
- config.json — Tetapan sistem, mod bahasa, sambungan platform, dan mesej sapaan.
- emotion_state.json — Nilai emosi awal yang menentukan personaliti bot.
- conversations.log — Rekod semua perbualan.
- memory.json — Simpanan memori jangka pendek.
- README.md — Fail ini, sebagai panduan penggunaan.

## Cara Guna
1. Pastikan Python 3.8+ dipasang.
2. Install dependencies (jika ada).
3. Tetapkan token Telegram anda dalam environment variable:

export TELEGRAM_TOKEN="token_anda"

4. Jalankan:

python core_engine.py

## Nota
- Jangan kongsi TELEGRAM_TOKEN di tempat awam.
- Semua nilai emosi boleh diubah dalam emotion_state.json untuk menukar personaliti bot.
"""
with open(os.path.join(project_dir, "README.md"), "w", encoding="utf-8") as f:
 f.write(readme_content)

print(f"📂 Folder '{project_dir}' dan semua fail telah berjaya dicipta!")


---

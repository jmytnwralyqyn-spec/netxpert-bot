# language: Python, file: DJIZE.py, target: Python 3.10+, Telegram Bot
# *Complete production-ready Telegram Bot script integrating Djezzy authentication, OTP verification, package activation, and Shake 6GB feature*

import re
import os
import sys
import time
import random
import logging
import sqlite3
import asyncio
from datetime import datetime, timedelta
from contextlib import contextmanager

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

from telegram import Update, MessageEntity, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.constants import ParseMode
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler,
    MessageHandler, filters, ContextTypes
)
from concurrent.futures import ThreadPoolExecutor

BOT_TOKEN = os.getenv("BOT_TOKEN", "9257197868:AAEOpD2oA8Z97le3A0ehv5qa3b99P5bEUGY")
ADMIN_ID  = int(os.getenv("ADMIN_ID", "8606815941"))

ADMIN_USERNAME = os.getenv("ADMIN_USERNAME", "l9bi7e_net")
DB_PATH   = "l9bi7e_net.db"
DEFAULT_PROOF_CHANNEL_LINK = os.getenv("DEFAULT_PROOF_CHANNEL_LINK", "https://t.me/NETXPERT_DZ_bot_TFIAL")

BASE_URL      = "https://apim.djezzy.dz/mobile-api"
TOKEN_URL     = f"{BASE_URL}/oauth2/token"
OTP_URL       = f"{BASE_URL}/oauth2/registration"
CLIENT_ID     = "87pIExRhxBb3_wGsA5eSEfyATloa"
CLIENT_SECRET = "uf82p68Bgisp8Yg1Uz8Pf6_v1XYa"

USER_AGENTS = [
    "Dalvik/2.1.0 (Linux; U; Android 6.0; PGN610 Build/MRA58K)",
    "Dalvik/2.1.0 (Linux; U; Android 8.0.0; SM-G950F Build/R16NW)",
    "Dalvik/2.1.0 (Linux; U; Android 9; SM-G973F Build/PPR1.180610.011)",
    "Dalvik/2.1.0 (Linux; U; Android 10; SM-G975F Build/QP1A.190711.020)",
    "Dalvik/2.1.0 (Linux; U; Android 11; SM-G991B Build/RP1A.200720.012)",
    "Dalvik/2.1.0 (Linux; U; Android 12; SM-G998B Build/SP1A.210812.016)",
    "MobileApp/3.0.0",
    "MobileApp/3.1.0",
    "MobileApp/3.0.4",
]
USER_AGENT = USER_AGENTS[0]
USER_AGENT_V2 = "MobileApp/3.0.4"

PKG_1GO = "GIFTWALKWIN1GO"
PKG_2GO = "GIFTWALKWIN2GO"
PKG_6GB = "GIFTWALKWIN6GDAY"
MAX_INVITES = 5

STATE_UNKNOWN = 0
STATE_READY   = 1
STATE_WAITING = 2
_UNSET_HISTORY = object()

EMOJI_BOT       = "(5327893397246077619)"
EMOJI_STAR      = "(5309928798882395910)"
EMOJI_CHECK     = "(6237704110628413424)"
EMOJI_WARNING   = "(6100541727280469939)"
EMOJI_ERROR     = "(5325550088859184283)"
EMOJI_CLOCK     = "(6120558272039620624)"
EMOJI_FIRE      = "(5328121086347351965)"
EMOJI_LOADING   = "(5271638912035494810)"
EMOJI_ROCKET    = "(5309928798882395910)"
EMOJI_CHAT      = "(6120558272039620624)"
EMOJI_STOP      = "(6097950169783866920)"
EMOJI_VIEW      = "(6098136562774579060)"
EMOJI_LIST      = "(6097880977860729697)"
EMOJI_NAME      = "(5327893397246077619)"
EMOJI_SEND      = "(5309928798882395910)"
EMOJI_LIKES     = "(6237704110628413424)"
EMOJI_PHONE     = "(6087128656084210204)"
EMOJI_SIM       = "(6129550284290006595)"
EMOJI_GIFT      = "(6129497211379129336)"
EMOJI_BALANCE   = "(6089104607328342288)"
EMOJI_CARD      = "(5305265301917549162)"
EMOJI_LOGIN     = "(5296369303661067030)"
EMOJI_LOGOUT    = "(6087162775304409688)"
EMOJI_BACK      = "(5461151367559141950)"
EMOJI_USERS     = "(5970053985103516666)"
EMOJI_STATS     = "(5231200819986047254)"
EMOJI_MEGAPHONE = "(5424818078833715060)"
EMOJI_BAN       = "(6086741365998227951)"
EMOJI_UNBAN     = "(6086954160152908450)"
EMOJI_BROOM     = "(5445267414562389170)"
EMOJI_SETTINGS  = "(5341715473882955310)"
EMOJI_SHIELD    = "(6089003761496232797)"
EMOJI_BOX       = "(6089140105233044310)"
EMOJI_TAG       = "(6089104607328342288)"
EMOJI_LIGHTNING = "(6087079590377820415)"
EMOJI_BOOK      = "(6129432481927010933)"
EMOJI_PIN       = "(5397782960512444700)"
EMOJI_HOURGLASS = "(6129779562529168023)"
EMOJI_PARTY     = "(5461151367559141950)"
EMOJI_SUPPORT   = "(6120558272039620624)"
EMOJI_REPORT    = "(6100541727280469939)"
EMOJI_REPLY     = "(6120558272039620624)"

emoji_pattern = re.compile(r"\((\d+)\)")
_EMOJI_CACHE: dict = {}
_bold_pattern = re.compile(r"<b>(.*?)</b>", re.DOTALL)
_code_pattern = re.compile(r"<code>(.*?)</code>", re.DOTALL)


def utf16_len(text: str) -> int:
    return len(text.encode("utf-16-le")) // 2


async def build_ce_text(bot, raw_text: str):
    if not raw_text:
        return "", []

    code_blocks = []
    def _stash_code(m):
        idx = len(code_blocks)
        code_blocks.append(m.group(1))
        return f"\x00C{idx}\x00"
    text = _code_pattern.sub(_stash_code, raw_text)

    bold_blocks = []
    def _stash_bold(m):
        idx = len(bold_blocks)
        bold_blocks.append(m.group(1))
        return f"\x00B{idx}\x00"
    text = _bold_pattern.sub(_stash_bold, text)

    all_ids = set()
    for m in emoji_pattern.finditer(raw_text):
        all_ids.add(m.group(1))
    missing = [cid for cid in all_ids if cid not in _EMOJI_CACHE]
    if missing:
        try:
            stickers = await bot.get_custom_emoji_stickers(missing)
            for s in stickers:
                _EMOJI_CACHE[str(s.custom_emoji_id)] = s
        except Exception:
            pass

    result = []
    entities = []
    pos = 0
    i = 0

    combined = re.compile(r"\((\d+)\)|\x00C(\d+)\x00|\x00B(\d+)\x00")

    def _append_plain(s):
        nonlocal pos
        j = 0
        for im in emoji_pattern.finditer(s):
            before = s[j:im.start()]
            if before:
                result.append(before)
                pos += utf16_len(before)
            cid = im.group(1)
            st = _EMOJI_CACHE.get(cid)
            if st and st.emoji:
                emoji = st.emoji
                result.append(emoji)
                entities.append(MessageEntity(
                    type=MessageEntity.CUSTOM_EMOJI,
                    offset=pos,
                    length=utf16_len(emoji),
                    custom_emoji_id=cid,
                ))
                pos += utf16_len(emoji)
            j = im.end()
        tail = s[j:]
        if tail:
            result.append(tail)
            pos += utf16_len(tail)

    for m in combined.finditer(text):
        before = text[i:m.start()]
        if before:
            _append_plain(before)

        if m.group(1):
            cid = m.group(1)
            st = _EMOJI_CACHE.get(cid)
            if st and st.emoji:
                emoji = st.emoji
                result.append(emoji)
                entities.append(MessageEntity(
                    type=MessageEntity.CUSTOM_EMOJI,
                    offset=pos,
                    length=utf16_len(emoji),
                    custom_emoji_id=cid,
                ))
                pos += utf16_len(emoji)

        elif m.group(2):
            idx = int(m.group(2))
            content = code_blocks[idx]
            start_pos = pos
            _append_plain(content)
            entities.append(MessageEntity(
                type=MessageEntity.CODE,
                offset=start_pos,
                length=pos - start_pos,
            ))

        elif m.group(3):
            idx = int(m.group(3))
            content = bold_blocks[idx]
            start_pos = pos
            _append_plain(content)
            entities.append(MessageEntity(
                type=MessageEntity.BOLD,
                offset=start_pos,
                length=pos - start_pos,
            ))

        i = m.end()

    tail = text[i:]
    if tail:
        _append_plain(tail)

    return "".join(result), entities


async def send_ce(bot, chat_id, text: str, reply_markup=None, reply_to_message_id=None):
    final_text, entities = await build_ce_text(bot, text)
    try:
        return await bot.send_message(
            chat_id=chat_id,
            text=final_text,
            entities=entities,
            reply_markup=reply_markup,
            reply_to_message_id=reply_to_message_id,
        )
    except Exception as e:
        logger.error(f"send_ce entities failed: {e}")
        try:
            clean = re.sub(r"\(\d{10,}\)", "", text)
            clean = clean.replace("<b>", "*").replace("</b>", "*")
            clean = clean.replace("<code>", "`").replace("</code>", "`")
            return await bot.send_message(
                chat_id=chat_id,
                text=clean,
                reply_markup=reply_markup,
                reply_to_message_id=reply_to_message_id,
                parse_mode=None,
            )
        except Exception as e2:
            logger.error(f"send_ce fallback failed: {e2}")
            return None


async def edit_ce(bot, chat_id, message_id, text: str, reply_markup=None):
    final_text, entities = await build_ce_text(bot, text)
    try:
        await bot.edit_message_text(
            chat_id=chat_id,
            message_id=message_id,
            text=final_text,
            entities=entities,
            reply_markup=reply_markup,
        )
        return
    except Exception as e:
        logger.warning(f"edit_ce entities failed: {e}")
    try:
        clean = re.sub(r"\(\d{10,}\)", "", text)
        clean = clean.replace("<b>", "*").replace("</b>", "*")
        clean = clean.replace("<code>", "`").replace("</code>", "`")
        await bot.edit_message_text(
            chat_id=chat_id,
            message_id=message_id,
            text=clean,
            reply_markup=reply_markup,
            parse_mode=None,
        )
    except Exception as e2:
        logger.error(f"edit_ce fallback failed: {e2}")


def _make_ib(text, **kwargs):
    try:
        return InlineKeyboardButton(text, **kwargs)
    except TypeError:
        kwargs.pop("style", None)
        return InlineKeyboardButton(text, **kwargs)


DANGER_WORDS = [
    "رجوع", "عودة", "back", "إلغاء", "الغاء", "cancel",
    "حذف", "delete", "remove", "حظر", "ban",
    "تصفير", "مسح", "clear", "reset", "تغيير الرقم", "change_number",
    "خروج", "logout", "إيقاف", "stop"
]

SUCCESS_WORDS = [
    "تأكيد", "تاكيد", "confirm",
    "تفعيل", "activate", "شراء", "buy",
    "رصيد", "balance",
    "تسجيل الدخول", "login",
    "عرض", "offer",
    "بطاقتي", "account",
    "حزمة", "walk",
    "دعوة", "invite",
    "إعادة", "retry",
    "فك الحظر", "unban"
]


def btn(text: str, callback_data: str = None, url: str = None) -> InlineKeyboardButton:
    check = (text or "").lower() + " " + (callback_data or "").lower()
    style = "primary"
    if any(w.lower() in check for w in DANGER_WORDS):
        style = "danger"
    elif any(w.lower() in check for w in SUCCESS_WORDS):
        style = "success"

    if url:
        return _make_ib(text, url=url, style=style)
    return _make_ib(text, callback_data=callback_data, style=style)


OFFERS = {
    "OFFREJEUNE1200":   {"name": "📦 IZZI 1200 — 10GB + Illimité",  "display": "10GB + Illimité + 1000DA", "price": "1200 DA", "duration": "30 يوم", "cat": "izzy"},
    "OFFREJEUNEBASE500":{"name": "📦 IZZI 500 — 5GB + Insta",      "display": "5GB + Insta + 500DA",     "price": "500 DA",  "duration": "30 يوم", "cat": "izzy"},
    "OFFREJEUNEBASE300":{"name": "📦 IZZI 300 — 3GB",              "display": "3GB + مكالمات",            "price": "300 DA",  "duration": "15 يوم", "cat": "izzy"},
    "OFFREJEUNE50":     {"name": "📦 IZZI 50 — 1GB",               "display": "1GB + مكالمات",            "price": "50 DA",   "duration": "24 ساعة", "cat": "izzy"},
    "OPTION500":        {"name": "📦 Option 500 — 5GB + Insta",    "display": "5GB + Insta",              "price": "500 DA",  "duration": "15 يوم", "cat": "izzy"},
    "lowValue1000":     {"name": "⚡ ZID 1000 — 10GB",             "display": "10GB + 1300DA",            "price": "1000 DA", "duration": "30 يوم", "cat": "zid"},
    "lowValue800":      {"name": "⚡ ZID 800 — 8GB",               "display": "8GB + 1000DA",             "price": "800 DA",  "duration": "30 يوم", "cat": "zid"},
    "lowValue400":      {"name": "⚡ ZID 400 — 4GB",               "display": "4GB + 500DA",              "price": "400 DA",  "duration": "15 يوم", "cat": "zid"},
    "lowValue200":      {"name": "⚡ ZID 200 — 2GB",               "display": "2GB + 200DA",              "price": "200 DA",  "duration": "7 أيام", "cat": "zid"},
    "lowValue50":       {"name": "⚡ ZID 50 — 500MB",              "display": "500MB + 50DA",             "price": "50 DA",   "duration": "24 ساعة", "cat": "zid"},
    "HAYLABEZZEFMONTH2000PREP": {"name": "🔥 BEZZEF 2000 — 60GB",  "display": "60GB + 7000DA",            "price": "2000 DA", "duration": "30 يوم", "cat": "bezzef"},
    "HAYLABEZZEFMONTH1500PREP": {"name": "🔥 BEZZEF 1500 — 40GB",  "display": "40GB + 3000DA",            "price": "1500 DA", "duration": "30 يوم", "cat": "bezzef"},
    "HAYLABEZZEFMONTH1200PREP": {"name": "🔥 BEZZEF 1200 — 10GB",  "display": "10GB + 1500DA",            "price": "1200 DA", "duration": "30 يوم", "cat": "bezzef"},
    "HAYLABEZZEFWEEK500PREP":   {"name": "🔥 BEZZEF 500 — 3GB",    "display": "3GB + 750DA",              "price": "500 DA",  "duration": "15 يوم", "cat": "bezzef"},
    "HAYLABEZZEFDAILY150PREP":  {"name": "🔥 BEZZEF 150 — 2GB",    "display": "2GB + 300DA",              "price": "150 DA",  "duration": "24 ساعة", "cat": "bezzef"},
    "HAYLABEZZEFDAY100PREP":    {"name": "🔥 BEZZEF 100 — 500MB",  "display": "500MB + 150DA",            "price": "100 DA",  "duration": "24 ساعة", "cat": "bezzef"},
    "MAXIHAYLA2000":    {"name": "🌟 MAXI 2000 — 60GB",            "display": "60GB + 4000DA",            "price": "2000 DA", "duration": "30 يوم", "cat": "maxi"},
    "MAXIHAYLA1500":    {"name": "🌟 MAXI 1500 — 40GB",            "display": "40GB + 3000DA",            "price": "1500 DA", "duration": "30 يوم", "cat": "maxi"},
    "MAXIHAYLA1200":    {"name": "🌟 MAXI 1200 — 12GB",            "display": "12GB + 1500DA",            "price": "1200 DA", "duration": "30 يوم", "cat": "maxi"},
    "MAXIHAYLA500":     {"name": "🌟 MAXI 500 — 7GB",              "display": "7GB + 750DA",              "price": "500 DA",  "duration": "7 أيام", "cat": "maxi"},
    "MAXIHAYLA100":     {"name": "🌟 MAXI 100 — 1GB",              "display": "1GB + 150DA",              "price": "100 DA",  "duration": "24 ساعة", "cat": "maxi"},
    "BTLINTSPEEDDAY2Go":      {"name": "⚡ 4GB (70 DA)",  "display": "4GB | 70 DA | 24 ساعة",   "price": "70 DA",  "duration": "24 ساعة", "cat": "imtiyaz"},
    "BTL500MBDAY":            {"name": "⚡ 5GB (90 DA)",  "display": "5GB | 90 DA | 24 ساعة",   "price": "90 DA",  "duration": "24 ساعة", "cat": "imtiyaz"},
    "BTL4GBDAY":              {"name": "🔥 10GB (190 DA)","display": "10GB | 190 DA | 3 أيام",   "price": "190 DA", "duration": "3 أيام",  "cat": "imtiyaz"},
    "BTLHYBMONTHLY800DA":     {"name": "👑 800 DA — 15GB", "display": "15GB + 2000DA",           "price": "800 DA", "duration": "30 يوم",  "cat": "imtiyaz"},
    "BTLHYBMONTHLY1000DA":    {"name": "👑 1000 DA — 20GB","display": "20GB + 3000DA",           "price": "1000 DA","duration": "30 يوم",  "cat": "imtiyaz"},
    "BTLHYBMONTHLY1200DA":    {"name": "👑 1200 DA — 30GB","display": "30GB + 2500DA",           "price": "1200 DA","duration": "30 يوم",  "cat": "imtiyaz"},
    "BTLHYBMONTHLY1500DA":    {"name": "👑 1500 DA — 40GB","display": "40GB + 3000DA",           "price": "1500 DA","duration": "30 يوم",  "cat": "imtiyaz"},
    "BTL2C2000":              {"name": "👑 2000 DA — 70GB","display": "70GB + مكالمات",          "price": "2000 DA","duration": "30 يوم",  "cat": "imtiyaz"},
}

OFFERS_IZZY    = [k for k,v in OFFERS.items() if v["cat"] == "izzy"]
OFFERS_ZID     = [k for k,v in OFFERS.items() if v["cat"] == "zid"]
OFFERS_BEZZEF  = [k for k,v in OFFERS.items() if v["cat"] == "bezzef"]
OFFERS_MAXI    = [k for k,v in OFFERS.items() if v["cat"] == "maxi"]
OFFERS_IMTIYAZ = [k for k,v in OFFERS.items() if v["cat"] == "imtiyaz"]

log_handlers = [logging.StreamHandler(sys.stdout)]
try:
    log_handlers.append(logging.FileHandler("bot.log", encoding="utf-8"))
except Exception as e:
    print(f"⚠️ تحذير: تعذر إنشاء ملف bot.log ({e}). سيتم الطباعة في الكونسول فقط.")

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    handlers=log_handlers,
)
logger = logging.getLogger("L9bi7eBot")

sqlite3.register_adapter(datetime, lambda dt: dt.isoformat())
sqlite3.register_converter("timestamp", lambda b: datetime.fromisoformat(b.decode()))


def init_db():
    conn = sqlite3.connect(DB_PATH, detect_types=sqlite3.PARSE_DECLTYPES)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY, name TEXT, username TEXT, phone TEXT,
        registered_at TIMESTAMP, last_active TIMESTAMP, banned INTEGER DEFAULT 0)''')
    c.execute('''CREATE TABLE IF NOT EXISTS tokens (
        user_id INTEGER PRIMARY KEY, access_token TEXT,
        refresh_token TEXT, expires_at TIMESTAMP)''')
    c.execute('''CREATE TABLE IF NOT EXISTS activations (
        id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, phone TEXT,
        offer_type TEXT, activated_at TIMESTAMP)''')
    c.execute('''CREATE TABLE IF NOT EXISTS invitations (
        id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER,
        invited_phone TEXT, invited_at TIMESTAMP)''')
    c.execute('''CREATE TABLE IF NOT EXISTS scheduled (
        id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, phone TEXT,
        offer_type TEXT, remind_at TIMESTAMP, sent INTEGER DEFAULT 0)''')
    c.execute('''CREATE TABLE IF NOT EXISTS force_channels (
        channel_id TEXT PRIMARY KEY, channel_link TEXT, channel_name TEXT,
        channel_type TEXT DEFAULT 'telegram', added_at TIMESTAMP)''')
    c.execute('''CREATE TABLE IF NOT EXISTS bot_settings (
        key TEXT PRIMARY KEY, value TEXT)''')
    c.execute('''CREATE TABLE IF NOT EXISTS reports (
        id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, username TEXT,
        name TEXT, message TEXT, status TEXT DEFAULT 'open',
        admin_reply TEXT, created_at TIMESTAMP, replied_at TIMESTAMP)''')
    conn.commit()
    try:
        cols = [r[1] for r in conn.execute("PRAGMA table_info(force_channels)").fetchall()]
        if "channel_type" not in cols:
            conn.execute("ALTER TABLE force_channels ADD COLUMN channel_type TEXT DEFAULT 'telegram'")
            conn.commit()
    except Exception:
        pass
    conn.close()


@contextmanager
def get_db():
    conn = sqlite3.connect(DB_PATH, detect_types=sqlite3.PARSE_DECLTYPES)
    conn.row_factory = sqlite3.Row
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()


def db_add_user(uid, name="", username="", phone=None):
    with get_db() as conn:
        existing = conn.execute(
            "SELECT phone, banned FROM users WHERE user_id=?", (uid,)).fetchone()
        saved_phone = phone if phone else (existing["phone"] if existing else None)
        banned_val = existing["banned"] if existing else 0
        conn.execute(
            "INSERT OR REPLACE INTO users "
            "(user_id,name,username,phone,registered_at,last_active,banned) "
            "VALUES(?,?,?,?,?,?,?)",
            (uid, name, username, saved_phone, datetime.now(), datetime.now(), banned_val))


def db_get_user(uid):
    with get_db() as conn:
        return conn.execute("SELECT * FROM users WHERE user_id=?", (uid,)).fetchone()


def db_get_phone(uid):
    with get_db() as conn:
        r = conn.execute("SELECT phone FROM users WHERE user_id=?", (uid,)).fetchone()
        return r["phone"] if r else None


def db_update_phone(uid, phone):
    with get_db() as conn:
        conn.execute("UPDATE users SET phone=? WHERE user_id=?", (phone, uid))


def db_update_name(uid, name):
    with get_db() as conn:
        conn.execute("UPDATE users SET name=? WHERE user_id=?", (name, uid))


def db_update_username(uid, username):
    with get_db() as conn:
        conn.execute("UPDATE users SET username=? WHERE user_id=?", (username, uid))


def db_is_banned(uid):
    with get_db() as conn:
        r = conn.execute("SELECT banned FROM users WHERE user_id=?", (uid,)).fetchone()
        return bool(r["banned"]) if r else False


def db_ban(uid):
    with get_db() as conn:
        conn.execute("UPDATE users SET banned=1 WHERE user_id=?", (uid,))


def db_unban(uid):
    with get_db() as conn:
        conn.execute("UPDATE users SET banned=0 WHERE user_id=?", (uid,))


def db_all_users():
    with get_db() as conn:
        return [dict(r) for r in conn.execute("SELECT * FROM users")]


def db_user_count():
    with get_db() as conn:
        return conn.execute("SELECT COUNT(*) FROM users").fetchone()[0]


def db_save_tokens(uid, access, refresh=None, expires_in=3600):
    with get_db() as conn:
        conn.execute("INSERT OR REPLACE INTO tokens VALUES(?,?,?,?)",
                     (uid, access, refresh, datetime.now() + timedelta(seconds=expires_in)))


def db_get_token_row(uid):
    with get_db() as conn:
        return conn.execute(
            "SELECT access_token,refresh_token,expires_at FROM tokens WHERE user_id=?",
            (uid,)).fetchone()


def db_clear_tokens(uid):
    with get_db() as conn:
        conn.execute("DELETE FROM tokens WHERE user_id=?", (uid,))


def db_clear_all_tokens():
    with get_db() as conn:
        conn.execute("DELETE FROM tokens")


def db_save_activation(uid, phone, offer):
    with get_db() as conn:
        conn.execute(
            "INSERT INTO activations(user_id,phone,offer_type,activated_at) VALUES(?,?,?,?)",
            (uid, phone, offer, datetime.now()))


def db_last_activation(phone, offer):
    with get_db() as conn:
        return conn.execute(
            "SELECT activated_at FROM activations WHERE phone=? AND offer_type=? "
            "ORDER BY activated_at DESC LIMIT 1", (phone, offer)).fetchone()


def db_total_activations():
    with get_db() as conn:
        return conn.execute("SELECT COUNT(*) FROM activations").fetchone()[0]


def db_invite_count(uid):
    with get_db() as conn:
        return conn.execute("SELECT COUNT(*) FROM invitations WHERE user_id=?", (uid,)).fetchone()[0]


def db_add_invitation(uid, phone):
    with get_db() as conn:
        if conn.execute("SELECT id FROM invitations WHERE user_id=? AND invited_phone=?",
                        (uid, phone)).fetchone():
            return False, "هذا الرقم تمت دعوته مسبقاً"
        count = conn.execute("SELECT COUNT(*) FROM invitations WHERE user_id=?", (uid,)).fetchone()[0]
        if count >= MAX_INVITES:
            return False, f"وصلت للحد الأقصى ({MAX_INVITES} دعوات)"
        conn.execute("INSERT INTO invitations(user_id,invited_phone,invited_at) VALUES(?,?,?)",
                     (uid, phone, datetime.now()))
        return True, f"تم الإرسال ({count+1}/{MAX_INVITES})"


def db_schedule_reminder(uid, phone, offer, seconds):
    with get_db() as conn:
        conn.execute("DELETE FROM scheduled WHERE user_id=? AND phone=? AND offer_type=? AND sent=0",
                     (uid, phone, offer))
        conn.execute("INSERT INTO scheduled(user_id,phone,offer_type,remind_at) VALUES(?,?,?,?)",
                     (uid, phone, offer, datetime.now() + timedelta(seconds=seconds)))


def db_due_reminders():
    with get_db() as conn:
        return [dict(r) for r in conn.execute(
            "SELECT id,user_id,offer_type FROM scheduled WHERE remind_at<=? AND sent=0",
            (datetime.now(),)).fetchall()]


def db_mark_reminder_sent(rid):
    with get_db() as conn:
        conn.execute("UPDATE scheduled SET sent=1 WHERE id=?", (rid,))


def db_add_channel(channel_id, channel_link, channel_name, channel_type="telegram"):
    with get_db() as conn:
        conn.execute(
            "INSERT OR REPLACE INTO force_channels"
            "(channel_id,channel_link,channel_name,channel_type,added_at) "
            "VALUES(?,?,?,?,?)",
            (str(channel_id), channel_link, channel_name, channel_type, datetime.now()))


def db_remove_channel(channel_id):
    with get_db() as conn:
        conn.execute("DELETE FROM force_channels WHERE channel_id=?", (str(channel_id),))


def db_get_channels():
    with get_db() as conn:
        return [dict(r) for r in conn.execute("SELECT * FROM force_channels").fetchall()]


def db_clear_channels():
    with get_db() as conn:
        conn.execute("DELETE FROM force_channels")


def db_set_setting(key, value):
    with get_db() as conn:
        conn.execute("INSERT OR REPLACE INTO bot_settings(key,value) VALUES(?,?)",
                     (key, str(value)))


def db_get_setting(key, default=None):
    with get_db() as conn:
        r = conn.execute("SELECT value FROM bot_settings WHERE key=?", (key,)).fetchone()
        return r["value"] if r else default


def db_del_setting(key):
    with get_db() as conn:
        conn.execute("DELETE FROM bot_settings WHERE key=?", (key,))


def db_add_report(uid, username, name, message):
    with get_db() as conn:
        cur = conn.execute(
            "INSERT INTO reports(user_id,username,name,message,status,created_at) "
            "VALUES(?,?,?,?,?,?)",
            (uid, username or "", name or "", message, "open", datetime.now()))
        return cur.lastrowid


def db_get_open_reports():
    with get_db() as conn:
        return [dict(r) for r in conn.execute(
            "SELECT * FROM reports WHERE status='open' ORDER BY created_at DESC").fetchall()]


def db_get_report(rid):
    with get_db() as conn:
        r = conn.execute("SELECT * FROM reports WHERE id=?", (rid,)).fetchone()
        return dict(r) if r else None


def db_reply_report(rid, reply):
    with get_db() as conn:
        conn.execute(
            "UPDATE reports SET status='replied', admin_reply=?, replied_at=? WHERE id=?",
            (reply, datetime.now(), rid))


def get_proof_channel():
    cid = db_get_setting("proof_channel_id")
    if not cid:
        return None
    return {
        "id": cid,
        "link": db_get_setting("proof_channel_link", "https://t.me/NETXPERT_DZ_bot_TFIAL"),
        "name": db_get_setting("proof_channel_name", "قناة الإثبات"),
    }


def set_proof_channel(channel_id, channel_link, channel_name):
    db_set_setting("proof_channel_id", channel_id)
    db_set_setting("proof_channel_link", channel_link)
    db_set_setting("proof_channel_name", channel_name)
    db_add_channel(channel_id, channel_link, channel_name, "telegram")


def clear_proof_channel():
    old_id = db_get_setting("proof_channel_id")
    if old_id:
        db_remove_channel(old_id)
    db_del_setting("proof_channel_id")
    db_del_setting("proof_channel_link")
    db_del_setting("proof_channel_name")


def db_get_user_phones(uid, limit=3):
    with get_db() as conn:
        rows = conn.execute(
            "SELECT DISTINCT phone FROM activations WHERE user_id=? "
            "ORDER BY activated_at DESC LIMIT ?", (uid, limit)).fetchall()
        phones = [r["phone"] for r in rows if r["phone"]]
        main_phone = db_get_phone(uid)
        if main_phone and main_phone not in phones:
            phones.insert(0, main_phone)
        return phones[:limit]


def format_phone(p):
    d = re.sub(r"\D", "", str(p))
    if not d: return ""
    if d.startswith("213"): d = d[3:]
    if len(d) == 9: d = "0" + d
    if d.startswith("0") and len(d) == 10: return "213" + d[1:]
    return "213" + d


def mask_phone(p):
    d = re.sub(r"\D", "", str(p))
    if len(d) >= 12: return d[:3] + "●●●●" + d[-3:]
    if len(d) >= 10: return d[:2] + "●●●●●" + d[-3:]
    return "07●●●●●"


def fmt_time(ms):
    if ms <= 0: return "جاهز"
    s = ms // 1000
    d, h, m = s // 86400, (s % 86400) // 3600, (s % 3600) // 60
    if d: return f"{d}ي {h}س"
    if h: return f"{h}س {m}د"
    return f"{m}د"


def fmt_seconds(sec):
    if sec <= 0: return "0 ثانية"
    d, h, m, s = sec//86400, (sec%86400)//3600, (sec%3600)//60, sec%60
    parts = []
    if d: parts.append(f"{d} يوم")
    if h: parts.append(f"{h} ساعة")
    if m: parts.append(f"{m} دقيقة")
    if s and not d and not h: parts.append(f"{s} ثانية")
    return " و".join(parts)


def fmt_seconds_detailed(sec):
    if sec <= 0:
        return "جاهز الآن"
    d = int(sec) // 86400
    h = (int(sec) % 86400) // 3600
    m = (int(sec) % 3600) // 60
    s = int(sec) % 60
    parts = []
    if d: parts.append(f"{d} يوم")
    if h: parts.append(f"{h} ساعة")
    if m: parts.append(f"{m} دقيقة")
    if s: parts.append(f"{s} ثانية")
    return " و ".join(parts) if parts else "0 ثانية"


def is_valid_dz_number(raw):
    return bool(re.match(r"^07\d{8}$", re.sub(r"\D","",raw)))


def get_offer_state(phone, offer):
    row = db_last_activation(phone, offer)
    if not row: return STATE_UNKNOWN, 0
    elapsed = datetime.now() - row["activated_at"]
    cooldown = timedelta(days=1) if offer in ("1g", "shake_6gb") else timedelta(days=7)
    if elapsed >= cooldown: return STATE_READY, 0
    return STATE_WAITING, int((cooldown - elapsed).total_seconds() * 1000)


def _make_session():
    s = requests.Session()
    r = Retry(total=3, backoff_factor=0.5,
              status_forcelist=[500,502,503,504],
              allowed_methods=["GET","POST"])
    s.mount("https://", HTTPAdapter(max_retries=r))
    s.mount("http://",  HTTPAdapter(max_retries=r))
    return s


_session = _make_session()
_BASE_H = {"User-Agent": USER_AGENT, "Accept": "application/json", "accept-language": "ar"}


def _extract_djezzy_error(resp):
    if resp is None: return None
    try: body = resp.json()
    except Exception: return None
    if not isinstance(body, dict): return None
    errors = body.get("errors")
    if isinstance(errors, list) and errors:
        e0 = errors[0]
        if isinstance(e0, dict):
            d = e0.get("detail") or e0.get("title") or e0.get("message") or ""
            if d: return str(d)
    msg = body.get("message")
    if isinstance(msg, dict):
        return msg.get("ar") or msg.get("en") or None
    if isinstance(msg, str) and msg: return msg
    fault = body.get("fault", {})
    if isinstance(fault, dict):
        fs = fault.get("faultstring", "")
        if fs: return str(fs)
    return None

_EARLY_REFRESH_BUFFER = timedelta(seconds=90)


def _refresh_access_token(uid, refresh_tok):
    try:
        resp = _session.post(TOKEN_URL,
            headers={**_BASE_H, "Content-Type": "application/x-www-form-urlencoded"},
            data={"grant_type":"refresh_token","refresh_token":refresh_tok,
                  "client_id":CLIENT_ID,"client_secret":CLIENT_SECRET,"scope":"openid"},
            timeout=12)
        if resp.status_code == 200:
            data = resp.json()
            db_save_tokens(uid, data["access_token"], data.get("refresh_token", refresh_tok),
                           data.get("expires_in", 3600))
            return f"Bearer {data['access_token']}"
    except Exception as e:
        logger.warning(f"refresh err: {e}")
    return None


def get_token(uid):
    row = db_get_token_row(uid)
    if not row: return None
    exp = row["expires_at"]
    if isinstance(exp, str):
        try: exp = datetime.fromisoformat(exp)
        except Exception: exp = None
    if exp and exp - _EARLY_REFRESH_BUFFER > datetime.now():
        return f"Bearer {row['access_token']}"
    if row["refresh_token"]:
        new_tok = _refresh_access_token(uid, row["refresh_token"])
        if new_tok: return new_tok
        if exp and exp > datetime.now(): return f"Bearer {row['access_token']}"
    return None


def api_request_otp(msisdn):
    max_attempts = 4
    wait_times = [0, 20, 45, 90]

    for attempt in range(max_attempts):
        if attempt > 0:
            wait = wait_times[attempt]
            logger.info(f"OTP retry {attempt}/{max_attempts-1} — انتظار {wait}s")
            time.sleep(wait)

        headers = {
            "User-Agent": USER_AGENTS[attempt % len(USER_AGENTS)],
            "Accept": "application/json",
            "accept-language": "ar",
            "Content-Type": "application/json",
        }

        try:
            r = _session.post(
                OTP_URL,
                params={'msisdn': msisdn, 'client_id': CLIENT_ID, 'scope': "smsotp"},
                json={"consent-agreement": [{"marketing-notifications": False}], "is-consent": True},
                headers=headers,
                timeout=25
            )

            logger.info(f"OTP RESPONSE | attempt={attempt+1} | status={r.status_code} | body={r.text[:300]}")

            if r.status_code in [200, 201, 202]:
                logger.info(f"OTP sent OK → {msisdn[:6]}***")
                return True, "OK"

            if r.status_code == 429:
                logger.warning(f"OTP 429 attempt {attempt+1}")
                continue

            if r.status_code == 400:
                try:
                    body = r.json()
                    msg = str(body.get("message", "")).lower()
                    if "not known" in msg or "not registered" in msg:
                        return False, "❌ الرقم غير مسجل في شبكة جيزي\nتأكد من الرقم (يبدأ بـ 07)"
                except:
                    pass
                return False, "❌ الرقم غير صحيح أو غير مسجل"

            if r.status_code in (401, 403):
                logger.warning(f"OTP {r.status_code} attempt {attempt+1}")
                continue

            if 500 <= r.status_code < 600:
                logger.warning(f"OTP {r.status_code} attempt {attempt+1}")
                continue

            return False, f"خطأ {r.status_code}"

        except requests.exceptions.Timeout:
            logger.warning(f"OTP timeout attempt {attempt+1}")
            continue
        except requests.exceptions.RequestException as e:
            logger.error(f"OTP network error: {e}")
            continue
        except Exception as e:
            logger.error(f"OTP unexpected: {e}")
            continue

    return False, (
        "⏳ جيزي رفضت الطلب (محاولات كثيرة)\n\n"
        "🔸 انتظر 5-10 دقائق على الأقل\n"
        "🔸 جرّب بعدها مرة أخرى\n\n"
        "💡 نصيحة: لا تطلب الرمز أكثر من مرة كل 5 دقائق"
    )


def api_verify_otp(msisdn, otp, uid):
    max_attempts = 3
    for attempt in range(max_attempts):
        try:
            resp = _session.post(TOKEN_URL,
                headers={
                    "User-Agent": USER_AGENTS[attempt % len(USER_AGENTS)],
                    "Content-Type": "application/x-www-form-urlencoded",
                    "Accept": "application/json",
                },
                data={"otp":otp,"mobileNumber":msisdn,"scope":"djezzyAppV2",
                      "client_id":CLIENT_ID,"client_secret":CLIENT_SECRET,"grant_type":"mobile"},
                timeout=20)

            if resp.status_code == 200:
                data = resp.json()
                db_save_tokens(uid, data["access_token"], data.get("refresh_token"),
                               data.get("expires_in",3600))
                return True, data.get("firstName") or data.get("displayName") or "", "OK"

            if resp.status_code == 400:
                return False, None, "❌ الرمز غير صحيح — تحقق منه وأعد المحاولة"

            if resp.status_code == 429:
                if attempt < max_attempts - 1:
                    time.sleep(20)
                    continue
                return False, None, "⏳ محاولات كثيرة — انتظر دقيقة وأعد المحاولة"

            if 500 <= resp.status_code < 600:
                if attempt < max_attempts - 1:
                    time.sleep(5)
                    continue
                return False, None, "خطأ في خادم جيزي — أعد المحاولة"

            return False, None, f"خطأ {resp.status_code}"

        except requests.exceptions.Timeout:
            if attempt < max_attempts - 1:
                time.sleep(5)
                continue
            return False, None, "انتهت مهلة الاتصال — أعد المحاولة"
        except Exception as e:
            logger.error(f"verify_otp: {e}")
            if attempt < max_attempts - 1:
                time.sleep(3)
                continue
            return False, None, "خطأ في الاتصال — أعد المحاولة"

    return False, None, "تعذر التحقق — حاول لاحقاً"


def api_get_balance(uid, phone):
    token = get_token(uid)
    if not token: return None, "سجل دخول"
    headers = {**_BASE_H, "Authorization": token}
    for url in [f"{BASE_URL}/api/v1/subscribers/main-balance/{phone}",
                f"{BASE_URL}/api/v1/account/summary"]:
        try:
            resp = _session.get(url, headers=headers, timeout=15)
            if resp.status_code == 200:
                data = resp.json()
                bal = (data.get("balance",{}).get("mainBalance")
                    or data.get("balance",{}).get("availableBalance")
                    or data.get("data",{}).get("mainBalance"))
                if bal is not None: return str(bal), None
        except Exception: pass
    return None, "تعذر الجلب"


def api_get_full_account_info(uid, phone):
    token = get_token(uid)
    if not token: return None
    url = (f"{BASE_URL}/api/v1/subscribers/main-balance/{phone}"
           "?include=surprise-products,subscription-type-illegibility,connected-products,"
           "subscription-history,flash-products,supplementary-informations,available-services")
    headers = {**_BASE_H, "Authorization": token, "Connection": "Keep-Alive"}
    try:
        resp = _session.get(url, headers=headers, timeout=15)
        if resp.status_code != 200: return None
        raw = resp.json()
        data = raw.get("data", {}) or {}
        info = data.get("customerInformations", {}) or {}
        pt_raw = (info.get("paymentType") or "").strip()
        payment_type = "Prepaid" if pt_raw.upper() == "PREPAID" else (
                       "Postpaid" if pt_raw.upper() == "POSTPAID" else (pt_raw or "—"))
        sub_type = info.get("subscriptionType", {})
        sim_type_code = ""
        offer_name = "—"
        if isinstance(sub_type, dict):
            names = sub_type.get("name", {})
            sim_type_code = str(sub_type.get("code", "") or "").strip()
            if isinstance(names, dict):
                offer_name = names.get("ar") or names.get("en") or sim_type_code or "—"
            else:
                offer_name = sim_type_code or "—"
        connected = data.get("connectedProducts") or data.get("connected-products") or []
        connected_names = []
        if isinstance(connected, list):
            for cp in connected[:5]:
                if isinstance(cp, dict):
                    n = cp.get("name") or cp.get("description") or cp.get("code", "")
                    if isinstance(n, dict): n = n.get("ar") or n.get("en") or ""
                    if n: connected_names.append(str(n))
        _bal_raw = data.get("mainBalance")
        real_sim_name = offer_name
        if offer_name and offer_name != "—" and sim_type_code and sim_type_code.upper() not in offer_name.upper():
            real_sim_name = f"{offer_name} ({sim_type_code})"
        elif offer_name == "—" and sim_type_code:
            real_sim_name = sim_type_code
        return {
            "balance": str(_bal_raw) if _bal_raw is not None else None,
            "payment_type": payment_type,
            "real_sim_name": real_sim_name,
            "connected_products": connected_names,
        }
    except Exception as e:
        logger.warning(f"full_info: {e}")
    return None


def api_subscription_history(uid, phone):
    token = get_token(uid)
    if not token: return None
    try:
        resp = _session.get(
            f"{BASE_URL}/api/v1/subscribers/subscription-history/{phone}",
            headers={**_BASE_H, "Authorization": token, "accept-language":"fr"},
            timeout=15)
        if resp.status_code == 200:
            data = resp.json()
            if isinstance(data,dict) and data.get("status") == 200:
                return data.get("data",[])
            return data if isinstance(data,list) else None
    except Exception: pass
    return None


def _last_walk_win_date_typed(history, pkg_code):
    for item in (history or []):
        if item.get("packageCode","") == pkg_code:
            dt_str = item.get("subscriptionDateTime")
            if dt_str:
                try:
                    from dateutil import parser as dp
                    return int(dp.parse(dt_str).timestamp())
                except Exception:
                    try:
                        return int(datetime.fromisoformat(dt_str[:19]).timestamp())
                    except Exception: pass
    return None


def get_offer_state_real(uid, phone, offer, history=_UNSET_HISTORY):
    pkg_code = PKG_1GO if offer == "1g" else (PKG_6GB if offer == "shake_6gb" else PKG_2GO)
    cooldown_s = 86400 if offer in ("1g", "shake_6gb") else 7*24*3600
    if history is _UNSET_HISTORY:
        history = api_subscription_history(uid, phone)
    if history is not None:
        last_ts = _last_walk_win_date_typed(history, pkg_code)
        if not last_ts: return STATE_READY, 0, True
        elapsed = time.time() - last_ts
        if elapsed >= cooldown_s: return STATE_READY, 0, True
        return STATE_WAITING, int((cooldown_s - elapsed) * 1000), True
    st, rem = get_offer_state(phone, offer)
    return st, rem, False


_NEED_PAID_PKG_HINTS = ("غير مؤهل", "not eligible", "eligibility not found", "eligib",
                        "تعذر معالجة", "لا وجود للمكافأة", "no active", "باقة")


def api_activate_walk(uid, phone, pkg_code):
    token = get_token(uid)
    if not token: return False, "سجل دخول"
    headers = {**_BASE_H, "Content-Type":"application/json",
               "Authorization":token, "User-Agent":USER_AGENT_V2}
    urls = [f"{BASE_URL}/api/v1/services/walk/activate-reward/{phone}",
            f"{BASE_URL}/api/v2/services/walk/activate-reward/{phone}"]
    last_msg = None
    for attempt in range(1, 4):
        for url in urls:
            try:
                resp = _session.post(url, headers=headers, json={"packageCode":pkg_code}, timeout=15)
                if resp.status_code in (200,201,204): return True, "OK"
                if resp.status_code in (402, 403): return False, "NEED_100DZ"
                if resp.status_code == 429:
                    time.sleep(0.7); continue
                try: body = resp.json()
                except Exception: body = {}
                fault = body.get("fault", {})
                if isinstance(fault, dict) and int(fault.get("code", 0)) == 900901:
                    row = db_get_token_row(uid)
                    if row and row["refresh_token"]:
                        new_tok = _refresh_access_token(uid, row["refresh_token"])
                        if new_tok: headers["Authorization"] = new_tok
                    continue
                if resp.status_code in (400, 404, 422):
                    real = _extract_djezzy_error(resp)
                    if real and any(h.lower() in real.lower() for h in _NEED_PAID_PKG_HINTS):
                        return False, "NEED_100DZ"
                    if real: last_msg = real
            except Exception as e:
                logger.warning(f"walk: {e}")
        if attempt < 3: time.sleep(0.3)
    return False, (last_msg or "فشل التفعيل — حاول لاحقاً")


def api_activate_shake_6gb(uid, phone):
    token = get_token(uid)
    if not token: 
        return False, "سجل دخول"
    
    url = f"{BASE_URL}/api/v1/services/shake/{phone}"
    headers = {
        **_BASE_H, 
        "Content-Type": "application/json",
        "Authorization": token, 
        "User-Agent": USER_AGENT_V2
    }
    
    payload = {"packageCode": PKG_6GB}
    
    for attempt in range(1, 4):
        try:
            resp = _session.post(url, headers=headers, json=payload, timeout=15)
            if resp.status_code in (200, 201, 204): 
                return True, "OK"
            
            if resp.status_code == 429:
                time.sleep(0.7)
                continue
                
            try: 
                body = resp.json()
            except Exception: 
                body = {}
                
            fault = body.get("fault", {})
            if isinstance(fault, dict) and int(fault.get("code", 0)) == 900901:
                row = db_get_token_row(uid)
                if row and row["refresh_token"]:
                    new_tok = _refresh_access_token(uid, row["refresh_token"])
                    if new_tok: 
                        headers["Authorization"] = new_tok
                continue
                
            real = _extract_djezzy_error(resp)
            if real: 
                return False, real
                
        except Exception as e:
            logger.warning(f"shake 6gb activation error: {e}")
            
        if attempt < 3: 
            time.sleep(0.3)
            
    return False, "فشل تفعيل حزمة 6GB — حاول لاحقاً"


def api_activate_product(uid, phone, pkg_code):
    token = get_token(uid)
    if not token: return False, "سجل دخول"
    url = f"{BASE_URL}/api/v1/subscribers/activate-product/{phone}"
    headers = {**_BASE_H, "Content-Type":"application/json",
               "Authorization":token, "User-Agent":USER_AGENT_V2, "accept-language":"fr"}
    for attempt in range(1, 3):
        try:
            resp = _session.post(url, headers=headers, json={"packageCode":pkg_code}, timeout=12)
            if resp.status_code in (200,201,204): return True, "OK"
            if resp.status_code == 402:
                real = _extract_djezzy_error(resp)
                try:
                    bal_data = resp.json()
                    balance = (bal_data.get("data") or {}).get("mainBalance")
                    if balance is not None:
                        return False, (real or "رصيدك غير كافٍ") + f"\nرصيدك: {balance} دج"
                except Exception: pass
                return False, real or "رصيدك غير كافٍ"
            if resp.status_code == 403:
                real = _extract_djezzy_error(resp)
                return False, real or "يلزمك الاشتراك في باقة 100دج أو أكثر"
            if resp.status_code in (400, 404, 422):
                real = _extract_djezzy_error(resp)
                if real: return False, real
            if resp.status_code == 429: time.sleep(0.7); continue
            try:
                data = resp.json()
                fault = data.get("fault",{})
                if isinstance(fault,dict) and int(fault.get("code",0)) == 900901:
                    row = db_get_token_row(uid)
                    if row and row["refresh_token"]:
                        new_tok = _refresh_access_token(uid, row["refresh_token"])
                        if new_tok: headers["Authorization"] = new_tok
                    continue
                real = _extract_djezzy_error(resp)
                if real: return False, real
            except Exception: pass
        except Exception as e:
            logger.warning(f"buy: {e}")
    return False, "تعذر التفعيل"


def api_fetch_mgm_invitations(uid, phone):
    token = get_token(uid)
    if not token: return None
    try:
        resp = _session.get(
            f"{BASE_URL}/api/v1/services/mgm/invitations/{phone}",
            headers={**_BASE_H, "Authorization":token, "User-Agent":USER_AGENT_V2},
            timeout=15)
        if resp.status_code == 200:
            data = resp.json()
            if isinstance(data,dict) and data.get("status") == 200:
                return data.get("data",{})
            return data
    except Exception: pass
    return None


def api_send_mgm_invitation(uid, sender, receiver):
    token = get_token(uid)
    if not token: return False, "سجل دخول"
    url = f"{BASE_URL}/api/v1/services/mgm/send-invitation/{sender}"
    headers = {**_BASE_H, "Content-Type":"application/json",
               "Authorization":token, "User-Agent":USER_AGENT_V2}
    try:
        resp = _session.post(url, headers=headers, json={"msisdnReciever":receiver}, timeout=15)
        if resp.status_code in (200,201): return True, "OK"
        if resp.status_code == 400:
            try:
                data = resp.json()
                msg = data.get("message",{})
                ar = msg.get("ar","") if isinstance(msg,dict) else str(msg)
                if "وصلت إلى الحد الأقصى" in ar: return False, "وصلت للحد الأقصى (5 دعوات)"
                if "تمت دعوة هذا المستلم" in ar or "هذه العملية غير متوفرة" in ar:
                    return False, "هذا الرقم سبق دعوته"
                if "العميل غير موجود" in ar: return False, "الرقم غير موجود في Djezzy"
            except Exception: pass
            real = _extract_djezzy_error(resp)
            return False, real or "خطأ في الإرسال"
        if resp.status_code == 403: return False, "هذا الرقم سبق دعوته"
        real = _extract_djezzy_error(resp)
        return False, real or f"خطأ {resp.status_code}"
    except Exception as e:
        logger.error(f"send_mgm: {e}")
        return False, "خطأ في الاتصال"


def api_delete_pending_invitations(uid, phone, pending_ids):
    token = get_token(uid)
    if not token or not pending_ids: return 0, 0
    base_url = f"{BASE_URL}/api/v1/services/mgm/delete-invitation/{phone}"
    headers_json = {**_BASE_H, "Content-Type":"application/json",
                    "Authorization":token, "User-Agent":USER_AGENT_V2}
    ok_count = 0
    for inv_id in pending_ids:
        if inv_id is None: continue
        deleted = False
        attempts = [
            ("POST", base_url, {"invitationId": inv_id}),
            ("POST", base_url, {"invitationId": inv_id, "msisdn": phone}),
            ("DELETE", base_url, {"invitationId": inv_id}),
        ]
        for method, url, body in attempts:
            try:
                if method == "DELETE":
                    resp = _session.delete(url, headers=headers_json, json=body, timeout=15)
                else:
                    resp = _session.post(url, headers=headers_json, json=body, timeout=15)
                if resp.status_code in (200, 201, 204, 404):
                    deleted = True; break
            except Exception: pass
        if deleted: ok_count += 1
    return ok_count, len(pending_ids) - ok_count


def api_activate_mgm_reward(uid, phone, pkg_code):
    token = get_token(uid)
    if not token: return "ERROR"
    url = f"{BASE_URL}/api/v1/services/mgm/activate-reward/{phone}"
    headers = {**_BASE_H, "Content-Type":"application/json",
               "Authorization":token, "User-Agent":USER_AGENT_V2}
    for attempt in range(1, 4):
        try:
            resp = _session.post(url, headers=headers, json={"packageCode":pkg_code}, timeout=15)
            if resp.status_code in (200,201):
                return "SUCCESS"
            if resp.status_code == 404:
                try:
                    data = resp.json()
                    msg = data.get("message","")
                    if isinstance(msg,str) and "Eligibility not found" in msg:
                        return "ALREADY_CLAIMED"
                    if isinstance(msg,dict) and "لا وجود للمكافأة" in msg.get("ar",""):
                        return "REWARD_NOT_EXIST"
                except Exception: pass
                return "ALREADY_CLAIMED"
            if resp.status_code == 400:
                try:
                    data = resp.json()
                    msg = data.get("message",{})
                    ar = msg.get("ar","") if isinstance(msg,dict) else str(msg)
                    if "تعذر معالجة طلبك" in ar: return "ALREADY_CLAIMED"
                except Exception: pass
        except Exception: pass
        time.sleep(0.3)
    return "ERROR"


def api_try_mgm_bonus(uid, phone):
    r1 = api_activate_mgm_reward(uid, phone, "MGMBONUS1Go")
    if r1 == "SUCCESS":         return "SUCCESS_1GO"
    if r1 == "ALREADY_CLAIMED": return "ALREADY_CLAIMED"
    if r1 == "REWARD_NOT_EXIST":
        r2 = api_activate_mgm_reward(uid, phone, "MGMBONUS500Mo")
        if r2 == "SUCCESS":         return "SUCCESS_500MO"
        if r2 == "ALREADY_CLAIMED": return "ALREADY_CLAIMED"
        return "REWARD_NOT_EXIST"
    return "ERROR"


REQUIRED_CHANNELS = [
    {"id": "@NETXPERT_DZ_bot_TFIAL", "link": "https://t.me/NETXPERT_DZ_bot_TFIAL", "name": "NETXPERT DZ TFIAL"},
    {"id": "@Dark_1_DZ", "link": "https://t.me/Dark_1_DZ", "name": "Dark 1 DZ"}
]

async def check_force_subscription(bot, uid):
    for channel in REQUIRED_CHANNELS:
        try:
            member = await bot.get_chat_member(chat_id=channel["id"], user_id=uid)
            if member.status in ['left', 'kicked']:
                return False, channel
        except Exception as e:
            logger.warning(f"Error checking subscription for {channel['id']}: {e}")
            pass
    return True, None


async def send_proof_to_channel(bot, user, offer_name, offer_size, phone, note=None):
    proof = get_proof_channel()
    if not proof:
        return False, "قناة الإثبات غير مُعدّة"

    uname = f"@{user.username}" if user.username else "بدون يوزر"
    uid = user.id
    name = user.first_name or "مستخدم"

    msg = (
        f"✅ <b>إثبات تفعيل جديد</b>\n"
        f"━━━━━━━━━━━━━━━━━━━\n"
        f"👤 الاسم: <b>{name}</b>\n"
        f"🆔 المستخدم: <code>{uid}</code>\n"
        f"📱 الحساب: {uname}\n"
        f"📞 الرقم: <code>{mask_phone(phone)}</code>\n"
        f"━━━━━━━━━━━━━━━━━━━\n"
        f"🎁 العرض: <b>{offer_name}</b>\n"
        f"📦 الحجم: <b>{offer_size}</b>\n"
        f"🕐 الوقت: <code>{datetime.now().strftime('%Y-%m-%d %H:%M')}</code>\n"
    )
    if note:
        msg += f"\n📝 ملاحظة: {note}\n"
    msg += f"\n🤖 ‏dz_net2_bot"

    try:
        await send_ce(bot, proof["id"], msg)
        return True, "OK"
    except Exception as e:
        logger.error(f"send_proof_to_channel: {e}")
        return False, str(e)


def msg_welcome(name):
    return (f"👑 مرحباً بك في بوت خدمات جيزي\n"
            f"━━━━━━━━━━━━━━━━━━━\n"
            f"⭐ <b>المميزات :</b>\n\n"
            f"🎁 • هدايا الإنترنت المجانية (بما فيها 6GB Shake)\n"
            f"💎 • باقات مدفوعة (IZZI, ZID, MAXI, BEZZEF, IMTIYAZ)\n"
            f"💰 • كشف الرصيد فوراً\n"
            f"📞 • دعم جميع أرقام جيزي\n"
            f"🚀 • تفعيل سريع وفوري\n"
            f"━━━━━━━━━━━━━━━━━━━\n"
            f"ℹ️ اختر الخدمة المناسبة من الأزرار أدناه\n\n"
            f"👤 أهلاً <b>{name}</b>")


def msg_login_required():
    return (f"{EMOJI_LOGIN} <b>يجب تسجيل الدخول أولاً</b>\n\n"
            f"{EMOJI_PHONE} أرسل رقم هاتفك المسجّل في Djezzy:\n"
            f"<code>07XXXXXXXX</code>\n\n"
            f"{EMOJI_CLOCK} الرمز سيصلك في رسالة SMS")


def msg_otp_sent(phone):
    return (f"{EMOJI_SEND} تم إرسال رمز التفعيل إلى <code>{phone}</code>\n\n"
            f"{EMOJI_CLOCK} الرمز صالح لمدة دقيقتين\n"
            f"اكتب الرمز المكوّن من 4 إلى 6 خانات:")


def msg_login_success(name, phone, username):
    uname = f"@{username}" if username else "غير محدّد"
    return (f"{EMOJI_CHECK} تم الدخول إلى حسابك يا <b>{name}</b>\n\n"
            f"الحساب: {uname}\n"
            f"الرقم: <code>{phone}</code>\n"
            f"المشغّل: Djezzy\n\n"
            f"اختر خدمة من القائمة")


def msg_offer_confirm(offer_name, size, phone):
    return (f"{EMOJI_GIFT} تأكيد طلب: <b>{offer_name}</b>\n\n"
            f"الرقم: <code>{phone}</code>\n"
            f"الحجم: {size}\n\n"
            f"هل تُتابع؟")


def msg_offer_waiting(offer_name, remain):
    return (f"{EMOJI_HOURGLASS} <b>{offer_name}</b> غير متاح الآن\n\n"
            f"الوقت المتبقي: <code>{remain}</code>\n"
            f"سنُعلمك فور توفّره")


def msg_offer_fail(offer_name, err):
    return (f"{EMOJI_ERROR} تعذّر تفعيل <b>{offer_name}</b>\n\n"
            f"السبب: {err}\n"
            f"جرّب عرضاً آخر أو أعد المحاولة لاحقاً")


def msg_balance(balance):
    return f"{EMOJI_BALANCE} رصيدك الحالي: <code>{balance} دج</code>"


def msg_account(name, username, phone, status1, status2, status6gb, invites, balance,
                sim_name=None, connected_products=None, saved_phones=None):
    uname = f"@{username}" if username else "غير محدّد"
    sim_line = f"\nنوع الشريحة: {sim_name}" if sim_name else ""
    active_line = ""
    if connected_products:
        active_line = "\n" + "\n".join(f"— {p}" for p in connected_products[:5])
    phones_line = ""
    if saved_phones and len(saved_phones) > 1:
        phones_line = "\n\n📱 <b>الأرقام المحفوظة:</b>\n" + "\n".join(
            f"— <code>{mask_phone(p)}</code>" for p in saved_phones)
    return (f"{EMOJI_CARD} <b>بطاقة الحساب</b>\n\n"
            f"الاسم: <b>{name}</b>\n"
            f"الحساب: {uname}\n"
            f"الرقم: <code>{phone}</code>\n"
            f"المشغّل: Djezzy\n"
            f"الرصيد: <code>{balance}</code>{sim_line}\n\n"
            f"{EMOJI_GIFT} الحزم المجانية\n"
            f"— 1GB: {status1}\n"
            f"— 2GB: {status2}\n"
            f"— 6GB Shake: {status6gb}\n"
            f"— دعوات مُرسَلة: <code>{invites}</code>{active_line}{phones_line}")


def msg_invite_start(count, remaining):
    return (f"{EMOJI_PARTY} <b>دعوة مستخدم جديد</b>\n\n"
            f"المُرسَل: <code>{count}/{MAX_INVITES}</code>\n"
            f"المتبقي: <code>{remaining}</code>\n\n"
            f"أرسل رقم صديقك (Djezzy):\n"
            f"<code>07XXXXXXXX</code>")


def msg_invite_result(success, m, count, remaining):
    if success:
        return f"{EMOJI_CHECK} {m}\nالإجمالي: <code>{count}/{MAX_INVITES}</code> — المتبقي: <code>{remaining}</code>"
    return f"{EMOJI_ERROR} {m}"


def msg_offers_menu():
    return (f"{EMOJI_BOX} <b>الباقات المدفوعة المتاحة</b>\n\n"
            f"{EMOJI_PIN} 📦 IZZI\n"
            f"{EMOJI_PIN} ⚡ ZID\n"
            f"{EMOJI_PIN} 🔥 BEZZEF\n"
            f"{EMOJI_PIN} 🌟 MAXI\n"
            f"{EMOJI_PIN} 👑 IMTIYAZ\n\n"
            f"اختر الفئة المناسبة")


def kb_main(uid=None, admin=False):
    phone = db_get_phone(uid) if uid else None
    rows = [
        [btn("حزمة 1GB", "walk_1g"), btn("حزمة 2GB", "walk_2g")],
        [btn("🎁 تفعيل 6GB Shake", "shake_6gb")],
        [btn("📊 حالة العروض", "offer_status")],
        [btn("دعوة مستخدم", "invite")],
        [btn("💎 الباقات المدفوعة", "show_offers")],
        [btn("الرصيد", "balance"), btn("بطاقتي", "account")],
        [btn("🚨 بلاغ عن مشكل", "report_issue"), btn("🎧 دعم فني", "tech_support")],
    ]
    if phone:
        rows.append([btn("🔄 تغيير الرقم", "change_number")])
    else:
        rows.append([btn("تسجيل الدخول", "login")])
    if admin:
        rows.append([btn("الإدارة", "admin_panel")])
    return InlineKeyboardMarkup(rows)


def kb_login_required():
    return InlineKeyboardMarkup([
        [btn("تسجيل الدخول الآن", "login")],
        [btn("رجوع للقائمة", "back_menu")],
    ])


def kb_confirm(action):
    return InlineKeyboardMarkup([
        [btn("تأكيد", f"confirm_{action}"), btn("إلغاء", "back_menu")],
    ])


def kb_retry():
    return InlineKeyboardMarkup([
        [btn("🔄 إعادة الإرسال", "retry_otp")],
        [btn("📱 رقم مختلف", "change_number")],
        [btn("🏠 رجوع", "back_menu")],
    ])


def kb_offers_categories():
    return InlineKeyboardMarkup([
        [btn("📦 IZZI", "offers_cat_izzy"), btn("⚡ ZID", "offers_cat_zid")],
        [btn("🔥 BEZZEF", "offers_cat_bezzef"), btn("🌟 MAXI", "offers_cat_maxi")],
        [btn("👑 IMTIYAZ", "offers_cat_imtiyaz")],
        [btn("رجوع", "back_menu")],
    ])


def _offers_list_kb(pkg_list):
    rows = [[btn(OFFERS.get(c,{}).get("name",c), f"buy_{c}")] for c in pkg_list]
    rows.append([btn("رجوع", "show_offers")])
    return InlineKeyboardMarkup(rows)


def kb_izzy():    return _offers_list_kb(OFFERS_IZZY)
def kb_zid():     return _offers_list_kb(OFFERS_ZID)
def kb_bezzef():  return _offers_list_kb(OFFERS_BEZZEF)
def kb_maxi():    return _offers_list_kb(OFFERS_MAXI)
def kb_imtiyaz(): return _offers_list_kb(OFFERS_IMTIYAZ)


def kb_cancel_invite():
    return InlineKeyboardMarkup([[btn("إلغاء العملية", "back_menu")]])


def kb_cancel_report():
    return InlineKeyboardMarkup([[btn("إلغاء البلاغ", "back_menu")]])


def kb_admin():
    return InlineKeyboardMarkup([
        [btn("إعلان جماعي", "admin_broadcast"), btn("لوحة الأرقام", "admin_stats")],
        [btn("سجل المستخدمين", "admin_users"), btn("حظر", "admin_ban")],
        [btn("فك الحظر", "admin_unban"), btn("تصفير الجلسات", "admin_clear_tokens")],
        [btn("📸 إدارة قناة الإثبات", "admin_proof_channel")],
        [btn("🚨 البلاغات", "admin_reports")],
        [btn("رجوع", "back_menu")],
    ])


def kb_admin_report(report_id):
    return InlineKeyboardMarkup([
        [btn("✍️ رد على المشكل", f"admin_reply_report_{report_id}")],
        [btn("🗑 تجاهل البلاغ", f"admin_ignore_report_{report_id}")],
    ])


executor = ThreadPoolExecutor(max_workers=10)


async def run_sync(fn, *args):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(executor, fn, *args)


def _admin_stats():
    proof = get_proof_channel()
    proof_line = f"📸 قناة الإثبات: <code>{proof['name']}</code>" if proof else "📸 قناة الإثبات: غير مُعدّة"
    open_reports = len(db_get_open_reports())
    return (f"{EMOJI_SHIELD} <b>لوحة الإدارة</b>\n\n"
            f"{EMOJI_USERS} عدد المستخدمين: <code>{db_user_count()}</code>\n"
            f"{EMOJI_GIFT} عدد التفعيلات: <code>{db_total_activations()}</code>\n"
            f"🚨 بلاغات مفتوحة: <code>{open_reports}</code>\n"
            f"{proof_line}")


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        if update.effective_chat.type != "private": return
        uid = update.effective_user.id
        if db_is_banned(uid):
            await send_ce(context.bot, update.effective_chat.id, f"{EMOJI_BAN} حسابك محظور.")
            return

        is_subbed, missing_ch = await check_force_subscription(context.bot, uid)
        if not is_subbed and missing_ch:
            kb = InlineKeyboardMarkup([
                [btn("📢 اشترك في القناة الأولى", url=REQUIRED_CHANNELS[0]["link"])],
                [btn("📢 اشترك في القناة الثانية", url=REQUIRED_CHANNELS[1]["link"])],
                [btn("🔄 تحقق من الاشتراك", "start")]
            ])
            await send_ce(context.bot, update.effective_chat.id,
                          f"⚠️ <b>عذراً، يجب عليك الاشتراك في قنواتنا أولاً لاستخدام البوت:</b>\n\n"
                          f"1️⃣ {REQUIRED_CHANNELS[0]['name']}\n"
                          f"2️⃣ {REQUIRED_CHANNELS[1]['name']}\n\n"
                          f"بعد الاشتراك، اضغط على زر <b>تحقق من الاشتراك</b> بالأسفل 👇",
                          reply_markup=kb)
            return

        name = update.effective_user.first_name or "صديقي"
        username = update.effective_user.username or ""
        db_add_user(uid, name, username)

        phone = db_get_phone(uid)
        if phone:
            await send_ce(context.bot, update.effective_chat.id,
                          f"{EMOJI_CHECK} أهلاً بك <b>{name}</b>\n\n"
                          f"رقمك: <code>{mask_phone(phone)}</code>",
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
        else:
            await send_ce(context.bot, update.effective_chat.id,
                          msg_welcome(name),
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
    except Exception as e:
        logger.exception(f"start error: {e}")
        try:
            await context.bot.send_message(
                update.effective_chat.id,
                "⚠️ حدث خطأ غير متوقع. جرّب /start مرة أخرى."
            )
        except Exception:
            pass


async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.type != "private": return
    await send_ce(context.bot, update.effective_chat.id,
                  f"{EMOJI_BOOK} <b>دليل الاستخدام</b>\n\n"
                  f"1) اضغط <b>تسجيل الدخول</b>\n"
                  f"2) أرسل رقم هاتفك <code>07XXXXXXXX</code>\n"
                  f"3) أدخل رمز OTP\n"
                  f"4) استخدم الأزرار لتفعيل 6GB Shake أو الحزم الأخرى\n"
                  f"5) استعمل زر <b>بلاغ عن مشكل</b> لأي مشكلة\n"
                  f"6) استعمل زر <b>دعم فني</b> للتواصل المباشر")


async def admin_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.type != "private": return
    if update.effective_user.id != ADMIN_ID:
        await send_ce(context.bot, update.effective_chat.id, f"{EMOJI_BAN} للإدارة فقط"); return
    await send_ce(context.bot, update.effective_chat.id, _admin_stats(),
                  reply_markup=kb_admin())


async def on_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    try:
        await q.answer()
    except Exception:
        pass
    try:
        await _handle_callback(update, context)
    except Exception as e:
        logger.exception(f"on_callback error: {e}")
        try:
            await context.bot.send_message(q.message.chat.id, "⚠️ حدث خطأ، حاول مجدداً.")
        except Exception:
            pass


async def _handle_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    uid = q.from_user.id
    data = q.data
    chat_id = q.message.chat.id

    if db_is_banned(uid):
        await q.answer("محظور", show_alert=True); return

    if data == "start":
        await start(update, context)
        return

    if data == "login" or data == "change_number":
        if data == "change_number":
            db_clear_tokens(uid)
        context.user_data["waiting"] = "phone"
        await edit_ce(context.bot, chat_id, q.message.message_id, msg_login_required())
        return

    if data == "back_menu":
        name = q.from_user.first_name or "صديقي"
        context.user_data.pop("waiting", None)
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_BOT} <b>القائمة الرئيسية</b>\n\nأهلاً {name}",
                      reply_markup=kb_main(uid, uid == ADMIN_ID))
        return

    if data == "retry_otp":
        context.user_data["waiting"] = "phone"
        await edit_ce(context.bot, chat_id, q.message.message_id, msg_login_required())
        return

    if data == "report_issue":
        context.user_data["waiting"] = "report"
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_REPORT} <b>بلاغ عن مشكل</b>\n\n"
                      f"✍️ اكتب وصف المشكل بالتفصيل وسيصل إلى الإدارة فوراً:\n\n"
                      f"{EMOJI_PIN} يمكنك إرسال:\n"
                      f"— نص فقط\n"
                      f"— صورة مع وصف\n"
                      f"— ملف أو مستند\n\n"
                      f"{EMOJI_WARNING} <i>سيتم الرد عليك من قبل الإدارة قريباً</i>",
                      reply_markup=kb_cancel_report())
        return

    if data == "tech_support":
        support_url = f"https://t.me/{ADMIN_USERNAME}"
        kb = InlineKeyboardMarkup([
            [btn(f"🎧 مراسلة الدعم الفني", url=support_url)],
            [btn("رجوع", "back_menu")],
        ])
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_SUPPORT} <b>الدعم الفني</b>\n\n"
                      f"👤 للتواصل المباشر مع الإدارة:\n"
                      f"@{ADMIN_USERNAME}\n\n"
                      f"اضغط الزر أدناه للمراسلة الفورية 👇",
                      reply_markup=kb)
        return

    phone = db_get_phone(uid)
    if not phone:
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_LOGIN} <b>يجب تسجيل الدخول أولاً</b>\n\n"
                      f"{EMOJI_PHONE} أضغط على الزر بالأسفل:",
                      reply_markup=kb_login_required())
        return

    if data == "walk_1g":
        st, rem = get_offer_state(phone, "1g")
        if st == STATE_WAITING:
            await edit_ce(context.bot, chat_id, q.message.message_id,
                          msg_offer_waiting("حزمة 1GB", fmt_time(rem)),
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
            return
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      msg_offer_confirm("حزمة 1GB","1GB",mask_phone(phone)),
                      reply_markup=kb_confirm("walk_1g"))
        return

    if data == "walk_2g":
        st, rem = get_offer_state(phone, "2g")
        if st == STATE_WAITING:
            await edit_ce(context.bot, chat_id, q.message.message_id,
                          msg_offer_waiting("حزمة 2GB", fmt_time(rem)),
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
            return
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      msg_offer_confirm("حزمة 2GB","2GB",mask_phone(phone)),
                      reply_markup=kb_confirm("walk_2g"))
        return

    if data == "shake_6gb":
        st, rem = get_offer_state(phone, "shake_6gb")
        if st == STATE_WAITING:
            await edit_ce(context.bot, chat_id, q.message.message_id,
                          msg_offer_waiting("حزمة 6GB Shake", fmt_time(rem)),
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
            return
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      msg_offer_confirm("حزمة 6GB (Shake)", "6GB", mask_phone(phone)),
                      reply_markup=kb_confirm("shake_6gb"))
        return

    if data == "offer_status":
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_HOURGLASS} جارٍ التحقق من حالة العروض...")

        history = await run_sync(api_subscription_history, uid, phone)

        st1, rem1, conf1 = get_offer_state_real(uid, phone, "1g", history=history)
        st2, rem2, conf2 = get_offer_state_real(uid, phone, "2g", history=history)
        st6, rem6, conf6 = get_offer_state_real(uid, phone, "shake_6gb", history=history)

        msg = (f"📱 <b>الرقم :</b> <code>{mask_phone(phone)}</code>\n"
               f"━━━━━━━━━━━━━━━━━━━\n\n")

        msg += f"🎁 <b>عرض 1 جيغا</b>\n"
        if st1 == STATE_READY:
            msg += f"📆 الحالة : ✅ جاهز للتفعيل\n"
        elif st1 == STATE_WAITING:
            msg += f"📆 الحالة : ⏳ لم يكتمل اليوم\n"
            msg += f"⏳ متبقي : <code>{fmt_seconds_detailed(rem1 // 1000)}</code>\n"
        else:
            msg += f"📆 الحالة : ❓ لم يُفعَّل\n"
        if not conf1:
            msg += "ℹ️ <i>(تقدير محلي)</i>\n"

        msg += f"\n━━━━━━━━━━━━━━━━━━━\n\n"

        msg += f"🎉 <b>عرض 2 جيغا</b>\n"
        if st2 == STATE_READY:
            msg += f"📆 الحالة : ✅ جاهز للتفعيل\n"
        elif st2 == STATE_WAITING:
            msg += f"📆 الحالة : ⏳ لم يكتمل الأسبوع\n"
            msg += f"⏳ متبقي : <code>{fmt_seconds_detailed(rem2 // 1000)}</code>\n"
        else:
            msg += f"📆 الحالة : ❓ لم يُفعَّل\n"
        if not conf2:
            msg += "ℹ️ <i>(تقدير محلي)</i>\n"

        msg += f"\n━━━━━━━━━━━━━━━━━━━\n\n"

        msg += f"⚡ <b>عرض 6GB Shake</b>\n"
        if st6 == STATE_READY:
            msg += f"📆 الحالة : ✅ جاهز للتفعيل\n"
        elif st6 == STATE_WAITING:
            msg += f"📆 الحالة : ⏳ لم يكتمل اليوم\n"
            msg += f"⏳ متبقي : <code>{fmt_seconds_detailed(rem6 // 1000)}</code>\n"
        else:
            msg += f"📆 الحالة : ❓ لم يُفعَّل\n"
        if not conf6:
            msg += "ℹ️ <i>(تقدير محلي)</i>\n"

        await edit_ce(context.bot, chat_id, q.message.message_id, msg,
                      reply_markup=kb_main(uid, uid == ADMIN_ID))
        return

    if data == "invite":
        await _do_invite_flow(update, context, uid, phone)
        return

    if data == "balance":
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_HOURGLASS} جارٍ استرجاع الرصيد...")
        bal, err = await run_sync(api_get_balance, uid, phone)
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      msg_balance(bal if bal else (err or "غير معروف")),
                      reply_markup=kb_main(uid, uid == ADMIN_ID))
        return

    if data == "account":
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_HOURGLASS} جارٍ تحديث بيانات الحساب...")
        user = db_get_user(uid)
        name = (user["name"] if user else None) or q.from_user.first_name or "مستخدم"
        username = (user["username"] if user else None) or q.from_user.username or ""

        history = await run_sync(api_subscription_history, uid, phone)
        st1, rem1, conf1 = get_offer_state_real(uid, phone, "1g", history=history)
        st2, rem2, conf2 = get_offer_state_real(uid, phone, "2g", history=history)
        st6, rem6, conf6 = get_offer_state_real(uid, phone, "shake_6gb", history=history)

        def _s(st, rem, confirmed):
            base = ("جاهز" if st == STATE_READY else
                    (f"{fmt_time(rem)}" if st == STATE_WAITING else "لم يُفعَّل"))
            return base if confirmed else f"{base} (تقدير محلي)"

        inv_data = await run_sync(api_fetch_mgm_invitations, uid, phone)
        if isinstance(inv_data, dict):
            max_inv = inv_data.get("campaign", {}).get("maxInvitation", MAX_INVITES)
            inv_list = inv_data.get("invitations", [])
            done = sum(1 for i in inv_list if i.get("status") == "DONE")
            invites_str = f"{done}/{max_inv}"
        else:
            invites_str = f"{db_invite_count(uid)}/{MAX_INVITES} (محلي)"

        info = await run_sync(api_get_full_account_info, uid, phone)
        if info:
            balance_str = f"{info['balance']} دج" if info.get("balance") else "غير معروف"
            sim_name = info.get("real_sim_name")
            connected = info.get("connected_products")
        else:
            bal, _ = await run_sync(api_get_balance, uid, phone)
            balance_str = f"{bal} دج" if bal else "غير معروف"
            sim_name = None
            connected = None

        saved_phones = db_get_user_phones(uid)

        await edit_ce(context.bot, chat_id, q.message.message_id,
                      msg_account(name, username, mask_phone(phone),
                                  _s(st1, rem1, conf1), _s(st2, rem2, conf2), _s(st6, rem6, conf6),
                                  invites_str, balance_str,
                                  sim_name=sim_name, connected_products=connected,
                                  saved_phones=saved_phones),
                      reply_markup=kb_main(uid, uid == ADMIN_ID))
        return

    if data == "show_offers":
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      msg_offers_menu(), reply_markup=kb_offers_categories())
        return

    if data.startswith("offers_cat_"):
        cat = data.replace("offers_cat_", "")
        cat_map = {
            "izzy":    ("📦 عروض IZZI",    kb_izzy()),
            "zid":     ("⚡ عروض ZID",     kb_zid()),
            "bezzef":  ("🔥 عروض BEZZEF",  kb_bezzef()),
            "maxi":    ("🌟 عروض MAXI",    kb_maxi()),
            "imtiyaz": ("👑 عروض IMTIYAZ", kb_imtiyaz()),
        }
        title, kb = cat_map.get(cat, ("العروض", kb_offers_categories()))
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_BOX} <b>{title}</b>\n\nاختر العرض:",
                      reply_markup=kb)
        return

    if data.startswith("confirm_walk_"):
        act = data.replace("confirm_walk_", "")
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_HOURGLASS} جارٍ تفعيل العرض...")
        asyncio.create_task(_run_walk(update, context, uid, phone, act))
        return

    if data == "confirm_shake_6gb":
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_HOURGLASS} جارٍ تفعيل باقة 6GB Shake...")
        asyncio.create_task(_run_shake_6gb(update, context, uid, phone))
        return

    if data.startswith("buy_") and not data.startswith("buy_confirm_"):
        pkg = data.replace("buy_", "")
        info = OFFERS.get(pkg, {})
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_BOX} <b>{info.get('name',pkg)}</b>\n"
                      f"{info.get('display','')}\n"
                      f"💰 {info.get('price','')}\n"
                      f"⏰ {info.get('duration','')}\n"
                      f"{EMOJI_PHONE} <code>{mask_phone(phone)}</code>\n\n"
                      f"نُتابع التفعيل؟",
                      reply_markup=InlineKeyboardMarkup([
                          [btn("تأكيد الشراء", f"buy_confirm_{pkg}")],
                          [btn("إلغاء", "back_menu")],
                      ]))
        return

    if data.startswith("buy_confirm_"):
        pkg = data.replace("buy_confirm_", "")
        info = OFFERS.get(pkg, {})
        label = info.get("name", pkg)
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_HOURGLASS} جارٍ تفعيل {label}...")
        asyncio.create_task(_run_buy(update, context, uid, phone, pkg))
        return

    if data == "admin_panel" or data == "admin_stats":
        if uid != ADMIN_ID: return
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      _admin_stats(), reply_markup=kb_admin())
        return

    if data == "admin_users":
        if uid != ADMIN_ID: return
        users = db_all_users()
        lines = []
        for u in users[:20]:
            uname = f"@{u['username']}" if u.get("username") else "—"
            phn = mask_phone(u["phone"]) if u.get("phone") else "—"
            ban = "محظور " if u.get("banned") else ""
            lines.append(f"{ban}<code>{u['user_id']}</code> {u.get('name','—')} | {uname} | {phn}")
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_USERS} <b>المستخدمون ({len(users)})</b>\n\n" + "\n".join(lines),
                      reply_markup=kb_admin())
        return

    if data == "admin_reports":
        if uid != ADMIN_ID: return
        reports = db_get_open_reports()
        if not reports:
            await edit_ce(context.bot, chat_id, q.message.message_id,
                          f"{EMOJI_CHECK} لا توجد بلاغات مفتوحة",
                          reply_markup=kb_admin())
            return
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"🚨 <b>البلاغات المفتوحة ({len(reports)})</b>",
                      reply_markup=kb_admin())
        for r in reports[:10]:
            uname = f"@{r['username']}" if r.get('username') else "بدون يوزر"
            txt = (f"🚨 <b>بلاغ #{r['id']}</b>\n"
                   f"👤 {r.get('name') or 'مستخدم'}\n"
                   f"🆔 <code>{r['user_id']}</code>\n"
                   f"📱 {uname}\n"
                   f"🕐 {r['created_at']}\n"
                   f"━━━━━━━━━━━━━━━━━━━\n"
                   f"📝 {r['message'][:600]}")
            await send_ce(context.bot, chat_id, txt,
                          reply_markup=kb_admin_report(r['id']))
        return

    if data.startswith("admin_reply_report_"):
        if uid != ADMIN_ID: return
        rid = int(data.replace("admin_reply_report_", ""))
        r = db_get_report(rid)
        if not r:
            await q.answer("البلاغ غير موجود", show_alert=True); return
        context.user_data["waiting"] = "admin_reply_report"
        context.user_data["reply_report_id"] = rid
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"✍️ <b>الرد على البلاغ #{rid}</b>\n\n"
                      f"👤 {r.get('name','مستخدم')}\n"
                      f"📝 {r['message'][:200]}\n\n"
                      f"اكتب ردك الآن وسيصل للمستخدم فوراً:")
        return

    if data.startswith("admin_ignore_report_"):
        if uid != ADMIN_ID: return
        rid = int(data.replace("admin_ignore_report_", ""))
        db_reply_report(rid, "[تم تجاهل البلاغ من الإدارة]")
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_CHECK} تم تجاهل البلاغ #{rid}",
                      reply_markup=kb_admin())
        return

    if data == "admin_broadcast":
        if uid != ADMIN_ID: return
        context.user_data["waiting"] = "admin_bc"
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_MEGAPHONE} أرسل الرسالة (أو /cancel للإلغاء):")
        return

    if data == "admin_ban":
        if uid != ADMIN_ID: return
        context.user_data["waiting"] = "admin_ban"
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_BAN} أرسل ID المستخدم للحظر:")
        return

    if data == "admin_unban":
        if uid != ADMIN_ID: return
        context.user_data["waiting"] = "admin_unban"
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_UNBAN} أرسل ID المستخدم لرفع الحظر:")
        return

    if data == "admin_clear_tokens":
        if uid != ADMIN_ID: return
        db_clear_all_tokens()
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_BROOM} تم مسح جميع التوكنات",
                      reply_markup=kb_admin())
        return

    if data == "admin_proof_channel":
        if uid != ADMIN_ID: return
        proof = get_proof_channel()
        if proof:
            info = (f"📸 <b>قناة الإثبات</b>\n\n"
                    f"✅ مُعدّة\n"
                    f"📢 الاسم: <b>{proof['name']}</b>\n"
                    f"🆔 ID: <code>{proof['id']}</code>\n"
                    f"🔗 الرابط: {proof['link']}")
        else:
            info = f"📸 <b>قناة الإثبات</b>\n\n❌ غير مُعدّة بعد"
        await edit_ce(context.bot, chat_id, q.message.message_id, info,
                      reply_markup=InlineKeyboardMarkup([
                          [btn("➕ تعيين قناة الإثبات", "admin_set_proof")],
                          [btn("🗑 حذف قناة الإثبات", "admin_clear_proof")],
                          [btn("رجوع", "admin_panel")],
                      ]))
        return

    if data == "admin_set_proof":
        if uid != ADMIN_ID: return
        context.user_data["waiting"] = "admin_set_proof"
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"📸 <b>تعيين قناة الإثبات</b>\n\n"
                      f"أرسل رابط القناة أو يوزرنيم:\n"
                      f"<code>https://t.me/dznet3</code>\n\n"
                      f"⚠️ البوت يجب أن يكون <b>أدمن</b> (لينشر فيها)")
        return

    if data == "admin_clear_proof":
        if uid != ADMIN_ID: return
        clear_proof_channel()
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_BROOM} تم حذف قناة الإثبات",
                      reply_markup=InlineKeyboardMarkup([
                          [btn("رجوع", "admin_proof_channel")]]))
        return


async def _run_walk(update, context, uid, phone, act):
    try:
        pkg_map = {"1g": PKG_1GO, "2g": PKG_2GO}
        name_map = {"1g": "حزمة 1GB", "2g": "حزمة 2GB"}
        size_map = {"1g": "1GB", "2g": "2GB"}
        if act not in pkg_map: return

        pkg, name, size = pkg_map[act], name_map[act], size_map[act]
        cooldown_s = 86400 if act == "1g" else 7*24*3600

        history = await run_sync(api_subscription_history, uid, phone)
        if history is not None:
            last_ts = _last_walk_win_date_typed(history, pkg)
            if last_ts:
                elapsed = time.time() - last_ts
                if elapsed < cooldown_s:
                    rem = int(cooldown_s - elapsed)
                    await send_ce(context.bot, uid, msg_offer_waiting(name, fmt_seconds(rem)),
                                  reply_markup=kb_main(uid, uid == ADMIN_ID))
                    return

        ok, err = await run_sync(api_activate_walk, uid, phone, pkg)
        if ok:
            db_save_activation(uid, phone, act)
            await send_ce(context.bot, uid,
                          f"{EMOJI_GIFT} <b>تم تفعيل عرض {name}...</b>")
            await asyncio.sleep(1)
            await send_ce(context.bot, uid,
                          f"{EMOJI_CHECK} <b>نجاح!</b>\n\n"
                          f"✅ {name}\n"
                          f"📦 {size}\n"
                          f"📞 <code>{mask_phone(phone)}</code>",
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
            db_schedule_reminder(uid, phone, act, cooldown_s)
            user = update.effective_user
            await send_proof_to_channel(context.bot, user, name, size, phone)
        elif err == "NEED_100DZ":
            kb = InlineKeyboardMarkup([
                [btn("فعّل باقة مدفوعة", "show_offers")],
                [btn("رجوع", "back_menu")],
            ])
            await send_ce(context.bot, uid,
                          f"{EMOJI_WARNING} <b>{name}</b> يحتاج شرط قبل التفعيل\n\n"
                          f"لازم يكون عندك باقة مدفوعة نشطة (شرط من Djezzy).\n\n"
                          f"1) فعّل باقة مدفوعة من القائمة\n"
                          f"2) رجع وفعّل الهدية المجانية",
                          reply_markup=kb)
        else:
            await send_ce(context.bot, uid, msg_offer_fail(name, err),
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
    except Exception as e:
        logger.exception(f"_run_walk error: {e}")
        try:
            await context.bot.send_message(uid, "⚠️ حدث خطأ في التفعيل.")
        except Exception:
            pass


async def _run_shake_6gb(update, context, uid, phone):
    try:
        cooldown_s = 86400
        history = await run_sync(api_subscription_history, uid, phone)
        if history is not None:
            last_ts = _last_walk_win_date_typed(history, PKG_6GB)
            if last_ts:
                elapsed = time.time() - last_ts
                if elapsed < cooldown_s:
                    rem = int(cooldown_s - elapsed)
                    await send_ce(context.bot, uid, msg_offer_waiting("حزمة 6GB Shake", fmt_seconds(rem)),
                                  reply_markup=kb_main(uid, uid == ADMIN_ID))
                    return

        ok, err = await run_sync(api_activate_shake_6gb, uid, phone)
        if ok:
            db_save_activation(uid, phone, "shake_6gb")
            await send_ce(context.bot, uid, f"{EMOJI_GIFT} <b>تم إرسال طلب تفعيل 6GB Shake...</b>")
            await asyncio.sleep(1)
            await send_ce(context.bot, uid,
                          f"{EMOJI_CHECK} <b>نجاح تفعيل باقة 6GB!</b>\n\n"
                          f"📦 العرض: Shake 6GB\n"
                          f"📞 الرقم: <code>{mask_phone(phone)}</code>",
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
            db_schedule_reminder(uid, phone, "shake_6gb", cooldown_s)
            user = update.effective_user
            await send_proof_to_channel(context.bot, user, "Shake 6GB", "6GB", phone)
        else:
            await send_ce(context.bot, uid,
                          f"{EMOJI_ERROR} <b>تعذّر تفعيل باقة 6GB</b>\n\nالسبب: {err}",
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
    except Exception as e:
        logger.exception(f"_run_shake_6gb error: {e}")
        try:
            await context.bot.send_message(uid, "⚠️ حدث خطأ في التفعيل.")
        except Exception:
            pass


async def _run_buy(update, context, uid, phone, pkg):
    try:
        info = OFFERS.get(pkg, {})
        label = info.get("name", pkg)
        ok, err = await run_sync(api_activate_product, uid, phone, pkg)
        if ok:
            await send_ce(context.bot, uid,
                          f"{EMOJI_GIFT} <b>تم تفعيل {label}...</b>")
            await asyncio.sleep(1)
            await send_ce(context.bot, uid,
                          f"{EMOJI_CHECK} <b>نجاح!</b>\n\n"
                          f"✅ {label}\n"
                          f"📦 {info.get('display','')}\n"
                          f"💰 {info.get('price','')}\n"
                          f"📞 <code>{mask_phone(phone)}</code>",
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
            try:
                user = update.effective_user
                uname = f"@{user.username}" if user.username else "بدون يوزر"
                await send_ce(context.bot, ADMIN_ID,
                              f"💎 <b>عملية شراء جديدة</b>\n\n"
                              f"👤 {user.first_name or 'مستخدم'}\n"
                              f"🆔 <code>{uid}</code>\n"
                              f"📱 {uname}\n"
                              f"📞 <code>{mask_phone(phone)}</code>\n"
                              f"📦 {label}\n"
                              f"💰 {info.get('price','')}")
            except Exception: pass
        else:
            await send_ce(context.bot, uid, msg_offer_fail(label, err),
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
    except Exception as e:
        logger.exception(f"_run_buy error: {e}")


async def _do_invite_flow(update, context, uid, phone):
    q = update.callback_query
    chat_id = q.message.chat.id
    await edit_ce(context.bot, chat_id, q.message.message_id,
                  f"{EMOJI_HOURGLASS} جارٍ فحص المكافآت المعلّقة...")

    bonus = await run_sync(api_try_mgm_bonus, uid, phone)
    if bonus in ("SUCCESS_1GO", "SUCCESS_500MO", "ALREADY_CLAIMED"):
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      msg_mgm_bonus(bonus), reply_markup=kb_main(uid, uid == ADMIN_ID))
        return

    await edit_ce(context.bot, chat_id, q.message.message_id,
                  f"{EMOJI_HOURGLASS} جارٍ مراجعة الدعوات المتاحة...")
    data = await run_sync(api_fetch_mgm_invitations, uid, phone)
    if data is None:
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_WARNING} تعذّر جلب البيانات، حاول لاحقاً",
                      reply_markup=kb_main(uid, uid == ADMIN_ID))
        return

    max_inv = data.get("campaign", {}).get("maxInvitation", 5) if isinstance(data, dict) else 5
    inv_list = data.get("invitations", []) if isinstance(data, dict) else []
    done_count = sum(1 for i in inv_list if i.get("status") == "DONE")
    pending_ids = [i.get("id") for i in inv_list if i.get("status") == "PENDING"]

    if pending_ids:
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_BROOM} جارٍ إخلاء الدعوات المعلّقة...")
        await run_sync(api_delete_pending_invitations, uid, phone, pending_ids)

    if done_count >= max_inv:
        await edit_ce(context.bot, chat_id, q.message.message_id,
                      f"{EMOJI_BAN} <b>وصلت للحد الأقصى ({max_inv})</b>",
                      reply_markup=kb_main(uid, uid == ADMIN_ID))
        return

    count = db_invite_count(uid)
    context.user_data["waiting"] = "invite"
    await edit_ce(context.bot, chat_id, q.message.message_id,
                  msg_invite_start(count, MAX_INVITES - count),
                  reply_markup=kb_cancel_invite())


async def on_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        await _handle_message(update, context)
    except Exception as e:
        logger.exception(f"on_message error: {e}")


async def _handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.type != "private": return
    uid = update.effective_user.id
    chat_id = update.effective_chat.id
    if db_is_banned(uid): return

    is_subbed, missing_ch = await check_force_subscription(context.bot, uid)
    if not is_subbed and missing_ch:
        kb = InlineKeyboardMarkup([
            [btn("📢 اشترك في القناة الأولى", url=REQUIRED_CHANNELS[0]["link"])],
            [btn("📢 اشترك في القناة الثانية", url=REQUIRED_CHANNELS[1]["link"])],
            [btn("🔄 تحقق من الاشتراك", "start")]
        ])
        await send_ce(context.bot, chat_id,
                      f"⚠️ <b>عذراً، يجب عليك الاشتراك في قنواتنا أولاً لاستخدام البوت:</b>\n\n"
                      f"1️⃣ {REQUIRED_CHANNELS[0]['name']}\n"
                      f"2️⃣ {REQUIRED_CHANNELS[1]['name']}\n\n"
                      f"بعد الاشتراك، اضغط على زر <b>تحقق من الاشتراك</b> بالأسفل 👇",
                      reply_markup=kb)
        return

    waiting = context.user_data.get("waiting")
    if not waiting: return

    msg_obj = update.message
    text = (msg_obj.text or msg_obj.caption or "").strip()

    if text == "/cancel":
        context.user_data.pop("waiting", None)
        context.user_data.pop("reply_report_id", None)
        await send_ce(context.bot, chat_id, f"{EMOJI_BAN} أُلغيت العملية",
                      reply_markup=kb_main(uid, uid == ADMIN_ID))
        return

    if waiting == "report":
        if not text and not msg_obj.photo and not msg_obj.document:
            await send_ce(context.bot, chat_id,
                          f"{EMOJI_WARNING} أرسل نص المشكل أو صورة/ملف مع وصف",
                          reply_markup=kb_cancel_report())
            return

        username = update.effective_user.username or ""
        name = update.effective_user.first_name or "مستخدم"
        report_text = text or "(بدون نص - مرفق)"

        rid = db_add_report(uid, username, name, report_text)
        context.user_data.pop("waiting", None)

        await send_ce(context.bot, chat_id,
                      f"{EMOJI_CHECK} <b>تم إرسال بلاغك بنجاح!</b>\n\n"
                      f"🆔 رقم البلاغ: <code>#{rid}</code>\n\n"
                      f"{EMOJI_CLOCK} سيتم الرد عليك من قبل الإدارة قريباً",
                      reply_markup=kb_main(uid, uid == ADMIN_ID))

        try:
            uname = f"@{username}" if username else "بدون يوزر"
            admin_txt = (f"🚨 <b>بلاغ جديد #{rid}</b>\n"
                         f"━━━━━━━━━━━━━━━━━━━\n"
                         f"👤 {name}\n"
                         f"🆔 <code>{uid}</code>\n"
                         f"📱 {uname}\n"
                         f"🕐 {datetime.now().strftime('%Y-%m-%d %H:%M')}\n"
                         f"━━━━━━━━━━━━━━━━━━━\n"
                         f"📝 {report_text}")
            await send_ce(context.bot, ADMIN_ID, admin_txt,
                          reply_markup=kb_admin_report(rid))

            if msg_obj.photo:
                await context.bot.forward_message(
                    chat_id=ADMIN_ID,
                    from_chat_id=chat_id,
                    message_id=msg_obj.message_id)
            elif msg_obj.document:
                await context.bot.forward_message(
                    chat_id=ADMIN_ID,
                    from_chat_id=chat_id,
                    message_id=msg_obj.message_id)
        except Exception as e:
            logger.error(f"forward report to admin: {e}")
        return

    if waiting == "admin_reply_report":
        if uid != ADMIN_ID: return
        rid = context.user_data.get("reply_report_id")
        if not rid:
            context.user_data.pop("waiting", None)
            return
        r = db_get_report(rid)
        if not r:
            await send_ce(context.bot, chat_id, f"{EMOJI_ERROR} البلاغ غير موجود",
                          reply_markup=kb_admin())
            context.user_data.pop("waiting", None)
            return
        if not text:
            await send_ce(context.bot, chat_id,
                          f"{EMOJI_WARNING} أرسل نص الرد")
            return

        db_reply_report(rid, text)

        try:
            user_txt = (f"🎧 <b>رد على بلاغك #{rid}</b>\n"
                        f"━━━━━━━━━━━━━━━━━━━\n"
                        f"📝 بلاغك: {r['message'][:200]}\n"
                        f"━━━━━━━━━━━━━━━━━━━\n"
                        f"💬 <b>رد الإدارة:</b>\n"
                        f"{text}")
            await send_ce(context.bot, r['user_id'], user_txt,
                          reply_markup=kb_main(r['user_id'], r['user_id'] == ADMIN_ID))
            await send_ce(context.bot, chat_id,
                          f"{EMOJI_CHECK} تم إرسال الرد للمستخدم #{r['user_id']}",
                          reply_markup=kb_admin())
        except Exception as e:
            logger.error(f"send reply to user: {e}")
            await send_ce(context.bot, chat_id,
                          f"{EMOJI_WARNING} تعذر إرسال الرد للمستخدم\nالسبب: {e}",
                          reply_markup=kb_admin())
        context.user_data.pop("waiting", None)
        context.user_data.pop("reply_report_id", None)
        return

    if waiting == "phone":
        raw_phone = re.sub(r"\D", "", text)

        if not re.match(r"^07\d{8}$", raw_phone):
            if raw_phone.startswith("05") or raw_phone.startswith("06"):
                await send_ce(context.bot, chat_id,
                              f"{EMOJI_ERROR} <b>هذا الرقم ليس جيزي!</b>\n\n"
                              f"📞 الرقم: <code>{raw_phone}</code>\n\n"
                              f"{EMOJI_WARNING} البوت يقبل فقط أرقام جيزي (07)\n"
                              f"💡 أرقام جيزي: <code>07XXXXXXXX</code>",
                              reply_markup=kb_main(uid, uid == ADMIN_ID))
                return

            if len(raw_phone) != 10:
                await send_ce(context.bot, chat_id,
                              f"{EMOJI_WARNING} <b>الرقم غير مكتمل</b>\n\n"
                              f"يجب أن يكون 10 أرقام\n"
                              f"مثال: <code>07XXXXXXXX</code>\n"
                              f"الرقم: <code>{raw_phone}</code> ({len(raw_phone)} أرقام)",
                              reply_markup=kb_main(uid, uid == ADMIN_ID))
                return

            await send_ce(context.bot, chat_id,
                          f"{EMOJI_WARNING} الرقم غير صحيح\nمثال: <code>07XXXXXXXX</code>",
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
            return

        phone = format_phone(text)
        if not phone or len(phone) != 12:
            await send_ce(context.bot, chat_id,
                          f"{EMOJI_WARNING} الرقم غير صحيح\nمثال: <code>07XXXXXXXX</code>")
            return

        db_update_phone(uid, phone)

        wait_msg = await send_ce(context.bot, chat_id,
                                 f"{EMOJI_HOURGLASS} جارٍ إرسال الرمز، يرجى الانتظار...")

        ok, err = await run_sync(api_request_otp, phone)

        if ok:
            context.user_data["waiting"] = "otp"
            if wait_msg:
                await edit_ce(context.bot, chat_id, wait_msg.message_id, msg_otp_sent(mask_phone(phone)))
            else:
                await send_ce(context.bot, chat_id, msg_otp_sent(mask_phone(phone)))
        else:
            if "غير مسجل" in err or "غير صحيح" in err:
                if wait_msg:
                    await edit_ce(context.bot, chat_id, wait_msg.message_id,
                                  f"{EMOJI_ERROR} <b>تعذّر الإرسال</b>\n\n{err}",
                                  reply_markup=kb_main(uid, uid == ADMIN_ID))
                else:
                    await send_ce(context.bot, chat_id,
                                  f"{EMOJI_ERROR} <b>تعذّر الإرسال</b>\n\n{err}",
                                  reply_markup=kb_main(uid, uid == ADMIN_ID))
            else:
                if wait_msg:
                    await edit_ce(context.bot, chat_id, wait_msg.message_id,
                                  f"{EMOJI_ERROR} <b>تعذّر الإرسال</b>\n\n{err}",
                                  reply_markup=kb_retry())
                else:
                    await send_ce(context.bot, chat_id,
                                  f"{EMOJI_ERROR} <b>تعذّر الإرسال</b>\n\n{err}",
                                  reply_markup=kb_retry())
        return

    if waiting == "otp":
        otp = re.sub(r"\D", "", text)
        if len(otp) < 4:
            await send_ce(context.bot, chat_id,
                          f"{EMOJI_WARNING} الرمز من 4 إلى 6 أرقام")
            return
        phone = db_get_phone(uid)
        if not phone:
            context.user_data["waiting"] = "phone"
            await send_ce(context.bot, chat_id, f"{EMOJI_ERROR} حدث خطأ، أعد الإرسال")
            return
        ok, name, err = await run_sync(api_verify_otp, phone, otp, uid)
        if ok:
            name = name or update.effective_user.first_name or "مستخدم"
            username = update.effective_user.username or ""
            db_update_name(uid, name)
            db_update_username(uid, username)
            db_add_user(uid, name, username, phone)
            context.user_data.pop("waiting", None)
            await send_ce(context.bot, chat_id,
                          msg_login_success(name, mask_phone(phone), username),
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
            for off, size in [("1g","حزمة 1GB"),("2g","حزمة 2GB"),("shake_6gb","حزمة 6GB Shake")]:
                st, _ = get_offer_state(phone, off)
                if st == STATE_READY:
                    await send_ce(context.bot, chat_id,
                                  f"{EMOJI_GIFT} <b>عرض {size} جاهز للتفعيل!</b>")
        else:
            await send_ce(context.bot, chat_id,
                          f"{EMOJI_ERROR} فشل التحقق\n\n{err}",
                          reply_markup=kb_retry())
        return

    if waiting == "invite":
        if not is_valid_dz_number(text):
            await send_ce(context.bot, chat_id,
                          f"{EMOJI_WARNING} رقم غير صحيح\nمثال: <code>07XXXXXXXX</code>",
                          reply_markup=kb_cancel_invite())
            return
        phone = db_get_phone(uid)
        if not phone:
            context.user_data.pop("waiting", None)
            await send_ce(context.bot, chat_id, f"{EMOJI_ERROR} أعد تسجيل الدخول",
                          reply_markup=kb_main(uid, uid == ADMIN_ID))
            return
        receiver = format_phone(text)
        if receiver == phone:
            await send_ce(context.bot, chat_id, f"{EMOJI_ERROR} لا يمكن دعوة رقمك",
                          reply_markup=kb_cancel_invite())
            return
        ok, api_msg = await run_sync(api_send_mgm_invitation, uid, phone, receiver)
        if ok:
            db_ok, db_msg = db_add_invitation(uid, text)
        else:
            db_ok, db_msg = False, api_msg
        count = db_invite_count(uid)
        context.user_data.pop("waiting", None)
        await send_ce(context.bot, chat_id,
                      msg_invite_result(db_ok, db_msg, count, MAX_INVITES - count),
                      reply_markup=kb_main(uid, uid == ADMIN_ID))
        return

    if waiting == "admin_bc":
        if uid != ADMIN_ID: return
        if not text:
            context.user_data.pop("waiting", None)
            return
        users = db_all_users()
        s = f = 0
        for u in users:
            try:
                await context.bot.send_message(u["user_id"], f"<b>إعلان</b>\n\n{text}",
                                                parse_mode=ParseMode.HTML)
                s += 1; await asyncio.sleep(0.05)
            except Exception: f += 1
        context.user_data.pop("waiting", None)
        await send_ce(context.bot, chat_id,
                      f"{EMOJI_CHECK} أُرسلت: {s}\n{EMOJI_ERROR} فشل: {f}",
                      reply_markup=kb_admin())
        return

    if waiting == "admin_ban":
        if uid != ADMIN_ID: return
        try:
            db_ban(int(text))
            await send_ce(context.bot, chat_id, f"{EMOJI_BAN} تم حظر {text}",
                          reply_markup=kb_admin())
        except ValueError:
            await send_ce(context.bot, chat_id, f"{EMOJI_WARNING} ID غير صحيح",
                          reply_markup=kb_admin())
        context.user_data.pop("waiting", None)
        return

    if waiting == "admin_unban":
        if uid != ADMIN_ID: return
        try:
            db_unban(int(text))
            await send_ce(context.bot, chat_id, f"{EMOJI_UNBAN} تم رفع الحظر عن {text}",
                          reply_markup=kb_admin())
        except ValueError:
            await send_ce(context.bot, chat_id, f"{EMOJI_WARNING} ID غير صحيح",
                          reply_markup=kb_admin())
        context.user_data.pop("waiting", None)
        return

    if waiting == "admin_set_proof":
        if uid != ADMIN_ID: return
        raw = text.strip()
        username = None
        if raw.startswith("https://t.me/") or raw.startswith("http://t.me/"):
            username = raw.split("t.me/")[-1].split("?")[0].strip("/")
        elif raw.startswith("https://telegram.me/") or raw.startswith("http://telegram.me/"):
            username = raw.split("telegram.me/")[-1].split("?")[0].strip("/")
        elif raw.startswith("@"):
            username = raw[1:].strip()
        elif re.match(r"^[a-zA-Z][a-zA-Z0-9_]{3,}$", raw):
            username = raw
        elif raw.startswith("-100"):
            username = raw

        if not username:
            await send_ce(context.bot, chat_id,
                          f"{EMOJI_WARNING} صيغة غير صحيحة\n"
                          f"مثال: <code>https://t.me/dznet3</code>")
            return

        try:
            chat_obj = await context.bot.get_chat(
                username if username.startswith("-100") else f"@{username}"
            )
            ch_id = str(chat_obj.id)
            ch_name = chat_obj.title or username
            if chat_obj.username:
                ch_link = f"https://t.me/{chat_obj.username}"
            else:
                ch_link = f"https://t.me/c/{ch_id.replace('-100', '')}"
        except Exception as e:
            await send_ce(context.bot, chat_id,
                          f"{EMOJI_ERROR} تعذّر جلب القناة\n"
                          f"السبب: {e}\n\n"
                          f"⚠️ تأكد أن البوت <b>أدمن</b> في القناة")
            return

        set_proof_channel(ch_id, ch_link, ch_name)
        context.user_data.pop("waiting", None)
        await send_ce(context.bot, chat_id,
                      f"{EMOJI_CHECK} تم تعيين قناة الإثبات\n\n"
                      f"📸 الاسم: <b>{ch_name}</b>\n"
                      f"🆔 ID: <code>{ch_id}</code>\n"
                      f"🔗 الرابط: {ch_link}",
                      reply_markup=kb_admin())
        return


async def reminder_loop(app: Application):
    name_map = {"1g": "حزمة 1GB", "2g": "حزمة 2GB", "shake_6gb": "حزمة 6GB Shake"}
    while True:
        try:
            for r in db_due_reminders():
                label = name_map.get(r["offer_type"], r["offer_type"])
                try:
                    await send_ce(app.bot, r["user_id"],
                                  f"{EMOJI_GIFT} <b>تذكير!</b>\n\nعرض <b>{label}</b> جاهز للتفعيل الآن!",
                                  reply_markup=kb_main(r["user_id"], r["user_id"] == ADMIN_ID))
                    db_mark_reminder_sent(r["id"])
                except Exception as e:
                    logger.warning(f"reminder: {e}")
        except Exception as e:
            logger.error(f"reminder_loop: {e}")
        await asyncio.sleep(60)


async def post_init(app: Application):
    try:
        await app.bot.set_my_commands([
            ("start", "تشغيل البوت"),
            ("help", "مساعدة"),
            ("admin", "لوحة الإدارة"),
        ])
    except Exception as e:
        logger.warning(f"set_my_commands: {e}")

    asyncio.create_task(reminder_loop(app))

    if DEFAULT_PROOF_CHANNEL_LINK and not get_proof_channel():
        try:
            username = DEFAULT_PROOF_CHANNEL_LINK.split("t.me/")[-1].strip("/")
            chat_obj = await app.bot.get_chat(f"@{username}")
            ch_id = str(chat_obj.id)
            ch_name = chat_obj.title or username
            ch_link = f"https://t.me/{chat_obj.username}" if chat_obj.username else DEFAULT_PROOF_CHANNEL_LINK
            set_proof_channel(ch_id, ch_link, ch_name)
            logger.info(f"✅ تم تعيين قناة الإثبات الافتراضية: {ch_name} ({ch_id})")
        except Exception as e:
            logger.warning(f"⚠️ تعذّر تعيين قناة الإثبات الافتراضية: {e}")

    try:
        proof = get_proof_channel()
        proof_line = f"\n📸 قناة الإثبات: <b>{proof['name']}</b>" if proof else "\n⚠️ قناة الإثبات: غير مُعدّة"
        await send_ce(app.bot, ADMIN_ID,
                      f"{EMOJI_CHECK} <b>البوت يعمل الآن</b>{proof_line}\n"
                      f"🎧 دعم فني: @{ADMIN_USERNAME}")
        logger.info("Bot started successfully")
    except Exception as e:
        logger.error(f"post_init admin msg failed: {e}")


async def error_handler(update: object, context: ContextTypes.DEFAULT_TYPE):
    logger.error(f"Update {update} caused error: {context.error}", exc_info=context.error)


def main():
    init_db()
    print("l9bi7e net")
    logger.info("l9bi7e net starting...")

    app = Application.builder().token(BOT_TOKEN).post_init(post_init).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_cmd))
    app.add_handler(CommandHandler("admin", admin_cmd))
    app.add_handler(CallbackQueryHandler(on_callback))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, on_message))
    app.add_handler(MessageHandler(filters.PHOTO, on_message))
    app.add_handler(MessageHandler(filters.Document.ALL, on_message))
    app.add_error_handler(error_handler)

    app.run_polling(
        allowed_updates=["message", "callback_query"],
        drop_pending_updates=True,
    )


if __name__ == "__main__":
    main()

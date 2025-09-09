
# full_feature_bot.py
# Requirements: pytelegrambotapi, requests
# pip install pytelegrambotapi requests

import sqlite3
import requests
import time
import random
import string
import os
from datetime import datetime, timedelta
import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton

# ---------------- CONFIG ----------------
# Fill these or set as ENV variables (for Railway use os.getenv)
BOT_TOKEN = os.getenv("BOT_TOKEN") or "8264688551:AAFkMjycsx4R9GebuOYTTJpITlW4aE7g2rc"
SUPER_ADMIN_ID = int(os.getenv("SUPER_ADMIN_ID") or 7798676542)  # replace with your Telegram ID
DEFAULT_ADMIN_PASSWORD = os.getenv("ADMIN_PASSWORD") or "rolex@7786"

# LeakOSINT (optional) - set if you want get info to call API
LEAKOSINT_URL = os.getenv("LEAKOSINT_URL") or "https://leakosintapi.com/"
LEAKOSINT_API = os.getenv("LEAKOSINT_API") or "8451537591:E4O0Mx4O"  # set your LeakOSINT token if available
LEAKOSINT_LANG = "en"
LEAKOSINT_LIMIT = 300

# ---------------- BOT INIT ----------------
bot = telebot.TeleBot(BOT_TOKEN, parse_mode="HTML")

# ---------------- DB helpers (thread-safe) ----------------
DB_PATH = "bot.db"

def get_db():
    conn = sqlite3.connect(DB_PATH, timeout=30, check_same_thread=False)
    c = conn.cursor()
    return conn, c

def init_db():
    conn, c = get_db()
    c.execute("""CREATE TABLE IF NOT EXISTS users(
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        name TEXT,
        credits REAL DEFAULT 0,
        referrals INTEGER DEFAULT 0,
        last_bonus INTEGER DEFAULT 0
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS admins(
        user_id INTEGER PRIMARY KEY
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS giftcodes(
        code TEXT PRIMARY KEY,
        credits REAL,
        used_by INTEGER DEFAULT 0
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS force_channels(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        channel TEXT UNIQUE
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS settings(
        key TEXT PRIMARY KEY,
        value TEXT
    )""")
    c.execute("""CREATE TABLE IF NOT EXISTS logs(
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        who INTEGER,
        action TEXT,
        target TEXT,
        amount REAL,
        time TEXT
    )""")
    # seed admin password and referral rate (if not exists)
    c.execute("INSERT OR IGNORE INTO settings(key,value) VALUES(?,?)", ("admin_password", DEFAULT_ADMIN_PASSWORD))
    c.execute("INSERT OR IGNORE INTO settings(key,value) VALUES(?,?)", ("referral_rate", "1.5"))  # credits per reward step
    c.execute("INSERT OR IGNORE INTO settings(key,value) VALUES(?,?)", ("referral_needed", "2"))
    # Ensure super admin is in admins
    c.execute("INSERT OR IGNORE INTO admins(user_id) VALUES(?)", (SUPER_ADMIN_ID,))
    conn.commit()
    conn.close()

init_db()

# ---------------- Utility functions ----------------
def get_setting(key, default=None):
    conn, c = get_db()
    c.execute("SELECT value FROM settings WHERE key=?", (key,))
    row = c.fetchone()
    conn.close()
    return row[0] if row else default

def set_setting(key, value):
    conn, c = get_db()
    c.execute("INSERT OR REPLACE INTO settings(key,value) VALUES(?,?)", (key, str(value)))
    conn.commit()
    conn.close()

def add_user(user):
    conn, c = get_db()
    c.execute("INSERT OR IGNORE INTO users(user_id, username, name) VALUES(?,?,?)",
              (user.id, user.username or "", user.first_name or ""))
    conn.commit()
    conn.close()

def update_username_name(user):
    conn, c = get_db()
    c.execute("UPDATE users SET username=?, name=? WHERE user_id=?", (user.username or "", user.first_name or "", user.id))
    conn.commit()
    conn.close()

def get_user_row(user_id):
    conn, c = get_db()
    c.execute("SELECT user_id, username, name, credits, referrals, last_bonus FROM users WHERE user_id=?", (user_id,))
    row = c.fetchone()
    conn.close()
    return row

def add_credits_to(uid, amount):
    conn, c = get_db()
    c.execute("UPDATE users SET credits = credits + ? WHERE user_id=?", (amount, uid))
    conn.commit()
    conn.close()

def set_credits(uid, amount):
    conn, c = get_db()
    c.execute("UPDATE users SET credits = ? WHERE user_id=?", (amount, uid))
    conn.commit()
    conn.close()

def get_all_users():
    conn, c = get_db()
    c.execute("SELECT user_id, username, name, credits, referrals FROM users")
    rows = c.fetchall()
    conn.close()
    return rows

def create_giftcode(credits):
    code = ''.join(random.choices(string.ascii_uppercase + string.digits, k=8))
    conn, c = get_db()
    c.execute("INSERT INTO giftcodes(code, credits) VALUES(?,?)", (code, credits))
    conn.commit()
    conn.close()
    return code

def redeem_giftcode(uid, code):
    conn, c = get_db()
    c.execute("SELECT credits, used_by FROM giftcodes WHERE code=?", (code,))
    row = c.fetchone()
    if not row:
        conn.close()
        return False, "Invalid code."
    credits, used_by = row
    if used_by and used_by != 0:
        conn.close()
        return False, "Code already used."
    c.execute("UPDATE giftcodes SET used_by=? WHERE code=?", (uid, code))
    c.execute("UPDATE users SET credits = credits + ? WHERE user_id=?", (credits, uid))
    conn.commit()
    conn.close()
    return True, credits

def add_admin(uid):
    conn, c = get_db()
    c.execute("INSERT OR IGNORE INTO admins(user_id) VALUES(?)", (uid,))
    conn.commit()
    conn.close()

def remove_admin(uid):
    conn, c = get_db()
    c.execute("DELETE FROM admins WHERE user_id=?", (uid,))
    conn.commit()
    conn.close()

def get_admins():
    conn, c = get_db()
    c.execute("SELECT user_id FROM admins")
    rows = [r[0] for r in c.fetchall()]
    conn.close()
    return rows

def add_force_channel(ch):
    conn, c = get_db()
    c.execute("INSERT OR IGNORE INTO force_channels(channel) VALUES(?)", (ch,))
    conn.commit()
    conn.close()

def remove_force_channel(ch):
    conn, c = get_db()
    c.execute("DELETE FROM force_channels WHERE channel=?", (ch,))
    conn.commit()
    conn.close()

def list_force_channels():
    conn, c = get_db()
    c.execute("SELECT channel FROM force_channels")
    rows = [r[0] for r in c.fetchall()]
    conn.close()
    return rows

def log_action(who, action, target="", amount=0):
    conn, c = get_db()
    c.execute("INSERT INTO logs(who, action, target, amount, time) VALUES(?,?,?,?,?)",
              (who, action, str(target), float(amount), datetime.now().isoformat()))
    conn.commit()
    conn.close()

# ---------------- In-memory states (for flows) ----------------
pending_reply = {}         # admin_id -> user_id (to whom admin will reply)
pending_broadcast = {}     # admin_id -> True (next message to admin is broadcast text)
pending_addcredit = {}     # admin_id -> expects "user_id amount"
pending_zerocredit = {}    # admin_id -> expects user id
pending_channel_add = {}   # admin_id -> expects channel username
pending_channel_remove = {}# admin_id -> expects channel username
pending_referral_rate = {} # admin_id -> expects new rate
pending_gen_gift = {}      # admin_id -> expects credits to create giftcode

# ---------------- Helpers for UI ----------------
def user_menu_markup():
    kb = InlineKeyboardMarkup()
    kb.row_width = 2
    kb.add(
        InlineKeyboardButton("👩‍💻 Profile", callback_data="profile"),
        InlineKeyboardButton("💎 Add Fund", callback_data="addfund"),
    )
    kb.add(
        InlineKeyboardButton("🔍 Get Information", callback_data="getinfo"),
    )
    kb.add(
        InlineKeyboardButton("⚡ Referral", callback_data="referral"),
        InlineKeyboardButton("🎁 Bonus", callback_data="bonus"),
    )
    kb.add(
        InlineKeyboardButton("📊 Status", callback_data="status"),
        InlineKeyboardButton("🧧 Gift Code", callback_data="giftcode"),
    )
    kb.add(
        InlineKeyboardButton("📞 Support", callback_data="support"),
        InlineKeyboardButton("🆘 Complaint", callback_data="complaint"),
    )
    return kb

def admin_panel_markup():
    kb = InlineKeyboardMarkup()
    kb.row_width = 2
    kb.add(
        InlineKeyboardButton("➕ Add Credit", callback_data="admin_addcredit"),
        InlineKeyboardButton("❌ Zero Credit", callback_data="admin_zerocredit"),
    )
    kb.add(
        InlineKeyboardButton("👥 User List", callback_data="admin_userlist"),
        InlineKeyboardButton("📢 Broadcast Users", callback_data="admin_broadcast"),
    )
    kb.add(
        InlineKeyboardButton("📣 Broadcast Admins", callback_data="admin_broadcast_admins"),
        InlineKeyboardButton("📨 Complaints", callback_data="admin_complaints"),
    )
    kb.add(
        InlineKeyboardButton("🔧 Referral Rate", callback_data="admin_referral_rate"),
        InlineKeyboardButton("📢 Channel Manager", callback_data="admin_channel_manager"),
    )
    return kb

def superadmin_panel_markup():
    kb = InlineKeyboardMarkup()
    kb.add(InlineKeyboardButton("👮 Admins List", callback_data="sa_admin_list"))
    kb.add(InlineKeyboardButton("🚪 Logout Admin", callback_data="sa_logout_admin"))
    kb.add(InlineKeyboardButton("🔑 Change Password", callback_data="sa_change_pass"))
    kb.add(InlineKeyboardButton("🎟 Generate Gift Code", callback_data="sa_gen_gift"))
    kb.add(InlineKeyboardButton("📢 Broadcast Admins", callback_data="sa_broadcast_admins"))
    return kb

# ---------------- Start handler ----------------
@bot.message_handler(commands=["start"])
def handle_start(message):
    user = message.from_user
    add_user(user)
    update_username_name(user)

    # Referral param: /start REFID
    parts = message.text.split()
    if len(parts) > 1:
        try:
            ref_id = int(parts[1])
            if ref_id != user.id:
                conn, c = get_db()
                # ensure referrer exists
                c.execute("SELECT user_id FROM users WHERE user_id=?", (ref_id,))
                if c.fetchone():
                    # increment referrals and maybe give reward if threshold reached
                    c.execute("UPDATE users SET referrals = referrals + 1 WHERE user_id=?", (ref_id,))
                    conn.commit()
                    # check threshold
                    c.execute("SELECT referrals FROM users WHERE user_id=?", (ref_id,))
                    rcount = c.fetchone()[0]
                    needed = int(get_setting("referral_needed", "2"))
                    if rcount % needed == 0:
                        reward = float(get_setting("referral_rate", "1.5"))
                        c.execute("UPDATE users SET credits = credits + ? WHERE user_id=?", (reward, ref_id))
                        log_action(ref_id, "Referral reward", user.id, reward)
                    conn.commit()
                conn.close()
        except:
            pass

    # Force channel check
    missing = []
    for ch in list_force_channels():
        try:
            member = bot.get_chat_member(ch, user.id)
            if member.status in ["left", "kicked"]:
                missing.append(ch)
        except Exception:
            missing.append(ch)

    if missing:
        kb = InlineKeyboardMarkup()
        for ch in missing:
            kb.add(InlineKeyboardButton(f"👉 Join {ch}", url=f"https://t.me/{ch.replace('@','')}"))
        kb.add(InlineKeyboardButton("✅ I have joined", callback_data="joined_confirm"))
        bot.send_message(message.chat.id, "⚠️ You must join all required channels before using this bot.", reply_markup=kb)
        return

    bot.send_message(message.chat.id, "Welcome! Use the menu below:", reply_markup=user_menu_markup())

# ---------------- Callback Query Handler ----------------
@bot.callback_query_handler(func=lambda call: True)
def callback_handler(call):
    user = call.from_user
    uid = user.id
    data = call.data

    # Confirm joined channels
    if data == "joined_confirm":
        missing = []
        for ch in list_force_channels():
            try:
                member = bot.get_chat_member(ch, uid)
                if member.status in ["left", "kicked"]:
                    missing.append(ch)
            except:
                missing.append(ch)
        if missing:
            bot.answer_callback_query(call.id, "You still did not join required channels.", show_alert=True)
        else:
            bot.edit_message_text("✅ Thank you. You can use the bot now.", chat_id=call.message.chat.id, message_id=call.message.message_id)
            bot.send_message(uid, "Welcome! Use the menu below:", reply_markup=user_menu_markup())
        return

    # ---------- USER BUTTONS ----------
    if data == "profile":
        row = get_user_row(uid)
        if not row:
            bot.answer_callback_query(call.id, "No data.")
            return
        _id, username, name, credits, referrals, last_bonus = row
        username = username if username else "N/A"
        text = f"👤 <b>Profile</b>\n\nName: {name}\nUsername: @{username}\nUser ID: {_id}\nCredits: {credits}\nReferrals: {referrals}"
        bot.send_message(call.message.chat.id, text)
        return

    if data == "addfund":
        text = ("💳 <b>Buy Bot Credits</b>\n\n"
                "[ϟ] 50 Rs = 30 Credits\n"
                "[ϟ] 100 Rs = 60 Credits\n"
                "[ϟ] 200 Rs = 120 Credits\n"
                "[ϟ] 500 Rs = 240 Credits\n"
                "[ϟ] 1000 Rs = 400 Credits\n\n"
                "💫 Use credits to enjoy OSINT bots with unlimited features.\n"
                "⚡ More credits = More power\n\n"
                "📩 To purchase, message 👉 @Royal_smart_boy")
        bot.send_message(call.message.chat.id, text)
        return

    if data == "getinfo":
        bot.send_message(call.message.chat.id, "🔍 Send the mobile number (without +91). Example: 98xxxxxxxx")
        bot.register_next_step_handler(call.message, handle_getinfo)
        return

    if data == "referral":
        row = get_user_row(uid)
        if not row:
            bot.answer_callback_query(call.id, "No data.")
            return
        ref_link = f"https://t.me/{bot.get_me().username}?start={uid}"
        needed = get_setting("referral_needed", "2")
        rate = get_setting("referral_rate", "1.5")
        bot.send_message(call.message.chat.id, f"👥 Your referral link:\n{ref_link}\n\nReward: Every {needed} referrals => {rate} credits")
        return

    if data == "bonus":
        row = get_user_row(uid)
        if not row:
            bot.answer_callback_query(call.id, "No data.")
            return
        last_bonus = row[5] or 0
        now = int(time.time())
        if now - last_bonus >= 86400:
            add_credits_to(uid, 0.5)
            conn, c = get_db()
            c.execute("UPDATE users SET last_bonus=? WHERE user_id=?", (now, uid))
            conn.commit()
            conn.close()
            log_action(uid, "Daily Bonus", "", 0.5)
            bot.send_message(call.message.chat.id, "🎁 You have received 0.5 credits! Come back after 24 hours.")
        else:
            left = int((86400 - (now - last_bonus)) / 3600)
            bot.send_message(call.message.chat.id, f"⏳ Bonus already claimed. Try again in {left} hours.")
        return

    if data == "status":
        if uid in get_admins() or uid == SUPER_ADMIN_ID:
            users_count = len(get_all_users())
            bot.send_message(call.message.chat.id, f"📊 Real Users: {users_count}")
        else:
            bot.send_message(call.message.chat.id, "📊 Users: 600+ Active")
        return

    if data == "giftcode":
        bot.send_message(call.message.chat.id, "🎟 Send /redeem CODE to redeem a gift code.")
        return

    if data == "support":
        # open chat to super admin (or designated support user). We'll send a quick message template for user to write and we'll forward to admins.
        bot.send_message(call.message.chat.id, "📞 To contact support, send your message and it will be forwarded to admins. Type your message now:")
        bot.register_next_step_handler(call.message, handle_support_message)
        return

    if data == "complaint":
        bot.send_message(call.message.chat.id, "✍️ Please write your complaint:")
        bot.register_next_step_handler(call.message, handle_complaint_from_user)
        return

    # ---------- ADMIN / SUPER ADMIN ----------

    # Admin panel open
    if data == "admin_panel":
        # check admin
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ You are not admin.")
            return
        bot.send_message(call.message.chat.id, "⚙️ Admin Panel:", reply_markup=admin_panel_markup())
        return

    # Superadmin panel open
    if data == "super_panel":
        if uid != SUPER_ADMIN_ID:
            bot.answer_callback_query(call.id, "❌ Only Super Admin.")
            return
        bot.send_message(call.message.chat.id, "👑 Super Admin Panel:", reply_markup=superadmin_panel_markup())
        return

    # Admin actions
    if data == "admin_addcredit":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        bot.send_message(uid, "💳 Send target user_id and amount (format: user_id amount). Example: 12345678 5")
        pending_addcredit[uid] = True
        return

    if data == "admin_zerocredit":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        bot.send_message(uid, "❌ Send user_id to reset credits to 0.")
        pending_zerocredit[uid] = True
        return

    if data == "admin_userlist":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        rows = get_all_users()
        text = f"👥 Users ({len(rows)}):\n"
        for r in rows[:200]:  # show first 200 to avoid huge messages
            text += f"{r[0]} | @{r[1]} | {r[2]} | Credits: {r[3]} | Refs: {r[4]}\n"
        bot.send_message(uid, text)
        return

    if data == "admin_broadcast":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        bot.send_message(uid, "📢 Send the broadcast message to send to all users:")
        pending_broadcast[uid] = True
        return

    if data == "admin_broadcast_admins":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        bot.send_message(uid, "📢 Send the message to broadcast to all admins:")
        pending_broadcast[f"admins_{uid}"] = True
        return

    if data == "admin_complaints":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        # For simplicity we don't store complaints permanently in DB in this code; admins receive complaints live.
        bot.send_message(uid, "📨 Complaints are forwarded to your chat when users send them.")
        return

    if data == "admin_referral_rate":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        bot.send_message(uid, "⚙️ Send new referral reward credits (example: 1.5) and optionally referrals needed (example: 2). Format: credits needed (e.g. 1.5 2)")
        pending_referral_rate[uid] = True
        return

    if data == "admin_channel_manager":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        kb = InlineKeyboardMarkup()
        kb.add(InlineKeyboardButton("➕ Add Channel", callback_data="chan_add"))
        kb.add(InlineKeyboardButton("➖ Remove Channel", callback_data="chan_remove"))
        kb.add(InlineKeyboardButton("📋 List Channels", callback_data="chan_list"))
        bot.send_message(uid, "Channel Manager:", reply_markup=kb)
        return

    # Channel manager handlers
    if data == "chan_add":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        bot.send_message(uid, "📢 Send channel username to add (format: @channelname)")
        pending_channel_add[uid] = True
        return

    if data == "chan_remove":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        bot.send_message(uid, "📢 Send channel username to remove (format: @channelname)")
        pending_channel_remove[uid] = True
        return

    if data == "chan_list":
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Only admins.")
            return
        chs = list_force_channels()
        if not chs:
            bot.send_message(uid, "❌ No force-join channels set.")
        else:
            bot.send_message(uid, "📋 Force-join channels:\n" + "\n".join(chs))
        return

    # Super Admin actions
    if data == "sa_admin_list":
        if uid != SUPER_ADMIN_ID:
            bot.answer_callback_query(call.id, "❌ Only Super Admin.")
            return
        admins_list = get_admins()
        text = "👮 Admins:\n" + "\n".join(str(a) for a in admins_list)
        # create buttons to logout each admin
        kb = InlineKeyboardMarkup()
        for a in admins_list:
            if a == SUPER_ADMIN_ID:
                kb.add(InlineKeyboardButton(f"Super Admin → {a}", callback_data="noop"))
            else:
                kb.add(InlineKeyboardButton(f"Logout {a}", callback_data=f"sa_logout_{a}"))
        bot.send_message(uid, text, reply_markup=kb)
        return

    if data.startswith("sa_logout_"):
        if uid != SUPER_ADMIN_ID:
            bot.answer_callback_query(call.id, "❌ Only Super Admin.")
            return
        try:
            tid = int(data.split("_")[-1])
            remove_admin(tid)
            bot.answer_callback_query(call.id, f"🚪 Admin {tid} logged out.")
            bot.send_message(tid, "🚪 You have been logged out by Super Admin.")
        except:
            bot.answer_callback_query(call.id, "❌ Error.")
        return

    if data == "sa_change_pass":
        if uid != SUPER_ADMIN_ID:
            bot.answer_callback_query(call.id, "❌ Only Super Admin.")
            return
        bot.send_message(uid, "🔑 Send new admin password:")
        pending_referral_rate[f"sa_pass_{uid}"] = True  # reusing pending dict for single-step flow
        return

    if data == "sa_gen_gift":
        if uid != SUPER_ADMIN_ID:
            bot.answer_callback_query(call.id, "❌ Only Super Admin.")
            return
        bot.send_message(uid, "🎟 Send credits amount for gift code generation (e.g. 5):")
        pending_gen_gift[uid] = True
        return

    if data == "sa_broadcast_admins":
        if uid != SUPER_ADMIN_ID:
            bot.answer_callback_query(call.id, "❌ Only Super Admin.")
            return
        bot.send_message(uid, "📢 Send message to broadcast to all admins:")
        pending_broadcast[f"admins_sa_{uid}"] = True
        return

    # logout noop or other placeholders
    if data == "noop":
        bot.answer_callback_query(call.id, "Info")
        return

    # reply-to-user action from admin on complaints
    if data.startswith("replyto_"):
        # callback like replyto_<user_id>
        if uid not in get_admins():
            bot.answer_callback_query(call.id, "❌ Not admin.")
            return
        try:
            target_user = int(data.split("_", 1)[1])
            pending_reply[uid] = target_user
            bot.send_message(uid, f"✍️ Write your reply to user {target_user}:")
        except:
            bot.answer_callback_query(call.id, "❌ Error.")
        return

    # unknown callback
    bot.answer_callback_query(call.id, "Unknown action.")

# ---------------- Message handlers for flows & commands ----------------

# Admin login command: /admin
@bot.message_handler(commands=["admin"])
def admin_login_cmd(message):
    bot.send_message(message.chat.id, "🔑 Send admin password:")
    bot.register_next_step_handler(message, admin_password_check)

def admin_password_check(message):
    text = message.text.strip()
    pwd = get_setting("admin_password", DEFAULT_ADMIN_PASSWORD)
    if text == pwd:
        add_admin(message.from_user.id)
        log_action(message.from_user.id, "Admin login")
        bot.send_message(message.chat.id, "✅ You are now an admin.", reply_markup=admin_panel_markup())
    else:
        bot.send_message(message.chat.id, "❌ Wrong password.")

# Superadmin command
@bot.message_handler(commands=["superadmin"])
def superadmin_cmd(message):
    if message.from_user.id != SUPER_ADMIN_ID:
        return
    bot.send_message(message.chat.id, "👑 Super Admin Panel:", reply_markup=superadmin_panel_markup())

# handle many pending states via generic message handler
@bot.message_handler(func=lambda m: True, content_types=['text'])
def global_message_handler(message):
    uid = message.from_user.id
    text = message.text.strip()

    # pending add credit
    if uid in pending_addcredit:
        try:
            parts = text.split()
            target = int(parts[0]); amount = float(parts[1])
            add_credits_to(target, amount)
            log_action(uid, "Add Credit", target, amount)
            bot.send_message(uid, f"✅ Added {amount} credits to {target}.")
            try:
                bot.send_message(target, f"💳 Admin added {amount} credits to your account.")
            except:
                pass
        except Exception as e:
            bot.send_message(uid, f"❌ Invalid format. Use: user_id amount")
        del pending_addcredit[uid]
        return

    # pending zero credit
    if uid in pending_zerocredit:
        try:
            target = int(text)
            set_credits(target, 0)
            log_action(uid, "Zero Credit", target, 0)
            bot.send_message(uid, f"⭕ Credits set to 0 for {target}.")
            try:
                bot.send_message(target, "⭕ Your credits were reset by an admin.")
            except:
                pass
        except:
            bot.send_message(uid, "❌ Invalid user id.")
        del pending_zerocredit[uid]
        return

    # pending broadcast to users
    if uid in pending_broadcast:
        msg = text
        rows = get_all_users()
        count = 0
        for r in rows:
            try:
                bot.send_message(r[0], f"📢 Broadcast from Admin:\n\n{msg}")
                count += 1
            except:
                pass
        bot.send_message(uid, f"✅ Broadcast sent to {count} users.")
        del pending_broadcast[uid]
        return

    # pending broadcast to admins (admins_{uid} key)
    if f"admins_{uid}" in pending_broadcast:
        msg = text
        admin_rows = get_admins()
        count = 0
        for a in admin_rows:
            try:
                bot.send_message(a, f"📢 Broadcast to Admins:\n\n{msg}")
                count += 1
            except:
                pass
        bot.send_message(uid, f"✅ Broadcast sent to {count} admins.")
        del pending_broadcast[f"admins_{uid}"]
        return

    # pending broadcast by superadmin (admins_sa)
    if f"admins_sa_{uid}" in pending_broadcast:
        msg = text
        admin_rows = get_admins()
        count = 0
        for a in admin_rows:
            try:
                bot.send_message(a, f"📢 Message from Super Admin:\n\n{msg}")
                count += 1
            except:
                pass
        bot.send_message(uid, f"✅ Sent to {count} admins.")
        del pending_broadcast[f"admins_sa_{uid}"]
        return

    # pending referral rate change
    if uid in pending_referral_rate:
        try:
            parts = text.split()
            if len(parts) == 1:
                new_rate = float(parts[0])
                set_setting("referral_rate", str(new_rate))
                bot.send_message(uid, f"✅ Referral reward updated to {new_rate} credits per threshold.")
            else:
                new_rate = float(parts[0]); needed = int(parts[1])
                set_setting("referral_rate", str(new_rate))
                set_setting("referral_needed", str(needed))
                bot.send_message(uid, f"✅ Referral updated: every {needed} refs => {new_rate} credits.")
        except:
            bot.send_message(uid, "❌ Invalid format. Example: 1.5 2")
        del pending_referral_rate[uid]
        return

    # pending channel add
    if uid in pending_channel_add:
        ch = text.strip()
        if not ch.startswith("@"):
            bot.send_message(uid, "❌ Invalid format. Use @channelusername")
        else:
            add_force_channel(ch)
            bot.send_message(uid, f"✅ Channel {ch} added as force-join.")
        del pending_channel_add[uid]
        return

    # pending channel remove
    if uid in pending_channel_remove:
        ch = text.strip()
        remove_force_channel(ch)
        bot.send_message(uid, f"✅ Channel {ch} removed (if existed).")
        del pending_channel_remove[uid]
        return

    # pending superadmin change password
    if f"sa_pass_{uid}" in pending_referral_rate:
        # next message is new password
        set_setting("admin_password", text)
        bot.send_message(uid, "✅ Admin password changed.")
        del pending_referral_rate[f"sa_pass_{uid}"]
        return

    # pending generate gift code
    if uid in pending_gen_gift:
        try:
            credits = float(text)
            code = create_giftcode(credits)
            bot.send_message(uid, f"🎟 Gift code created: <code>{code}</code> = {credits} credits")
            log_action(uid, "Create Gift", code, credits)
        except:
            bot.send_message(uid, "❌ Invalid amount.")
        del pending_gen_gift[uid]
        return

    # pending admin reply to user (complaint flow)
    if uid in pending_reply:
        target = pending_reply[uid]
        try:
            bot.send_message(target, f"🔔 <b>Admin Reply:</b>\n{message.text}")
            bot.send_message(uid, "✅ Reply sent to user.")
        except Exception as e:
            bot.send_message(uid, f"❌ Error sending reply: {e}")
        del pending_reply[uid]
        return

    # pending support message from user (forward to admins)
    # we'll treat any message typed after support prompt as support message
    # To keep it simple: if user types "support:" or was prompted recently - we won't track per-user state here
    # For safety, fallback: normal messages ignored
    # If user typed a number for getinfo we handle below.

    # GET INFO numeric handler (if user previously was asked to send number)
    # For simplicity: if message is digits of length >=6 assume it's a getinfo query
    if text.isdigit() and len(text) >= 6:
        # treat as info query
        handle_getinfo(message)
        return

    # Default: ignore or help
    # Provide menu again
    bot.send_message(uid, "Use the buttons below:", reply_markup=user_menu_markup())

# ---------------- Flow handlers ----------------

def handle_getinfo(message):
    # Called when user sends mobile number for getinfo
    user = message.from_user
    query = message.text.strip()
    bot.send_message(message.chat.id, "⏳ Searching...")
    if LEAKOSINT_API:
        # call LeakOSINT
        try:
            payload = {"token": LEAKOSINT_API, "request": query, "limit": LEAKOSINT_LIMIT, "lang": LEAKOSINT_LANG}
            r = requests.post(LEAKOSINT_URL, json=payload, timeout=20)
            data = r.json()
            # Format results in pages if necessary. Simplest: show top info or raw.
            if "List" in data:
                msg = f"🔍 Results for {query}:\n"
                for db_name, val in data["List"].items():
                    msg += f"\n<b>{db_name}</b>\n{val.get('InfoLeak','')}\n"
                bot.send_message(message.chat.id, msg)
            else:
                bot.send_message(message.chat.id, "No results or unexpected response.")
        except Exception as e:
            bot.send_message(message.chat.id, f"Error calling LeakOSINT: {e}")
    else:
        bot.send_message(message.chat.id, "LeakOSINT not configured. Please set LEAKOSINT_API to enable this feature.")

def handle_complaint_from_user(message):
    user = message.from_user
    text = message.text.strip()
    # forward complaint to all admins
    admins = get_admins()
    for a in admins:
        try:
            kb = InlineKeyboardMarkup()
            kb.add(InlineKeyboardButton("📨 Reply to user", callback_data=f"replyto_{user.id}"))
            bot.send_message(a, f"📩 New Complaint\nFrom: {user.first_name} (@{user.username or 'N/A'})\nID: {user.id}\n\n{text}", reply_markup=kb)
        except:
            pass
    bot.send_message(user.id, "✅ Your complaint was sent to admins. They will reply here.")
    log_action(user.id, "Complaint", text, 0)

def handle_support_message(message):
    user = message.from_user
    text = message.text.strip()
    # forward support to super admin and all admins
    admins = get_admins()
    recipients = set(admins) | {SUPER_ADMIN_ID}
    for a in recipients:
        try:
            kb = InlineKeyboardMarkup()
            kb.add(InlineKeyboardButton("📨 Reply to user", callback_data=f"replyto_{user.id}"))
            bot.send_message(a, f"📞 Support Message\nFrom: {user.first_name} (@{user.username or 'N/A'})\nID: {user.id}\n\n{text}", reply_markup=kb)
        except:
            pass
    bot.send_message(user.id, "✅ Your message has been forwarded to support/admins.")
    log_action(user.id, "Support", text, 0)

# ---------------- Redeem gift code command ----------------
@bot.message_handler(commands=["redeem"])
def redeem_cmd(message):
    parts = message.text.split(maxsplit=1)
    if len(parts) < 2:
        bot.send_message(message.chat.id, "Usage: /redeem CODE")
        return
    code = parts[1].strip().upper()
    ok, res = redeem_giftcode(message.from_user.id, code)
    if ok:
        bot.send_message(message.chat.id, f"🎉 Redeemed! You got {res} credits.")
        log_action(message.from_user.id, "Redeem Gift", code, res)
    else:
        bot.send_message(message.chat.id, f"❌ {res}")

# ---------------- Admin generate gift command (optional) ----------------
@bot.message_handler(commands=["gen_gift"])
def admin_gen_gift_cmd(message):
    if message.from_user.id != SUPER_ADMIN_ID and message.from_user.id not in get_admins():
        bot.send_message(message.chat.id, "❌ Only admins.")
        return
    bot.send_message(message.chat.id, "🎟 Send credits amount to create a gift code:")
    pending_gen_gift[message.from_user.id] = True

# ---------------- Utility commands ----------------
@bot.message_handler(commands=["mycredits"])
def mycredits_cmd(message):
    row = get_user_row(message.from_user.id)
    if row:
        bot.send_message(message.chat.id, f"💳 Your credits: {row[3]}")
    else:
        bot.send_message(message.chat.id, "No record found. Send /start first.")

@bot.message_handler(commands=["admins"])
def list_admins_cmd(message):
    if message.from_user.id != SUPER_ADMIN_ID:
        return
    admins = get_admins()
    bot.send_message(message.chat.id, "Admins:\n" + "\n".join(str(a) for a in admins))

# ---------------- Run ----------------
print("🤖 Bot started...")
bot.infinity_polling()

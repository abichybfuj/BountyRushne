import telebot
from telebot import types
import json
import os
import datetime
import feedparser
import random
import threading
import time

# ===== التوكن من Render =====
TOKEN = os.getenv("8686210830:AAGZmXopHLTELnyszWr7wsyhouPmr74vVjk")
bot = telebot.TeleBot(TOKEN)

# ===== ملفات =====
USERS_FILE = "users.json"
LAST_VIDEO_FILE = "last_video.json"

# ===== تحميل المستخدمين =====
if os.path.exists(USERS_FILE):
    with open(USERS_FILE, "r") as f:
        users = json.load(f)
else:
    users = {}

# ===== تحميل آخر فيديو =====
if os.path.exists(LAST_VIDEO_FILE):
    with open(LAST_VIDEO_FILE, "r") as f:
        last_video = json.load(f)
else:
    last_video = {"id": ""}

# ===== العروض =====
OFFERS = ["حساب باونتي #1","حساب باونتي #2","حساب باونتي #3"]

# ===== RSS =====
YOUTUBE_RSS = "https://www.youtube.com/feeds/videos.xml?channel_id=UCqly9F4Fr_jf2Y1Cy5hacRg"

# ===== حفظ =====
def save_users():
    with open(USERS_FILE, "w") as f:
        json.dump(users, f, indent=4)

def save_last_video():
    with open(LAST_VIDEO_FILE, "w") as f:
        json.dump(last_video, f, indent=4)

# ===== الأزرار =====
def main_keyboard():
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.row("🎁 الهدية اليومية", "🛒 العروض")
    kb.row("📰 الأخبار", "👥 عدد المشتركين")
    kb.row("💎 نقاطي")
    return kb

# ===== فلترة فيديوهات =====
def check_new_video():
    global last_video
    feed = feedparser.parse(YOUTUBE_RSS)

    if feed.entries:
        latest = feed.entries[0]
        video_id = latest.id
        title = latest.title.lower()

        keywords = ["bounty rush", "opbr", "one piece bounty"]

        if video_id != last_video["id"] and any(k in title for k in keywords):
            last_video["id"] = video_id
            save_last_video()

            for chat_id in users.keys():
                try:
                    bot.send_message(chat_id, f"🔥 فيديو جديد!\n\n🎬 {latest.title}\n{latest.link}")
                except:
                    pass

# ===== لوب التحقق =====
def loop():
    while True:
        try:
            check_new_video()
        except:
            pass
        time.sleep(300)

threading.Thread(target=loop, daemon=True).start()

# ===== الرسائل =====
@bot.message_handler(func=lambda m: True)
def handle_message(message):
    chat_id = str(message.chat.id)
    text = message.text

    # تسجيل مستخدم
    if chat_id not in users:
        users[chat_id] = {"points":0,"last_daily":""}
        save_users()
        bot.send_message(chat_id, "🔥 أهلاً بك!", reply_markup=main_keyboard())

    today = datetime.date.today().isoformat()

    if text == "🎁 الهدية اليومية":
        if users[chat_id]["last_daily"] != today:
            users[chat_id]["points"] += 1
            users[chat_id]["last_daily"] = today
            save_users()
            bot.send_message(chat_id, f"✅ أخذت نقطة! نقاطك: {users[chat_id]['points']}", reply_markup=main_keyboard())
        else:
            bot.send_message(chat_id, "⚠️ تعال بكرة!", reply_markup=main_keyboard())

    elif text == "🛒 العروض":
        if users[chat_id]["points"] >= 10:
            offer = random.choice(OFFERS)
            users[chat_id]["points"] -= 10
            save_users()

            bot.send_message(
                chat_id,
                f"🎉 اشتريت: {offer}\n💎 نقاطك الآن: {users[chat_id]['points']}",
                reply_markup=main_keyboard()
            )
        else:
            bot.send_message(chat_id, "❌ تحتاج 10 نقاط", reply_markup=main_keyboard())

    elif text == "📰 الأخبار":
        feed = feedparser.parse(YOUTUBE_RSS)
        msgs = []
        for e in feed.entries[:5]:
            if any(k in e.title.lower() for k in ["bounty rush", "opbr"]):
                msgs.append(f"🎬 {e.title}\n{e.link}")
        if msgs:
            bot.send_message(chat_id, "\n\n".join(msgs), reply_markup=main_keyboard())
        else:
            bot.send_message(chat_id, "❌ ما فيه أخبار حالياً", reply_markup=main_keyboard())

    elif text == "👥 عدد المشتركين":
        bot.send_message(chat_id, f"👥 العدد: {len(users)}", reply_markup=main_keyboard())

    elif text == "💎 نقاطي":
        bot.send_message(chat_id, f"💎 نقاطك: {users[chat_id]['points']}", reply_markup=main_keyboard())

    else:
        bot.send_message(chat_id, "⚠️ استخدم الأزرار فقط", reply_markup=main_keyboard())

# ===== تشغيل البوت =====
bot.infinity_polling()

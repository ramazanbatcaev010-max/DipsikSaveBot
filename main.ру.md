# main.ру  
  
import os  
import re  
import json  
import asyncio  
from flask import Flask  
from threading import Thread  
from aiogram import Bot, Dispatcher, types, F  
from aiogram.filters import Command  
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton  
from aiogram.enums import ParseMode  
import yt_dlp  
  
# ========== НАСТРОЙКИ ==========  
BOT_TOKEN = os.getenv("BOT_TOKEN")  
CHANNEL_ID = os.getenv("CHANNEL_ID")  
ADMIN_ID = 5070313143  
  
bot = Bot(token=BOT_TOKEN)  
dp = Dispatcher()  
  
STATS_FILE = "stats.json"  
user_languages = {}  
  
texts = {  
    "ru": {  
        "not_subscribed": "❌ Ты не подписан на канал {channel}\n📢 Подпишись и нажми кнопку ниже!",  
        "subscribed": "✅ Спасибо за подписку! Скачиваю видео...",  
        "downloading": "⏳ Скачиваю видео, подожди немного...",  
        "error": "❌ Ошибка при скачивании. Проверь ссылку или попробуй позже.",  
        "too_big": "❌ Видео слишком большое (больше 50 МБ).",  
        "start": "👋 Привет! Я бот для скачивания видео из TikTok, YouTube, Instagram, Pinterest, Likee.\n\n📥 Просто отправь мне ссылку на видео.\n\n📢 Подпишись на канал {channel} чтобы пользоваться ботом.",  
        "help": "📥 ИНСТРУКЦИЯ:\n\n1. Открой TikTok/YouTube/Instagram/Pinterest/Likee\n2. Нажми «Поделиться» → «Скопировать ссылку»\n3. Отправь ссылку боту\n4. Получи видео без водяного знака",  
        "stats": "📊 СТАТИСТИКА\n\n👥 Пользователей: {users}\n📥 Скачиваний: {total}\n\n🎯 Платформы:\n{tops}",  
        "no_access": "⛔ Нет доступа.",  
        "lang_changed": "🌐 Язык: Русский",  
        "choose_lang": "🌐 Выбери язык:"  
    },  
    "en": {  
        "not_subscribed": "❌ Subscribe to {channel}\n📢 Click below!",  
        "subscribed": "✅ Thanks! Downloading...",  
        "downloading": "⏳ Downloading, please wait...",  
        "error": "❌ Error. Check link or try again.",  
        "too_big": "❌ Video too large (>50 MB).",  
        "start": "👋 Hi! Download videos from TikTok, YouTube, Instagram, Pinterest, Likee.\n\n📥 Send me a link.\n\n📢 Subscribe to {channel}",  
        "help": "📥 INSTRUCTION:\n\n1. Open TikTok/YouTube/Instagram/Pinterest/Likee\n2. Tap Share → Copy link\n3. Send link to bot\n4. Get video without watermark",  
        "stats": "📊 STATISTICS\n\n👥 Users: {users}\n📥 Downloads: {total}\n\n🎯 Platforms:\n{tops}",  
        "no_access": "⛔ No access.",  
        "lang_changed": "🌐 Language: English",  
        "choose_lang": "🌐 Choose language:"  
    }  
}  
  
def get_lang(user_id):  
    return user_languages.get(user_id, "ru")  
  
def load_stats():  
    if os.path.exists(STATS_FILE):  
        with open(STATS_FILE, "r") as f:  
            return json.load(f)  
    return {"users": 0, "total": 0, "platforms": {"TikTok": 0, "YouTube": 0, "Instagram": 0, "Pinterest": 0, "Likee": 0}}  
  
def save_stats(stats):  
    with open(STATS_FILE, "w") as f:  
        json.dump(stats, f)  
  
def detect_platform(url):  
    if "tiktok.com" in url:  
        return "TikTok"  
    elif "youtube.com" in url or "youtu.be" in url:  
        return "YouTube"  
    elif "instagram.com" in url:  
        return "Instagram"  
    elif "pinterest.com" in url or "pin.it" in url:  
        return "Pinterest"  
    elif "likee.com" in url or "likee.video" in url:  
        return "Likee"  
    return None  
  
async def download_video(url, user_id):  
    stats = load_stats()  
    platform = detect_platform(url)  
      
    ydl_opts = {  
        'format': 'best[height<=720]/best',  
        'outtmpl': 'downloads/%(title)s_%(id)s.%(ext)s',  
        'quiet': True,  
        'no_warnings': True,  
        'max_filesize': 50 * 1024 * 1024,  
    }  
      
    try:  
        os.makedirs("downloads", exist_ok=True)  
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:  
            info = ydl.extract_info(url, download=True)  
            filename = ydl.prepare_filename(info)  
            if not os.path.exists(filename):  
                for ext in ['.mp4', '.webm', '.mkv']:  
                    test_path = filename.rsplit('.', 1)[0] + ext  
                    if os.path.exists(test_path):  
                        filename = test_path  
                        break  
              
            stats["total"] += 1  
            if platform:  
                stats["platforms"][platform] += 1  
            save_stats(stats)  
            return filename  
    except Exception as e:  
        if "filesize" in str(e).lower():  
            return "too_big"  
        return None  
  
def is_subscribed(user_id):  
    try:  
        import asyncio  
        result = asyncio.run(bot.get_chat_member(chat_id=CHANNEL_ID, user_id=user_id))  
        return result.status in ["member", "creator", "administrator"]  
    except:  
        return False  
  
# ========== КОМАНДЫ ==========  
@dp.message(Command("start"))  
async def start_cmd(message: types.Message):  
    lang = get_lang(message.from_user.id)  
    keyboard = InlineKeyboardMarkup(inline_keyboard=[  
        [InlineKeyboardButton(text="📢 Подписаться" if lang == "ru" else "📢 Subscribe", url=f"https://t.me/{CHANNEL_ID[1:]}")],  
        [InlineKeyboardButton(text="✅ Я подписался" if lang == "ru" else "✅ I subscribed", callback_data="check_sub")]  
    ])  
    await message.answer(texts[lang]["start"].format(channel=CHANNEL_ID), reply_markup=keyboard)  
  
@dp.message(Command("help"))  
async def help_cmd(message: types.Message):  
    lang = get_lang(message.from_user.id)  
    keyboard = InlineKeyboardMarkup(inline_keyboard=[  
        [InlineKeyboardButton(text="➕ Добавить бота в группу", url=f"https://t.me/{bot.username}?startgroup=start")]  
    ])  
    await message.answer(texts[lang]["help"], reply_markup=keyboard)  
  
@dp.message(Command("language"))  
async def language_cmd(message: types.Message):  
    keyboard = InlineKeyboardMarkup(inline_keyboard=[  
        [InlineKeyboardButton(text="🇷🇺 Русский", callback_data="lang_ru")],  
        [InlineKeyboardButton(text="🇬🇧 English", callback_data="lang_en")]  
    ])  
    await message.answer(texts[get_lang(message.from_user.id)]["choose_lang"], reply_markup=keyboard)  
  
@dp.message(Command("stats"))  
async def stats_cmd(message: types.Message):  
    if message.from_user.id != ADMIN_ID:  
        await message.answer(texts[get_lang(message.from_user.id)]["no_access"])  
        return  
    stats = load_stats()  
    tops = "\n".join([f"- {k}: {v}" for k, v in stats["platforms"].items()])  
    await message.answer(texts["ru"]["stats"].format(users=stats["users"], total=stats["total"], tops=tops))  
  
@dp.callback_query(lambda c: c.data.startswith("lang_"))  
async def change_lang(callback: types.CallbackQuery):  
    lang_code = "ru" if callback.data == "lang_ru" else "en"  
    user_languages[callback.from_user.id] = lang_code  
    await callback.message.edit_text(texts[lang_code]["lang_changed"])  
    await callback.answer()  
  
@dp.callback_query(lambda c: c.data == "check_sub")  
async def check_sub(callback: types.CallbackQuery):  
    lang = get_lang(callback.from_user.id)  
    if is_subscribed(callback.from_user.id):  
        await callback.message.edit_text(texts[lang]["subscribed"])  
        await callback.answer()  
    else:  
        await callback.answer(texts[lang]["not_subscribed"].format(channel=CHANNEL_ID), show_alert=True)  
  
@dp.message(F.text)  
async def handle_link(message: types.Message):  
    lang = get_lang(message.from_user.id)  
    url = message.text.strip()  
      
    if not is_subscribed(message.from_user.id):  
        keyboard = InlineKeyboardMarkup(inline_keyboard=[  
            [InlineKeyboardButton(text="📢 Подписаться" if lang == "ru" else "📢 Subscribe", url=f"https://t.me/{CHANNEL_ID[1:]}")],  
            [InlineKeyboardButton(text="✅ Я подписался" if lang == "ru" else "✅ I subscribed", callback_data="check_sub")]  
        ])  
        await message.answer(texts[lang]["not_subscribed"].format(channel=CHANNEL_ID), reply_markup=keyboard)  
        return  
      
    await message.answer(texts[lang]["downloading"])  
    video_path = await download_video(url, message.from_user.id)  
      
    if video_path == "too_big":  
        await message.answer(texts[lang]["too_big"])  
    elif video_path and os.path.exists(video_path):  
        with open(video_path, "rb") as video:  
            await message.answer_video(video, caption=f"❤️ Скачано в @{bot.username}")  
        os.remove(video_path)  
    else:  
        await message.answer(texts[lang]["error"])  
  
# ========== FLASK ДЛЯ RENDER ==========  
app = Flask(__name__)  
  
@app.route('/')  
@app.route('/health')  
def health():  
    return "OK", 200  
  
def run_flask():  
    port = int(os.environ.get("PORT", 8080))  
    app.run(host='0.0.0.0', port=port)  
  
# ========== ЗАПУСК ==========  
async def main():  
    stats = load_stats()  
    save_stats(stats)  
      
    flask_thread = Thread(target=run_flask)  
    flask_thread.start()  
      
    await dp.start_polling(bot)  
  
if __name__ == "__main__":  
    asyncio.run(main())  

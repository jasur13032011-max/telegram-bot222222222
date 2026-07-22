# telegram-bot222222222
Mana aiogram 3.x yordamida barcha turdagi media fayllar bilan ishlash, ularni yuklab olish, filterlash va jo'natishning to'liq kodi hamda file_id keshlash mexanizmi bo'yicha tushuntirish.

🛠 Telegram Bot Kodi (media_bot.py)
Python
import os
from pathlib import Path
from io import BytesIO

from aiogram import Bot, Dispatcher, F
from aiogram.filters import Command
from aiogram.types import (
    Message,
    FSInputFile,
    URLInputFile,
    BufferedInputFile,
    InlineKeyboardMarkup,
    InlineKeyboardButton,
)
from aiogram.utils.media_group import MediaGroupBuilder

# ------------------------------------------------------------------
# CONFIG & CACHE
# ------------------------------------------------------------------
BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"
UPLOAD_DIR = Path("uploads")
UPLOAD_DIR.mkdir(exist_ok=True)  # uploads/ papkasini yaratish

# File ID Cache Pattern (Server resurslarini tejash uchun)
FILE_CACHE: dict[str, str] = {}

dp = Dispatcher()

# ------------------------------------------------------------------
# 1. JO'NATISH HANDLERLARI (/photo, /doc, /album, /voice)
# ------------------------------------------------------------------

@dp.message(Command("photo"))
async def send_photo_demo(message: Message, bot: Bot):
    """
    Fayl yuborishning 4 xil varianti:
    1. FSInputFile — mahalliy xotiradan
    2. URLInputFile — internetdagi havoladan
    3. BufferedInputFile — xotiradagi (RAM) baytlardan
    4. file_id — Telegram serverida saqlangan keshdagi fayldan
    """
    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="🔗 Batafsil", url="https://core.telegram.org")]
        ]
    )

    # 4-variant: Agarda file_id keshda bo'lsa, tezkor jo'natamiz
    if "demo_photo" in FILE_CACHE:
        await message.answer_photo(
            photo=FILE_CACHE["demo_photo"],
            caption="⚡ **file_id (kesh)** orqali lahzada yuborildi!",
            reply_markup=keyboard,
            parse_mode="Markdown"
        )
        return

    # 2-variant: URLInputFile orqali yuborish
    url_file = URLInputFile("https://picsum.photos/800/600")
    
    sent_msg = await message.answer_photo(
        photo=url_file,
        caption="🖼 **URLInputFile** orqali yuklandi va yuborildi.",
        reply_markup=keyboard,
        parse_mode="Markdown"
    )

    # Yuborilgan faylning file_id'sini keshga saqlaymiz
    FILE_CACHE["demo_photo"] = sent_msg.photo[-1].file_id


@dp.message(Command("doc"))
async def send_doc_demo(message: Message):
    # BufferedInputFile (RAM'da fayl yaratib yuborish)
    content = b"Hello, this is a dynamically generated text document!"
    buffered_file = BufferedInputFile(file=content, filename="sample.txt")

    await message.answer_document(
        document=buffered_file,
        caption="📄 **BufferedInputFile** orqali RAM'dan yaratilgan fayl."
    )


@dp.message(Command("album"))
async def send_album_demo(message: Message):
    """MediaGroupBuilder yordamida 4+ rasmdan iborat albom yuborish."""
    album_builder = MediaGroupBuilder(caption="🖼 **4 ta rasmdan iborat albom**")

    # Turli manbalardan rasm qo'shish
    album_builder.add_photo(media=URLInputFile("https://picsum.photos/800/600?1"))
    album_builder.add_photo(media=URLInputFile("https://picsum.photos/800/600?2"))
    album_builder.add_photo(media=URLInputFile("https://picsum.photos/800/600?3"))
    album_builder.add_photo(media=URLInputFile("https://picsum.photos/800/600?4"))

    await message.answer_media_group(media=album_builder.build())


@dp.message(Command("voice"))
async def send_voice_demo(message: Message):
    # Keshda bo'lsa file_id orqali yuboramiz
    if "demo_voice" in FILE_CACHE:
        await message.answer_voice(voice=FILE_CACHE["demo_voice"], caption="🎙 Keshdagi ovozli xabar.")
        return

    await message.answer("ℹ️ Hali keshda voice mavjud emas. Botga ovozli xabar yuboring!")

# ------------------------------------------------------------------
# 2. QABUL QILISH VA FILTERLASH HANDLERLARI
# ------------------------------------------------------------------

# Rasm qabul qilish, yuklab olish va uploads/ papkasiga saqlash
@dp.message(F.photo)
async def handle_photo(message: Message, bot: Bot):
    # Eng yuqori sifatli rasmni olish (oxirgi element)
    photo = message.photo[-1]
    
    # Telegram serveridan fayl ma'lumotlarini olish
    file_info = await bot.get_file(photo.file_id)
    save_path = UPLOAD_DIR / f"{photo.file_unique_id}.jpg"

    # Local xotiraga yuklab olish
    await bot.download_file(file_info.file_path, destination=save_path)

    # Keshga saqlash
    FILE_CACHE["last_user_photo"] = photo.file_id

    await message.reply(
        f"✅ Rasm qabul qilindi va saqlandi!\n"
        f"📁 Fayl yo'li: `{save_path}`\n"
        f"🆔 File ID: `{photo.file_id[:20]}...`",
        parse_mode="Markdown"
    )


# Hujjat qabul qilish: Hajm (<= 5MB) va MIME Type filtri
@dp.message(F.document)
async def handle_document(message: Message):
    doc = message.document
    max_size = 5 * 1024 * 1024  # 5 MB

    # 1. Hajm filtri
    if doc.file_size > max_size:
        await message.reply("❌ Fayl hajmi juda katta! Maksimal ruxsat etilgan hajm: **5 MB**.")
        return

    # 2. MIME type filtri (Faqat PDF va ZIP ruxsat beriladi)
    allowed_mimes = ["application/pdf", "application/zip", "image/png"]
    if doc.mime_type not in allowed_mimes:
        await message.reply(
            f"❌ Noto'g'ri fayl formati (`{doc.mime_type}`).\n"
            f"Faqat **PDF**, **ZIP** yoki **PNG** yuborishingiz mumkin."
        )
        return

    await message.reply(f"📄 Hujjat qabul qilindi: **{doc.file_name}** ({doc.file_size / 1024:.1f} KB)")


# Ovozli xabar: Duration (Davomiyligi <= 30 soniya) filtri
@dp.message(F.voice)
async def handle_voice(message: Message):
    voice = message.voice
    max_duration = 30  # soniya

    if voice.duration > max_duration:
        await message.reply("⏱ Ovozli xabar juda uzun! Maksimal davomiylik: **30 soniya**.")
        return

    # Keshga saqlab qo'yamiz (/voice komandasi uchun)
    FILE_CACHE["demo_voice"] = voice.file_id
    await message.reply(f"🎙 Ovozli xabar qabul qilindi ({voice.duration}s) va keshga saqlandi!")


# Stiker: Kelgan stikerni o'ziga qaytarib yuborish (Echo)
@dp.message(F.sticker)
async def handle_sticker(message: Message):
    await message.answer_sticker(sticker=message.sticker.file_id)


# Lokatsiya: Google Maps havolasini yaratish
@dp.message(F.location)
async def handle_location(message: Message):
    lat = message.location.latitude
    lon = message.location.longitude
    
    google_maps_url = f"https://www.google.com/maps?q={lat},{lon}"

    kb = InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="🗺 Google Maps'da ochish", url=google_maps_url)]
        ]
    )

    await message.reply(
        f"📍 **Joylashuv qabul qilindi!**\n\n"
        f"• Kenglik (Lat): `{lat}`\n"
        f"• Uzoqlik (Lon): `{lon}`",
        reply_markup=kb,
        parse_mode="Markdown"
    )

# ------------------------------------------------------------------
# MAIN RUNNER
# ------------------------------------------------------------------
async def main():
    bot = Bot(token=BOT_TOKEN)
    await dp.start_polling(bot)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
⚡ File ID Cache Pattern Tushuntirishi
Telegram botlarda fayllar (rasm, video, hujjat, audio) bilan ishlashda file_id keshini qo'llash quyidagi afzalliklarni beradi:

🔄 Ishlash prinsipi:
[Fayl Birinchi Marta Yuborilganda]
Server Local/URL fayl ──> Telegram Serverga Yuklanadi ──> Telegram 'file_id' Qaytaradi ──> Keshga Saqlanadi

[Ayni Fayl Qayta Yuborilganda]
Keshdan 'file_id' Olinadi ──> Telegram Serverga 'file_id' Yuboriladi (Qayta yuklanmaydi) ──> Lahzada Foydalanuvchiga Yetadi
🎯 Nega file_id Kesh ishlatilishi kerak?
Tezlik (Latency):
Faylni local diskdan yoki URL'dan qayta-qayta Telegram'ga yuklash (upload) vaqt oladi. file_id ishlatilganda Telegram serveri allaqachon o'zida saqlangan faylni qayta yuklamasdan darhol foydalanuvchiga uzatadi.

Trafik va Resurs Tejamkorligi:
Serveringizning chiquvchi tarmoq grafigi (Outbound Bandwidth) va CPU yuklamasi sezilarli darajada kamayadi.

Telegram API Cheklovlari (Rate Limits):
Katta hajmdagi fayllarni qayta-qayta yuklash API cheklovlariga (FloodWait) tushish xavfini oshiradi. file_id bilan jo'natish bu xavfni kamaytiradi.

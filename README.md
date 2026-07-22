# telegram-bot222222222
Aiogram 3.x kutubxonasida Python'da yozilgan Telegram bot kodi. Kod 3 darajali menyu, Unsplash rasmlari, dict-bazasi va HTML formatlash standartlariga to'liq javob beradi.

Telegram Bot Kodi (bot.py)
Python
import asyncio
import logging
from aiogram import Bot, Dispatcher, F, types
from aiogram.enums import ParseMode
from aiogram.filters import CommandStart
from aiogram.types import (
    KeyboardButton,
    ReplyKeyboardMarkup,
    ReplyKeyboardRemove,
    URLInputFile,
)

# Bot tokeningizni kiriting
BOT_TOKEN = "BOT_TOKENINI_SHUYERGA_YOZING"

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# ----------------------------------------------------------------------
# 1. MA'LUMOTLAR BAZASI (PRODUCTS dict)
# ----------------------------------------------------------------------
PRODUCTS = {
    "Taomlar": {
        "🍕 Pizza": {
            "name": "Margarita Pizzasi",
            "price": "75,000 so'm",
            "desc": "Yangi pomidorlar, Motsarella pishlog'i va rayhon barglari bilan tayyorlangan klassik pissa.",
            "image": "https://images.unsplash.com/photo-1513104890138-7c749659a591?auto=format&fit=crop&w=800&q=80",
        },
        "🍔 Burger": {
            "name": "Cheeseburger Double",
            "price": "45,000 so'm",
            "desc": "Ikkita suvli mol go'shti kotleti, Cheddar pishlog'i va maxsus sous.",
            "image": "https://images.unsplash.com/photo-1568901346375-23c9450c58cd?auto=format&fit=crop&w=800&q=80",
        },
        "🌮 Taco": {
            "name": "Meksikancha Taco",
            "price": "35,000 so'm",
            "desc": "Qiyma go'sht, makkajo'xori, lobiya va achchiq sous bilan tortilgan Taco.",
            "image": "https://images.unsplash.com/photo-1565299585323-38d6b0865b47?auto=format&fit=crop&w=800&q=80",
        },
        "🍜 Lag'mon": {
            "name": "Uyg'urcha Lag'mon",
            "price": "40,000 so'm",
            "desc": "Klassik usulda cho'zilgan xamir, yangi sabzavotlar va mol go'shti.",
            "image": "https://images.unsplash.com/photo-1569718212165-3a8278d5f624?auto=format&fit=crop&w=800&q=80",
        },
    },
    "Ichimliklar": {
        "☕ Kofe": {
            "name": "Cappuccino",
            "price": "22,000 so'm",
            "desc": "Xushbo'y espresso va yumshoq sut ko'pigi.",
            "image": "https://images.unsplash.com/photo-1534778101976-62847782c213?auto=format&fit=crop&w=800&q=80",
        },
        "🍵 Choy": {
            "name": "Ko'k Choy (Limon bilan)",
            "price": "10,000 so'm",
            "desc": "Oliy navli ko'k choy va yangi kesilgan limon tilimlari.",
            "image": "https://images.unsplash.com/photo-1576092768241-dec231879fc3?auto=format&fit=crop&w=800&q=80",
        },
        "🧃 Sok": {
            "name": "Yangi Siqilgan Apelsin Sharbati",
            "price": "25,000 so'm",
            "desc": "100% tabiiy va yangi uzilgan apelsin sharbati.",
            "image": "https://images.unsplash.com/photo-1613478223719-2ab802602423?auto=format&fit=crop&w=800&q=80",
        },
    },
}

# ----------------------------------------------------------------------
# 2. TUGMALAR (KEYBOARDS)
# ----------------------------------------------------------------------

# 1-daraja: Bosh menyu
main_menu = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(text="🍕 Taomlar"), KeyboardButton(text="🥤 Ichimliklar")],
        [KeyboardButton(text="📞 Bog'lanish"), KeyboardButton(text="ℹ️ Bot haqida")],
    ],
    resize_keyboard=True,
)

# 2-daraja: Taomlar submenyu
taomlar_menu = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(text="🍕 Pizza"), KeyboardButton(text="🍔 Burger")],
        [KeyboardButton(text="🌮 Taco"), KeyboardButton(text="🍜 Lag'mon")],
        [KeyboardButton(text="⬅️ Orqaga")],
    ],
    resize_keyboard=True,
)

# 2-daraja: Ichimliklar submenyu
ichimliklar_menu = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(text="☕ Kofe"), KeyboardButton(text="🍵 Choy")],
        [KeyboardButton(text="🧃 Sok")],
        [KeyboardButton(text="⬅️ Orqaga")],
    ],
    resize_keyboard=True,
)

# 3-daraja: Kontent oynasidagi "Orqaga" tugmalari
back_to_taomlar = ReplyKeyboardMarkup(
    keyboard=[[KeyboardButton(text="⬅️ Taomlarga qaytish")]], resize_keyboard=True
)

back_to_ichimliklar = ReplyKeyboardMarkup(
    keyboard=[[KeyboardButton(text="⬅️ Ichimliklarga qaytish")]], resize_keyboard=True
)


# ----------------------------------------------------------------------
# 3. HANDLERLAR
# ----------------------------------------------------------------------


# /start buyrug'i
@dp.message(CommandStart())
async def cmd_start(message: types.Message):
    await message.answer(
        f"Xush kelibsiz, <b>{message.from_user.full_name}</b>! 👋\n\n"
        f"Kerakli bo'limni tanlang:",
        reply_markup=main_menu,
        parse_mode=ParseMode.HTML,
    )


# 1-DARAJA: Bosh menyu tugmalari
@dp.message(F.text == "🍕 Taomlar")
async def show_taomlar(message: types.Message):
    await message.answer("<b>Taomlar bo'limi:</b>", reply_markup=taomlar_menu, parse_mode=ParseMode.HTML)


@dp.message(F.text == "🥤 Ichimliklar")
async def show_ichimliklar(message: types.Message):
    await message.answer("<b>Ichimliklar bo'limi:</b>", reply_markup=ichimliklar_menu, parse_mode=ParseMode.HTML)


@dp.message(F.text == "📞 Bog'lanish")
async def show_contact(message: types.Message):
    text = (
        "<b>📞 Biz bilan bog'lanish:</b>\n\n"
        "📱 Telefon: +998 90 123 45 67\n"
        "💬 Telegram: @FastFoodSupportBot\n"
        "📍 Manzil: Toshkent sh., Amir Temur ko'chasi, 10-uy"
    )
    await message.answer(text, parse_mode=ParseMode.HTML)


@dp.message(F.text == "ℹ️ Bot haqida")
async def show_about(message: types.Message):
    text = (
        "<b>ℹ️ Bot haqida:</b>\n\n"
        "Bu bot orqali siz mazali taomlar va ichimliklarni osongina ko'rib chiqishingiz va buyurtma berishingiz mumkin.\n"
        "<i>Bot Aiogram 3.x texnologiyasi asosida ishlab chiqilgan.</i> 🚀"
    )
    await message.answer(text, parse_mode=ParseMode.HTML)


# 2-DARAJA → 1-DARAJA: Orqaga qaytish (Bosh menyuga)
@dp.message(F.text == "⬅️ Orqaga")
async def back_to_main(message: types.Message):
    await message.answer("<b>Bosh menyu:</b>", reply_markup=main_menu, parse_mode=ParseMode.HTML)


# 3-DARAJA → 2-DARAJA: Taomlar yoki Ichimliklarga qaytish
@dp.message(F.text == "⬅️ Taomlarga qaytish")
async def back_to_taomlar_handler(message: types.Message):
    await message.answer("<b>Taomlar bo'limi:</b>", reply_markup=taomlar_menu, parse_mode=ParseMode.HTML)


@dp.message(F.text == "⬅️ Ichimliklarga qaytish")
async def back_to_ichimliklar_handler(message: types.Message):
    await message.answer("<b>Ichimliklar bo'limi:</b>", reply_markup=ichimliklar_menu, parse_mode=ParseMode.HTML)


# 3-DARAJA KONTENT: Taomlar mahsulotlari
@dp.message(F.text.in_(["🍕 Pizza", "🍔 Burger", "🌮 Taco", "🍜 Lag'mon"]))
async def show_food_product(message: types.Message):
    product = PRODUCTS["Taomlar"][message.text]

    caption = (
        f"<b>{product['name']}</b>\n\n"
        f"📝 <b>Tavsif:</b> {product['desc']}\n"
        f"💰 <b>Narxi:</b> <u>{product['price']}</u>"
    )

    image_file = URLInputFile(product["image"])
    await message.answer_photo(
        photo=image_file,
        caption=caption,
        parse_mode=ParseMode.HTML,
        reply_markup=back_to_taomlar,
    )


# 3-DARAJA KONTENT: Ichimliklar mahsulotlari
@dp.message(F.text.in_(["☕ Kofe", "🍵 Choy", "🧃 Sok"]))
async def show_drink_product(message: types.Message):
    product = PRODUCTS["Ichimliklar"][message.text]

    caption = (
        f"<b>{product['name']}</b>\n\n"
        f"📝 <b>Tavsif:</b> {product['desc']}\n"
        f"💰 <b>Narxi:</b> <u>{product['price']}</u>"
    )

    image_file = URLInputFile(product["image"])
    await message.answer_photo(
        photo=image_file,
        caption=caption,
        parse_mode=ParseMode.HTML,
        reply_markup=back_to_ichimliklar,
    )


# ECHO HANDLER (Eng oxirida bo'lishi shart!)
@dp.message()
async def echo_handler(message: types.Message):
    await message.answer(
        "❓ Noma'lum buyruq. Iltimos, menyudagi tugmalardan birini tanlang.",
        reply_markup=main_menu,
    )


# ----------------------------------------------------------------------
# 4. BOTNI ISHGA TUSHIRISH
# ----------------------------------------------------------------------
async def main():
    logging.basicConfig(level=logging.INFO)
    print("Bot muvaffaqiyatli ishga tushdi...")
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
Navigatsiya sxemasi (3 daraja)
Plaintext
[ 1-Daraja: Bosh Menyu ]
   ├── 🍕 Taomlar
   │      ├── 🍕 Pizza ──────────> [ 3-Daraja: Rasm + Narx + Tavsif ]
   │      ├── 🍔 Burger ─────────> [ 3-Daraja: Rasm + Narx + Tavsif ]
   │      ├── 🌮 Taco ───────────> [ 3-Daraja: Rasm + Narx + Tavsif ]
   │      ├── 🍜 Lag'mon ────────> [ 3-Daraja: Rasm + Narx + Tavsif ]
   │      └── ⬅️ Orqaga ──────────> (Bosh Menyuga qaytadi)
   │
   ├── 🥤 Ichimliklar
   │      ├── ☕ Kofe ───────────> [ 3-Daraja: Rasm + Narx + Tavsif ]
   │      ├── 🍵 Choy ───────────> [ 3-Daraja: Rasm + Narx + Tavsif ]
   │      ├── 🧃 Sok ────────────> [ 3-Daraja: Rasm + Narx + Tavsif ]
   │      └── ⬅️ Orqaga ──────────> (Bosh Menyuga qaytadi)
   │
   ├── 📞 Bog'lanish
   └── ℹ️ Bot haqida
Muhim xususiyatlar:
Kutubxona: aiogram 3.x versiyasiga moslashtirilgan.

URLInputFile: Unsplash sahifalaridan olingan rasmlarni Telegram'ga yuklash uchun ishlatildi.

HTML Formatlash: Barcha matnlar <b>, <i>, va <u> teglaridan foydalanib bezatilgan.

Catch-all (Echo): Mos kelmaydigan har qanday xabarlar uchun eng pastki qismga joylashtirildi.

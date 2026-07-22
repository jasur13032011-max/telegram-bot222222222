# telegram-bot222222222
Mana barcha talablaringizga to'liq javob beradigan, aiogram 3.x bazasidagi to'liq loyiha kodi va uning arxitekturasi.

Loyihada 7 bosqichli FSM, inline tugmalar orqali tanlovlar, ma'lumotlarni qayta tahrirlash (boshidan yoki aniq bosqichni tanlab), pagination qilingan admin paneli hamda xatoliklarni ushlab turuvchi middleware o'rnatilgan.

🛠 Telegram Bot Kodi (bot.py)
Python
import asyncio
import logging
import re
from typing import Any, Awaitable, Callable, Dict, List

from aiogram import Bot, Dispatcher, F
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import (
    CallbackQuery,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    Message,
    TelegramObject,
)

# ------------------------------------------------------------------
# CONFIG & FAKE DATABASE
# ------------------------------------------------------------------
BOT_TOKEN = "YOUR_BOT_TOKEN"
ADMIN_IDS = [123456789]  # Admin ID-larini kiriting

APPLICATIONS: Dict[int, dict] = {}  # Application_ID -> Data

# ------------------------------------------------------------------
# 1. MIDDLEWARE (Logging & Error handling)
# ------------------------------------------------------------------
class LoggingMiddleware:
    """Har bir kelayotgan xabarni log qiladi va try/except bilan xatoni ushlaydi."""
    async def __call__(
        self,
        handler: Callable[[TelegramObject, Dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: Dict[str, Any]
    ) -> Any:
        try:
            if isinstance(event, Message):
                logging.info(f"[MSG] User: {event.from_user.id} | Text: {event.text}")
            elif isinstance(event, CallbackQuery):
                logging.info(f"[CB] User: {event.from_user.id} | Data: {event.data}")
            return await handler(event, data)
        except Exception as e:
            logging.error(f"[ERROR in Middleware]: {e}", exc_info=True)

# ------------------------------------------------------------------
# 2. FSM STATES
# ------------------------------------------------------------------
class Form(StatesGroup):
    name = State()       # 1/7
    surname = State()    # 2/7
    age = State()        # 3/7
    gender = State()     # 4/7
    city = State()       # 5/7
    phone = State()      # 6/7
    confirm = State()    # 7/7

# ------------------------------------------------------------------
# KEYBOARDS (INLINE)
# ------------------------------------------------------------------
def get_gender_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="👨 Erkak", callback_data="gender_Erkak"),
            InlineKeyboardButton(text="👩 Ayol", callback_data="gender_Ayol")
        ]
    ])

def get_city_kb():
    cities = ["Toshkent", "Samarqand", "Buxoro", "Andijon", "Farg'ona", "Namangan"]
    buttons = [
        [InlineKeyboardButton(text=city, callback_data=f"city_{city}")] for city in cities
    ]
    return InlineKeyboardMarkup(inline_keyboard=buttons)

def get_confirm_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Tasdiqlash", callback_data="confirm_yes")],
        [
            InlineKeyboardButton(text="🔄 Boshidan tahrirlash", callback_data="edit_start"),
            InlineKeyboardButton(text="⚙️ Bosqichni tanlash", callback_data="edit_select")
        ],
        [InlineKeyboardButton(text="❌ Bekor qilish", callback_data="cancel_form")]
    ])

def get_edit_steps_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="1. Ism", callback_data="step_name"), InlineKeyboardButton(text="2. Familiya", callback_data="step_surname")],
        [InlineKeyboardButton(text="3. Yosh", callback_data="step_age"), InlineKeyboardButton(text="4. Jins", callback_data="step_gender")],
        [InlineKeyboardButton(text="5. Shahar", callback_data="step_city"), InlineKeyboardButton(text="6. Telefon", callback_data="step_phone")]
    ])

# ------------------------------------------------------------------
# CANCEL HANDLER (/cancel har bosqichda ishlaydi)
# ------------------------------------------------------------------
dp = Dispatcher(storage=MemoryStorage())

@dp.message(Command("cancel"))
@dp.callback_query(F.data == "cancel_form")
async def cancel_handler(event: TelegramObject, state: FSMContext):
    current_state = await state.get_state()
    if current_state is None:
        text = "Sizda hech qanday faol jarayon yo'q."
    else:
        await state.clear()
        text = "🚫 Anketa to'ldirish bekor qilindi. Qayta boshlash uchun /start bosing."
    
    if isinstance(event, CallbackQuery):
        await event.message.edit_text(text)
    else:
        await event.answer(text)

# ------------------------------------------------------------------
# 3. FSM HANDLERS (7 Bosqich)
# ------------------------------------------------------------------
@dp.message(Command("start"))
async def cmd_start(message: Message, state: FSMContext):
    await state.set_state(Form.name)
    await message.answer("📊 **[1/7]** Anketa to'ldirishni boshlaymiz!\n\nIltimos, **ismingizni** kiriting:\n\n*(Jarayonni bekor qilish uchun /cancel bosing)*")

# Bosqich 1: Ism
@dp.message(Form.name)
async def process_name(message: Message, state: FSMContext):
    if not message.text.isalpha():
        await message.answer("⚠️ **Xatolik:** Ism faqat harflardan iborat bo'lishi kerak! Qayta kiriting:")
        return
    await state.update_data(name=message.text)
    
    # Agar faqat bitta bosqichni tahrirlayotgan bo'lsa
    data = await state.get_data()
    if data.get("editing_single"):
        await state.update_data(editing_single=False)
        await show_confirmation(message, state)
        return

    await state.set_state(Form.surname)
    await message.answer("📊 **[2/7]** Endi **familiyangizni** kiriting:")

# Bosqich 2: Familiya
@dp.message(Form.surname)
async def process_surname(message: Message, state: FSMContext):
    if not message.text.isalpha():
        await message.answer("⚠️ **Xatolik:** Familiya faqat harflardan iborat bo'lishi kerak! Qayta kiriting:")
        return
    await state.update_data(surname=message.text)

    data = await state.get_data()
    if data.get("editing_single"):
        await state.update_data(editing_single=False)
        await show_confirmation(message, state)
        return

    await state.set_state(Form.age)
    await message.answer("📊 **[3/7]** **Yoshiningizni** kiriting (masalan: 21):")

# Bosqich 3: Yosh (Validatsiya)
@dp.message(Form.age)
async def process_age(message: Message, state: FSMContext):
    if not message.text.isdigit() or not (10 <= int(message.text) <= 100):
        await message.answer("⚠️ **Xatolik:** Yosh faqat sonlarda (10 dan 100 gacha) bo'lishi kerak! Qayta kiriting:")
        return
    await state.update_data(age=int(message.text))

    data = await state.get_data()
    if data.get("editing_single"):
        await state.update_data(editing_single=False)
        await show_confirmation(message, state)
        return

    await state.set_state(Form.gender)
    await message.answer("📊 **[4/7]** **Jinsingizni** tanlang:", reply_markup=get_gender_kb())

# Bosqich 4: Jins (Inline)
@dp.callback_query(Form.gender, F.data.startswith("gender_"))
async def process_gender(callback: CallbackQuery, state: FSMContext):
    gender = callback.data.split("_")[1]
    await state.update_data(gender=gender)
    await callback.answer()

    data = await state.get_data()
    if data.get("editing_single"):
        await state.update_data(editing_single=False)
        await show_confirmation(callback.message, state, edit_mode=True)
        return

    await state.set_state(Form.city)
    await callback.message.edit_text("📊 **[5/7]** Yashaydigan **shahringizni** tanlang:", reply_markup=get_city_kb())

# Bosqich 5: Shahar (Inline)
@dp.callback_query(Form.city, F.data.startswith("city_"))
async def process_city(callback: CallbackQuery, state: FSMContext):
    city = callback.data.split("_")[1]
    await state.update_data(city=city)
    await callback.answer()

    data = await state.get_data()
    if data.get("editing_single"):
        await state.update_data(editing_single=False)
        await show_confirmation(callback.message, state, edit_mode=True)
        return

    await state.set_state(Form.phone)
    await callback.message.edit_text("📊 **[6/7]** **Telefon raqamingizni** kiriting (Masalan: +998901234567):")

# Bosqich 6: Telefon (Validatsiya Regex)
@dp.message(Form.phone)
async def process_phone(message: Message, state: FSMContext):
    pattern = r"^\+998\d{9}$"
    if not re.match(pattern, message.text):
        await message.answer("⚠️ **Xatolik:** Telefon raqami `+998XXXXXXXXX` formatida bo'lishi kerak! Qayta kiriting:")
        return
    await state.update_data(phone=message.text)

    await state.set_state(Form.confirm)
    await show_confirmation(message, state)

# Bosqich 7: Tasdiqlash sahifasi (7/7)
async def show_confirmation(message_or_cb: Any, state: FSMContext, edit_mode: bool = False):
    data = await state.get_data()
    text = (
        "📊 **[7/7] Ma'lumotlarni tasdiqlang:**\n\n"
        f"👤 **Ism:** {data.get('name')}\n"
        f"👤 **Familiya:** {data.get('surname')}\n"
        f"🎂 **Yosh:** {data.get('age')}\n"
        f"⚧ **Jins:** {data.get('gender')}\n"
        f"🏙 **Shahar:** {data.get('city')}\n"
        f"📞 **Telefon:** {data.get('phone')}\n\n"
        "Ma'lumotlar to'g'rimi?"
    )
    if edit_mode:
        await message_or_cb.edit_text(text, parse_mode="Markdown", reply_markup=get_confirm_kb())
    else:
        await message_or_cb.answer(text, parse_mode="Markdown", reply_markup=get_confirm_kb())

# ------------------------------------------------------------------
# TAHRIRLASH (Edit) HANDLERLARI
# ------------------------------------------------------------------
@dp.callback_query(Form.confirm, F.data == "edit_start")
async def edit_start(callback: CallbackQuery, state: FSMContext):
    """Boshidan tahrirlash"""
    await state.set_state(Form.name)
    await callback.message.edit_text("📊 **[1/7]** Qaytadan boshlaymiz.\n\nIltimos, **ismingizni** kiriting:")

@dp.callback_query(Form.confirm, F.data == "edit_select")
async def edit_select_step(callback: CallbackQuery):
    """Aynan qaysi bosqichni tahrirlashni tanlash"""
    await callback.message.edit_text("⚙️ Qaysi ma'lumotni o'zgartirmoqchisiz?", reply_markup=get_edit_steps_kb())

@dp.callback_query(Form.confirm, F.data.startswith("step_"))
async def process_step_choice(callback: CallbackQuery, state: FSMContext):
    step = callback.data.split("_")[1]
    await state.update_data(editing_single=True)
    
    if step == "name":
        await state.set_state(Form.name)
        await callback.message.edit_text("Yangi **ismingizni** kiriting:")
    elif step == "surname":
        await state.set_state(Form.surname)
        await callback.message.edit_text("Yangi **familiyangizni** kiriting:")
    elif step == "age":
        await state.set_state(Form.age)
        await callback.message.edit_text("Yangi **yoshingizni** kiriting:")
    elif step == "gender":
        await state.set_state(Form.gender)
        await callback.message.edit_text("Yangi **jinsingizni** tanlang:", reply_markup=get_gender_kb())
    elif step == "city":
        await state.set_state(Form.city)
        await callback.message.edit_text("Yangi **shahringizni** tanlang:", reply_markup=get_city_kb())
    elif step == "phone":
        await state.set_state(Form.phone)
        await callback.message.edit_text("Yangi **telefon raqamingizni** kiriting (+998...):")

# Save & Submit
@dp.callback_query(Form.confirm, F.data == "confirm_yes")
async def save_application(callback: CallbackQuery, state: FSMContext, bot: Bot):
    data = await state.get_data()
    app_id = len(APPLICATIONS) + 1
    data["app_id"] = app_id
    data["user_id"] = callback.from_user.id
    
    # Save to DICT
    APPLICATIONS[app_id] = data
    await state.clear()

    await callback.message.edit_text("✅ Anketangiz muvaffaqiyatli saqlandi! Rahmat.")

    # Admin'ga xabar yuborish
    admin_msg = (
        f"📥 **Yangi Ariza #{app_id}**\n\n"
        f"• User ID: `{data['user_id']}`\n"
        f"• Ism: {data['name']} {data['surname']}\n"
        f"• Yosh: {data['age']}\n"
        f"• Telefon: {data['phone']}"
    )
    for admin_id in ADMIN_IDS:
        try:
            await bot.send_message(admin_id, admin_msg, parse_mode="Markdown")
        except Exception as e:
            logging.error(f"Adminga xabar yuborishda xatolik ({admin_id}): {e}")

# ------------------------------------------------------------------
# 4. ADMIN PANEL (PAGINATION & MANAGEMENT)
# ------------------------------------------------------------------
def get_applications_page_kb(page: int = 0, items_per_page: int = 3):
    keys = list(APPLICATIONS.keys())
    total_pages = (len(keys) + items_per_page - 1) // items_per_page or 1

    start = page * items_per_page
    end = start + items_per_page
    page_keys = keys[start:end]

    builder = []
    # Arizalar tugmalari
    for app_id in page_keys:
        app = APPLICATIONS[app_id]
        builder.append([InlineKeyboardButton(
            text=f"📋 #{app_id} - {app['name']} ({app['phone']})", 
            callback_data=f"app_view_{app_id}_{page}"
        )])

    # Pagination tugmalari
    nav_buttons = []
    if page > 0:
        nav_buttons.append(InlineKeyboardButton(text="⬅️ Ortga", callback_data=f"app_page_{page - 1}"))
    nav_buttons.append(InlineKeyboardButton(text=f"📄 {page + 1}/{total_pages}", callback_data="ignore"))
    if page < total_pages - 1:
        nav_buttons.append(InlineKeyboardButton(text="Oldinga ➡️", callback_data=f"app_page_{page + 1}"))
    
    builder.append(nav_buttons)
    return InlineKeyboardMarkup(inline_keyboard=builder)

@dp.message(Command("applications"))
async def cmd_applications(message: Message):
    if message.from_user.id not in ADMIN_IDS:
        await message.answer("🚫 Bu buyruq faqat adminlar uchun!")
        return

    if not APPLICATIONS:
        await message.answer("📂 Arizalar ro'yxati bo'sh.")
        return

    await message.answer("📥 **Arizalar ro'yxati:**", parse_mode="Markdown", reply_markup=get_applications_page_kb(0))

@dp.callback_query(F.data.startswith("app_page_"))
async def process_app_page(callback: CallbackQuery):
    if callback.from_user.id not in ADMIN_IDS:
        return
    page = int(callback.data.split("_")[2])
    await callback.message.edit_text("📥 **Arizalar ro'yxati:**", parse_mode="Markdown", reply_markup=get_applications_page_kb(page))

@dp.callback_query(F.data.startswith("app_view_"))
async def process_app_view(callback: CallbackQuery):
    if callback.from_user.id not in ADMIN_IDS:
        return
    
    _, _, app_id, page = callback.data.split("_")
    app_id, page = int(app_id), int(page)
    
    app = APPLICATIONS.get(app_id)
    if not app:
        await callback.answer("❌ Ariza topilmadi!", show_alert=True)
        return

    text = (
        f"📄 **Ariza #{app['app_id']}**\n\n"
        f"👤 **Ism Familiya:** {app['name']} {app['surname']}\n"
        f"🎂 **Yosh:** {app['age']}\n"
        f"⚧ **Jins:** {app['gender']}\n"
        f"🏙 **Shahar:** {app['city']}\n"
        f"📞 **Telefon:** {app['phone']}\n"
        f"🆔 **User ID:** `{app['user_id']}`"
    )

    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🗑 O'chirish", callback_data=f"app_delete_{app_id}_{page}")],
        [InlineKeyboardButton(text="⬅️ Ro'yxatga qaytish", callback_data=f"app_page_{page}")]
    ])

    await callback.message.edit_text(text, parse_mode="Markdown", reply_markup=kb)

@dp.callback_query(F.data.startswith("app_delete_"))
async def process_app_delete(callback: CallbackQuery):
    if callback.from_user.id not in ADMIN_IDS:
        return

    _, _, app_id, page = callback.data.split("_")
    app_id, page = int(app_id), int(page)

    if app_id in APPLICATIONS:
        del APPLICATIONS[app_id]
        await callback.answer("🗑 Ariza o'chirildi!", show_alert=True)
    
    # Ro'yxatga qaytish
    if not APPLICATIONS:
        await callback.message.edit_text("📂 Arizalar ro'yxati bo'sh.")
    else:
        # Sahifa chegaradan chiqib ketmasligini tekshirish
        keys = list(APPLICATIONS.keys())
        total_pages = (len(keys) + 3 - 1) // 3
        if page >= total_pages:
            page = max(0, total_pages - 1)
        await callback.message.edit_text("📥 **Arizalar ro'yxati:**", parse_mode="Markdown", reply_markup=get_applications_page_kb(page))

# ------------------------------------------------------------------
# MAIN RUNNER
# ------------------------------------------------------------------
async def main():
    logging.basicConfig(level=logging.INFO)
    bot = Bot(token=BOT_TOKEN)

    # Middleware o'rnatish
    dp.message.outer_middleware(LoggingMiddleware())
    dp.callback_query.outer_middleware(LoggingMiddleware())

    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
💡 Yozilgan funksiyalarning tushuntirishi:
7 Bosqichli FSM & Progress Bar ([N/7]):

Har bir bosqich matnida [1/7], [2/7], ..., [7/7] belgilari orqali foydalanuvchiga qaysi bosqichdaligi aniq ko'rsatib boriladi.

Validatsiyalar & Xatolar:

Ism/Familiya: Faqat alifbo harflaridan (.isalpha()) iborat bo'lishi shart.

Yosh: Faqat son (.isdigit()) va 10 hamda 100 oralig'ida bo'lishi kerak.

Telefon: Telegram formati uchun mos keluvchi Regex ^\+998\d{9}$ orqali tekshiriladi (+998901234567).

Inline UI & Tahrirlash (Edit):

Jins va Shahar inline tugmalar orqali tanlanadi.

7/7 tasdiqlash bosqichida "🔄 Boshidan tahrirlash" butun FSM-ni 1-bosqichdan qayta ishga tushiradi.

"⚙️ Bosqichni tanlash" (Bonus) tugmasi bosilsa, foydalanuvchi faqat o'zi xohlagan bitta ma'lumotni (masalan, yoshini yoki shahrini) o'zgartiradi va avtomatiq ravishda yana tasdiqlash sahifasiga qaytadi.

APPLICATIONS Dict & Admin Notification:

Ma'lumot tasdiqlanganda ariza global dictionary-ga yoziladi va barcha ko'rsatilgan ADMIN_IDS ga bildirishnoma boradi.

Paginated Admin Panel (/applications):

Adminlar har bir sahifada 3 tadan arizani ko'rishadi va "⬅️ Ortga" / "Oldinga ➡️" tugmalari bilan varraqlashlari mumkin.

Har bir arizaga kirib uni batafsil ko'rish va o'chirib tashlash (Delete) imkoniyati mavjud.

/cancel Har qanday holatda:

Foydalanuvchi qaysi bosqichda bo'lishidan qat'i nazar, /cancel buyrug'ini yuborib FSM-ni tozalashi mumkin.

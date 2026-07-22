# telegram-bot222222222
Aiogram (Python) yordamida berilgan barcha talablarga javob beradigan to'liq FSM (Finite State Machine) kodi va so'ngida MemoryStorage vs RedisStorage taqqoslash hisoboti:

1. Telegram Bot Kodi (Aiogram 3.x)
Python
import asyncio
import logging
from aiogram import Bot, Dispatcher, F, Router
from aiogram.filters import Command, CommandStart
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import (
    CallbackQuery,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    KeyboardButton,
    Message,
    ReplyKeyboardMarkup,
    ReplyKeyboardRemove,
)

BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"

# 1. StatesGroup — 6 ta state
class RegistrationForm(StatesGroup):
    name = State()         # 1/6
    age = State()          # 2/6
    gender = State()       # 3/6
    city = State()         # 4/6
    phone = State()        # 5/6
    confirm = State()      # 6/6

router = Router()

# Klaviaturalar
def get_gender_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="Erkak", callback_data="gender_erkak"),
         InlineKeyboardButton(text="Ayol", callback_data="gender_ayol")]
    ])

def get_city_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="Toshkent", callback_data="city_Toshkent"),
         InlineKeyboardButton(text="Samarqand", callback_data="city_Samarqand")],
        [InlineKeyboardButton(text="Farg'ona", callback_data="city_Farg'ona"),
         InlineKeyboardButton(text="Boshqa", callback_data="city_Boshqa")]
    ])

def get_phone_keyboard():
    return ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text="📱 Kontaktni ulashish", request_contact=True)]],
        resize_keyboard=True,
        one_time_keyboard=True
    )

def get_confirm_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Tasdiqlash", callback_data="confirm_yes"),
         InlineKeyboardButton(text="❌ Bekor qilish", callback_data="confirm_no")]
    ])

# Database simulyatsiyasi
async def save_user(data: dict):
    print(f" Foydalanuvchi ma'lumotlar bazasiga saqlandi: {data}")

# /cancel komandasi (Har qanday bosqichda ishlaydi)
@router.message(Command("cancel"))
@router.callback_query(F.data == "confirm_no")
async def cancel_handler(event: Message | CallbackQuery, state: FSMContext):
    current_state = await state.get_state()
    if current_state is None:
        if isinstance(event, Message):
            await event.answer("Hozirda hech qanday jarayon yo'q.")
        return

    await state.clear()
    text = " Jarayon bekor qilindi. Qaytadan boshlash uchun /start bosing."
    
    if isinstance(event, Message):
        await event.answer(text, reply_markup=ReplyKeyboardRemove())
    else:
        await event.message.edit_text(text)

# Bosqich 0: /start
@router.message(CommandStart())
async def start_handler(message: Message, state: FSMContext):
    await state.set_state(RegistrationForm.name)
    await message.answer("Ro'yxatdan o'tish boshlandi!\n\n(1/6) **Ismingizni kiriting:** (2-50 ta belgi)")

# Bosqich 1: Ism validatsiyasi
@router.message(RegistrationForm.name)
async def process_name(message: Message, state: FSMContext):
    name = message.text.strip()
    if not (2 <= len(name) <= 50):
        await message.answer(" Ism uzunligi 2 dan 50 tagacha belgi bo'lishi kerak. Qaytadan kiriting:")
        return

    await state.update_data(name=name)
    await state.set_state(RegistrationForm.age)
    await message.answer("(2/6) **Yoshingizni kiriting:** (14 dan 100 gacha)")

# Bosqich 2: Yosh validatsiyasi
@router.message(RegistrationForm.age)
async def process_age(message: Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer(" Iltimos, yoshingizni faqat raqamda kiriting:")
        return

    age = int(message.text)
    if not (14 <= age <= 100):
        await message.answer(" Yosh 14 va 100 oraliqida bo'lishi kerak. Qaytadan kiriting:")
        return

    await state.update_data(age=age)
    await state.set_state(RegistrationForm.gender)
    await message.answer("(3/6) **Jinsingizni tanlang:**", reply_markup=get_gender_keyboard())

# Bosqich 3: Jins (Inline keyboard)
@router.callback_query(RegistrationForm.gender, F.data.startswith("gender_"))
async def process_gender(callback: CallbackQuery, state: FSMContext):
    gender = callback.data.split("_")[1].capitalize()
    await state.update_data(gender=gender)
    
    await callback.message.delete()
    await state.set_state(RegistrationForm.city)
    await callback.message.answer("(4/6) **Shahringizni tanlang:**", reply_markup=get_city_keyboard())

# Bosqich 4: Shahar (Inline keyboard)
@router.callback_query(RegistrationForm.city, F.data.startswith("city_"))
async def process_city(callback: CallbackQuery, state: FSMContext):
    city = callback.data.split("_")[1]
    await state.update_data(city=city)

    await callback.message.delete()
    await state.set_state(RegistrationForm.phone)
    await callback.message.answer(
        "(5/6) **Telefon raqamingizni yuboring:**",
        reply_markup=get_phone_keyboard()
    )

# Bosqich 5: Telefon raqam (Reply Keyboard Contact)
@router.message(RegistrationForm.phone, F.contact)
async def process_phone(message: Message, state: FSMContext):
    phone_number = message.contact.phone_number
    await state.update_data(phone=phone_number)
    
    data = await state.get_data()
    await state.set_state(RegistrationForm.confirm)
    
    summary_text = (
        "(6/6) **Ma'lumotlaringizni tasdiqlang:**\n\n"
        f"👤 **Ism:** {data['name']}\n"
        f"🎂 **Yosh:** {data['age']}\n"
        f"🚻 **Jins:** {data['gender']}\n"
        f"🏙 **Shahar:** {data['city']}\n"
        f"📞 **Tel:** {data['phone']}\n\n"
        "Ma'lumotlar to'g'rimi?"
    )
    
    await message.answer("Rahmat!", reply_markup=ReplyKeyboardRemove())
    await message.answer(summary_text, reply_markup=get_confirm_keyboard())

# Noto'g'ri kontakt yuborilganda
@router.message(RegistrationForm.phone)
async def process_phone_invalid(message: Message):
    await message.answer(" Iltimos, **📱 Kontaktni ulashish** tugmasini bosing.")

# Bosqich 6: Tasdiqlash va saqlash
@router.callback_query(RegistrationForm.confirm, F.data == "confirm_yes")
async def process_confirm(callback: CallbackQuery, state: FSMContext):
    user_data = await state.get_data()
    
    # Ma'lumotlarni saqlash
    await save_user(user_data)
    
    # State ni tozalash
    await state.clear()
    
    await callback.message.edit_text(" Muvaffaqiyatli ro'yxatdan o'tdingiz! Ma'lumotlaringiz saqlandi.")

# Botni ishga tushirish
async def main():
    # 1. MemoryStorage o'rnatish
    storage = MemoryStorage()
    bot = Bot(token=BOT_TOKEN)
    dp = Dispatcher(storage=storage)
    
    dp.include_router(router)
    await dp.start_polling(bot)

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    asyncio.run(main())
2. MemoryStorage vs RedisStorage Taqqoslash Hisoboti
Telegram botlarda FSM vaqtinchalik ma'lumotlarni (masalan: foydalanuvchi qaysi bosqichda, kiritgan javoblari) saqlash uchun Storage drayverlaridan foydalanadi.

Xususiyat	MemoryStorage	RedisStorage
Ma'lumot qayerda saqlanadi?	Serverning RAM (Operativ xotira) qismida	Alohida Redis xotira serverida (NoSQL)
Bot qayta yuklanganda (Restart)	Barcha ma'lumotlar yo'qoladi ❌	Ma'lumotlar saqlanib qoladi ✅
Ko'p jarayonlilik (Multi-process/Cluster)	Qo'llamaydi (faqat 1 ta bot jarayonida)	To'liq qo'llaydi (bir nechta bot nusxasi ishlashi mumkin)
Tizim talabi	Qo'shimcha dastur kerak emas	Redis serverini o'rnatish va sozlash talab qilinadi
Tavsiya etiladigan holat	Testlash, kichik va shaxsiy botlar uchun	Production, yuqori yuklamali va muhim botlar uchun
Qisqacha xulosa:
MemoryStorage loyihani ishlab chiqish (development) va kichik loyihalar uchun tezkor yechimdir. Bot tayyor bo'lgach yoki restart berilganda foydalanuvchilarning bosqichlari (states) va vaqtinchalik kiritgan ma'lumotlari o'chib ketadi.

RedisStorage esa haqiqiy botlar (production) uchun shart. Agarda serverda profilaktika bo'lib bot qayta tushsa ham, foydalanuvchilar qolgan joyidan muloqotni bemalol davom ettira oladilar.

# telegram-bot222222222
Mana so'ralgan barcha xususiyatlarni (CallbackData, Pagination, Dynamic inline keyboards, URL tugmalar va boshqalar) o'zida jamlagan Aiogram 3.x uchun to'liq va professional tayyorlangan kod:

Python
import asyncio
import logging
from typing import Optional

from aiogram import Bot, Dispatcher, F, html, types
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.filters import Command
from aiogram.filters.callback_data import CallbackData
from aiogram.types import InlineKeyboardButton, InlineKeyboardMarkup, Message

# ==========================
# KONFIGURATSIYA
# ==========================
BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"  # Bot tokeningizni kiriting

logging.basicConfig(level=logging.INFO)
bot = Bot(token=BOT_TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
dp = Dispatcher()


# ==========================
# 1. CALLBACK DATA KLASSLARI (Kamida 2 ta)
# ==========================
class OrderCB(CallbackData, prefix="order"):
    action: str  # view, edit, delete, confirm_delete
    order_id: int


class PaginationCB(CallbackData, prefix="page"):
    page: int


class RateCB(CallbackData, prefix="rate"):
    vote: str  # like, dislike


class StarCB(CallbackData, prefix="star"):
    rating: int


# ==========================
# FAKE MA'LUMOTLAR
# ==========================
FAKE_ORDERS = {
    101: {"item": "🍕 Margarita Pizza", "status": "Kutilmoqda"},
    102: {"item": "🍔 Cheeseburger", "status": "Yetkazilmoqda"},
    103: {"item": "💻 Python Kursi", "status": "To'langan"},
}

# 50 ta item yaratish
ITEMS = [f"📦 Mahsulot #{i}" for i in range(1, 51)]


# ==========================
# /rate HANDLER
# ==========================
@dp.message(Command("rate"))
async def cmd_rate(message: Message):
    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="👍 Like", callback_data=RateCB(vote="like").pack()
                ),
                InlineKeyboardButton(
                    text="👎 Dislike", callback_data=RateCB(vote="dislike").pack()
                ),
            ]
        ]
    )
    await message.answer("Ushbu xizmatimizga baho bering:", reply_markup=keyboard)


@dp.callback_query(RateCB.filter())
async def process_rate(call: types.CallbackQuery, callback_data: RateCB):
    # show_alert=True bilan bildirishnoma ko'rsatish
    res_text = "Rahmat! Like bosdingiz! 👍" if callback_data.vote == "like" else "Rahmat! Dislike bosdingiz! 👎"
    await call.answer(text=res_text, show_alert=True)


# ==========================
# /stars HANDLER (edit_text bilan)
# ==========================
@dp.message(Command("stars"))
async def cmd_stars(message: Message):
    buttons = [
        InlineKeyboardButton(
            text=f"⭐ {i}", callback_data=StarCB(rating=i).pack()
        )
        for i in range(1, 6)
    ]
    keyboard = InlineKeyboardMarkup(inline_keyboard=[buttons])
    await message.answer("Bahoingizni tanlang (1-5):", reply_markup=keyboard)


@dp.callback_query(StarCB.filter())
async def process_stars(call: types.CallbackQuery, callback_data: StarCB):
    await call.answer()  # Doim loading'ni to'xtatish
    await call.message.edit_text(
        f"Siz {callback_data.rating} ⭐ baho qo'ydingiz! Rahmat!"
    )


# ==========================
# /orders HANDLER (CRUD + Tasdiqlash tugmasi)
# ==========================
def get_orders_keyboard():
    keyboard = []
    for order_id, data in FAKE_ORDERS.items():
        row = [
            InlineKeyboardButton(
                text=f"{data['item']}",
                callback_data=OrderCB(action="view", order_id=order_id).pack(),
            ),
            InlineKeyboardButton(
                text="✏️",
                callback_data=OrderCB(action="edit", order_id=order_id).pack(),
            ),
            InlineKeyboardButton(
                text="🗑",
                callback_data=OrderCB(action="delete", order_id=order_id).pack(),
            ),
        ]
        keyboard.append(row)
    return InlineKeyboardMarkup(inline_keyboard=keyboard)


@dp.message(Command("orders"))
async def cmd_orders(message: Message):
    await message.answer("Buyurtmalaringiz ro'yxati:", reply_markup=get_orders_keyboard())


@dp.callback_query(OrderCB.filter(F.action == "view"))
async def view_order(call: types.CallbackQuery, callback_data: OrderCB):
    await call.answer()
    order = FAKE_ORDERS.get(callback_data.order_id)
    if order:
        await call.message.answer(
            f"ℹ️ <b>Buyurtma #{callback_data.order_id}</b>\n"
            f"Nomi: {order['item']}\nStatus: {order['status']}"
        )


@dp.callback_query(OrderCB.filter(F.action == "edit"))
async def edit_order(call: types.CallbackQuery, callback_data: OrderCB):
    await call.answer("Tahrirlash rejimi tanlandi")
    await call.message.answer(f"✏️ Buyurtma #{callback_data.order_id} tahrirlash uchun tanlandi.")


@dp.callback_query(OrderCB.filter(F.action == "delete"))
async def confirm_delete_order(call: types.CallbackQuery, callback_data: OrderCB):
    await call.answer()
    # Tasdiqlovchi tugma yaratish
    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="❌ Ha, o'chiring!",
                    callback_data=OrderCB(
                        action="confirm_delete", order_id=callback_data.order_id
                    ).pack(),
                ),
                InlineKeyboardButton(
                    text="Bekor qilish", callback_data="cancel_action"
                ),
            ]
        ]
    )
    await call.message.edit_text(
        f"⚠️ Haqiqatan ham #{callback_data.order_id} buyurtmani o'chirmoqchimisiz?",
        reply_markup=keyboard,
    )


@dp.callback_query(OrderCB.filter(F.action == "confirm_delete"))
async def process_delete(call: types.CallbackQuery, callback_data: OrderCB):
    await call.answer("O'chirildi!", show_alert=True)
    FAKE_ORDERS.pop(callback_data.order_id, None)
    await call.message.edit_text("✅ Buyurtma muvaffaqiyatli o'chirildi.")


@dp.callback_query(F.data == "cancel_action")
async def cancel_action(call: types.CallbackQuery):
    await call.answer("Bekor qilindi")
    await call.message.edit_text("Buyurtmalar ro'yxati:", reply_markup=get_orders_keyboard())


# ==========================
# /list HANDLER (PAGINATION - 50 ta item, 5 ta/sahifa)
# ==========================
ITEMS_PER_PAGE = 5


def get_pagination_keyboard(page: int = 1):
    total_pages = (len(ITEMS) + ITEMS_PER_PAGE - 1) // ITEMS_PER_PAGE
    buttons = []

    if page > 1:
        buttons.append(
            InlineKeyboardButton(
                text="⬅️ Orqaga", callback_data=PaginationCB(page=page - 1).pack()
            )
        )

    buttons.append(
        InlineKeyboardButton(
            text=f"{page}/{total_pages}", callback_data="ignore"
        )
    )

    if page < total_pages:
        buttons.append(
            InlineKeyboardButton(
                text="Oldinga ➡️", callback_data=PaginationCB(page=page + 1).pack()
            )
        )

    return InlineKeyboardMarkup(inline_keyboard=[buttons])


@dp.message(Command("list"))
async def cmd_list(message: Message):
    page = 1
    start = (page - 1) * ITEMS_PER_PAGE
    end = start + ITEMS_PER_PAGE
    current_items = ITEMS[start:end]

    text = f"<b>Mahsulotlar ro'yxati ({page}-sahifa):</b>\n\n" + "\n".join(current_items)
    await message.answer(text, reply_markup=get_pagination_keyboard(page))


@dp.callback_query(PaginationCB.filter())
async def process_pagination(call: types.CallbackQuery, callback_data: PaginationCB):
    await call.answer()
    page = callback_data.page
    start = (page - 1) * ITEMS_PER_PAGE
    end = start + ITEMS_PER_PAGE
    current_items = ITEMS[start:end]

    text = f"<b>Mahsulotlar ro'yxati ({page}-sahifa):</b>\n\n" + "\n".join(current_items)
    await call.message.edit_text(text, reply_markup=get_pagination_keyboard(page))


@dp.callback_query(F.data == "ignore")
async def ignore_click(call: types.CallbackQuery):
    await call.answer()  # Sahifa raqami bosilganda shunchaki yuklanishni to'xtatish


# ==========================
# /link HANDLER (URL Tugmalar)
# ==========================
@dp.message(Command("link"))
async def cmd_link(message: Message):
    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [InlineKeyboardButton(text="✈️ Telegram", url="https://t.me")],
            [InlineKeyboardButton(text="📸 Instagram", url="https://instagram.com")],
            [InlineKeyboardButton(text="🌐 GitHub", url="https://github.com")],
        ]
    )
    await message.answer("Ijtimoiy tarmoqlarimiz:", reply_markup=keyboard)


# ==========================
# ATAYLAB ANSWER'SIZ HANDLER (Tushuntirish)
# ==========================
@dp.message(Command("no_answer_demo"))
async def cmd_no_answer_demo(message: Message):
    keyboard = InlineKeyboardMarkup(
        inline_keyboard=[
            [
                InlineKeyboardButton(
                    text="⚠️ Meni bosing (call.answer'siz)",
                    callback_data="bad_button",
                )
            ]
        ]
    )
    await message.answer(
        "Quyidagi tugma ataylab answer'siz qoldirilgan:", reply_markup=keyboard
    )


@dp.callback_query(F.data == "bad_button")
async def bad_button_handler(call: types.CallbackQuery):
    # ATAYLAB await call.answer() YOZILMAGAN!
    # SABABI VA OQIBATI:
    # 1. Telegram foydalanuvchi interfeysida tugma ustidagi "soat/aylanayotgan yuklanish" (loading spinner)
    #    taxminan 10-60 soniya davomida yo'qolmay turib qoladi.
    # 2. Foydalanuvchiga bot qotib qolgandek yoki ishlamayotgandek tuyuladi.
    # 3. Telegram serveri botga shu callback'ni qayta-qayta yuborishi va keraksiz server resursi sarflanishi mumkin.
    await call.message.answer(
        "Siz tugmani bosdingiz, lekin `call.answer()` chaqirilmagani uchun tugmada yuklanish (loading) aylanib turibdi!"
    )


# ==========================
# BOTNI ISHGA TUSHIRISH
# ==========================
async def main():
    print("✅ Bot callback funksiyalar bilan ishga tushdi...")
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
Kod tarkibidagi muhim xususiyatlar bo'yicha tushuntirish:
CallbackData klasslari: OrderCB, PaginationCB, RateCB va StarCB orqali ma'lumotlarni xavfsiz va tizimli uzatish yo'lga qo'yilgan.

show_alert=True: /rate tugmalari bosilganda pop-up oyna ko'rinishida rasmiy ogohlantirish beradi.

/stars va edit_text: Tugma bosilganda yangi xabar yubormasdan, mavjud xabarning o'zini o'zgartiradi.

/orders va Tasdiqlash: Har bir buyurtma yonida ko'rish, tahrirlash va o'chirish tugmasi bor. O'chirish bosilganda "Ha, o'chiring!" va "Bekor qilish" tasdiq oynasiga o'tadi.

/list Pagination: 50 ta element 5 tadan bo'lib sahifalangan. Oldinga/Orqaga tugmalari va 1/10 ko'rinishidagi sahifa ko'rsatkichi bor.

/no_answer_demo: call.answer() yozilmasa tugmada qanday aylanuvchi yuklanish belgisi paydo bo'lishini va foydalanuvchiga bot qotgandek ko'rinishini amalda tushuntiradi.

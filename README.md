# telegram-bot222222222
Mana SQLAlchemy 2.0 (asyncio), aiosqlite va aiogram 3.x bazasida tayyorlangan, Service/Repository pattern arxitekturasiga ega to'liq Telegram bot loyihasi.

📁 Loyiha strukturasi
Plaintext
bot_project/
├── database.py       # Engine va sessionmaker
├── models.py         # User va Order modellari (Mapped pattern)
├── repositories.py   # Base, User va Order repozitariylari
├── services.py       # UserService, OrderService (Business logic & Transactions)
├── middlewares.py    # DBSessionMiddleware va UserContextMiddleware
├── handlers.py       # Bot komandalari va handler'lar
└── main.py           # Botni ishga tushirish va init_db()
1. DB sozlmalari va Modellari (database.py, models.py)
database.py
Python
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

# SQLite (Async) - PostgreSQL uchun: postgresql+asyncpg://user:pass@localhost/dbname
DATABASE_URL = "sqlite+aiosqlite:///./bot_database.db"

engine = create_async_engine(DATABASE_URL, echo=False)
async_session_maker = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_maker() as session:
        yield session
models.py
Python
from datetime import datetime
from typing import List, Optional
from sqlalchemy import BigInteger, ForeignKey, String, DateTime, Float, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)  # Telegram ID
    full_name: Mapped[str] = mapped_column(String(100))
    username: Mapped[Optional[str]] = mapped_column(String(50), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())

    # Relationship (Bir nechta buyurtmaga ega bo'lishi mumkin)
    orders: Mapped[List["Order"]] = relationship("Order", back_populates="user", cascade="all, delete-orphan")


class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    user_id: Mapped[int] = mapped_column(BigInteger, ForeignKey("users.id", ondelete="CASCADE"))
    item_name: Mapped[str] = mapped_column(String(100))
    price: Mapped[float] = mapped_column(Float)
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())

    # Relationship
    user: Mapped["User"] = relationship("User", back_populates="orders")
2. Repozitariylar va Servislar (repositories.py, services.py)
repositories.py
Python
from typing import List, Optional
from sqlalchemy import select, func
from sqlalchemy.orm import selectinload
from sqlalchemy.ext.asyncio import AsyncSession
from models import User, Order


class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get_by_id(self, user_id: int) -> Optional[User]:
        stmt = select(User).where(User.id == user_id)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    async def create(self, user_id: int, full_name: str, username: Optional[str]) -> User:
        user = User(id=user_id, full_name=full_name, username=username)
        self.session.add(user)
        return user

    async def get_user_with_orders(self, user_id: int) -> Optional[User]:
        # SELECTINLOAD yordamida N+1 muammosisiz yuklash
        stmt = select(User).options(selectinload(User.orders)).where(User.id == user_id)
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()


class OrderRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create_order(self, user_id: int, item_name: str, price: float) -> Order:
        order = Order(user_id=user_id, item_name=item_name, price=price)
        self.session.add(order)
        return order

    async def get_user_orders(self, user_id: int) -> List[Order]:
        stmt = select(Order).where(Order.user_id == user_id).order_by(Order.created_at.desc())
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    async def get_stats(self) -> dict:
        stmt_users = select(func.count(User.id))
        stmt_orders = select(func.count(Order.id), func.coalesce(func.sum(Order.price), 0.0))

        total_users = (await self.session.execute(stmt_users)).scalar() or 0
        orders_res = (await self.session.execute(stmt_orders)).one()
        
        return {
            "total_users": total_users,
            "total_orders": orders_res[0],
            "total_revenue": orders_res[1]
        }
services.py
Python
from sqlalchemy.ext.asyncio import AsyncSession
from repositories import UserRepository, OrderRepository
from models import User, Order


class UserService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self.user_repo = UserRepository(session)

    async def get_or_create_user(self, user_id: int, full_name: str, username: str = None) -> User:
        user = await self.user_repo.get_by_id(user_id)
        if not user:
            # Transaction context orqali avtomatik commit
            async with self.session.begin():
                user = await self.user_repo.create(user_id, full_name, username)
        return user


class OrderService:
    def __init__(self, session: AsyncSession):
        self.session = session
        self.order_repo = OrderRepository(session)

    async def create_purchase_transaction(self, user_id: int, item_name: str, price: float) -> Order:
        """
        Tranzaksiya misoli: Xaridni amalga oshirish va saqlash.
        Agarda jarayonda xatolik yuz bersa, session.begin() avtomatik ROLLBACK qiladi.
        """
        async with self.session.begin():
            order = await self.order_repo.create_order(user_id, item_name, price)
            # Bu yerda to'lov tizimi tekshiruvi yoki boshqa mantiq bo'lishi mumkin
            return order
3. Middlewares (middlewares.py)
Python
from typing import Any, Awaitable, Callable, Dict
from aiogram import BaseMiddleware
from aiogram.types import TelegramObject, User as TelegramUser
from sqlalchemy.ext.asyncio import AsyncSession
from database import async_session_maker
from services import UserService


class DBSessionMiddleware(BaseMiddleware):
    """Har bir so'rov uchun DB Session ochib beradi va yopadi."""
    async def __call__(
        self,
        handler: Callable[[TelegramObject, Dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: Dict[str, Any]
    ) -> Any:
        async with async_session_maker() as session:
            data["session"] = session
            return await handler(event, data)


class UserContextMiddleware(BaseMiddleware):
    """Userni DBdan qidiradi/yaratadi va data['db_user'] sifatida uzatadi."""
    async def __call__(
        self,
        handler: Callable[[TelegramObject, Dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: Dict[str, Any]
    ) -> Any:
        session: AsyncSession = data.get("session")
        event_user: TelegramUser = data.get("event_from_user")

        if session and event_user:
            user_service = UserService(session)
            db_user = await user_service.get_or_create_user(
                user_id=event_user.id,
                full_name=event_user.full_name,
                username=event_user.username
            )
            data["db_user"] = db_user

        return await handler(event, data)
4. Bot Handler'lari (handlers.py)
Python
from aiogram import Router, F
from aiogram.filters import Command
from aiogram.types import Message
from sqlalchemy.ext.asyncio import AsyncSession
from models import User
from repositories import UserRepository, OrderRepository
from services import OrderService

router = Router()


@router.message(Command("profile"))
async def cmd_profile(message: Message, session: AsyncSession, db_user: User):
    user_repo = UserRepository(session)
    
    # N+1 muammosiz foydalanuvchi va uning buyurtmalarini yuklash
    user_with_orders = await user_repo.get_user_with_orders(db_user.id)
    orders_count = len(user_with_orders.orders) if user_with_orders else 0

    await message.answer(
        f"👤 **Profil:**\n\n"
        f"• ID: `{db_user.id}`\n"
        f"• Ism: {db_user.full_name}\n"
        f"• Username: @{db_user.username or 'Yo'q'}\n"
        f"• Jami buyurtmalar: {orders_count} ta",
        parse_mode="Markdown"
    )


@router.message(Command("buy"))
async def cmd_buy(message: Message, session: AsyncSession, db_user: User):
    order_service = OrderService(session)
    
    # Misol uchun yangi mahsulot xarid qilish
    item_name = "Premium Obuna"
    price = 49000.0

    order = await order_service.create_purchase_transaction(
        user_id=db_user.id,
        item_name=item_name,
        price=price
    )

    await message.answer(
        f"✅ **Xarid muvaffaqiyatli amalga oshirildi!**\n\n"
        f"📦 Mahsulot: {order.item_name}\n"
        f"💰 Narxi: {order.price:,.0f} so'm\n"
        f"🆔 Buyurtma ID: #{order.id}"
    )


@router.message(Command("orders"))
async def cmd_orders(message: Message, session: AsyncSession, db_user: User):
    order_repo = OrderRepository(session)
    orders = await order_repo.get_user_orders(db_user.id)

    if not orders:
        await message.answer("🛒 Sizda hali buyurtmalar yo'q.")
        return

    text = "📋 **Sizning buyurtmalaringiz:**\n\n"
    for o in orders:
        text += f"• **#{o.id}** {o.item_name} - {o.price:,.0f} so'm ({o.created_at.strftime('%Y-%m-%d %H:%M')})\n"

    await message.answer(text, parse_mode="Markdown")


@router.message(Command("stats"))
async def cmd_stats(message: Message, session: AsyncSession):
    order_repo = OrderRepository(session)
    stats = await order_repo.get_stats()

    await message.answer(
        f"📊 **Bot Statistikasi:**\n\n"
        f"• Foydalanuvchilar: {stats['total_users']} ta\n"
        f"• Jami buyurtmalar: {stats['total_orders']} ta\n"
        f"• Jami tushum: {stats['total_revenue']:,.0f} so'm",
        parse_mode="Markdown"
    )
5. Main Runner va DB Initsializatsiyasi (main.py)
Python
import asyncio
import logging
from aiogram import Bot, Dispatcher
from database import engine
from models import Base
from middlewares import DBSessionMiddleware, UserContextMiddleware
from handlers import router

BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"


async def init_db():
    """Birinchi marta ishga tushganda jadvallarni yaratadi."""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)


async def main():
    logging.basicConfig(level=logging.INFO)

    # 1. Baza jadvallarini yaratish
    await init_db()

    bot = Bot(token=BOT_TOKEN)
    dp = Dispatcher()

    # 2. Middlewares o'rnatish
    dp.update.outer_middleware(DBSessionMiddleware())
    dp.update.outer_middleware(UserContextMiddleware())

    # 3. Router restratsiyasi
    dp.include_router(router)

    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
💡 N+1 Muammosi va selectinload Yechimi
❌ N+1 Muammosi nima?
Faraz qilaylik, 100 ta foydalanuvchi va ularning har birining buyurtmalarini ko'rmoqchimiz:

Python
# 1 ta so'rov: Barcha foydalanuvchilarni olish
users = await session.scalars(select(User))

for user in users:
    # HAR BIR foydalanuvchi uchun alohida DB so'rov yuboriladi! (100 ta so'rov)
    print(user.orders) 
Oqibati: 1 ta boshlang'ich so'rov + 100 ta qo'shimcha so'rov = 101 ta DB so me'yoriy yuklanish (Lazy Loading xatosi).

✅ selectinload bilan Yechim
SQLAlchemy Async rejimda default holatda Lazy Loading ishlamaydi (MissingGreenlet xatosi beradi). Bog'langan obyektlarni bitta samarali SQL zanjiri bilan yuklash uchun selectinload ishlatiladi:

Python
from sqlalchemy.orm import selectinload

# 2 ta umumiy so'rov bilan BARCHA ma'lumotlar yuklanadi (N ta so'rov o'rniga):
stmt = select(User).options(selectinload(User.orders))
users = await session.scalars(stmt)

for user in users:
    # Baza bilan qayta bog'lanish bo'lmaydi, ma'lumotlar keshdan olinadi
    print(user.orders)
Birinchi SQL so'rov: SELECT * FROM users;

Ikkinchi SQL so'rov: SELECT * FROM orders WHERE user_id IN (1, 2, 3, ...);
Natijada ma'lumotlar bir zumda xotiraga yuklanadi va bazaga keraksiz bosim tushmaydi.

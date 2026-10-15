[Uploading main.py…]()
import os
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request, HTTPException
from pydantic import BaseModel
from aiogram import Bot, Dispatcher
from aiogram.types import Message, InlineKeyboardMarkup, InlineKeyboardButton, WebAppInfo
from aiogram.filters import Command
from supabase import create_client, Client

# НАСТРОЙКИ (Заполнятся из переменных окружения на Render)
BOT_TOKEN = os.getenv("BOT_TOKEN")
SUPABASE_URL = os.getenv("SUPABASE_URL")
SUPABASE_KEY = os.getenv("SUPABASE_KEY")
WEBAPP_URL = os.getenv("WEBAPP_URL", "https://taupe-cupcake-588870.netlify.app/")

supabase = create_client(SUPABASE_URL, SUPABASE_KEY)

@asynccontextmanager
async def lifespan(app: FastAPI):
    bot = Bot(token=BOT_TOKEN)
    app.state.bot = bot
    app.state.dp = Dispatcher()
    
    # Команда /start для бота
    @app.state.dp.message(Command("start"))
    async def cmd_start(message: Message):
        # Регистрируем пользователя в базе, если его нет
        supabase.table("users").upsert(
            {"id": message.from_user.id, "username": message.from_user.username, "balance": 0}, 
            on_conflict="id"
        ).execute()
        
        keyboard = InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🚀 Открыть приложение", web_app=WebAppInfo(url=WEBAPP_URL))]
        ])
        await message.answer(
            "👋 Привет! Добро пожаловать в биржу взаимной рекламы.\n\n"
            "📺 Смотри рекламы по 1 минуте, получай баллы и публикуй свои ссылки!\n"
            "🔥 10 баллов = 1 бесплатная публикация.",
            reply_markup=keyboard
        )

    # Установка Webhook при запуске
    webhook_url = os.getenv("APP_URL") + "/webhook"
    await bot.set_webhook(webhook_url)
    print(f"✅ Бот запущен! Webhook установлен на: {webhook_url}")
    
    yield
    
    await bot.delete_webhook()
    await bot.session.close()

app = FastAPI(lifespan=lifespan)

# Обработчик обновлений от Telegram
@app.post("/webhook")
async def webhook(request: Request):
    update = await request.json()
    await app.state.dp.feed_webhook_update(app.state.bot, update)
    return {"ok": True}

# --- API ДЛЯ MINI APP (Сайт на Netlify) ---

class WatchRequest(BaseModel):
    user_id: int

@app.get("/api/get_user")
async def get_user(user_id: int):
    user = supabase.table("users").select("balance").eq("id", user_id).single().execute().data
    return {"balance": user["balance"] or 0}

@app.get("/api/get_ad")
async def get_ad(user_id: int):
    # priority DESC гарантирует, что твоя реклама (999) будет всегда первой!
    response = supabase.table("ads").select("*").eq("status", "active").order("priority", desc=True).limit(1).execute()
    if not response.data:
        raise HTTPException(status_code=404, detail="Нет доступной рекламы")
    ad = response.data[0]
    return {"ad": ad, "is_owner_ad": ad["user_id"] == 0}

@app.post("/api/complete_watch")
async def complete_watch(req: WatchRequest):
    user = supabase.table("users").select("balance").eq("id", req.user_id).single().execute().data
    new_balance = (user["balance"] or 0) + 1
    supabase.table("users").update({"balance": new_balance}).eq("id", req.user_id).execute()
    return {"success": True, "new_balance": new_balance}
web: uvicorn main:app --host 0.0.0.0 --port $PORT
fastapi==0.109.0
uvicorn==0.27.0
aiogram==3.4.1
supabase==2.3.2
pydantic==2.6.1

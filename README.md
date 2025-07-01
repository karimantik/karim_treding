
import asyncio
import logging
import requests
import pandas as pd
import numpy as np
import schedule
import nest_asyncio
import time

from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import (ApplicationBuilder, CommandHandler, MessageHandler,
                          ContextTypes, filters)
import ta

nest_asyncio.apply()
logging.basicConfig(level=logging.INFO)

TOKEN = "ضع_توكن_البوت_هنا"
CHAT_ID = None

ASSETS = {
    "Quickler": "BTCUSDT",
    "Asia Composite Index": "ETHUSDT",
    "Crypto Composite Index": "BNBUSDT",
    "Halal Market Axis": "XRPUSDT",
    "Europe Composite Index": "EURUSDT",
    "Gold OTC": "XAUUSDT",
    "Bitcoin": "BTCUSDT",
    "Fx Gold": "XAUUSD",
    "Fx Bitcoin": "BTCUSD",
}

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global CHAT_ID
    CHAT_ID = update.effective_chat.id
    keyboard = [["📊 التوصيات", "📈 تحليل أصل"],
                ["ℹ️ المساعدة", "🪙 ما وضع السوق؟"]]
    reply_markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
    await update.message.reply_text(
        "👋 أهلاً بك في بوت التداول الذكي!\nاختر من القائمة أو اكتب أمرًا مباشرة.",
        reply_markup=reply_markup)

def fetch_data(symbol, interval="1m", limit=50):
    try:
        url = f"https://api.binance.com/api/v3/klines?symbol={symbol}&interval={interval}&limit={limit}"
        data = requests.get(url, timeout=5).json()
        df = pd.DataFrame(data, columns=["time", "open", "high", "low", "close",
                                         "volume", "close_time", "qav", "trades",
                                         "tb_base", "tb_quote", "ignore"])
        df["close"] = df["close"].astype(float)
        return df
    except:
        return None

def analyze(df):
    df["rsi"] = ta.momentum.RSIIndicator(df["close"], window=14).rsi()
    macd = ta.trend.MACD(df["close"])
    df["macd"] = macd.macd()
    df["macd_signal"] = macd.macd_signal()
    df["ema12"] = ta.trend.ema_indicator(df["close"], window=12).ema_indicator()
    df["ema26"] = ta.trend.ema_indicator(df["close"], window=26).ema_indicator()
    last = df.iloc[-1]
    signal = "لا توصية"
    if last["rsi"] < 30 and last["macd"] > last["macd_signal"] and last["ema12"] > last["ema26"]:
        signal = "شراء (CALL)"
    elif last["rsi"] > 70 and last["macd"] < last["macd_signal"] and last["ema12"] < last["ema26"]:
        signal = "بيع (PUT)"
    return signal, round(last["rsi"], 2), last["close"]

def format_analysis(name, sig, rsi, price):
    return f"📍 {name}:\n🔔 توصية: {sig}\n💰 السعر الحالي: {price}$\n📊 RSI: {rsi}\n"

async def send_recommendations(application):
    global CHAT_ID
    if not CHAT_ID:
        return
    msg = "\u2728 توصيات التداول:\n"
    for name, symbol in ASSETS.items():
        df = fetch_data(symbol)
        if df is not None:
            sig, rsi, price = analyze(df)
            msg += format_analysis(name, sig, rsi, price) + "\n"
    await application.bot.send_message(chat_id=CHAT_ID, text=msg)

async def signals(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global CHAT_ID
    CHAT_ID = update.effective_chat.id
    await send_recommendations(context.application)

async def analyze_asset(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.strip()
    for name, symbol in ASSETS.items():
        if name.lower() in text.lower():
            df = fetch_data(symbol)
            if df is not None:
                sig, rsi, price = analyze(df)
                await update.message.reply_text(format_analysis(name, sig, rsi, price))
                return
    await update.message.reply_text("⚠️ لم أتعرف على الأصل الذي كتبته. جرب كتابة الاسم كما هو.")

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("🧠 الأوامر:\n/start - بدء\n/tawsiyat - عرض توصيات\n/ta7lil <اسم> - تحليل عملة\nاكتب اسم أصل وسأحلله مباشرة!")

async def text_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.strip()
    if "توصيات" in text:
        await signals(update, context)
    elif "تحليل" in text or any(asset.lower() in text.lower() for asset in ASSETS):
        await analyze_asset(update, context)
    elif "مساعدة" in text or "help" in text:
        await help_command(update, context)
    elif "سوق" in text:
        await signals(update, context)
    else:
        await update.message.reply_text("❓ لم أفهم طلبك. جرب الضغط على الأزرار أو كتابة /start")

def schedule_job(app):
    schedule.every(60).seconds.do(lambda: asyncio.run_coroutine_threadsafe(send_recommendations(app), asyncio.get_event_loop()))

def run_scheduler(app):
    schedule_job(app)
    while True:
        schedule.run_pending()
        time.sleep(1)

async def main():
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("tawsiyat", signals))
    app.add_handler(CommandHandler("ta7lil", analyze_asset))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, text_handler))

    loop = asyncio.get_event_loop()
    loop.create_task(asyncio.to_thread(run_scheduler, app))

    logging.info("🚀 البوت يعمل الآن...")
    await app.run_polling()

if __name__ == "__main__":
    import sys
    if sys.platform.startswith("win") and sys.version_info >= (3, 8):
        asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())

    asyncio.run(main())

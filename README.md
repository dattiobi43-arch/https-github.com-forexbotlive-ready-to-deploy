import os
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import requests
from datetime import datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, InputMediaPhoto
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes

# YOUR KEYS (already filled)
TELEGRAM_TOKEN = "8546430777:AAEOuGqdCglh4P1g5TcIm8P2xDbhaCb7v-M"
AV_KEY = "C91O68HMMB5UYSGT"

PAIRS = ["EURUSD","GBPUSD","USDJPY","AUDUSD","USDCAD","NZDUSD","EURGBP","EURJPY","GBPJPY"]

def get_fx_data(pair):
    url = f"https://www.alphavantage.co/query?function=FX_INTRADAY&from_symbol={pair[:3]}&to_symbol={pair[3:]}&interval=5min&apikey={AV_KEY}&outputsize=compact&datatype=csv"
    df = pd.read_csv(url)
    df['timestamp'] = pd.to_datetime(df['timestamp'])
    df = df.sort_values('timestamp')
    return df

async def start(update: Update, context):
    keyboard = [[InlineKeyboardButton(p, callback_data=p)] for p in PAIRS]
    await update.message.reply_text("Live Forex AI Bot ⚡\nChoose pair:", reply_markup=InlineKeyboardMarkup(keyboard))

async def button(update: Update, context):
    query = update.callback_query
    await query.answer()
    pair = query.data
    await query.edit_message_text("Fetching data...")

    df = get_fx_data(pair)
    if len(df) < 50:
        await query.edit_message_text("No data right now, try again")
        return

    last = df.iloc[-1]
    change = (last['close'] - df.iloc[-2]['close']) / df.iloc[-2]['close'] * 100

    plt.figure(figsize=(10,6), facecolor='black')
    plt.plot(df['timestamp'][-50:], df['close'][-50:], color='cyan')
    plt.title(f"{pair} • {last['close']:.5f} ({change:+.3f}%)", color='white')
    plt.xticks(rotation=45, color='white')
    plt.yticks(color='white')
    plt.grid(alpha=0.3)
    plt.tight_layout()
    plt.savefig("chart.png", facecolor='black')
    plt.close()

    caption = f"*{pair}* • {datetime.now().strftime('%H:%M')}\nPrice: `{last['close']:.5f}`\nChange: `{change:+.3f}%`"

    with open("chart.png", "rb") as photo:
        await query.edit_message_media(
            media=InputMediaPhoto(photo, caption=caption, parse_mode="Markdown"),
            reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Refresh", callback_data=pair)]])
        )

application = Application.builder().token(TELEGRAM_TOKEN).build()
application.add_handler(CommandHandler("start", start))
application.add_handler(CallbackQueryHandler(button))

print("Bot is running... Open Telegram and /start")
application.run_polling()

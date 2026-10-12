
import os
import logging
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    ContextTypes,
    filters,
)

logging.basicConfig(level=logging.INFO)
TOKEN = os.getenv("BOT_TOKEN")


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("🏴‍☠️ Մեր խումբը", url="https://t.me/sevdrosh_bot")],
        [InlineKeyboardButton("ℹ️ Օգնություն", callback_data="help")],
    ]
    await update.effective_message.reply_text(
        "🏴‍☠️ Բարի գալուստ Սև Դրոշ։\n\n"
        "⚓ Ես քո բոտն եմ։ Ընտրիր կոճակը ներքևում։",
        reply_markup=InlineKeyboardMarkup(keyboard),
    )


async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    if query.data == "help":
        await query.message.reply_text(
            "ℹ️ Օգնություն\n"
            "Օգտագործիր /start հրամանը՝ գլխավոր ընտրացանկը բացելու համար։"
        )


async def chat_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_message and update.effective_message.text:
        text = update.effective_message.text.strip().lower()

        if text in ("բարև", "բարեւ", "բարև ձեզ"):
            await update.effective_message.reply_text(
                "🏴‍☠️ Բարև, նավաստի՛։ Բարի գալուստ Սև Դրոշ։"
            )


def main():
    if not TOKEN:
        raise RuntimeError("BOT_TOKEN-ը կարգավորված չէ։")

    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(button_handler))
    app.add_handler(
        MessageHandler(filters.TEXT & ~filters.COMMAND, chat_handler)
    )

    print("Սև Դրոշ բոտը գործարկվում է...")
    app.run_polling()


if __name__ == "__main__":
    main()

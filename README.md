

import os
import logging

from telegram import (
    Update,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    ChatPermissions,
)
from telegram.error import TelegramError
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

GROUP_LINK = "https://t.me/+XLLyoXGPoBBmNDA5"

CHANNELS = [
    {
        "name": "🏴‍☠️ @Sevdrosh",
        "chat_id": "@Sevdrosh",
        "url": "https://t.me/Sevdrosh",
    },
    {
        "name": "⚓ Երկրորդ ալիք",
        "chat_id": os.getenv("CHANNEL2_ID", ""),
        "url": "https://t.me/+RfB5XnjF7PYwOTQ0",
    },
    {
        "name": "⚓ Երրորդ ալիք",
        "chat_id": os.getenv("CHANNEL3_ID", ""),
        "url": "https://t.me/+TTcMszInB0cyYjZk",
    },
]


def channels_ready():
    return all(str(c["chat_id"]).strip() for c in CHANNELS)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("🏴‍☠️ Մեր խումբը", url=GROUP_LINK)],
        [InlineKeyboardButton("ℹ️ Օգնություն", callback_data="help")],
    ]
    await update.effective_message.reply_text(
        "🏴‍☠️ Բարի գալուստ Սև Դրոշ։\n\n"
        "⚓ Ես քո բոտն եմ։ Ընտրիր կոճակը ներքևում։",
        reply_markup=InlineKeyboardMarkup(keyboard),
    )


async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.effective_message.reply_text(
        "ℹ️ Օգնություն\n"
        "/start — գլխավոր ընտրացանկ\n"
        "/help — օգնություն\n"
        "/ping — ստուգել բոտը\n"
        "/id — ցույց տալ ID-ները"
    )


async def ping(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.effective_message.reply_text("🏴‍☠️ Բոտը աշխատում է։")


async def show_id(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    chat_id = update.effective_chat.id
    await update.effective_message.reply_text(
        f"👤 Քո ID-ն՝ {user_id}\n💬 Չաթի ID-ն՝ {chat_id}"
    )


async def check_membership(context, user_id: int) -> bool:
    for channel in CHANNELS:
        try:
            member = await context.bot.get_chat_member(
                chat_id=channel["chat_id"],
                user_id=user_id,
            )
            if member.status not in ("member", "administrator", "creator"):
                if not (
                    member.status == "restricted"
                    and getattr(member, "is_member", False)
                ):
                    return False
        except TelegramError:
            logging.exception("Չհաջողվեց ստուգել %s", channel["name"])
            return False
    return True


def join_keyboard():
    buttons = [
        [InlineKeyboardButton(c["name"], url=c["url"])]
        for c in CHANNELS
    ]
    buttons.append([
        InlineKeyboardButton("✅ Ստուգել", callback_data="verify")
    ])
    return InlineKeyboardMarkup(buttons)


async def restrict_user(context, group_id: int, user_id: int):
    await context.bot.restrict_chat_member(
        chat_id=group_id,
        user_id=user_id,
        permissions=ChatPermissions(
            can_send_messages=False,
            can_send_audios=False,
            can_send_documents=False,
            can_send_photos=False,
            can_send_videos=False,
            can_send_video_notes=False,
            can_send_voice_notes=False,
            can_send_polls=False,
            can_send_other_messages=False,
            can_add_web_page_previews=False,
        ),
    )


async def allow_user(context, group_id: int, user_id: int):
    await context.bot.restrict_chat_member(
        chat_id=group_id,
        user_id=user_id,
        permissions=ChatPermissions(
            can_send_messages=True,
            can_send_audios=True,
            can_send_documents=True,
            can_send_photos=True,
            can_send_videos=True,
            can_send_video_notes=True,
            can_send_voice_notes=True,
            can_send_polls=True,
            can_send_other_messages=True,
            can_add_web_page_previews=True,
        ),
    )


async def new_members(update: Update, context: ContextTypes.DEFAULT_TYPE):
    message = update.effective_message
    if not message:
        return

    for user in message.new_chat_members or []:
        if user.is_bot:
            continue

        if not channels_ready():
            await message.reply_text(
                "⚙️ Ալիքների կարգավորումները դեռ ավարտված չեն։"
            )
            continue

        try:
            if not await check_membership(context, user.id):
                await restrict_user(context, message.chat.id, user.id)
                await message.reply_text(
                    f"🏴‍☠️ Բարի գալուստ, {user.first_name}։\n\n"
                    "Չաթում գրելու համար հետևիր բոլոր 3 ալիքներին, "
                    "ապա սեղմիր «Ստուգել»։",
                    reply_markup=join_keyboard(),
                )
        except TelegramError:
            logging.exception("Չհաջողվեց սահմանափակել նոր անդամին")
            await message.reply_text(
                "❗ Չհաջողվեց սահմանափակել անդամին։ "
                "Ստուգիր բոտի ադմինիստրատորի թույլտվությունները։"
            )


async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    if query.data == "help":
        await query.message.reply_text(
            "ℹ️ Օգնություն\n"
            "/start — գլխավոր ընտրացանկ\n"
            "/help — օգնություն\n"
            "/ping — ստուգել բոտը\n"
            "/id — ցույց տալ ID-ները"
        )

    elif query.data == "verify":
        if not channels_ready():
            await query.message.reply_text(
                "⚙️ Փակ ալիքների ID-ները դեռ կարգավորված չեն։"
            )
            return

        user_id = query.from_user.id
        group_id = query.message.chat.id

        try:
            if await check_membership(context, user_id):
                await allow_user(context, group_id, user_id)
                await query.message.reply_text(
                    f"✅ {query.from_user.first_name}, ստուգումն անցավ։ "
                    "Հիմա կարող ես գրել չաթում։"
                )
            else:
                await query.message.reply_text(
                    "❌ Դեռ բոլոր ալիքներին չես հետևել։ "
                    "Հետևիր բոլորին և նորից ստուգիր։",
                    reply_markup=join_keyboard(),
                )
        except TelegramError:
            logging.exception("Ստուգման սխալ")
            await query.message.reply_text(
                "❗ Ստուգումը չստացվեց։ Ստուգիր բոտի ադմինիստրատորի "
                "թույլտվությունները բոլոր ալիքներում և խմբում։"
            )


async def greeting(update: Update, context: ContextTypes.DEFAULT_TYPE):
    message = update.effective_message
    if not message or not message.text:
        return

    if message.text.strip().lower() in ("բարև", "բարեւ", "բարև ձեզ"):
        await message.reply_text(
            "🏴‍☠️ Բարև, նավաստի՛։ Բարի գալուստ Սև Դրոշ։"
        )


async def forwarded_channel_id(
    update: Update, context: ContextTypes.DEFAULT_TYPE
):
    message = update.effective_message
    origin = getattr(message, "forward_origin", None)

    if origin and getattr(origin, "type", None) == "channel":
        chat = origin.chat
        await message.reply_text(
            f"📢 Ալիք՝ {getattr(chat, 'title', 'Անանուն')}\n"
            f"🆔 Ալիքի ID՝ {chat.id}"
        )


def main():
    if not TOKEN:
        raise RuntimeError("BOT_TOKEN-ը կարգավորված չէ։")

    app = Application.builder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("ping", ping))
    app.add_handler(CommandHandler("id", show_id))
    app.add_handler(CallbackQueryHandler(button_handler))
    app.add_handler(
        MessageHandler(filters.StatusUpdate.NEW_CHAT_MEMBERS, new_members)
    )
    app.add_handler(
        MessageHandler(filters.FORWARDED, forwarded_channel_id)
    )
    app.add_handler(
        MessageHandler(filters.TEXT & ~filters.COMMAND, greeting)
    )

    print("Սև Դրոշ բոտը գործարկվում է...")
    app.run_polling()


if __name__ == "__main__":
    main()

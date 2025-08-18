import json
from telegram import Update, ChatPermissions
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackContext

# ----- معلومات البوت والقروب والحساب الشخصي -----
TOKEN = "8183729953:AAG1hPOTbQ0Bstq1wnnTfOta1zqCGS23Ifw"
CHAT_ID = "-1002917150685"
USER_ID = "1875496677"

# ----- ملف التحذيرات -----
WARNINGS_FILE = "warnings.json"

# ----- تحميل التحذيرات من الملف أو إنشاء ملف جديد -----
try:
    with open(WARNINGS_FILE, "r") as f:
        warnings = json.load(f)
except FileNotFoundError:
    warnings = {}

updater = Updater(token=TOKEN, use_context=True)
dispatcher = updater.dispatcher

# ----- التحقق من صاحب البوت فقط -----
def is_owner(update: Update):
    return str(update.effective_user.id) == USER_ID

# ----- حفظ التحذيرات ----- 
def save_warnings():
    with open(WARNINGS_FILE, "w") as f:
        json.dump(warnings, f)

# ----- أوامر البوت -----
def start(update: Update, context: CallbackContext):
    if is_owner(update):
        update.message.reply_text("بوت AI جاهز للعمل 🤖")
    else:
        update.message.reply_text("أنت غير مخوّل لاستخدام البوت 🚫")

def mute_user(update: Update, context: CallbackContext):
    if not is_owner(update):
        return
    if update.message.reply_to_message:
        user_id = update.message.reply_to_message.from_user.id
        context.bot.restrict_chat_member(
            chat_id=CHAT_ID,
            user_id=user_id,
            permissions=ChatPermissions(can_send_messages=False)
        )
        update.message.reply_text("تم ميوت العضو ✅")
    else:
        update.message.reply_text("رد على رسالة العضو لتعمل ميوت له!")

def unmute_user(update: Update, context: CallbackContext):
    if not is_owner(update):
        return
    if update.message.reply_to_message:
        user_id = update.message.reply_to_message.from_user.id
        context.bot.restrict_chat_member(
            chat_id=CHAT_ID,
            user_id=user_id,
            permissions=ChatPermissions(can_send_messages=True)
        )
        update.message.reply_text("تم إلغاء الميوت ✅")
    else:
        update.message.reply_text("رد على رسالة العضو لإلغاء الميوت!")

def delete_message(update: Update, context: CallbackContext):
    if not is_owner(update):
        return
    if update.message.reply_to_message:
        context.bot.delete_message(chat_id=CHAT_ID, message_id=update.message.reply_to_message.message_id)
        update.message.reply_text("تم حذف الرسالة ✅")
    else:
        update.message.reply_text("رد على الرسالة التي تريد حذفها!")

def warn_user(update: Update, context: CallbackContext):
    if not is_owner(update):
        return
    if update.message.reply_to_message:
        user_id = str(update.message.reply_to_message.from_user.id)
        warnings[user_id] = warnings.get(user_id, 0) + 1
        save_warnings()
        update.message.reply_text(f"تم تحذير العضو ⚠️ عدد التحذيرات: {warnings[user_id]}")
        if warnings[user_id] >= 3:
            context.bot.kick_chat_member(chat_id=CHAT_ID, user_id=int(user_id))
            update.message.reply_text("العضو تم طرده بعد 3 تحذيرات 🚫")
            warnings[user_id] = 0
            save_warnings()
    else:
        update.message.reply_text("رد على رسالة العضو لتحذيره!")

# ----- AI دردشة خاصة معك -----
def chat_private(update: Update, context: CallbackContext):
    if str(update.effective_chat.id) == USER_ID:
        user_msg = update.message.text
        reply = f"AI: فهمت كلامك '{user_msg}' 😉"
        update.message.reply_text(reply)

# ----- استقبال أي رسالة عامة (قروب) -----
def echo_group(update: Update, context: CallbackContext):
    pass  # يمكن إضافة فلترة كلمات أو ردود ذكية لاحقًا

# ----- إضافة الأوامر -----
dispatcher.add_handler(CommandHandler("start", start))
dispatcher.add_handler(CommandHandler("mute", mute_user))
dispatcher.add_handler(CommandHandler("unmute", unmute_user))
dispatcher.add_handler(CommandHandler("delete", delete_message))
dispatcher.add_handler(CommandHandler("warn", warn_user))
dispatcher.add_handler(MessageHandler(Filters.private & Filters.text, chat_private))
dispatcher.add_handler(MessageHandler(Filters.group & Filters.text, echo_group))

# ----- تشغيل البوت -----
updater.start_polling()
updater.idle()

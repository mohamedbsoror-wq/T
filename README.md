import sqlite3
import datetime
import random
import os  # مكتبة التعامل مع نظام التشغيل
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes, CallbackQueryHandler, MessageHandler, filters

# ==========================================================
# ⚙️ الإعدادات الآمنة (هذه القيم سيتم سحبها من الاستضافة)
# ==========================================================
TOKEN = os.getenv("BOT_TOKEN")  # سيسحب التوكن من الاستضافة
ADMIN_ID = int(os.getenv("ADMIN_ID", "0"))  # سيسحب أيدي الأدمن من الاستضافة
BOT_USERNAME = "Scripts127_bot" 
CHANNEL_ID = "@i_71j"           

# 🛒 قائمة السكربتات
SCRIPTS_LIST = {
    "sc1": {"name": "🚀 سكربت الطيران VIP", "price": 50, "content": "FLY_SCRIPT_V1"},
    "sc2": {"name": "🛡️ سكربت الحماية المطور", "price": 30, "content": "ANTI_BAN_V4"},
}

# ==========================================================
# 🗄️ وظائف قاعدة البيانات
# ==========================================================
def init_db():
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    cursor.execute('CREATE TABLE IF NOT EXISTS users (user_id INTEGER PRIMARY KEY, balance REAL DEFAULT 0, last_gift TEXT, is_banned INTEGER DEFAULT 0)')
    cursor.execute('CREATE TABLE IF NOT EXISTS codes (code_text TEXT PRIMARY KEY, amount REAL, limit_use INTEGER, expiry_time TEXT)')
    cursor.execute('CREATE TABLE IF NOT EXISTS code_history (user_id INTEGER, code_text TEXT)')
    conn.commit()
    conn.close()

def get_user(user_id):
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    cursor.execute("SELECT balance, last_gift, is_banned FROM users WHERE user_id = ?", (user_id,))
    res = cursor.fetchone()
    conn.close()
    return {"balance": res[0], "last_gift": res[1], "is_banned": res[2]} if res else None

def update_user(user_id, balance, last_gift=None):
    conn = sqlite3.connect('database.db')
    if last_gift:
        conn.execute("UPDATE users SET balance = ?, last_gift = ? WHERE user_id = ?", (balance, last_gift, user_id))
    else:
        conn.execute("UPDATE users SET balance = ? WHERE user_id = ?", (balance, user_id))
    conn.commit()
    conn.close()

init_db()

# ==========================================================
# 🛡️ فحص الاشتراك الإجباري
# ==========================================================
async def is_subscribed(context, user_id):
    try:
        member = await context.bot.get_chat_member(chat_id=CHANNEL_ID, user_id=user_id)
        return member.status in ['member', 'administrator', 'creator']
    except:
        return False

# ==========================================================
# 🕹️ لوحة المفاتيح الرئيسية
# ==========================================================
def main_keyboard(user_id):
    btns = [
        [InlineKeyboardButton("🚀 متجر السكربتات", callback_data='shop')],
        [InlineKeyboardButton("🎁 الهدية اليومية", callback_data='daily_gift'), InlineKeyboardButton("🎫 كود هدية", callback_data='enter_code')],
        [InlineKeyboardButton("👤 حسابي", callback_data='my_account'), InlineKeyboardButton("🤝 دعوة", callback_data='invite')]
    ]
    if user_id == ADMIN_ID:
        btns.append([InlineKeyboardButton("⚙️ لوحة التحكم", callback_data='admin_panel')])
    return InlineKeyboardMarkup(btns)

# ==========================================================
# 🚀 الأوامر الأساسية
# ==========================================================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    
    if not await is_subscribed(context, user_id):
        txt = (
            "🚸| عذراً عزيزي..\n"
            "🔰| عليك الاشتراك في قناة البوت لتتمكن من استخدامه\n\n"
            f"- {CHANNEL_ID}\n\n"
            "‼️| اشترك ثم ارسل /start"
        )
        btns = [[InlineKeyboardButton("✅ اضغط للاشتراك", url=f"https://t.me/{CHANNEL_ID[1:]}")]]
        return await update.message.reply_text(txt, reply_markup=InlineKeyboardMarkup(btns))

    user = get_user(user_id)
    if not user:
        conn = sqlite3.connect('database.db')
        conn.execute("INSERT INTO users (user_id, balance) VALUES (?, ?)", (user_id, 0))
        conn.commit()
        conn.close()
        user = {"balance": 0}
        if context.args and context.args[0].isdigit():
            inviter_id = int(context.args[0])
            inviter = get_user(inviter_id)
            if inviter and inviter_id != user_id:
                update_user(inviter_id, inviter['balance'] + 5)
                try: await context.bot.send_message(inviter_id, "🤝 مبروك! حصلت على 5 كوينز لدعوة صديق.")
                except: pass

    txt = f"👋 <b>مرحباً بك في متجر السكربتات</b>\n━━━━━━━━━━━━━━\n💰 رصيدك: <code>{user['balance']}</code> كوينز"
    await update.message.reply_text(txt, reply_markup=main_keyboard(user_id), parse_mode='HTML')

# ==========================================================
# 🛠️ أوامر المدير (Add, BC, Make Code)
# ==========================================================
async def add_balance(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID: return
    try:
        uid, amt = int(context.args[0]), float(context.args[1])
        u = get_user(uid)
        if u:
            update_user(uid, u['balance'] + amt)
            await update.message.reply_text(f"✅ تم إضافة {amt} كوينز للأيدي `{uid}`")
        else: await update.message.reply_text("❌ المستخدم غير موجود.")
    except: await update.message.reply_text("⚠️ `/add [ID] [Amt]`")

async def broadcast(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID: return
    msg = update.message.text.replace('/bc ', '')
    if not msg or msg == '/bc': return await update.message.reply_text("⚠️ اكتب الرسالة.")
    conn = sqlite3.connect('database.db'); users = conn.execute("SELECT user_id FROM users").fetchall(); conn.close()
    count = 0
    for u in users:
        try: await context.bot.send_message(u[0], f"📢 <b>رسالة من الإدارة:</b>\n\n{msg}", parse_mode='HTML'); count += 1
        except: pass
    await update.message.reply_text(f"✅ تم الإرسال لـ {count} مستخدم.")

async def make_code_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID: return
    context.user_data['making_code'] = True
    context.user_data['step'] = 'name'
    await update.message.reply_text("🎫 ارسل الآن **اسم الكود**:")

# ==========================================================
# 📩 معالج الرسائل
# ==========================================================
async def handle_messages(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    text = update.message.text
    if user_id == ADMIN_ID and context.user_data.get('making_code'):
        step = context.user_data.get('step')
        if step == 'name':
            context.user_data['c_name'] = text; context.user_data['step'] = 'amt'
            await update.message.reply_text(f"✅ الاسم: {text}\nارسل الآن **المبلغ**:")
        elif step == 'amt':
            context.user_data['c_amt'] = text; context.user_data['step'] = 'limit'
            await update.message.reply_text("✅ ارسل الآن **عدد مرات الاستخدام**:")
        elif step == 'limit':
            context.user_data['c_limit'] = text; context.user_data['step'] = 'mins'
            await update.message.reply_text("✅ ارسل الآن **عدد دقائق الصلاحية**:")
        elif step == 'mins':
            try:
                name, amt, limit, mins = context.user_data['c_name'], float(context.user_data['c_amt']), int(context.user_data['c_limit']), int(text)
                expiry = (datetime.datetime.now() + datetime.timedelta(minutes=mins)).strftime("%Y-%m-%d %H:%M:%S")
                conn = sqlite3.connect('database.db'); conn.execute("INSERT INTO codes VALUES (?, ?, ?, ?)", (name, amt, limit, expiry)); conn.commit(); conn.close()
                await update.message.reply_text(f"✅ تم إنشاء كود: <code>{name}</code> بنجاح!", parse_mode='HTML')
            except: await update.message.reply_text("❌ خطأ في البيانات!")
            context.user_data['making_code'] = False
        return
    if context.user_data.get('waiting_for_code'):
        context.user_data['waiting_for_code'] = False
        user = get_user(user_id)
        conn = sqlite3.connect('database.db'); cursor = conn.cursor()
        cursor.execute("SELECT amount, limit_use, expiry_time FROM codes WHERE code_text = ?", (text,))
        res = cursor.fetchone()
        if not res: return await update.message.reply_text("❌ الكود خاطئ أو انتهى!")
        amt, limit, exp = res
        update_user(user_id, user['balance'] + amt)
        cursor.execute("INSERT INTO code_history VALUES (?, ?)", (user_id, text))
        if limit - 1 <= 0: cursor.execute("DELETE FROM codes WHERE code_text = ?", (text,))
        else: cursor.execute("UPDATE codes SET limit_use = ? WHERE code_text = ?", (limit-1, text))
        conn.commit(); conn.close()
        await update.message.reply_text(f"🎉 مبروك حصلت على {amt} كوينز!")

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query; user_id = query.from_user.id; user = get_user(user_id); await query.answer()
    if query.data == 'back':
        await query.edit_message_text(f"👋 <b>القائمة الرئيسية</b>\n💰 رصيدك: <code>{user['balance']}</code> كوينز", reply_markup=main_keyboard(user_id), parse_mode='HTML')
    elif query.data == 'daily_gift':
        today = datetime.datetime.now().strftime("%Y-%m-%d")
        if user['last_gift'] == today: await context.bot.answer_callback_query(query.id, "❌ استلمتها اليوم بالفعل!", show_alert=True)
        else:
            gift = random.randint(10, 30); update_user(user_id, user['balance'] + gift, last_gift=today)
            await query.edit_message_text(f"🎁 حصلت على {gift} كوينز هدية!", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙", callback_data='back')]]))
    elif query.data == 'invite':
        await query.edit_message_text(f"🤝 رابطك: <code>https://t.me/{BOT_USERNAME}?start={user_id}</code>\nتحصل على 5 كوينز لكل دعوة.", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙", callback_data='back')]]), parse_mode='HTML')
    elif query.data == 'my_account':
        await query.edit_message_text(f"👤 حسابك:\n🆔: <code>{user_id}</code>\n💰: {user['balance']} كوينز", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙", callback_data='back')]]), parse_mode='HTML')
    elif query.data == 'enter_code':
        context.user_data['waiting_for_code'] = True
        await query.edit_message_text("⌨️ ارسل كود الهدية الآن:", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("❌ إلغاء", callback_data='back')]]))
    elif query.data == 'admin_panel' and user_id == ADMIN_ID:
        await query.edit_message_text("⚙️ لوحة التحكم:\n\n/bc الإذاعة\n/make_code صنع كود\n/add إضافة رصيد", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙", callback_data='back')]]))
    elif query.data == 'shop':
        btns = [[InlineKeyboardButton(f"{v['name']} | {v['price']} 💰", callback_data=f"conf_{k}")] for k,v in SCRIPTS_LIST.items()]
        btns.append([InlineKeyboardButton("🔙 رجوع", callback_data='back')])
        await query.edit_message_text("🛒 المتجر:", reply_markup=InlineKeyboardMarkup(btns))

if __name__ == '__main__':
    if not TOKEN:
        print("❌ خطأ: التوكن غير موجود! تأكد من ضبط الـ Environment Variables.")
    else:
        app = ApplicationBuilder().token(TOKEN).build()
        app.add_handler(CommandHandler("start", start))
        app.add_handler(CommandHandler("bc", broadcast))
        app.add_handler(CommandHandler("add", add_balance))
        app.add_handler(CommandHandler("make_code", make_code_start))
        app.add_handler(CallbackQueryHandler(button_handler))
        app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_messages))
        print("🚀 البوت الآمن يعمل الآن...")
        app.run_polling()

import os
import json
import time
from datetime import datetime, timedelta
import pytz
from prayer_times import PrayerTimes
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    filters,
    ContextTypes,
    JobQueue,
)

# تعريف المتغيرات الأساسية
TOKEN = "7352787407:AAEyRbUqFBCWdBMGzx71qHo7UrnuoSNjnDE"
BOT_USERNAME = "@ALAllaaA_bot"
DATA_FILE = "user_data.json"

# تهيئة بيانات المستخدمين
user_data = {}

# تحميل البيانات من الملف
def load_data():
    global user_data
    try:
        with open(DATA_FILE, "r") as f:
            user_data = json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        user_data = {}

# حفظ البيانات في الملف
def save_data():
    with open(DATA_FILE, "w") as f:
        json.dump(user_data, f, indent=4)

# هيكل البيانات الافتراضي للمستخدم
def default_user_data():
    return {
        "tasks": [],
        "active_sessions": {},
        "completed_sessions": [],
        "location": None,
        "prayer_times": None,
        "last_prayer_notified": None,
    }

# تأكد من وجود بيانات المستخدم
def ensure_user_data(user_id):
    if str(user_id) not in user_data:
        user_data[str(user_id)] = default_user_data()

# إدارة المهام 📋
async def add_tasks(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    ensure_user_data(user_id)
    
    text = update.message.text.replace("اضف", "").strip()
    tasks = [t.strip() for t in text.split("\n") if t.strip()]
    
    for task in tasks:
        user_data[str(user_id)]["tasks"].append({
            "text": task,
            "completed": False,
            "created_at": datetime.now().isoformat()
        })
    
    save_data()
    await update.message.reply_text(f"تمت إضافة {len(tasks)} مهمة بنجاح!")
    await show_tasks(update, context)

async def show_tasks(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    ensure_user_data(user_id)
    
    tasks = user_data[str(user_id)].get("tasks", [])
    if not tasks:
        await update.message.reply_text("لا توجد مهام حالية.")
        return
    
    tasks_text = "📋 قائمة المهام:\n\n"
    for i, task in enumerate(tasks, 1):
        status = "✅" if task["completed"] else "⏹️"
        tasks_text += f"{i}. {status} {task['text']}\n"
    
    keyboard = [
        [
            InlineKeyboardButton("▶️ بدء جلسة", callback_data="start_session"),
            InlineKeyboardButton("📊 إحصائيات", callback_data="show_stats"),
        ],
        [
            InlineKeyboardButton("✏️ تعديل", callback_data="edit_task"),
            InlineKeyboardButton("🗑️ حذف", callback_data="delete_task"),
        ]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await update.message.reply_text(tasks_text, reply_markup=reply_markup)

async def edit_task(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    ensure_user_data(user_id)
    
    try:
        parts = update.message.text.split()
        task_num = int(parts[1]) - 1
        new_text = " ".join(parts[2:])
        
        tasks = user_data[str(user_id)]["tasks"]
        if 0 <= task_num < len(tasks):
            tasks[task_num]["text"] = new_text
            save_data()
            await update.message.reply_text("تم تعديل المهمة بنجاح!")
            await show_tasks(update, context)
        else:
            await update.message.reply_text("رقم المهمة غير صحيح!")
    except (IndexError, ValueError):
        await update.message.reply_text("استخدم الأمر هكذا: تعديل 1 النص الجديد")

async def delete_task(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    ensure_user_data(user_id)
    
    try:
        task_num = int(update.message.text.split()[1]) - 1
        tasks = user_data[str(user_id)]["tasks"]
        if 0 <= task_num < len(tasks):
            deleted_task = tasks.pop(task_num)
            save_data()
            await update.message.reply_text(f"تم حذف المهمة: {deleted_task['text']}")
            await show_tasks(update, context)
        else:
            await update.message.reply_text("رقم المهمة غير صحيح!")
    except (IndexError, ValueError):
        await update.message.reply_text("استخدم الأمر هكذا: حذف 2")

async def toggle_task(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    ensure_user_data(user_id)
    
    try:
        task_num = int(query.data.split("_")[1]) - 1
        tasks = user_data[str(user_id)]["tasks"]
        if 0 <= task_num < len(tasks):
            tasks[task_num]["completed"] = not tasks[task_num]["completed"]
            save_data()
            await show_tasks(update, context)
    except (IndexError, ValueError):
        pass

# جلسات بومودورو ⏳
async def start_session(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    ensure_user_data(user_id)
    
    tasks = user_data[str(user_id)]["tasks"]
    if not tasks:
        await query.edit_message_text("لا توجد مهام لبدء جلسة!")
        return
    
    keyboard = []
    for i, task in enumerate(tasks, 1):
        keyboard.append([InlineKeyboardButton(
            f"{i}. {task['text']}",
            callback_data=f"select_task_{i}"
        )])
    
    reply_markup = InlineKeyboardMarkup(keyboard)
    await query.edit_message_text("اختر المهمة لبدء جلسة:", reply_markup=reply_markup)

async def select_task_for_session(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    ensure_user_data(user_id)
    
    try:
        task_num = int(query.data.split("_")[2]) - 1
        tasks = user_data[str(user_id)]["tasks"]
        if 0 <= task_num < len(tasks):
            user_data[str(user_id)]["active_sessions"][str(task_num)] = {
                "start_time": datetime.now().isoformat(),
                "task_index": task_num
            }
            save_data()
            
            # جدولة تذكير بعد 25 دقيقة (جلسة بومودورو قياسية)
            context.job_queue.run_once(
                end_session_reminder,
                25 * 60,
                chat_id=query.message.chat_id,
                user_id=user_id,
                task_num=task_num,
                data=query.message.message_id
            )
            
            await query.edit_message_text(
                f"بدأت جلسة العمل على المهمة: {tasks[task_num]['text']}\n"
                "⏳ جلسة عمل لمدة 25 دقيقة...\n\n"
                "اضغط ⏹️ لإنهاء الجلسة مبكراً",
                reply_markup=InlineKeyboardMarkup([
                    [InlineKeyboardButton("⏹️ إنهاء الجلسة", callback_data=f"end_session_{task_num}")]
                ])
            )
    except (IndexError, ValueError):
        pass

async def end_session(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    ensure_user_data(user_id)
    
    try:
        task_num = int(query.data.split("_")[2])
        end_session_for_task(user_id, task_num)
        
        tasks = user_data[str(user_id)]["tasks"]
        await query.edit_message_text(
            f"انتهت جلسة العمل على المهمة: {tasks[task_num]['text']}\n"
            "خذ استراحة لمدة 5 دقائق!",
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton("📋 عرض المهام", callback_data="show_tasks")]
            ])
        )
    except (IndexError, ValueError):
        pass

def end_session_for_task(user_id, task_num):
    user_id = str(user_id)
    task_num = str(task_num)
    
    if task_num in user_data[user_id]["active_sessions"]:
        session = user_data[user_id]["active_sessions"].pop(task_num)
        start_time = datetime.fromisoformat(session["start_time"])
        duration = (datetime.now() - start_time).total_seconds() / 60  # بالدقائق
        
        user_data[user_id]["completed_sessions"].append({
            "task_index": session["task_index"],
            "duration": duration,
            "date": datetime.now().isoformat()
        })
        save_data()

async def end_session_reminder(context: ContextTypes.DEFAULT_TYPE):
    job = context.job
    user_id = job.user_id
    task_num = job.task_num
    
    end_session_for_task(user_id, task_num)
    
    tasks = user_data[str(user_id)]["tasks"]
    await context.bot.send_message(
        job.chat_id,
        f"⏰ انتهت جلسة العمل على المهمة: {tasks[task_num]['text']}\n"
        "خذ استراحة لمدة 5 دقائق!",
        reply_to_message_id=job.data
    )

async def show_stats(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    ensure_user_data(user_id)
    
    completed_sessions = user_data[str(user_id)].get("completed_sessions", [])
    if not completed_sessions:
        await query.edit_message_text("لا توجد إحصائيات متاحة بعد.")
        return
    
    total_time = sum(s["duration"] for s in completed_sessions)
    tasks = user_data[str(user_id)]["tasks"]
    
    stats_text = "📊 إحصائيات الجلسات:\n\n"
    stats_text += f"إجمالي وقت العمل: {total_time:.1f} دقيقة\n\n"
    
    # تجميع الوقت لكل مهمة
    task_stats = {}
    for session in completed_sessions:
        task_idx = session["task_index"]
        if task_idx not in task_stats:
            task_stats[task_idx] = 0
        task_stats[task_idx] += session["duration"]
    
    for task_idx, duration in task_stats.items():
        if 0 <= task_idx < len(tasks):
            stats_text += f"{tasks[task_idx]['text']}: {duration:.1f} دقيقة\n"
    
    await query.edit_message_text(
        stats_text,
        reply_markup=InlineKeyboardMarkup([
            [InlineKeyboardButton("📋 عرض المهام", callback_data="show_tasks")]
        ])
    )

# تذكير مواقيت الصلاة 🕌
async def set_location(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    ensure_user_data(user_id)
    
    try:
        location = update.message.text.replace("موقع", "").strip()
        city, country = [part.strip() for part in location.split(",")]
        
        user_data[str(user_id)]["location"] = {
            "city": city,
            "country": country
        }
        
        # احصل على أوقات الصلاة
        try:
            prayer_times = PrayerTimes(city, country).get_times()
            user_data[str(user_id)]["prayer_times"] = prayer_times
            save_data()
            
            await update.message.reply_text(
                f"تم تعيين الموقع إلى: {city}, {country}\n"
                f"أوقات الصلاة اليوم:\n"
                f"الفجر: {prayer_times['fajr']}\n"
                f"الظهر: {prayer_times['dhuhr']}\n"
                f"العصر: {prayer_times['asr']}\n"
                f"المغرب: {prayer_times['maghrib']}\n"
                f"العشاء: {prayer_times['isha']}"
            )
            
            # جدولة التذكير بالصلاة
            schedule_prayer_reminders(context.job_queue, user_id, update.message.chat_id)
            
        except Exception as e:
            await update.message.reply_text(f"حدث خطأ في جلب أوقات الصلاة: {str(e)}")
    except (IndexError, ValueError):
        await update.message.reply_text("استخدم الأمر هكذا: موقع المدينة, البلد")

def schedule_prayer_reminders(job_queue: JobQueue, user_id: int, chat_id: int):
    user_id = str(user_id)
    if user_id not in user_data or not user_data[user_id]["prayer_times"]:
        return
    
    prayer_times = user_data[user_id]["prayer_times"]
    now = datetime.now(pytz.timezone("Asia/Riyadh"))  # افترض توقيت السعودية
    
    for prayer, time_str in prayer_times.items():
        prayer_time = datetime.strptime(time_str, "%H:%M").replace(
            year=now.year, month=now.month, day=now.day,
            tzinfo=pytz.timezone("Asia/Riyadh")
        )
        
        # تذكير قبل 5 دقائق
        reminder_time = prayer_time - timedelta(minutes=5)
        if reminder_time > now:
            job_queue.run_once(
                prayer_reminder,
                (reminder_time - now).total_seconds(),
                chat_id=chat_id,
                user_id=user_id,
                data=prayer
            )

async def prayer_reminder(context: ContextTypes.DEFAULT_TYPE):
    prayer = context.job.data
    user_id = context.job.user_id
    
    # أوقف أي جلسات نشطة
    if user_data[str(user_id)]["active_sessions"]:
        for task_num in list(user_data[str(user_id)]["active_sessions"].keys()):
            end_session_for_task(user_id, task_num)
    
    prayer_name = {
        "fajr": "الفجر",
        "dhuhr": "الظهر",
        "asr": "العصر",
        "maghrib": "المغرب",
        "isha": "العشاء"
    }.get(prayer, prayer)
    
    await context.bot.send_message(
        context.job.chat_id,
        f"⏰ تذكير: وقت صلاة {prayer_name} بعد 5 دقائق\n"
        "الرجاء التوقف عن العمل والاستعداد للصلاة"
    )

# معالجة الأزرار
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    data = query.data
    
    if data == "show_tasks":
        await show_tasks(update, context)
    elif data == "start_session":
        await start_session(update, context)
    elif data == "show_stats":
        await show_stats(update, context)
    elif data.startswith("select_task_"):
        await select_task_for_session(update, context)
    elif data.startswith("end_session_"):
        await end_session(update, context)
    elif data.startswith("toggle_"):
        await toggle_task(update, context)

# الأوامر الأساسية
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "مرحباً! أنا بوت الإنتاجية الخاص بك.\n\n"
        "يمكنني مساعدتك في:\n"
        "📋 إدارة المهام\n"
        "⏳ جلسات بومودورو\n"
        "🕌 تذكير مواقيت الصلاة\n\n"
        "استخدم /help لرؤية الأوامر المتاحة"
    )

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "🛠️ الأوامر المتاحة:\n\n"
        "📋 إدارة المهام:\n"
        "اضف [المهام] - إضافة مهام جديدة (سطر لكل مهمة)\n"
        "المهام - عرض جميع المهام\n"
        "تعديل [رقم] [نص] - تعديل مهمة\n"
        "حذف [رقم] - حذف مهمة\n\n"
        "⏳ جلسات بومودورو:\n"
        "جلسة - بدء جلسة عمل\n"
        "إحصائيات - عرض إحصائيات الجلسات\n\n"
        "🕌 تذكير الصلاة:\n"
        "موقع [المدينة], [البلد] - تعيين الموقع لأوقات الصلاة"
    )

# معالجة الأخطاء
async def error(update: Update, context: ContextTypes.DEFAULT_TYPE):
    print(f"حدث خطأ: {context.error}")

# تشغيل البوت
def main():
    load_data()
    
    print("Starting bot...")
    app = Application.builder().token(TOKEN).build()
    
    # الأوامر
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("المهام", show_tasks))
    app.add_handler(CommandHandler("إحصائيات", show_stats))
    
    # الرسائل
    app.add_handler(MessageHandler(filters.TEXT & filters.Regex(r"^اضف"), add_tasks))
    app.add_handler(MessageHandler(filters.TEXT & filters.Regex(r"^تعديل"), edit_task))
    app.add_handler(MessageHandler(filters.TEXT & filters.Regex(r"^حذف"), delete_task))
    app.add_handler(MessageHandler(filters.TEXT & filters.Regex(r"^موقع"), set_location))
    
    # الأزرار
    app.add_handler(CallbackQueryHandler(button_handler))
    
    # الأخطاء
    app.add_error_handler(error)
    
    # جدولة التذكيرات
    for user_id in user_data:
        if user_data[user_id]["prayer_times"]:
            schedule_prayer_reminders(app.job_queue, int(user_id), int(user_id))
    
    print("Polling...")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()

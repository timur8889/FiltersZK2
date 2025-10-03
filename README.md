import logging
from datetime import datetime, timedelta
from telegram import Update, ReplyKeyboardMarkup, ReplyKeyboardRemove
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    filters,
    ContextTypes,
    ConversationHandler,
)
import sqlite3

# Настройка логирования
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO
)
logger = logging.getLogger(__name__)

# Состояния для ConversationHandler
ADD_NAME, ADD_DAYS = range(2)

# Инициализация базы данных
def init_db():
    conn = sqlite3.connect('filters.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS filters
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  user_id INTEGER,
                  name TEXT NOT NULL,
                  replace_date DATE NOT NULL)''')
    conn.commit()
    conn.close()

# Добавление фильтра в базу
def add_filter(user_id, name, days):
    conn = sqlite3.connect('filters.db')
    c = conn.cursor()
    replace_date = datetime.now() + timedelta(days=days)
    c.execute("INSERT INTO filters (user_id, name, replace_date) VALUES (?, ?, ?)",
              (user_id, name, replace_date.strftime('%Y-%m-%d')))
    conn.commit()
    conn.close()

# Удаление фильтра
def delete_filter(user_id, filter_id):
    conn = sqlite3.connect('filters.db')
    c = conn.cursor()
    c.execute("DELETE FROM filters WHERE id = ? AND user_id = ?", (filter_id, user_id))
    conn.commit()
    conn.close()

# Получение списка фильтров пользователя
def get_user_filters(user_id):
    conn = sqlite3.connect('filters.db')
    c = conn.cursor()
    c.execute("SELECT id, name, replace_date FROM filters WHERE user_id = ?", (user_id,))
    filters = c.fetchall()
    conn.close()
    return filters

# Проверка просроченных фильтров
def check_expired_filters(context):
    conn = sqlite3.connect('filters.db')
    c = conn.cursor()
    c.execute("SELECT user_id, name FROM filters WHERE replace_date <= date('now')")
    expired_filters = c.fetchall()
    conn.close()
    
    for user_id, name in expired_filters:
        context.bot.send_message(
            chat_id=user_id,
            text=f"🔔 Требуется замена фильтра: {name}!"
        )

# Команда /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [['Добавить фильтр', 'Список фильтров']]
    reply_markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
    await update.message.reply_text(
        'Добро пожаловать! Управляйте заменой фильтров с помощью кнопок ниже:',
        reply_markup=reply_markup
    )

# Начало процесса добавления фильтра
async def add_filter_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        'Введите название фильтра:',
        reply_markup=ReplyKeyboardRemove()
    )
    return ADD_NAME

# Сохранение названия фильтра
async def add_filter_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['filter_name'] = update.message.text
    await update.message.reply_text('Введите интервал замены в днях:')
    return ADD_DAYS

# Сохранение интервала и завершение добавления
async def add_filter_days(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        days = int(update.message.text)
        if days <= 0:
            raise ValueError
        add_filter(update.effective_user.id, context.user_data['filter_name'], days)
        await update.message.reply_text('Фильтр успешно добавлен!')
        return ConversationHandler.END
    except ValueError:
        await update.message.reply_text('Пожалуйста, введите корректное число дней:')
        return ADD_DAYS

# Отмена операции
async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text('Операция отменена.')
    return ConversationHandler.END

# Показать список фильтров
async def show_filters(update: Update, context: ContextTypes.DEFAULT_TYPE):
    filters = get_user_filters(update.effective_user.id)
    if not filters:
        await update.message.reply_text('У вас нет активных фильтров.')
        return

    text = "Ваши фильтры:\n\n"
    for filter_id, name, replace_date in filters:
        days_left = (datetime.strptime(replace_date, '%Y-%m-%d') - datetime.now()).days
        text += f"• {name} (ID: {filter_id})\n  Замена через: {days_left} дней\n  Дата замены: {replace_date}\n\n"
    
    text += "Для удаления фильтра отправьте его ID числом"
    await update.message.reply_text(text)

# Обработчик удаления фильтра
async def delete_filter_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        filter_id = int(update.message.text)
        delete_filter(update.effective_user.id, filter_id)
        await update.message.reply_text('Фильтр успешно удален!')
    except ValueError:
        await update.message.reply_text('Пожалуйста, введите корректный ID фильтра:')

# Основная функция
def main():
    init_db()
    
    application = Application.builder().token("8031772805:AAFDfZoaR2xF_jpE-1PfEWaxtW1OUzBe2RU").build()

    # Добавление периодической проверки
    job_queue = application.job_queue
    job_queue.run_repeating(
        check_expired_filters,
        interval=timedelta(hours=24),
        first=10
    )

    # Обработчик диалога добавления фильтра
    conv_handler = ConversationHandler(
        entry_points=[MessageHandler(filters.Regex('^Добавить фильтр$'), add_filter_start)],
        states={
            ADD_NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, add_filter_name)],
            ADD_DAYS: [MessageHandler(filters.TEXT & ~filters.COMMAND, add_filter_days)],
        },
        fallbacks=[CommandHandler('cancel', cancel)]
    )

    application.add_handler(CommandHandler("start", start))
    application.add_handler(conv_handler)
    application.add_handler(MessageHandler(filters.Regex('^Список фильтров$'), show_filters))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, delete_filter_handler))

    application.run_polling()

if __name__ == '__main__':
    main()

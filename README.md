import requests
import sys
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# 🔎 Рекурсивный поиск area_id по названию города
def find_city_recursive(areas, city_name):
    for area in areas:
        if city_name.lower() in area['name'].lower():
            return area['id']
        if area.get('areas'):
            result = find_city_recursive(area['areas'], city_name)
            if result:
                return result
    return None

def get_area_id(city_name):
    response = requests.get("https://api.hh.ru/areas")
    areas = response.json()
    return find_city_recursive(areas, city_name) or 1  # fallback — Алматы

def fetch_vacancies(keyword, city):
    area_id = get_area_id(city)
    url = f"https://api.hh.ru/vacancies?text={keyword}&area={area_id}"
    response = requests.get(url)
    data = response.json()
    vacancies = []
    for item in data["items"][:5]:  # Ограничим 5 вакансиями
        title = item["name"]
        url = item["alternate_url"]
        vacancies.append(f"{title}\n{url}")
    return vacancies

# ▶️ Команда /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Привет! Напиши /help, чтобы узнать как пользоваться ботом.")

# ℹ️ Команда /help
async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = (
        "ℹ️ Как пользоваться ботом:\n\n"
        "✏️ Напиши команду в формате:\n"
        "`/find вакансия, страна, город`\n\n"
        "📌 Примеры:\n"
        "`/find Pentester, Казахстан, Астана`\n"
        "`/find Python, Казахстан, Алматы`\n\n"
        "Пропиши /stop бот автоматически остановиться."
        "Бот найдёт актуальные вакансии с HeadHunter по твоим параметрам."
    )
    await update.message.reply_text(msg, parse_mode='Markdown')

async def find(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if context.args:
        raw_input = ' '.join(context.args)
        parts = [p.strip() for p in raw_input.split(",")]

        if len(parts) < 2:
            await update.message.reply_text("Пожалуйста, укажи хотя бы вакансию и город. Пример: /find Python, Алматы")
            return

        keyword = parts[0]
        city = parts[-1]

        vacancies = fetch_vacancies(keyword, city)
        if vacancies:
            message = "\n\n".join(vacancies)
        else:
            message = "Ничего не найдено 😞"

        await update.message.reply_text(message)
    else:
        await update.message.reply_text("Формат: /find вакансия, страна, город")

async def stop(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("⛔️ Бот останавливается...")
    await context.application.shutdown()
    sys.exit()

# 🚀 Запуск бота
app = ApplicationBuilder().token("7564730542:AAGU7ME3sF1kvRmsAY6BCjYV3FbhVg2110M").build()
app.add_handler(CommandHandler("start", start))
app.add_handler(CommandHandler("help", help_command))
app.add_handler(CommandHandler("find", find))
app.add_handler(CommandHandler("stop", stop))

print("✅ Бот запущен и ждёт команды...")
app.run_polling()

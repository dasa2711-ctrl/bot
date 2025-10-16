import os
import re
import random
import logging
from datetime import time, datetime
from typing import Optional

from telegram import Update
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    ConversationHandler,
    PicklePersistence,
    ContextTypes,
    filters,
)

# ====== Настройки по умолчанию ======
DEFAULT_TIMEZONE = os.getenv("TIMEZONE", "Europe/Moscow")

DEFAULT_FEED_TIME = os.getenv("FEED_TIME", "09:00")  # формат HH:MM
DEFAULT_WALK_TIME = os.getenv("WALK_TIME", "19:00")  # формат HH:MM

FUNNY_HOURS_START = 10
FUNNY_HOURS_END = 18
FUNNY_EVERY_MINUTES = 60     # запуск проверки раз в N минут
FUNNY_SEND_PROBABILITY = 0.45  # вероятность отправки «мысли» каждый запуск

PERSISTENCE_FILE = os.getenv("PERSISTENCE_FILE", "petbot.pickle")

ASK_PET_NAME = 1

# ====== Логирование ======
logging.basicConfig(
    format="%(asctime)s %(levelname)s:%(name)s: %(message)s", level=logging.INFO
)
logger = logging.getLogger("PetBot")

# ====== Таймзона ======
try:
    # Python 3.9+: стандартная библиотека
    from zoneinfo import ZoneInfo
    TZ = ZoneInfo(DEFAULT_TIMEZONE)
except Exception:
    # Фолбэк на pytz, если нужно
    import pytz
    TZ = pytz.timezone(DEFAULT_TIMEZONE)

# ====== Утилиты времени ======
TIME_RE = re.compile(r"^(2[0-3]|[01]?\d):([0-5]\d)$")

def parse_hhmm(hhmm: str) -> Optional[time]:
    m = TIME_RE.match(hhmm.strip())
    if not m:
        return None
    hh, mm = map(int, m.groups())
    # Важно: указываем tzinfo для корректного ежедневного расписания
    return time(hour=hh, minute=mm, tzinfo=TZ)

def now_tz() -> datetime:
    return datetime.now(TZ)

# ====== Фразы ======
PET_REPLIES = [
    "Это {pet} 🐾. Я всё понял: «{msg}». Кстати, скучаю 🥺",
    "{pet} на связи! Ты сказал(а): «{msg}». Я тут сижу и смотрю в окно…",
    "Гав-мяу! Это {pet}. Заметил твоё «{msg}». Когда ты вернёшься? 👀",
    "Эй, я {pet}. Принял(а): «{msg}». Я был(а) молодцом сегодня? 😇",
    "{pet} докладывает: «{msg}» принято. Миска кажется подозрительно пустой…",
]

FUNNY_THOUGHTS = [
    "{pet}: Я только что победил(а) шуршащий пакет. Мир теперь в безопасности.",
    "{pet}: Солнце переехало на 3 см. Пойду лягу туда снова.",
    "{pet}: Пытался(лась) дружить с пылью под диваном. Она немногословна.",
    "{pet}: Если миска пустая, это же не моя вина, правда? 🤔",
    "{pet}: Соседи опять что‑то уронили. Я им мысленно гавкнул(а)!",
    "{pet}: Я придумал(а) план побега… Нужны ласки и корм. Много корма.",
    "{pet}: Я охраняю твою квартиру от злобных пакетов. Не благодари.",
]

FEED_TEMPLATES = [
    "{pet}: Время подкрепиться! Миска ждёт 😋",
    "{pet}: Пора-пора покушать. Я был(а) очень хорошим(ей) сегодня!",
]

WALK_TEMPLATES = [
    "{pet}: Выгуляемся? Я уже с поводком у двери! 🐕‍🦺",
    "{pet}: Пора на прогулку! Обещаю вести себя прилично (наверное) 😅",
]

# ====== Помощники по джобам ======
def job_name(kind: str, chat_id: int) -> str:
    return f"{kind}-{chat_id}"

def cancel_jobs_for_chat(context: ContextTypes.DEFAULT_TYPE, chat_id: int):
    jq = context.application.job_queue
    for kind in ("feed", "walk", "funny"):
        name = job_name(kind, chat_id)
        for j in jq.get_jobs_by_name(name):
            j.schedule_removal()

def schedule_jobs_for_chat(context: ContextTypes.DEFAULT_TYPE, chat_id: int, pet_name: str, feed_time: time, walk_time: time):
    jq = context.application.job_queue

    # Сначала отменим предыдущие
    cancel_jobs_for_chat(context, chat_id)

    # Кормление — каждый день
    jq.run_daily(
        callback=send_feed_reminder,
        time=feed_time,
        name=job_name("feed", chat_id),
        chat_id=chat_id,
        data={"pet": pet_name},
    )

    # Прогулка — каждый день
    jq.run_daily(
        callback=send_walk_reminder,
        time=walk_time,
        name=job_name("walk", chat_id),
        chat_id=chat_id,
        data={"pet": pet_name},
    )

    # Забавные комментарии — каждые N минут, с вероятностью
    jq.run_repeating(
        callback=send_funny_comment,
        interval=FUNNY_EVERY_MINUTES * 60,
        first=60,  # старт через минуту после запуска
        name=job_name("funny", chat_id),
        chat_id=chat_id,
        data={"pet": pet_name},
    )
    logger.info("Jobs scheduled for chat %s: feed %s, walk %s", chat_id, feed_time, walk_time)

# ====== Команды ======
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    chat_data = context.chat_data

    # Если уже знаем имя — просто приветствуем
    if "pet_name" in chat_data:
        pet = chat_data["pet_name"]
        await update.message.reply_text(
            f"Эй! Это {pet} 🐾. Я уже тут. Напиши мне что‑нибудь.\n"
            f"Мои напоминания: кормление в {chat_data.get('feed_time', DEFAULT_FEED_TIME)}, прогулка в {chat_data.get('walk_time', DEFAULT_WALK_TIME)}.\n"
            f"Сменить времена: /set_times HH:MM HH:MM\nПосмотреть настройки: /my_settings"
        )
        return ConversationHandler.END

    await update.message.reply_text("Привет! Как зовут твоего питомца? Напиши имя:")
    return ASK_PET_NAME

async def set_pet_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    pet = (update.message.text or "").strip()
    if not pet:
        await update.message.reply_text("Имя не распознано. Введите, пожалуйста, имя питомца:")
        return ASK_PET_NAME

    chat_id = update.effective_chat.id
    context.chat_data["pet_name"] = pet

    # Установим времена (из чата или из дефолтов)
    feed_hhmm = context.chat_data.get("feed_time", DEFAULT_FEED_TIME)
    walk_hhmm = context.chat_data.get("walk_time", DEFAULT_WALK_TIME)
    feed_t = parse_hhmm(feed_hhmm) or parse_hhmm(DEFAULT_FEED_TIME)
    walk_t = parse_hhmm(walk_hhmm) or parse_hhmm(DEFAULT_WALK_TIME)

    schedule_jobs_for_chat(context, chat_id, pet, feed_t, walk_t)

    await update.message.reply_text(
        f"Отлично! Теперь я {pet} и буду писать от его имени.\n"
        f"Напоминания: кормление в {feed_hhmm}, прогулка в {walk_hhmm}.\n"
        f"Изменить времена: /set_times HH:MM HH:MM\nПосмотреть настройки: /my_settings"
    )
    return ConversationHandler.END

async def set_times(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    if "pet_name" not in context.chat_data:
        await update.message.reply_text("Сначала запусти /start и укажи имя питомца.")
        return

    args = (update.message.text or "").split()
    if len(args) != 3:
        await update.message.reply_text("Использование: /set_times HH:MM HH:MM\nНапример: /set_times 08:30 20:15")
        return

    feed_s, walk_s = args[1], args[2]
    feed_t, walk_t = parse_hhmm(feed_s), parse_hhmm(walk_s)
    if not feed_t or not walk_t:
        await update.message.reply_text("Ошибочный формат времени. Используй HH:MM, например 09:00 19:30.")
        return

    context.chat_data["feed_time"] = feed_s
    context.chat_data["walk_time"] = walk_s

    pet = context.chat_data["pet_name"]
    schedule_jobs_for_chat(context, chat_id, pet, feed_t, walk_t)

    await update.message.reply_text(f"Готово! Теперь кормление в {feed_s}, прогулка в {walk_s}.")

async def my_settings(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    pet = context.chat_data.get("pet_name")
    if not pet:
        await update.message.reply_text("Сначала /start — введи имя питомца.")
        return

    feed_s = context.chat_data.get("feed_time", DEFAULT_FEED_TIME)
    walk_s = context.chat_data.get("walk_time", DEFAULT_WALK_TIME)

    await update.message.reply_text(
        f"Мои настройки:\n"
        f"- Имя питомца: {pet}\n"
        f"- Кормление: {feed_s}\n"
        f"- Прогулка: {walk_s}\n"
        f"- Часовой пояс: {DEFAULT_TIMEZONE}"
    )

# ====== Диалог/Чат от лица питомца ======
async def pet_chat_reply(update: Update, context: ContextTypes.DEFAULT_TYPE):
    pet = context.chat_data.get("pet_name", "Твой питомец")
    text = (update.message.text or "").strip()
    template = random.choice(PET_REPLIES)
    reply = template.format(pet=pet, msg=text)
    await update.message.reply_text(reply)

# ====== Коллбеки джобов ======
async def send_feed_reminder(context: ContextTypes.DEFAULT_TYPE):
    job = context.job
    chat_id = job.chat_id
    data = job.data or {}
    pet = data.get("pet", "Питомец")
    msg = random.choice(FEED_TEMPLATES).format(pet=pet)
    await context.bot.send_message(chat_id=chat_id, text=msg)

async def send_walk_reminder(context: ContextTypes.DEFAULT_TYPE):
    job = context.job
    chat_id = job.chat_id
    data = job.data or {}
    pet = data.get("pet", "Питомец")
    msg = random.choice(WALK_TEMPLATES).format(pet=pet)
    await context.bot.send_message(chat_id=chat_id, text=msg)

async def send_funny_comment(context: ContextTypes.DEFAULT_TYPE):
    job = context.job
    chat_id = job.chat_id
    data = job.data or {}
    pet = data.get("pet", "Питомец")

    now = now_tz()
    # Только в рабочие часы, чтобы не будить по ночам
    if FUNNY_HOURS_START <= now.hour < FUNNY_HOURS_END:
        if random.random() < FUNNY_SEND_PROBABILITY:
            msg = random.choice(FUNNY_THOUGHTS).format(pet=pet)
            await context.bot.send_message(chat_id=chat_id, text=msg)

# ====== Инициализация приложения и восстановление расписания ======
async def post_init(app):
    # Восстанавливаем джобы для всех чатов, где задано имя
    for chat_id, chat_data in app.chat_data.items():
        pet = chat_data.get("pet_name")
        if not pet:
            continue
        feed_s = chat_data.get("feed_time", DEFAULT_FEED_TIME)
        walk_s = chat_data.get("walk_time", DEFAULT_WALK_TIME)
        feed_t = parse_hhmm(feed_s) or parse_hhmm(DEFAULT_FEED_TIME)
        walk_t = parse_hhmm(walk_s) or parse_hhmm(DEFAULT_WALK_TIME)

        # Нужен временный контекст-объект, но у нас есть app -> обойдемся прямым доступом
        # Сымитируем context через объект с атрибутом application
        class _Tmp:
            application = app
        schedule_jobs_for_chat(_Tmp(), chat_id, pet, feed_t, walk_t)
    logger.info("Post-init scheduling complete")

def main():
    token = os.getenv("TELEGRAM_BOT_TOKEN")
    if not token:
        raise RuntimeError("Не задан TELEGRAM_BOT_TOKEN")

    persistence = PicklePersistence(filepath=PERSISTENCE_FILE)

    app = (
        ApplicationBuilder()
        .token(token)
        .persistence(persistence)
        .build()
    )

    # Диалог для имени питомца
    conv = ConversationHandler(
        entry_points=[CommandHandler("start", start)],
        states={
            ASK_PET_NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, set_pet_name)]
        },
        fallbacks=[],
        name="pet_name_conv",
        persistent=True,
    )

    app.add_handler(conv)
    app.add_handler(CommandHandler("set_times", set_times))
    app.add_handler(CommandHandler("my_settings", my_settings))

    # Ответы от лица питомца на любые текстовые сообщения (после команд)
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, pet_chat_reply))

    # Восстановим расписание после запуска
    app.post_init = post_init

    logger.info("Starting bot...")
    app.run_polling(close_loop=False)

if __name__ == "__main__":
    main()

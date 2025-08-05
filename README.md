import time
import threading
from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    filters,
    ContextTypes,
)
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options

# ========== CONFIG ==========
TELEGRAM_TOKEN = "8261249645:AAHKZU8xNzP3V5FmAQOO9CBkDIukjVnstgw"
YOUR_TELEGRAM_ID = 8103868210
IVASMS_EMAIL = "tmoneydavid2@gmail.com"
IVASMS_PASSWORD = "Davidmicheal21"
IVASMS_URL = "https://www.ivasms.com/login"
# ============================

# Global variables
driver = None
acquired_numbers = []
monitoring = False


def start_browser():
    global driver
    chrome_options = Options()
    chrome_options.add_argument("--headless")
    chrome_options.add_argument("--disable-gpu")
    chrome_options.add_argument("--no-sandbox")
    driver = webdriver.Chrome(options=chrome_options)


def login_ivasms():
    driver.get(IVASMS_URL)
    time.sleep(3)
    driver.find_element(By.NAME, "email").send_keys(IVASMS_EMAIL)
    driver.find_element(By.NAME, "password").send_keys(IVASMS_PASSWORD)
    driver.find_element(By.XPATH, "//button[contains(text(), 'Login')]").click()
    time.sleep(5)


def get_available_numbers():
    driver.get("https://www.ivasms.com/test-number")  # Adjust if needed
    time.sleep(3)
    numbers = []
    rows = driver.find_elements(By.XPATH, "//table//tr")
    for row in rows[1:]:
        cols = row.find_elements(By.TAG_NAME, "td")
        if cols:
            country = cols[0].text.strip()
            number = cols[1].text.strip()
            numbers.append(f"{country} {number}")
    return numbers


def acquire_all_numbers():
    driver.get("https://www.ivasms.com/test-number")  # Adjust if needed
    time.sleep(3)
    buttons = driver.find_elements(By.XPATH, "//button[contains(text(), 'Acquire')]")
    for btn in buttons:
        try:
            btn.click()
            time.sleep(1)
        except:
            continue


def check_for_otps(application):
    global acquired_numbers
    while True:
        try:
            driver.get("https://www.ivasms.com/client-active-sms")  # or SMS statistics
            time.sleep(3)
            messages = driver.find_elements(By.XPATH, "//table//tr")
            for msg in messages[1:]:
                cols = msg.find_elements(By.TAG_NAME, "td")
                if cols and "OTP" in cols[2].text:
                    number = cols[0].text.strip()
                    content = cols[2].text.strip()
                    application.bot.send_message(
                        chat_id=YOUR_TELEGRAM_ID,
                        text=f"📩 OTP for {number}: {content}",
                    )
        except Exception as e:
            print("Error checking OTPs:", e)
        time.sleep(60)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("🔄 Logging into iVASMS...")
    start_browser()
    login_ivasms()
    numbers = get_available_numbers()
    if not numbers:
        await update.message.reply_text("❌ No numbers available.")
        return
    context.user_data["numbers"] = numbers
    keyboard = [["yes", "no"]]
    reply_markup = ReplyKeyboardMarkup(keyboard, one_time_keyboard=True)
    await update.message.reply_text(
        f"Hey Teez Plug Bot is active. Do you want to acquire these numbers?\n\n{', '.join(numbers)}",
        reply_markup=reply_markup,
    )


async def message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global monitoring
    text = update.message.text.lower()
    if text == "yes":
        acquire_all_numbers()
        acquired_numbers.extend(context.user_data.get("numbers", []))
        await update.message.reply_text("✅ Numbers acquired. Monitoring for OTPs...")
        if not monitoring:
            monitoring = True
            threading.Thread(target=check_for_otps, args=(context.application,), daemon=True).start()
    elif text == "no":
        await update.message.reply_text("❌ Okay, cancelled.")
    else:
        await update.message.reply_text("❓ Please reply 'yes' or 'no'.")


async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "/start - Start the bot\n"
        "/help - Show this help message"
    )


if __name__ == "__main__":
    app = ApplicationBuilder().token(TELEGRAM_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), message_handler))

    print("🤖 Bot is running...")
    app.run_polling()

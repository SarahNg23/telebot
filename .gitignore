import os
import requests
import asyncio
from dotenv import load_dotenv
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    filters,
    ContextTypes,
)
from openai import OpenAI  # Sử dụng client mới

# Fix lỗi event loop cho Jupyter Notebook
try:
    import nest_asyncio
    nest_asyncio.apply()
except ImportError:
    pass

# Load environment variables
load_dotenv("bot.env")

# Configuration
TELEGRAM_TOKEN = os.getenv('TELEGRAM_TOKEN')
CMC_API_KEY = os.getenv('COINMARKETCAP_API_KEY')
OPENAI_API_KEY = os.getenv('OPENAI_API_KEY')

# Khởi tạo OpenAI client
client = OpenAI(api_key=OPENAI_API_KEY)

# Endpoints
CMC_TOP10_URL = "https://pro-api.coinmarketcap.com/v1/cryptocurrency/listings/latest"
CMC_SINGLE_URL = "https://pro-api.coinmarketcap.com/v2/cryptocurrency/quotes/latest"

# Command: /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("Top 10 Cryptos", callback_data='top10')],
        [InlineKeyboardButton("BTC Analysis", callback_data='btc')],
        [InlineKeyboardButton("ETH Analysis", callback_data='eth')],
        [InlineKeyboardButton("Custom Analysis", callback_data='custom')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await update.message.reply_text(
        "Chọn loại phân tích bạn muốn:",
        reply_markup=reply_markup
    )

# Handle button clicks
async def handle_button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    if query.data == 'top10':
        analysis = await get_top10()
    elif query.data == 'btc':
        analysis = await get_analysis('BTC')
    elif query.data == 'eth':
        analysis = await get_analysis('ETH')
    elif query.data == 'custom':
        analysis = "Ghi mã hoặc tên crypto bạn muốn phân tích (ví dụ: BTC, ETH, SOL)."
        await query.edit_message_text(text=analysis)
        return
    
    await query.edit_message_text(text=analysis)

# Fetch top 10 cryptocurrencies
async def get_top10():
    parameters = {
        "start": "1",
        "limit": "10",
        "convert": "USD"
    }
    headers = {
        "Accepts": "application/json",
        "X-CMC_PRO_API_KEY": CMC_API_KEY
    }

    try:
        response = requests.get(CMC_TOP10_URL, headers=headers, params=parameters)
        data = response.json()
        
        if 'data' not in data:
            return "❌ Lỗi: Không tìm thấy dữ liệu từ API."
        
        analysis = "📊 Top 10 Cryptocurrencies:\n\n"
        for crypto in data['data']:
            change_24h = crypto['quote']['USD']['percent_change_24h']
            change_emoji = "🟢" if change_24h >= 0 else "🔴"
            
            analysis += (
                f"{crypto['symbol']}: ${crypto['quote']['USD']['price']:,.2f}\n"
                f"  24h Change: {change_emoji} {change_24h:.2f}%\n"
                f"  Market Cap: ${crypto['quote']['USD']['market_cap']:,.0f}\n"
                f"  Volume 24h: ${crypto['quote']['USD']['volume_24h']:,.0f}\n\n"
            )
        return analysis
    
    except Exception as e:
        return f"❌ Lỗi khi lấy dữ liệu: {str(e)}"

async def get_analysis(symbol: str):
    parameters = {
        "symbol": symbol,
        "convert": "USD"
    }
    headers = {
        "Accepts": "application/json",
        "X-CMC_PRO_API_KEY": CMC_API_KEY
    }

    try:
        response = requests.get(CMC_SINGLE_URL, headers=headers, params=parameters)
        data = response.json()

        if "data" not in data or symbol not in data["data"]:
            return f"❌ Không tìm thấy dữ liệu cho {symbol}."

        crypto = data["data"][symbol][0]
        price = crypto['quote']['USD']['price']
        change_24h = crypto['quote']['USD']['percent_change_24h']
        market_cap = crypto['quote']['USD']['market_cap']
        volume_24h = crypto['quote']['USD']['volume_24h']

        # Phân tích bằng ChatGPT
        prompt = (
            f"Phân tích crypto {symbol} với các thông tin sau:\n"
            f"- Giá hiện tại: ${price:,.2f}\n"
            f"- Thay đổi 24h: {change_24h:.2f}%\n"
            f"- Vốn hóa thị trường: ${market_cap:,.0f}\n"
            f"- Khối lượng 24h: ${volume_24h:,.0f}\n\n"
            f"Đưa ra phân tích kỹ thuật và dự đoán ngắn hạn."
        )

        # Sử dụng client mới
        chatgpt_response = client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": prompt}]
        )

        analysis = chatgpt_response.choices[0].message.content
        return f"📈 Phân tích {symbol}:\n\n{analysis}"

    except Exception as e:
        return f"❌ Lỗi khi lấy dữ liệu: {str(e)}"

# Handle custom crypto analysis
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    symbol = update.message.text.upper()
    analysis = await get_analysis(symbol)
    await update.message.reply_text(analysis)

def main():
    application = Application.builder().token(TELEGRAM_TOKEN).build()
    
    application.add_handler(CommandHandler('start', start))
    application.add_handler(CallbackQueryHandler(handle_button))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    
    print("Bot is running...")
    application.run_polling()

if __name__ == '__main__':
    main()

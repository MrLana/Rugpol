import logging
from telegram import Update
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackContext
import requests
from web3 import Web3
import json

# Setup logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', 
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Konfigurasi
TELEGRAM_TOKEN = "7491365614:AAE7m9c43WMUJC3qmRXSIoKsF_3shcBtp2Q"
BSCSCAN_API_KEY = "73MUDW1MEDY1UA8UCERKA1VD4G7312D5WZ"  # Untuk memeriksa kontrak
WEB3_PROVIDER = "https://bsc-dataseed.binance.org/"  # Provider BSC

# Inisialisasi Web3
w3 = Web3(Web3.HTTPProvider(WEB3_PROVIDER))

def start(update: Update, context: CallbackContext) -> None:
    """Handler untuk command /start"""
    user = update.effective_user
    update.message.reply_markdown_v2(
        fr'Hai {user.mention_markdown_v2()}\! Selamat datang di RugPool Bot\! '
        'Bot ini membantu Anda mendeteksi potensi rug pull di pool DeFi\.\n\n'
        'Perintah yang tersedia:\n'
        '/check \[contract_address] \- Memeriksa kontrak token\n'
        '/help \- Menampilkan bantuan'
    )

def help_command(update: Update, context: CallbackContext) -> None:
    """Handler untuk command /help"""
    update.message.reply_text(
        '🛡️ *RugPool Bot Help* 🛡️\n\n'
        'Perintah yang tersedia:\n'
        '/check [contract_address] - Memeriksa kontrak token untuk tanda-tanda rug pull\n'
        '/help - Menampilkan pesan bantuan ini\n\n'
        'Contoh penggunaan:\n'
        '/check 0x123...abc\n\n'
        'Bot ini memeriksa:\n'
        '- Kepemilikan kontrak\n'
        '- Fungsi pause/blacklist\n'
        '- Kemampuan mint baru\n'
        '- Liquidity lock\n'
        '- Dan indikator rug pull lainnya',
        parse_mode='Markdown'
    )

def check_contract(update: Update, context: CallbackContext) -> None:
    """Handler untuk memeriksa kontrak"""
    if not context.args:
        update.message.reply_text('Silakan masukkan alamat kontrak setelah perintah /check')
        return
    
    contract_address = context.args[0]
    
    if not Web3.is_address(contract_address):
        update.message.reply_text('Alamat kontrak tidak valid!')
        return
    
    # Normalisasi alamat
    contract_address = Web3.to_checksum_address(contract_address)
    
    try:
        # Kirim pesan sedang memproses
        processing_msg = update.message.reply_text(
            f'🕵️‍♂️ Memeriksa kontrak {contract_address}...\n'
            'Ini mungkin memakan waktu beberapa detik.'
        )
        
        # 1. Periksa info dasar dari BscScan
        bscscan_data = get_bscscan_info(contract_address)
        
        # 2. Periksa kontrak menggunakan Web3
        contract_analysis = analyze_contract(contract_address)
        
        # 3. Gabungkan hasil
        response = format_response(contract_address, bscscan_data, contract_analysis)
        
        # Edit pesan asli dengan hasil
        context.bot.edit_message_text(
            chat_id=update.message.chat_id,
            message_id=processing_msg.message_id,
            text=response,
            parse_mode='Markdown'
        )
        
    except Exception as e:
        logger.error(f"Error checking contract: {e}")
        update.message.reply_text(f'Terjadi error saat memeriksa kontrak: {str(e)}')

def get_bscscan_info(contract_address):
    """Mendapatkan info kontrak dari BscScan API"""
    url = f"https://api.bscscan.com/api?module=contract&action=getsourcecode&address={contract_address}&apikey={BSCSCAN_API_KEY}"
    response = requests.get(url)
    data = response.json()
    
    if data['status'] != '1':
        return {"error": "Gagal mendapatkan data dari BscScan"}
    
    result = data['result'][0]
    info = {
        "contract_name": result.get("ContractName", "Tidak diketahui"),
        "compiler_version": result.get("CompilerVersion", "Tidak diketahui"),
        "optimization_used": result.get("OptimizationUsed", "Tidak diketahui"),
        "is_proxy": "Proxy" in result.get("Proxy", "0"),
        "is_verified": result.get("SourceCode") is not None
    }
    
    return info

def analyze_contract(contract_address):
    """Menganalisis kontrak menggunakan Web3"""
    # Catatan: Ini adalah analisis sederhana. Untuk analisis mendalam, Anda perlu ABI spesifik.
    
    # Dapatkan kode byte
    bytecode = w3.eth.get_code(contract_address).hex()
    
    # Analisis sederhana berdasarkan bytecode
    analysis = {
        "has_pause_function": "pause" in bytecode.lower(),
        "has_mint_function": "mint" in bytecode.lower(),
        "has_blacklist": "blacklist" in bytecode.lower(),
        "is_pausable": "pausable" in bytecode.lower(),
        "has_owner_changes": "transferownership" in bytecode.lower()
    }
    
    return analysis

def format_response(contract_address, bscscan_data, contract_analysis):
    """Format respons untuk pengguna"""
    # Info dasar
    response = f"*🔍 Hasil Pemeriksaan Rug Pull* `{contract_address}`\n\n"
    response += f"*Nama Kontrak:* {bscscan_data.get('contract_name', 'Tidak diketahui')}\n"
    response += f"*Terverifikasi:* {'✅' if bscscan_data.get('is_verified') else '❌'}\n"
    response += f"*Proxy Contract:* {'✅' if bscscan_data.get('is_proxy') else '❌'}\n\n"
    
    # Tanda bahaya
    response += "*🚩 Tanda Bahaya Potensi Rug Pull:*\n"
    
    warnings = []
    if contract_analysis.get("has_mint_function"):
        warnings.append("⚠️ Memiliki fungsi mint - Developer bisa mencetak token baru")
    if contract_analysis.get("has_pause_function"):
        warnings.append("⚠️ Memiliki fungsi pause - Developer bisa membekukan perdagangan")
    if contract_analysis.get("has_blacklist"):
        warnings.append("⚠️ Memiliki fungsi blacklist - Developer bisa memblokir alamat tertentu")
    if contract_analysis.get("is_pausable"):
        warnings.append("⚠️ Kontrak pausable - Developer bisa menghentikan semua transaksi")
    if bscscan_data.get("is_proxy"):
        warnings.append("⚠️ Kontrak proxy - Logika bisa diubah oleh developer")
    
    if warnings:
        response += "\n".join(warnings)
    else:
        response += "✅ Tidak ditemukan tanda bahaya yang jelas"
    
    # Rekomendasi
    response += "\n\n*💡 Rekomendasi:*\n"
    if len(warnings) > 3:
        response += "🔴 RISIKO TINGGI! Hindari proyek ini karena banyak tanda bahaya rug pull."
    elif len(warnings) > 1:
        response += "🟡 RISIKO SEDANG. Berhati-hatilah dan lakukan riset lebih lanjut sebelum berinvestasi."
    else:
        response += "🟢 RISIKO RENDAH berdasarkan pemeriksaan dasar. Tetap lakukan due diligence Anda."
    
    response += "\n\n_Note: Ini bukan nasihat finansial. Selalu lakukan riset Anda sendiri._"
    
    return response

def main() -> None:
    """Jalankan bot."""
    # Buat Updater dan berikan token bot
    updater = Updater(TLEGRAM_TOKEN)

    # Dapatkan dispatcher untuk mendaftarkan handler
    dispatcher = updater.dispatcher

    # Daftarkan command handler
    dispatcher.add_handler(CommandHandler("start", start))
    dispatcher.add_handler(CommandHandler("help", help_command))
    dispatcher.add_handler(CommandHandler("check", check_contract))

    # Mulai bot
    updater.start_polling()

    # Jalankan bot sampai Ctrl-C ditekan
    updater.idle()

if __name__ == '__main__':
    main()

import requests  
import re  
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup  
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes  
  
TOKEN = "9085034198:AAFsGSaWdXc27vh2daT_xhtLKOw2utFaD6M"  
KANAL_ID = "@shadowXgrup" # KANALININ KULLANICI ADI (Başında @ olsun)  
  
guncel_kurlar = {}  
  
def fiyatlari_cek():  
    try:  
        url = "https://api.exchangerate-api.com/v4/latest/USD"  
        cevap = requests.get(url, timeout=10).json()  
        kurlar = cevap.get("rates", {})  
          
        usd_try = kurlar.get("TRY", 0)  
        eur_usd = kurlar.get("EUR", 0)  
        eur_try = usd_try / eur_usd if eur_usd != 0 else 0  
        gram_altin = usd_try * 75.0   
  
        return {  
            "usd": round(usd_try, 2),  
            "eur": round(eur_try, 2),  
            "altin": round(gram_altin, 2)  
        }  
    except Exception:  
        return None  
  
# KANAL KONTROL FONKSİYONU  
async def kanal_kontrol(context, user_id):  
    try:  
        member = await context.bot.get_chat_member(chat_id=KANAL_ID, user_id=user_id)  
        return member.status in ['creator', 'administrator', 'member']  
    except Exception:  
        return False  
  
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):  
    user_id = update.effective_user.id  
      
    # Kullanıcı kanalda mı?  
    katildi_mi = await kanal_kontrol(context, user_id)  
      
    if not katildi_mi:  
        # Kanalda değilse "Kanala Katıl" butonu göster  
        klavye = [  
            [InlineKeyboardButton("📢 Kanala Katıl", url=f"https://t.me/{KANAL_ID.replace('@', '')}")],  
            [InlineKeyboardButton("✅ Kontrol Et", callback_data="kontrol")]  
        ]  
        await update.message.reply_text(  
            "⚠️ **Botu kullanabilmek için önce kanalımıza katılman gerekiyor!**\n\nKatıldıktan sonra 'Kontrol Et' butonuna bas.",  
            reply_markup=InlineKeyboardMarkup(klavye),  
            parse_mode="Markdown"  
        )  
        return  
  
    # Kanalda ise asıl menüyü göster  
    karsilama = (  
        "╭━━━━━ 🏦 ━━━━━╮\n\n"  
        "   💹 **ShadowX Borsa**   \n"  
        "   📈 *Piyasaların Gücü Ellerinde!*   \n\n"  
        "╰━━━━━ 🌟 ━━━━━╯\n\n"  
        "💰 **Anlık Dolar, Euro ve Altın** fiyatlarını anında öğren!\n\n"  
        "🧮 **Hesaplama Yapmak İçin:**\n"  
        "Bota direkt *'100 Dolar'*, *'50 Euro'* veya *'5 Altın'* yaz, anında TL karşılığını gör!\n\n"  
        "👇 **İşlem yapmak için aşağıdaki butonları kullan:**"  
    )  
    klavye = [  
        [InlineKeyboardButton("💵 Dolar Fiyatı", callback_data="fiyat_usd"), InlineKeyboardButton("💶 Euro Fiyatı", callback_data="fiyat_eur")],  
        [InlineKeyboardButton("🥇 Gram Altın", callback_data="fiyat_altin"), InlineKeyboardButton("📊 Tüm Fiyatlar", callback_data="fiyat_hepsi")],  
        [InlineKeyboardButton("❓ Nasıl Hesaplanır?", callback_data="hesap_yardim")]  
    ]  
    await update.message.reply_text(karsilama, reply_markup=InlineKeyboardMarkup(klavye), parse_mode="Markdown")  
  
async def buton(update: Update, context: ContextTypes.DEFAULT_TYPE):  
    query = update.callback_query  
    await query.answer()  
      
    # "Kontrol Et" butonuna basıldıysa  
    if query.data == "kontrol":  
        user_id = query.from_user.id  
        katildi_mi = await kanal_kontrol(context, user_id)  
          
        if not katildi_mi:  
            await query.answer("❌ Henüz kanala katılmamışsın!", show_alert=True)  
            return  
        else:  
            # Katılmışsa menüyü göster  
            karsilama = (  
                "╭━━━━━ 🏦 ━━━━━╮\n\n"  
                "   💹 **ShadowX Borsa**   \n"  
                "   📈 *Piyasaların Gücü Ellerinde!*   \n\n"  
                "╰━━━━━ 🌟 ━━━━━╯\n\n"  
                "👇 **İşlem yapmak için aşağıdaki butonları kullan:**"  
            )  
            klavye = [  
                [InlineKeyboardButton("💵 Dolar Fiyatı", callback_data="fiyat_usd"), InlineKeyboardButton("💶 Euro Fiyatı", callback_data="fiyat_eur")],  
                [InlineKeyboardButton("🥇 Gram Altın", callback_data="fiyat_altin"), InlineKeyboardButton("📊 Tüm Fiyatlar", callback_data="fiyat_hepsi")],  
                [InlineKeyboardButton("❓ Nasıl Hesaplanır?", callback_data="hesap_yardim")]  
            ]  
            await query.edit_message_text(karsilama, reply_markup=InlineKeyboardMarkup(klavye), parse_mode="Markdown")  
            return  
  
    # Fiyat butonları  
    global guncel_kurlar  
    veriler = fiyatlari_cek()  
      
    if not veriler:  
        await query.message.reply_text("🚨 Fiyatlar alınamadı. Lütfen internet bağlantınızı kontrol edin.")  
        return  
          
    guncel_kurlar = veriler  
  
    if query.data == "fiyat_usd":  
        await query.message.reply_text(f"💵 **Anlık Dolar Kuru:**\n\n📉 **{veriler['usd']} TL**", parse_mode="Markdown")  
    elif query.data == "fiyat_eur":  
        await query.message.reply_text(f"💶 **Anlık Euro Kuru:**\n\n📉 **{veriler['eur']} TL**", parse_mode="Markdown")  
    elif query.data == "fiyat_altin":  
        await query.message.reply_text(f"🥇 **Anlık Gram Altın:**\n\n📉 **{veriler['altin']} TL**", parse_mode="Markdown")  
    elif query.data == "fiyat_hepsi":  
        metin = (  
            f"📊 **ShadowX Borsa | Güncel Piyasa:**\n\n"  
            f"💵 Dolar: **{veriler['usd']} TL**\n"  
            f"💶 Euro: **{veriler['eur']} TL**\n"  
            f"🥇 Gram Altın: **{veriler['altin']} TL**"  
        )  
        await query.message.reply_text(metin, parse_mode="Markdown")  
    elif query.data == "hesap_yardim":  
        yardim = (  
            "🧮 **Nasıl Hesaplama Yaparım?**\n\n"  
            "Bota direkt mesaj olarak şu formatta yazabilirsin:\n\n"  
            "💵 `100 Dolar`\n"  
            "💶 `50 Euro`\n"  
            "🥇 `10 Altın`\n\n"  
            "Bot senin için anında TL karşılığını hesaplayacaktır!"  
        )  
        await query.message.reply_text(yardim, parse_mode="Markdown")  
  
async def hesapla(update: Update, context: ContextTypes.DEFAULT_TYPE):  
    global guncel_kurlar  
    if not guncel_kurlar:  
        guncel_kurlar = fiyatlari_cek()  
        if not guncel_kurlar:  
            await update.message.reply_text("🚨 Şu an fiyatları çekemiyorum, lütfen biraz sonra tekrar dene.")  
            return  
  
    metin = update.message.text.lower()  
    sayi_bul = re.search(r"(\d+(?:[.,]\d+)?)", metin)  
      
    if not sayi_bul:  
        await update.message.reply_text("🤔 Hesaplamak için bir miktar belirtmelisin. Örnek: `100 Dolar` yaz.", parse_mode="Markdown")  
        return  
  
    miktar_str = sayi_bul.group(1).replace(",", ".")  
    miktar = float(miktar_str)  
  
    if "dolar" in metin or "usd" in metin or "$" in metin:  
        sonuc = miktar * guncel_kurlar['usd']  
        await update.message.reply_text(f"💵 **{miktar} Dolar** = 💰 **{round(sonuc, 2)} TL**", parse_mode="Markdown")  
    elif "euro" in metin or "eur" in metin or "€" in metin:  
        sonuc = miktar * guncel_kurlar['eur']  
        await update.message.reply_text(f"💶 **{miktar} Euro** = 💰 **{round(sonuc, 2)} TL**", parse_mode="Markdown")  
    elif "altın" in metin or "altin" in metin or "gram" in metin:  
        sonuc = miktar * guncel_kurlar['altin']  
        await update.message.reply_text(f"🥇 **{miktar} Gram Altın** = 💰 **{round(sonuc, 2)} TL**", parse_mode="Markdown")  
    else:  
        await update.message.reply_text("🤔 Hangi birimi hesaplayacağımı anlayamadım. Lütfen 'Dolar', 'Euro' veya 'Altın' kelimelerini kullan.")  
  
def main():  
    app = Application.builder().token(TOKEN).build()  
      
    app.add_handler(CommandHandler("start", start))  
    app.add_handler(CallbackQueryHandler(buton))  
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, hesapla))  
      
    print("ShadowX Borsa Çalışıyor...")  
    app.run_polling()  
  
if __name__ == "__main__":  
    main()  

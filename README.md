import telebot
import json
import os
from telebot.types import ReplyKeyboardMarkup, KeyboardButton

TOKEN = "8388573022:AAFNnVzQroLFi99mv5aMuTo697zlzndeP18"
ADMIN_ID = 7843448826
CHANNEL_ID = "@ironman_mod"  # kanal username

bot = telebot.TeleBot(TOKEN)

DB_FILE = "data.json"

def load_data():
    if not os.path.exists(DB_FILE):
        return {"mods": [], "maps": [], "shaders": []}
    with open(DB_FILE, "r") as f:
        return json.load(f)

def save_data(data):
    with open(DB_FILE, "w") as f:
        json.dump(data, f, indent=4)

data = load_data()

def menu(user_id):
    markup = ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add("📦 Mods", "🗺 Maps", "🌈 Shaders")
    markup.add("🔍 Qidiruv")

    if user_id == ADMIN_ID:
        markup.add("⚙️ Admin panel")

    return markup

@bot.message_handler(commands=['start'])
def start(message):
    bot.send_message(message.chat.id, "Xush kelibsiz!", reply_markup=menu(message.from_user.id))

# CATEGORY
def show_category(message, category):
    for item in data[category]:
        text = f"📌 {item['name']}\n\n{item['desc']}"
        
        if "photo" in item:
            bot.send_photo(message.chat.id, item["photo"], caption=text)
        else:
            bot.send_message(message.chat.id, text)

        bot.send_document(message.chat.id, item['file_id'])

@bot.message_handler(func=lambda m: m.text == "📦 Mods")
def mods(message):
    show_category(message, "mods")

@bot.message_handler(func=lambda m: m.text == "🗺 Maps")
def maps(message):
    show_category(message, "maps")

@bot.message_handler(func=lambda m: m.text == "🌈 Shaders")
def shaders(message):
    show_category(message, "shaders")

# ADMIN PANEL
@bot.message_handler(func=lambda m: m.text == "⚙️ Admin panel")
def admin_panel(message):
    markup = ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add("➕ Qo‘shish", "❌ O‘chirish")
    markup.add("⬅️ Orqaga")
    bot.send_message(message.chat.id, "Admin panel", reply_markup=markup)

# ADD
user_state = {}

@bot.message_handler(func=lambda m: m.text == "➕ Qo‘shish")
def add_start(message):
    markup = ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add("Mods", "Maps", "Shaders")
    bot.send_message(message.chat.id, "Bo‘lim tanla:", reply_markup=markup)
    user_state[message.chat.id] = {}

@bot.message_handler(func=lambda m: m.text in ["Mods", "Maps", "Shaders"])
def choose_category(message):
    user_state[message.chat.id]["category"] = message.text.lower()
    bot.send_message(message.chat.id, "Nom:")
    bot.register_next_step_handler(message, get_name)

def get_name(message):
    user_state[message.chat.id]["name"] = message.text
    bot.send_message(message.chat.id, "Tavsif:")
    bot.register_next_step_handler(message, get_desc)

def get_desc(message):
    user_state[message.chat.id]["desc"] = message.text
    bot.send_message(message.chat.id, "Rasm yubor:")
    bot.register_next_step_handler(message, get_photo)

def get_photo(message):
    user_state[message.chat.id]["photo"] = message.photo[-1].file_id
    bot.send_message(message.chat.id, "File yubor:")
    bot.register_next_step_handler(message, get_file)

def get_file(message):
    user = user_state[message.chat.id]

    item = {
        "name": user["name"],
        "desc": user["desc"],
        "photo": user["photo"],
        "file_id": message.document.file_id
    }

    data[user["category"]].append(item)
    save_data(data)

    # AUTO POST CHANNEL
    text = f"🔥 {item['name']}\n\n{item['desc']}"
    bot.send_photo(CHANNEL_ID, item["photo"], caption=text)
    bot.send_document(CHANNEL_ID, item["file_id"])

    bot.send_message(message.chat.id, "✅ Qo‘shildi va kanalga tashlandi!", reply_markup=menu(message.from_user.id))

# DELETE
@bot.message_handler(func=lambda m: m.text == "❌ O‘chirish")
def delete_start(message):
    bot.send_message(message.chat.id, "O‘chirish uchun nom yoz:")
    bot.register_next_step_handler(message, delete_item)

def delete_item(message):
    name = message.text.lower()

    for category in data:
        for item in data[category]:
            if item["name"].lower() == name:
                data[category].remove(item)
                save_data(data)
                bot.send_message(message.chat.id, "❌ O‘chirildi")
                return

    bot.send_message(message.chat.id, "Topilmadi 😅")

# SEARCH
@bot.message_handler(func=lambda m: m.text == "🔍 Qidiruv")
def search_start(message):
    bot.send_message(message.chat.id, "Nimani qidiryapsan?")
    bot.register_next_step_handler(message, search)

def search(message):
    text = message.text.lower()
    found = False

    for category in data:
        for item in data[category]:
            if text in item["name"].lower():
                found = True
                bot.send_photo(message.chat.id, item["photo"], caption=item["name"]+"\n\n"+item["desc"])
                bot.send_document(message.chat.id, item["file_id"])

    if not found:
        bot.send_message(message.chat.id, "Hech narsa topilmadi 😅")

# BACK
@bot.message_handler(func=lambda m: m.text == "⬅️ Orqaga")
def back(message):
    bot.send_message(message.chat.id, "Menu", reply_markup=menu(message.from_user.id))

print("Bot ishlayapti...")
bot.polling()

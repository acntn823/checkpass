import os
try:
    import requests, MedoSigner, SignerPy, telebot, concurrent.futures
except:
    os.system("pip install requests pycryptodome MedoSigner SignerPy telebot") 
import requests, threading, random, time, secrets, binascii, os, uuid, telebot, json, concurrent.futures
from urllib.parse import urlencode 
from MedoSigner import Argus, Gorgon, md5, Ladon 
from SignerPy import sign, get 

spam = {}
Token = "8476003009:AAHe1IKeO7QzFcwM1AlspjHhtuCMCLyV0wg"
bot = telebot.TeleBot(Token)
ADMIN_ID = 1568917766 
CHANNEL_USERNAME = "@ntnnotification"  
SUBSCRIBE_REQUIRED = True

CHANNEL_CONFIG_FILE = "channel_config.json"
TOKENS_FILE = "user_tokens.json"
SUBSCRIPTION_FILE = "user_subscriptions.json"  
def load_channel_config():
    if os.path.exists(CHANNEL_CONFIG_FILE):
        try:
            with open(CHANNEL_CONFIG_FILE, 'r') as f:
                return json.load(f)
        except:
            return {"channel": CHANNEL_USERNAME}
    return {"channel": CHANNEL_USERNAME}

def save_channel_config(config):
    with open(CHANNEL_CONFIG_FILE, 'w') as f:
        json.dump(config, f)

def load_user_tokens():
    if os.path.exists(TOKENS_FILE):
        try:
            with open(TOKENS_FILE, 'r') as f:
                return json.load(f)
        except:
            return {}
    return {}

def save_user_tokens(tokens):
    with open(TOKENS_FILE, 'w') as f:
        json.dump(tokens, f)

def load_user_subscriptions():
    if os.path.exists(SUBSCRIPTION_FILE):
        try:
            with open(SUBSCRIPTION_FILE, 'r') as f:
                return json.load(f)
        except:
            return {}
    return {}

def save_user_subscriptions(subscriptions):
    with open(SUBSCRIPTION_FILE, 'w') as f:
        json.dump(subscriptions, f)

config = load_channel_config()
user_tokens = load_user_tokens()
user_subscriptions = load_user_subscriptions()  
ASIAN_DOMAINS = [
    "https://api16-normal-c-alisg.tiktokv.com", 
    "https://api16-normal-no1a.tiktokv.eu",
    "https://api19-normal-c-alisg.tiktokv.com",
    "https://api16-normal-c-useast1a.tiktokv.com",
    "https://api19-normal-c-useast1a.tiktokv.com",
    "https://api16-normal-c-useast2a.tiktokv.com",
    "https://api19-normal-c-useast2a.tiktokv.com",
    "https://api16-normal-useast5.us.tiktokv.com",
    "https://api16-core-c-useast1a.tiktokv.com",
    "https://api16-core-c-useast2a.tiktokv.com",
]

def check_time_subscription(user_id):
    user_id_str = str(user_id)
    if user_id_str in user_subscriptions:
        expiry_time = user_subscriptions[user_id_str]
        if time.time() < expiry_time:
            return True, None
        else:
            del user_subscriptions[user_id_str]
            save_user_subscriptions(user_subscriptions)
            return False, "⏰ Gói đăng ký của bạn đã hết hạn! Vui lòng gia hạn."
    return False, None

def find_working_host_parallel(domains, request_func, max_workers=5):
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        future_to_domain = {
            executor.submit(request_func, domain): domain 
            for domain in domains
        }
        
        for future in concurrent.futures.as_completed(future_to_domain):
            domain = future_to_domain[future]
            try:
                result = future.result()
                if result:
                    return result, domain
            except Exception as e:
                continue
    return None, None

def check_subscription(user_id):
    if not SUBSCRIBE_REQUIRED or CHANNEL_USERNAME is None:
        return True, None
    if user_id == ADMIN_ID:
        return True, None
    

    time_sub, time_msg = check_time_subscription(user_id)
    if time_sub:
        return True, None
    
    try:
        channel_username_clean = CHANNEL_USERNAME.replace('@', '')
        chat_member = bot.get_chat_member(f"@{channel_username_clean}", user_id)
        valid_statuses = ['member', 'administrator', 'creator']        
        if chat_member.status in valid_statuses:
            return True, None
        else:
            return False, "You have left the channel or were banned. Please join again."            
    except Exception as e:
        error_msg = str(e).lower()
        if "user not found" in error_msg or "user not participant" in error_msg:
            return False, "You are not subscribed to the channel."
        elif "chat not found" in error_msg:
            return False, f"The channel {CHANNEL_USERNAME} was not found. Please contact admin."
        elif "bot is not a member" in error_msg:
            return False, f"The bot is not a member of {CHANNEL_USERNAME}. Please add the bot as admin to the channel."
        else:
            print(f"Subscription check error: {e} - ntnbotforpasskey_laco.py:133")
            return False, "Error checking subscription. Please try again later."

def deduct_token(user_id):
    if user_id == ADMIN_ID:
        return True
    time_sub, _ = check_time_subscription(user_id)
    if time_sub:
        return True
    
    if str(user_id) not in user_tokens:
        return False
    
    if user_tokens[str(user_id)] <= 0:
        return False
    
    user_tokens[str(user_id)] -= 1
    save_user_tokens(user_tokens)
    return True

user_commands = [
    telebot.types.BotCommand("start", "Bắt đầu sử dụng bot"),
    telebot.types.BotCommand("check", "Kiểm tra tài khoản TikTok"),
    telebot.types.BotCommand("buytoken", "Mua token"),
    telebot.types.BotCommand("profile", "Profile"),
    telebot.types.BotCommand("mystatus", "Kiểm tra trạng thái gói đăng ký của tôi")
]
bot.set_my_commands(user_commands)
@bot.message_handler(commands=["removetime"])
def remove_time_subscription(message):
    if message.chat.type != 'private':
        bot.reply_to(message, "❌ Bot này chỉ hoạt động trong tin nhắn riêng tư!")
        return
        
    if message.from_user.id != ADMIN_ID:
        bot.reply_to(message, "❌ Bạn không được ủy quyền để sử dụng lệnh này!")
        return
        
    parts = message.text.split()
    if len(parts) < 3:
        bot.reply_to(message, "❌ Cách sử dụng: /removetime <user_id> <số_ngày>")
        return
        
    try:
        user_id = int(parts[1])
        days = int(parts[2])
    except:
        bot.reply_to(message, "❌ ID người dùng hoặc số ngày không hợp lệ!")
        return
    
    user_id_str = str(user_id)
    if user_id_str not in user_subscriptions:
        bot.reply_to(message, f"❌ Người dùng {user_id} không có đăng ký đang hoạt động!")
        return
    current_expiry = user_subscriptions[user_id_str]
    new_expiry = current_expiry - (days * 24 * 60 * 60)
    if new_expiry <= time.time():
        del user_subscriptions[user_id_str]
        save_user_subscriptions(user_subscriptions)
        bot.reply_to(message, f"✅ Đã xóa {days} ngày khỏi đăng ký của người dùng {user_id}. Đăng ký đã kết thúc.")
        
        try:
            bot.send_message(user_id, f"⏰ Thời gian đăng ký của bạn đã giảm {days} ngày. Đăng ký của bạn đã kết thúc.")
        except:
            pass
    else:
        user_subscriptions[user_id_str] = new_expiry
        save_user_subscriptions(user_subscriptions)
        
        new_expiry_date = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(new_expiry))
        bot.reply_to(message, f"✅ Đã xóa {days} ngày khỏi đăng ký của người dùng {user_id}. Đăng ký mới đến: {new_expiry_date}")
        
        try:
            bot.send_message(user_id, f"⏰ Thời gian đăng ký của bạn đã giảm {days} ngày. Đăng ký mới của bạn đến: {new_expiry_date}")
        except:
            pass
@bot.message_handler(commands=["mystatus"])
def my_status_command(message):
    if message.chat.type != 'private':
        bot.reply_to(message, "❌ Bot này chỉ hoạt động trong tin nhắn riêng tư!")
        return
        
    user_id = message.from_user.id
    time_sub, time_msg = check_time_subscription(user_id)
    
    if time_sub:
        expiry_time = user_subscriptions[str(user_id)]
        expiry_date = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(expiry_time))
        bot.reply_to(message, f"✅ Bạn có gói đăng ký đang hoạt động đến hết: {expiry_date}")
    else:
        tokens = user_tokens.get(str(user_id), 0)
        bot.reply_to(message, f"❌ Bạn không có gói đăng ký đang hoạt động. Số token khả dụng: {tokens}")

@bot.message_handler(commands=["profile"])
def profile_command(message):
    if message.chat.type != 'private':
        bot.reply_to(message, "❌ Bot này chỉ hoạt động trong tin nhắn riêng tư!")
        return
        
    sub_status, error_msg = check_subscription(message.from_user.id)
    if not sub_status:
        if error_msg:
            bot.reply_to(message, f"⚠️ {error_msg}\n\nVui lòng tham gia: {CHANNEL_USERNAME}")
        else:
            bot.reply_to(message, f"⚠️ Bạn phải đăng ký kênh trước:\n{CHANNEL_USERNAME}")
        return
    
    user_id = message.from_user.id
    tokens = user_tokens.get(str(user_id), 0)
    time_sub, time_msg = check_time_subscription(user_id)
    subscription_status = "✅ Gói đăng ký đang hoạt động" if time_sub else "❌ Không có gói đăng ký"
    
    if time_sub:
        expiry_time = user_subscriptions[str(user_id)]
        expiry_date = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(expiry_time))
        subscription_status += f" Đến hết: {expiry_date}"
    
    try:
        user_profile = bot.get_chat(user_id)
        username = f"@{user_profile.username}" if user_profile.username else "❌ Không có username"
        first_name = user_profile.first_name or ""
        last_name = user_profile.last_name or ""
        full_name = f"{first_name} {last_name}".strip()
        profile_photos = bot.get_user_profile_photos(user_id, limit=1)
        photo_id = None
        if profile_photos.total_count > 0:
            photo_id = profile_photos.photos[0][0].file_id
        profile_msg = f"👤 **Thông tin hồ sơ**\n\n"
        profile_msg += f"🆔 **ID:** `{user_id}`\n"
        profile_msg += f"👤 Username: {username}\n"
        profile_msg += f"📛 **Tên:** {full_name}\n"
        profile_msg += f"💳 **Số token:** {tokens}\n"
        profile_msg += f"📊 **Trạng thái đăng ký:** {subscription_status}\n\n"
        profile_msg += f"📅 **Ngày tham gia:** {time.strftime('%Y-%m-%d %H:%M:%S')}"
        if photo_id:
            bot.send_photo(message.chat.id, photo_id, caption=profile_msg, parse_mode="Markdown")
        else:
            bot.send_message(message.chat.id, profile_msg, parse_mode="Markdown")
            
    except Exception as e:
        print(f"Lỗi trong lệnh profile: {e} - ntnbotforpasskey_laco.py:273")
        bot.reply_to(message, "❌ Đã xảy ra lỗi khi lấy thông tin hồ sơ")

@bot.message_handler(commands=["start"])
def start(message):
    bot.set_my_commands(user_commands, scope=telebot.types.BotCommandScopeChat(message.chat.id))
    if message.chat.type != 'private':
        bot.reply_to(message, "❌ Bot này chỉ hoạt động trong tin nhắn riêng tư!")
        return
        
    sub_status, error_msg = check_subscription(message.from_user.id)
    if not sub_status:
        if error_msg:
            bot.send_message(
                message.chat.id,
                f"⚠️ {error_msg}\n\nVui lòng tham gia: {CHANNEL_USERNAME}"
            )
        else:
            bot.send_message(
                message.chat.id,
                f"⚠️ Bạn phải đăng ký kênh trước:\n{CHANNEL_USERNAME}"
            )
        return
    
    user_id = message.from_user.id
    tokens = user_tokens.get(str(user_id), 0)
    time_sub, time_msg = check_time_subscription(user_id)
    subscription_info = "✅ Bạn có gói đăng ký đang hoạt động" if time_sub else "❌ Không có gói đăng ký đang hoạt động"
    
    if time_sub:
        expiry_time = user_subscriptions[str(user_id)]
        expiry_date = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(expiry_time))
        subscription_info += f" Đến hết: {expiry_date}"
    
    welcome_msg = f"""Xin chào! 👋

Bot này có phí. Sử dụng /buytoken để nạp tiền hoặc /check để kiểm tra liên kết bên ngoài.

Thông tin của bạn:
• Telegram ID: {user_id}
• 💳 Số token của bạn: {tokens}
• 📊 Trạng thái gói đăng ký: {subscription_info}"""
    bot.send_message(message.chat.id, welcome_msg)

@bot.message_handler(commands=["buytoken"])
def buy_tokens(message):
    if message.chat.type != 'private':
        bot.reply_to(message, "❌ Bot này chỉ hoạt động trong trò chuyện riêng tư!")
        return
        
    buy_msg = """Sau khi thanh toán, nhắn tin riêng cho Admin và gửi ảnh chuyển khoản để được xử lý nhanh nhất:
👉 @ntn2k3

Nội dung cần gửi:

Ngân hàng: Ảnh chụp màn hình biên lai (có ghi rõ TELEGRAM ID).

Binance: Ảnh chụp màn hình giao dịch + nội dung TELEGRAM ID của bạn.

⚠️ Lưu ý: KHÔNG gửi thông tin thanh toán tại đây.

Token sẽ được gửi cho bạn ngay sau khi Admin xác nhận thành công"""
    
    markup = telebot.types.InlineKeyboardMarkup()
    button = telebot.types.InlineKeyboardButton(
        text="💬 Nhắn tin cho Admin", 
        url="https://t.me/ntn2k3"
    )
    markup.add(button)
    
    bot.send_message(message.chat.id, buy_msg, reply_markup=markup)

@bot.message_handler(commands=["addtoken"])
def add_token(message):
    if message.chat.type != 'private':
        bot.reply_to(message, "❌ This bot only works in private chats!")
        return
        
    if message.from_user.id != ADMIN_ID:
        bot.reply_to(message, "❌ You are not authorized to use this command!")
        return
        
    parts = message.text.split()
    if len(parts) < 3:
        bot.reply_to(message, "❌ Usage: /addtoken <user_id> <amount>")
        return
        
    try:
        user_id = int(parts[1])
        amount = float(parts[2])
    except:
        bot.reply_to(message, "❌ Invalid user ID or amount!")
        return
        
    if str(user_id) not in user_tokens:
        user_tokens[str(user_id)] = 0
        
    user_tokens[str(user_id)] += amount
    save_user_tokens(user_tokens)
    
    bot.send_message(message.chat.id, f"✅ Đã thêm {amount} token cho {user_id}")
    
    try:
        bot.send_message(user_id, f"✅ Đã thêm {amount} token vào tài khoản của bạn, sử dụng /check <username>")
    except:
        pass

@bot.message_handler(commands=["removetoken"])
def remove_token(message):
    if message.chat.type != 'private':
        bot.reply_to(message, "❌ Bot này chỉ hoạt động trong tin nhắn riêng tư!")
        return
        
    if message.from_user.id != ADMIN_ID:
        bot.reply_to(message, "❌ You are not authorized to use this command!")
        return
        
    parts = message.text.split()
    if len(parts) < 3:
        bot.reply_to(message, "❌ Cách sử dụng: /removetoken <user_id> <số_lượng>")
        return
        
    try:
        user_id = int(parts[1])
        amount = float(parts[2])
    except:
        bot.reply_to(message, "❌ ID người dùng hoặc số tiền không hợp lệ!")
        return
        
    if str(user_id) not in user_tokens:
        user_tokens[str(user_id)] = 0
        
    user_tokens[str(user_id)] = max(0, user_tokens[str(user_id)] - amount)
    save_user_tokens(user_tokens)
    
    bot.send_message(message.chat.id, f"✅ Đã xóa {amount} token từ {user_id}")
@bot.message_handler(commands=["addtime"])
def add_time_subscription(message):
    if message.chat.type != 'private':
        bot.reply_to(message, "❌ This bot only works in private chats!")
        return
        
    if message.from_user.id != ADMIN_ID:
        bot.reply_to(message, "❌ You are not authorized to use this command!")
        return
        
    parts = message.text.split()
    if len(parts) < 3:
        bot.reply_to(message, "❌ Usage: /addtime <user_id> <days>")
        return
        
    try:
        user_id = int(parts[1])
        days = int(parts[2])
    except:
        bot.reply_to(message, "❌ Invalid user ID or days!")
        return
    expiry_time = time.time() + (days * 24 * 60 * 60)
    user_subscriptions[str(user_id)] = expiry_time
    save_user_subscriptions(user_subscriptions)
    
    expiry_date = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(expiry_time))
    bot.send_message(message.chat.id, f"✅ Đã kích hoạt gói đăng ký cho người dùng {user_id} trong {days} ngày, đến hết: {expiry_date}")
    
    try:
        bot.send_message(user_id, f"✅ Gói đăng ký của bạn đã được kích hoạt trong {days} ngày. Bây giờ bạn có thể sử dụng bot không giới hạn cho đến: {expiry_date}")
    except:
        pass

@bot.message_handler(commands=["check"])
def check_command(message):
    if message.chat.type != 'private':
        bot.reply_to(message, "❌ This bot only works in private chats!")
        return
        
    sub_status, error_msg = check_subscription(message.from_user.id)
    if not sub_status:
        if error_msg:
            bot.reply_to(message, f"⚠️ {error_msg}\n\nPlease join: {CHANNEL_USERNAME}")
        else:
            bot.reply_to(message, f"⚠️ You must subscribe to the channel first:\n{CHANNEL_USERNAME}")
        return
    
    if not deduct_token(message.from_user.id):
        bot.reply_to(message, "💸 Cần 1.00 token Sử dụng /buytoken\n• 1 token/kiểm tra = 3,000VNĐ")
        return
    
    if len(message.text.split()) > 1:
        username = message.text.split()[1].strip().lstrip('@')
        processing_msg = bot.reply_to(message, "⏳ Đang trích xuất thông tin...")
        process_username_check(message, username, processing_msg.message_id)
    else:
        bot.reply_to(message, "❌ Vui lòng gửi username sau lệnh (ví dụ: /check username)")

@bot.message_handler(func=lambda message: not message.text.startswith('/') and message.chat.type == 'private', content_types=['text'])
def check_private(message):
    sub_status, error_msg = check_subscription(message.from_user.id)
    if not sub_status:
        if error_msg:
            bot.reply_to(message, f"⚠️ {error_msg}\n\nPlease join: {CHANNEL_USERNAME}")
        else:
            bot.reply_to(message, f"⚠️ You must subscribe to the channel first:\n{CHANNEL_USERNAME}")
        return
    
    if not deduct_token(message.from_user.id):
        bot.reply_to(message, "💸 Cần 1.00 token Sử dụng /buytoken\n• 1 token/kiểm tra = 3,000VNĐ")
        return
    
    username = message.text.strip().lstrip('@')
    if username:
        processing_msg = bot.reply_to(message, "⏳ Đang trích xuất thông tin...")
        process_username_check(message, username, processing_msg.message_id)

def process_username_check(message, username, processing_msg_id=None):
    try:
        response = TikTok.send(username)
        if processing_msg_id:
            try:
                bot.delete_message(message.chat.id, processing_msg_id)
            except:
                pass      
        
        if response:
            data = response.get('data', {})
            has_email = data.get('has_email', False)
            has_mobile = data.get('has_mobile', False)
            has_oauth = data.get('has_oauth', False)
            has_passkey = data.get('has_passkey', False)
            oauth_platforms = data.get('oauth_platforms', [])
            account_data = get_account_info(username)
            
            hidden_binding = "No hidden binding in account 🟢" if not has_passkey else "Account has hidden binding ⚠️"
            external_binding = f"has external binding ({', '.join(oauth_platforms)}) 🔴" if oauth_platforms else "No external binding in account 🟢"
            phone_status = "✔️" if has_mobile else "❌"
            email_status = "✔️" if has_email else "❌"
            
            if account_data and all(account_data.values()):
                flag = account_data.get('flag', '🏴‍☠️')
                followers = account_data.get('followers', 'N/A')
                date_create = account_data.get('date_create', 'N/A')
                
                message_text = f"Account • @{username} {flag}\n\n"
                message_text += f"{hidden_binding}\n"
                message_text += f"{external_binding}\n\n"
                message_text += f"Followers: ({followers}) ~ Date: ({date_create})\n\n"
                message_text += f"Phone status({phone_status})-Email status({email_status})"
            else:
                message_text = f"Account • @{username}\n\n"
                message_text += f"{hidden_binding}\n"
                message_text += f"{external_binding}\n\n"
                message_text += f"Phone status({phone_status})-Email status({email_status})"
            
            bot.reply_to(message, message_text)
        else:
            bot.reply_to(message, f"Username error: {username}")
    except Exception as e:
        print(e)
        if processing_msg_id:
            try:
                bot.delete_message(message.chat.id, processing_msg_id)
            except:
                pass
        bot.reply_to(message, "Username error!")

def get_account_info(username):
    try:
        import datetime
        
        response = requests.get(
            f'https://www.tiktok.com/@{username}',
            headers={
                "User-Agent": "Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.198 Mobile Safari/537.36"
            },
            timeout=3
        ).text
        
        data = response.split('"userInfo":{"user":{')[1].split("</sc")[0]
        followers = data.split('"followerCount":')[1].split(',')[0]
        create_time = data.split('"createTime":')[1].split(',')[0]
        date_create = str(datetime.datetime.fromtimestamp(int(create_time))).split(' ')[0]
        try:
            region = data.split('"region":"')[1].split('"')[0]
            flag = ''.join(chr(0x1F1E6 + ord(c) - ord('A')) for c in region.upper())
        except:
            flag = "🏴‍☠️"
        
        return {
            'followers': followers,
            'date_create': date_create,
            'flag': flag
        }
        
    except Exception as e:
        print(f"Error getting account info: {e} - ntnbotforpasskey_laco.py:566")
        return None

class Encrption:
    @staticmethod
    def xor(string: str) -> str:
        return "".join([hex(ord(_) ^ 5)[2:] for _ in string]) 
    
    @staticmethod
    def sign(params, payload: str = None, sec_device_id: str = "", cookie: str or None = None, aid: int = 567753, license_id: int = 1611921764, sdk_version_str: str = "2.3.1.i18n", sdk_version: int =2, platform: int = 19, unix: int = None):
        x_ss_stub = md5(payload.encode('utf-8')).hexdigest() if payload != None else None
        data=payload
        if not unix: unix = int(time.time())
        return Gorgon(params, unix, payload, cookie).get_value() | { "x-ladon"   : Ladon.encrypt(unix, license_id, aid),"x-argus"   : Argus.get_sign(params, x_ss_stub, unix,platform        = platform,aid             = aid,license_id      = license_id,sec_device_id   = sec_device_id,sdk_version     = sdk_version_str, sdk_version_int = sdk_version)}

class TikTok:
    @staticmethod
    def send(user):
        secret = secrets.token_hex(16)
        cookies = {"passport_csrf_token": secret, "passport_csrf_token_default": secret}
        params = {'request_tag_from': "h5", 'passport-sdk-version': "6031690", 'iid': str(random.randint(1, 10**19)), 'device_id': str(random.randint(1, 10**19)), 'ac': "MOBILE", 'channel': "googleplay", 'aid': "567753", 'app_name': "tiktok_studio", 'version_code': "370301", 'version_name': "37.3.1", 'device_platform': "android", 'os': "android", 'ab_version': "37.3.1", 'ssmix': "a", 'device_type': "Redmi Note 8 Pro", 'device_brand': "Redmi", 'language': "ar", 'os_api': "30", 'os_version': "11", 'openudid': str(binascii.hexlify(os.urandom(8)).decode()), 'manifest_version_code': "370301", 'resolution': "1080*2220", 'dpi': "440", 'update_version_code': "370301", '_rticket': str(round(random.uniform(1.2, 1.6) * 100000000) * -1) + "4632", 'is_pad': "0", 'current_region': "YE", 'app_type': "normal", 'sys_region': "EG", 'last_install_time': "1735989433", 'mcc_mnc': "42103", 'timezone_name': "Asia/Aden", 'residence': "YE", 'app_language': "ar", 'carrier_region': "YE", 'ac2': "lte", 'uoo': "1", 'op_region': "YE", 'timezone_offset': "10800", 'build_number': "37.3.1", 'host_abi': "arm64-v8a", 'locale': "ar", 'region': "EG", 'ts': str(round(random.uniform(1.2, 1.6) * 100000000) * -1), 'cdid': str(uuid.uuid4()), 'support_webview': "1", 'cronet_version': "f6248591_2024-09-11", 'ttnet_version': "4.2.195.9-tiktok", 'use_store_region_cookie': "1"}
        s = get(params=params)
        data = {'mix_mode': "1", 'username': Encrption.xor(user)}
        headers = {'User-Agent': "com.ss.android.tt.creator/370301 (Linux; U; Android 11; ar; Redmi Note 8 Pro; Build/RP1A.200720.011; Cronet/TTNetVersion:f6248591 2024-09-11 QuicVersion:182d68c8 2024-05-28)", 'Accept': "application/json, text/plain, */*", 'x-tt-passport-csrf-token': secret, 'content-type': "application/x-www-form-urlencoded"}
        signature = sign(params=s, payload=data)
        headers.update({'x-ss-req-ticket': signature['x-ss-req-ticket'], 'x-ss-stub': signature['x-ss-stub'], 'x-argus': signature["x-argus"], 'x-gorgon': signature["x-gorgon"], 'x-khronos': signature["x-khronos"], 'x-ladon': signature["x-ladon"]})
        def try_domain(domain):
            try:
                response = requests.post(f"{domain}/passport/find_account/tiktok_username/?", 
                                       params=s, data=data, headers=headers, cookies=cookies, timeout=5)
                if response.status_code == 200 and "data" in response.json() and "token" in response.json()["data"]:
                    return response.json()["data"]["token"]
            except:
                return None
            return None

        tok = None
        with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
            future_to_domain = {executor.submit(try_domain, domain): domain for domain in ASIAN_DOMAINS}
            for future in concurrent.futures.as_completed(future_to_domain):
                result = future.result()
                if result:
                    tok = result
                    executor.shutdown(wait=False)
                    break
        
        if not tok: 
            return False
        
        m = Encrption.sign(params=urlencode(params), payload="", cookie=urlencode(cookies))
        params.update({'not_login_ticket': tok})
        headers.update({'x-argus': m["x-argus"], 'x-gorgon': m["x-gorgon"], 'x-khronos': m["x-khronos"], 'x-ladon': m["x-ladon"]})
        
        def try_domain_final(domain):
            try:
                res = requests.post(f"{domain}/passport/auth/available_ways/?", 
                                  params=s, headers=headers, cookies=cookies, timeout=5)
                if res.status_code == 200: 
                    return res.json()
            except:
                return None
            return None
        result = None
        with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
            future_to_domain = {executor.submit(try_domain_final, domain): domain for domain in ASIAN_DOMAINS}
            for future in concurrent.futures.as_completed(future_to_domain):
                result = future.result()
                if result:
                    executor.shutdown(wait=False)
                    break
        
        return result if result else False

while True:
    try: 
        bot.set_my_commands(user_commands)
        bot.infinity_polling(none_stop=True)
    except Exception: 
        time.sleep(60)

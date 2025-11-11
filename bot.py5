import os
import logging
import asyncio
from datetime import datetime, timezone
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ChatMemberHandler,
    ContextTypes,
    filters,
)
from motor.motor_asyncio import AsyncIOMotorClient
from dotenv import load_dotenv
from pathlib import Path

# Load environment variables
ROOT_DIR = Path(__file__).parent
load_dotenv(ROOT_DIR / '.env')

# Configuration
TELEGRAM_BOT_TOKEN = os.environ.get('TELEGRAM_BOT_TOKEN', '8012410295:AAE33t3wNvtXYT9M7BE2RLjUctYHgFD_ToQ')
ADMIN_ID = int(os.environ.get('ADMIN_ID', '872863489'))
MONGO_URL = os.environ.get('MONGO_URL')
DB_NAME = os.environ.get('DB_NAME', 'telegram_bot_db')

# محدودیت کاراکتر
MAX_MESSAGE_LENGTH = 300

# MongoDB setup
client = AsyncIOMotorClient(MONGO_URL)
db = client[DB_NAME]

# Logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Store conversation states and pending messages
user_states = {}
pending_messages = {}


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /start command"""
    user = update.effective_user
    
    # Save user to database
    await db.users.update_one(
        {'user_id': user.id},
        {
            '$set': {
                'username': user.username,
                'first_name': user.first_name,
                'last_name': user.last_name,
                'last_active': datetime.now(timezone.utc).isoformat(),
                'has_blocked': False,
            },
            '$setOnInsert': {
                'user_id': user.id,
                'is_blocked': False,
                'created_at': datetime.now(timezone.utc).isoformat(),
            }
        },
        upsert=True
    )
    
    # Check if user is admin
    if user.id == ADMIN_ID:
        keyboard = [
            [InlineKeyboardButton("📊 آمار", callback_data="admin_stats")],
            [InlineKeyboardButton("👥 لیست کاربران", callback_data="admin_users_list")],
            [InlineKeyboardButton("🚫 کاربران ترک‌کننده", callback_data="admin_blocked_users")],
            [InlineKeyboardButton("📢 پیام همگانی", callback_data="admin_broadcast")],
            [InlineKeyboardButton("💬 پیام به کاربر", callback_data="admin_send_to_user")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        await update.message.reply_text(
            "👋 *سلام ادمین!*\n\n"
            "به پنل مدیریت خوش آمدید",
            reply_markup=reply_markup,
            parse_mode='Markdown'
        )
    else:
        keyboard = [
            [InlineKeyboardButton("✉️ ارسال پیام ناشناس", callback_data="user_send_message")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        await update.message.reply_text(
            f"💬 *سلام {user.first_name}!*\n\n"
            "✨ به ربات ارسال پیام ناشناس خوش آمدید\n\n"
            "پیام‌های شما به صورت کاملاً ناشناس ارسال می‌شود\n"
            "📌 حداکثر هر پیام: 300 کاراکتر",
            reply_markup=reply_markup,
            parse_mode='Markdown'
        )


async def admin_panel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Show admin panel"""
    user = update.effective_user
    
    if user.id != ADMIN_ID:
        await update.message.reply_text("⛔️ دسترسی غیرمجاز!")
        return
    
    keyboard = [
        [InlineKeyboardButton("📊 آمار", callback_data="admin_stats")],
        [InlineKeyboardButton("👥 لیست کاربران", callback_data="admin_users_list")],
        [InlineKeyboardButton("🚫 کاربران ترک‌کننده", callback_data="admin_blocked_users")],
        [InlineKeyboardButton("📢 پیام همگانی", callback_data="admin_broadcast")],
        [InlineKeyboardButton("💬 پیام به کاربر", callback_data="admin_send_to_user")],
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await update.message.reply_text(
        "🎛 *پنل مدیریت*\n\n"
        "یکی از گزینه‌ها را انتخاب کنید:",
        reply_markup=reply_markup,
        parse_mode='Markdown'
    )


async def handle_user_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle messages from users"""
    user = update.effective_user
    message = update.message
    
    # Check if user is blocked
    user_doc = await db.users.find_one({'user_id': user.id})
    if user_doc and user_doc.get('is_blocked', False):
        await message.reply_text("⛔️ شما از استفاده از بات محروم شده‌اید")
        return
    
    # Update has_blocked to False (user is active)
    await db.users.update_one(
        {'user_id': user.id},
        {'$set': {'has_blocked': False, 'last_active': datetime.now(timezone.utc).isoformat()}}
    )
    
    # Admin message handling
    if user.id == ADMIN_ID:
        if user.id in user_states and user_states[user.id].get('mode') == 'waiting_for_user_id':
            if message.text and message.text.isdigit():
                target_user_id = int(message.text)
                target_user = await db.users.find_one({'user_id': target_user_id})
                if target_user:
                    user_states[user.id] = {'mode': 'sending_to_user', 'target_user_id': target_user_id}
                    await message.reply_text(
                        f"✅ کاربر یافت شد: {target_user.get('first_name', 'کاربر')}\n\n"
                        f"پیام خود را ارسال کنید:"
                    )
                else:
                    await message.reply_text("❌ کاربر یافت نشد")
            else:
                await message.reply_text("❌ لطفاً فقط عدد وارد کنید")
            return
        
        elif user.id in user_states and user_states[user.id].get('mode') == 'sending_to_user':
            target_user_id = user_states[user.id]['target_user_id']
            try:
                if message.text:
                    await context.bot.send_message(target_user_id, f"📩 پیام از ادمین:\n\n{message.text}")
                elif message.photo:
                    await context.bot.send_photo(target_user_id, message.photo[-1].file_id, caption=f"📩 {message.caption or ''}")
                elif message.video:
                    await context.bot.send_video(target_user_id, message.video.file_id, caption=f"📩 {message.caption or ''}")
                
                await message.reply_text("✅ پیام ارسال شد!")
                user_states[user.id] = {}
            except Exception as e:
                logger.error(f"Error sending to user: {e}")
                await message.reply_text(f"❌ خطا: {str(e)}")
            return
        
        elif user.id in user_states and user_states[user.id].get('mode') == 'replying':
            target_user_id = user_states[user.id]['target_user_id']
            try:
                if message.text:
                    await context.bot.send_message(target_user_id, f"📩 پاسخ ادمین:\n\n{message.text}")
                elif message.photo:
                    await context.bot.send_photo(target_user_id, message.photo[-1].file_id, caption=f"📩 {message.caption or ''}")
                elif message.video:
                    await context.bot.send_video(target_user_id, message.video.file_id, caption=f"📩 {message.caption or ''}")
                
                await message.reply_text("✅ پیام ارسال شد!")
                user_states[user.id] = {}
            except Exception as e:
                logger.error(f"Error replying: {e}")
                await message.reply_text(f"❌ خطا: {str(e)}")
            return
        
        elif user.id in user_states and user_states[user.id].get('mode') == 'broadcasting':
            users = await db.users.find({'is_blocked': False, 'user_id': {'$ne': ADMIN_ID}}).to_list(10000)
            
            success_count = 0
            fail_count = 0
            
            for user_doc in users:
                try:
                    if message.text:
                        await context.bot.send_message(user_doc['user_id'], f"📢 پیام همگانی:\n\n{message.text}")
                    elif message.photo:
                        await context.bot.send_photo(user_doc['user_id'], message.photo[-1].file_id, caption=f"📢 {message.caption or ''}")
                    elif message.video:
                        await context.bot.send_video(user_doc['user_id'], message.video.file_id, caption=f"📢 {message.caption or ''}")
                    
                    success_count += 1
                    await asyncio.sleep(0.05)
                except Exception as e:
                    fail_count += 1
                    logger.error(f"Failed to send to {user_doc['user_id']}: {e}")
            
            await message.reply_text(
                f"✅ پیام همگانی ارسال شد\n\n"
                f"✅ موفق: {success_count}\n"
                f"❌ ناموفق: {fail_count}"
            )
            user_states[user.id] = {}
            return
    
    # Regular user messaging
    if user.id in user_states and user_states[user.id].get('mode') == 'composing_message':
        # بررسی محدودیت کاراکتر
        if message.text and len(message.text) > MAX_MESSAGE_LENGTH:
            await message.reply_text(f"⚠️ پیام بیش از حد طولانی است!\n\nحداکثر: {MAX_MESSAGE_LENGTH} کاراکتر")
            return
        
        if (message.photo or message.video) and message.caption and len(message.caption) > MAX_MESSAGE_LENGTH:
            await message.reply_text(f"⚠️ توضیحات بیش از حد طولانی است!\n\nحداکثر: {MAX_MESSAGE_LENGTH} کاراکتر")
            return
        
        # ذخیره پیام
        if user.id not in pending_messages:
            pending_messages[user.id] = []
        
        message_id = len(pending_messages[user.id])
        message_data = {
            'id': message_id,
            'user_id': user.id,
            'username': user.username,
            'first_name': user.first_name,
            'last_name': user.last_name,
            'timestamp': datetime.now(timezone.utc).isoformat(),
            'is_deleted': False,
        }
        
        if message.text:
            message_data['type'] = 'text'
            message_data['content'] = message.text
        elif message.photo:
            message_data['type'] = 'photo'
            message_data['file_id'] = message.photo[-1].file_id
            message_data['caption'] = message.caption
        elif message.video:
            message_data['type'] = 'video'
            message_data['file_id'] = message.video.file_id
            message_data['caption'] = message.caption
        
        pending_messages[user.id].append(message_data)
        
        count = len([m for m in pending_messages[user.id] if not m['is_deleted']])
        
        keyboard = [
            [InlineKeyboardButton("✅ ارسال", callback_data="send_to_admin")],
            [InlineKeyboardButton("✏️ ویرایش", callback_data="edit_last"), InlineKeyboardButton("🗑 حذف", callback_data="delete_last")],
            [InlineKeyboardButton("❌ لغو", callback_data="cancel")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await message.reply_text(
            f"✅ *دریافت شد*\n\n"
            f"📊 تعداد پیام‌ها: {count}\n\n"
            f"می‌توانید پیام دیگری بفرستید یا:",
            reply_markup=reply_markup,
            parse_mode='Markdown'
        )
    
    # ویرایش پیام
    elif user.id in user_states and user_states[user.id].get('mode') == 'editing':
        edit_msg_id = user_states[user.id]['edit_msg_id']
        
        if edit_msg_id < len(pending_messages[user.id]):
            if message.text and len(message.text) > MAX_MESSAGE_LENGTH:
                await message.reply_text(f"⚠️ پیام بیش از حد طولانی است!")
                return
            
            if (message.photo or message.video) and message.caption and len(message.caption) > MAX_MESSAGE_LENGTH:
                await message.reply_text(f"⚠️ توضیحات بیش از حد طولانی است!")
                return
            
            if message.text:
                pending_messages[user.id][edit_msg_id]['type'] = 'text'
                pending_messages[user.id][edit_msg_id]['content'] = message.text
            elif message.photo:
                pending_messages[user.id][edit_msg_id]['type'] = 'photo'
                pending_messages[user.id][edit_msg_id]['file_id'] = message.photo[-1].file_id
                pending_messages[user.id][edit_msg_id]['caption'] = message.caption
            elif message.video:
                pending_messages[user.id][edit_msg_id]['type'] = 'video'
                pending_messages[user.id][edit_msg_id]['file_id'] = message.video.file_id
                pending_messages[user.id][edit_msg_id]['caption'] = message.caption
            
            pending_messages[user.id][edit_msg_id]['timestamp'] = datetime.now(timezone.utc).isoformat()
            user_states[user.id] = {'mode': 'composing_message'}
            
            count = len([m for m in pending_messages[user.id] if not m['is_deleted']])
            
            keyboard = [
                [InlineKeyboardButton("✅ ارسال", callback_data="send_to_admin")],
                [InlineKeyboardButton("✏️ ویرایش", callback_data="edit_last"), InlineKeyboardButton("🗑 حذف", callback_data="delete_last")],
                [InlineKeyboardButton("❌ لغو", callback_data="cancel")],
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await message.reply_text(
                f"✅ *ویرایش شد*\n\n"
                f"📊 تعداد پیام‌ها: {count}",
                reply_markup=reply_markup,
                parse_mode='Markdown'
            )
    
    else:
        keyboard = [
            [InlineKeyboardButton("✉️ ارسال پیام ناشناس", callback_data="user_send_message")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        await message.reply_text(
            "💡 لطفاً ابتدا روی دکمه زیر کلیک کنید:",
            reply_markup=reply_markup
        )


async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle button callbacks"""
    query = update.callback_query
    await query.answer()
    
    data = query.data
    user = query.from_user
    
    # User buttons
    if data.startswith("user_"):
        if data == "user_send_message":
            user_states[user.id] = {'mode': 'composing_message'}
            pending_messages[user.id] = []
            
            await query.edit_message_text(
                "✉️ *حالت ارسال پیام*\n\n"
                "✨ پیام خود را بنویسید\n"
                "📸 یا عکس/ویدیو بفرستید\n\n"
                "📌 حداکثر: 300 کاراکتر",
                parse_mode='Markdown'
            )
        return
    
    # لغو
    if data == "cancel":
        if user.id in user_states:
            del user_states[user.id]
        if user.id in pending_messages:
            del pending_messages[user.id]
        
        keyboard = [
            [InlineKeyboardButton("✉️ ارسال پیام ناشناس", callback_data="user_send_message")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            "❌ *لغو شد*\n\n"
            "می‌توانید دوباره شروع کنید:",
            reply_markup=reply_markup,
            parse_mode='Markdown'
        )
        return
    
    # ویرایش آخرین پیام
    if data == "edit_last":
        if user.id not in pending_messages or len(pending_messages[user.id]) == 0:
            await query.answer("❌ پیامی وجود ندارد!", show_alert=True)
            return
        
        active_msgs = [msg for msg in pending_messages[user.id] if not msg['is_deleted']]
        if not active_msgs:
            await query.answer("❌ پیام فعالی وجود ندارد!", show_alert=True)
            return
        
        last_msg = active_msgs[-1]
        user_states[user.id] = {'mode': 'editing', 'edit_msg_id': last_msg['id']}
        
        await query.edit_message_text(
            "✏️ *ویرایش پیام*\n\n"
            "پیام جدید خود را ارسال کنید:",
            parse_mode='Markdown'
        )
        return
    
    # حذف آخرین پیام
    if data == "delete_last":
        if user.id not in pending_messages or len(pending_messages[user.id]) == 0:
            await query.answer("❌ پیامی وجود ندارد!", show_alert=True)
            return
        
        active_msgs = [msg for msg in pending_messages[user.id] if not msg['is_deleted']]
        if not active_msgs:
            await query.answer("❌ پیام فعالی وجود ندارد!", show_alert=True)
            return
        
        last_msg = active_msgs[-1]
        pending_messages[user.id][last_msg['id']]['is_deleted'] = True
        
        remaining = len([m for m in pending_messages[user.id] if not m['is_deleted']])
        
        if remaining > 0:
            keyboard = [
                [InlineKeyboardButton("✅ ارسال", callback_data="send_to_admin")],
                [InlineKeyboardButton("✏️ ویرایش", callback_data="edit_last"), InlineKeyboardButton("🗑 حذف", callback_data="delete_last")],
                [InlineKeyboardButton("❌ لغو", callback_data="cancel")],
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await query.edit_message_text(
                f"🗑 *حذف شد*\n\n"
                f"📊 پیام‌های باقی‌مانده: {remaining}",
                reply_markup=reply_markup,
                parse_mode='Markdown'
            )
        else:
            if user.id in user_states:
                del user_states[user.id]
            if user.id in pending_messages:
                del pending_messages[user.id]
            
            keyboard = [
                [InlineKeyboardButton("✉️ ارسال پیام ناشناس", callback_data="user_send_message")],
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await query.edit_message_text(
                "🗑 *همه پیام‌ها حذف شدند*",
                reply_markup=reply_markup,
                parse_mode='Markdown'
            )
        return
    
    # ارسال به ادمین
    if data == "send_to_admin":
        if user.id not in pending_messages or len(pending_messages[user.id]) == 0:
            await query.edit_message_text("❌ پیامی برای ارسال وجود ندارد!")
            return
        
        active_messages = [msg for msg in pending_messages[user.id] if not msg['is_deleted']]
        
        if not active_messages:
            await query.edit_message_text("❌ پیام فعالی برای ارسال وجود ندارد!")
            return
        
        # ذخیره در دیتابیس
        for msg_data in pending_messages[user.id]:
            await db.messages.insert_one(msg_data)
        
        deleted_messages = [msg for msg in pending_messages[user.id] if msg['is_deleted']]
        
        # ارسال پیام‌های فعال به ادمین
        try:
            for i, msg_data in enumerate(active_messages, 1):
                admin_header = (
                    f"📨 *پیام جدید {i}/{len(active_messages)}*\n\n"
                    f"👤 {msg_data.get('first_name', 'N/A')} {msg_data.get('last_name', '') or ''}\n"
                    f"🆔 @{msg_data.get('username') or 'ندارد'}\n"
                    f"🔢 ID: `{msg_data['user_id']}`\n"
                    f"⏰ {datetime.fromisoformat(msg_data['timestamp']).strftime('%Y-%m-%d %H:%M')}\n\n"
                )
                
                keyboard = [
                    [InlineKeyboardButton("💬 پاسخ", callback_data=f"reply_{msg_data['user_id']}"),
                     InlineKeyboardButton("🚫 بلاک", callback_data=f"block_{msg_data['user_id']}")],
                ]
                reply_markup = InlineKeyboardMarkup(keyboard)
                
                if msg_data['type'] == 'text':
                    await context.bot.send_message(
                        ADMIN_ID,
                        admin_header + f"💬 {msg_data['content']}",
                        reply_markup=reply_markup,
                        parse_mode='Markdown'
                    )
                elif msg_data['type'] == 'photo':
                    caption_text = admin_header
                    if msg_data.get('caption'):
                        caption_text += f"💬 {msg_data['caption']}"
                    else:
                        caption_text += "📸 عکس"
                    
                    await context.bot.send_photo(
                        ADMIN_ID,
                        msg_data['file_id'],
                        caption=caption_text,
                        reply_markup=reply_markup,
                        parse_mode='Markdown'
                    )
                elif msg_data['type'] == 'video':
                    caption_text = admin_header
                    if msg_data.get('caption'):
                        caption_text += f"💬 {msg_data['caption']}"
                    else:
                        caption_text += "🎥 ویدیو"
                    
                    await context.bot.send_video(
                        ADMIN_ID,
                        msg_data['file_id'],
                        caption=caption_text,
                        reply_markup=reply_markup,
                        parse_mode='Markdown'
                    )
                
                await asyncio.sleep(0.1)
            
            logger.info(f"Successfully sent {len(active_messages)} messages to admin from user {user.id}")
            
        except Exception as e:
            logger.error(f"Failed to send messages to admin: {e}")
            await query.edit_message_text(f"❌ خطا در ارسال: {str(e)}")
            return
        
        # ارسال پیام‌های حذف شده (اگر وجود داشته باشند)
        if deleted_messages:
            try:
                for i, msg_data in enumerate(deleted_messages, 1):
                    if msg_data['type'] == 'text':
                        await context.bot.send_message(ADMIN_ID, f"🗑 پیام حذف شده {i}:\n{msg_data['content']}")
                    elif msg_data['type'] == 'photo':
                        await context.bot.send_photo(ADMIN_ID, msg_data['file_id'], caption=f"🗑 عکس حذف شده {i}")
                    elif msg_data['type'] == 'video':
                        await context.bot.send_video(ADMIN_ID, msg_data['file_id'], caption=f"🗑 ویدیو حذف شده {i}")
                    await asyncio.sleep(0.1)
            except Exception as e:
                logger.error(f"Failed to send deleted messages: {e}")
        
        # پاک کردن state
        if user.id in user_states:
            del user_states[user.id]
        if user.id in pending_messages:
            del pending_messages[user.id]
        
        keyboard = [
            [InlineKeyboardButton("✉️ ارسال پیام جدید", callback_data="user_send_message")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            "✅ *پیام ارسال شد!*\n\n"
            "به زودی پاسخ می‌دهیم",
            reply_markup=reply_markup,
            parse_mode='Markdown'
        )
        return
    
    # Admin commands
    if user.id != ADMIN_ID:
        await query.edit_message_text("⛔️ دسترسی غیرمجاز!")
        return
    
    # Admin stats
    if data == "admin_stats":
        total_users = await db.users.count_documents({'user_id': {'$ne': ADMIN_ID}})
        blocked_users = await db.users.count_documents({'is_blocked': True})
        active_users = total_users - blocked_users
        total_messages = await db.messages.count_documents({})
        deleted_messages = await db.messages.count_documents({'is_deleted': True})
        left_users = await db.users.count_documents({'has_blocked': True})
        
        stats_text = (
            f"📊 *آمار بات*\n\n"
            f"👥 کل کاربران: {total_users}\n"
            f"✅ فعال: {active_users}\n"
            f"🚫 بلاک شده: {blocked_users}\n"
            f"🚪 ترک‌کننده: {left_users}\n"
            f"💬 کل پیام‌ها: {total_messages}\n"
            f"🗑 حذف شده: {deleted_messages}"
        )
        
        keyboard = [[InlineKeyboardButton("🔙 بازگشت", callback_data="admin_back")]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(stats_text, reply_markup=reply_markup, parse_mode='Markdown')
    
    elif data == "admin_blocked_users":
        left_users = await db.users.find({'has_blocked': True}).sort('last_active', -1).limit(20).to_list(20)
        
        if not left_users:
            keyboard = [[InlineKeyboardButton("🔙 بازگشت", callback_data="admin_back")]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            await query.edit_message_text("✅ کاربر ترک‌کننده‌ای وجود ندارد!", reply_markup=reply_markup)
            return
        
        users_text = "🚫 *کاربران ترک‌کننده*\n\n"
        for i, u in enumerate(left_users, 1):
            last_active = u.get('last_active', 'نامشخص')
            if last_active != 'نامشخص':
                try:
                    last_active = datetime.fromisoformat(last_active).strftime('%Y-%m-%d')
                except:
                    pass
            
            users_text += f"{i}. {u.get('first_name', 'N/A')}\n"
            users_text += f"   @{u.get('username', 'ندارد')} | {last_active}\n\n"
        
        keyboard = [[InlineKeyboardButton("🔙 بازگشت", callback_data="admin_back")]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(users_text, reply_markup=reply_markup, parse_mode='Markdown')
    
    elif data == "admin_broadcast":
        user_states[user.id] = {'mode': 'broadcasting'}
        keyboard = [[InlineKeyboardButton("❌ لغو", callback_data="admin_cancel")]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            "📢 *پیام همگانی*\n\n"
            "پیام خود را ارسال کنید:",
            reply_markup=reply_markup,
            parse_mode='Markdown'
        )
    
    elif data == "admin_users_list":
        users = await db.users.find({'user_id': {'$ne': ADMIN_ID}}).sort('created_at', -1).limit(15).to_list(15)
        
        if not users:
            keyboard = [[InlineKeyboardButton("🔙 بازگشت", callback_data="admin_back")]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            await query.edit_message_text("کاربری ثبت نشده است", reply_markup=reply_markup)
            return
        
        users_text = "👥 *آخرین کاربران*\n\n"
        for i, u in enumerate(users, 1):
            status = "🚫" if u.get('is_blocked', False) else "✅"
            left = "🚪" if u.get('has_blocked', False) else ""
            users_text += f"{i}. {status}{left} {u.get('first_name', 'N/A')}\n   @{u.get('username', 'ندارد')} | `{u['user_id']}`\n\n"
        
        keyboard = [[InlineKeyboardButton("🔙 بازگشت", callback_data="admin_back")]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(users_text, reply_markup=reply_markup, parse_mode='Markdown')
    
    elif data == "admin_send_to_user":
        user_states[user.id] = {'mode': 'waiting_for_user_id'}
        keyboard = [[InlineKeyboardButton("❌ لغو", callback_data="admin_cancel")]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            "💬 *ارسال به کاربر*\n\n"
            "User ID کاربر را وارد کنید:",
            reply_markup=reply_markup,
            parse_mode='Markdown'
        )
    
    elif data.startswith("reply_"):
        target_user_id = int(data.split('_')[1])
        user_states[user.id] = {'mode': 'replying', 'target_user_id': target_user_id}
        
        target_user = await db.users.find_one({'user_id': target_user_id})
        
        await query.edit_message_reply_markup(reply_markup=None)
        await context.bot.send_message(
            ADMIN_ID,
            f"💬 حالت پاسخ به {target_user.get('first_name', 'کاربر')}\n\nپیام خود را ارسال کنید:"
        )
    
    elif data.startswith("block_"):
        target_user_id = int(data.split('_')[1])
        user_doc = await db.users.find_one({'user_id': target_user_id})
        is_blocked = user_doc.get('is_blocked', False) if user_doc else False
        
        await db.users.update_one(
            {'user_id': target_user_id},
            {'$set': {'is_blocked': not is_blocked}}
        )
        
        if is_blocked:
            status_text = "✅ رفع بلاک شد"
            new_button_text = "🚫 بلاک"
        else:
            status_text = "🚫 بلاک شد"
            new_button_text = "✅ رفع بلاک"
        
        keyboard = [
            [InlineKeyboardButton("💬 پاسخ", callback_data=f"reply_{target_user_id}"),
             InlineKeyboardButton(new_button_text, callback_data=f"block_{target_user_id}")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_reply_markup(reply_markup=reply_markup)
        await context.bot.send_message(ADMIN_ID, status_text)
    
    elif data == "admin_back" or data == "admin_cancel":
        if data == "admin_cancel" and user.id in user_states:
            user_states[user.id] = {}
        
        keyboard = [
            [InlineKeyboardButton("📊 آمار", callback_data="admin_stats")],
            [InlineKeyboardButton("👥 لیست کاربران", callback_data="admin_users_list")],
            [InlineKeyboardButton("🚫 کاربران ترک‌کننده", callback_data="admin_blocked_users")],
            [InlineKeyboardButton("📢 پیام همگانی", callback_data="admin_broadcast")],
            [InlineKeyboardButton("💬 پیام به کاربر", callback_data="admin_send_to_user")],
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            "🎛 *پنل مدیریت*\n\n"
            "یکی از گزینه‌ها را انتخاب کنید:",
            reply_markup=reply_markup,
            parse_mode='Markdown'
        )


async def error_handler(update: object, context: ContextTypes.DEFAULT_TYPE):
    """Log errors"""
    logger.error(f"Exception: {context.error}", exc_info=context.error)


async def my_chat_member_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Track when users block/unblock the bot"""
    if update.my_chat_member:
        user_id = update.my_chat_member.from_user.id
        new_status = update.my_chat_member.new_chat_member.status
        
        if new_status in ['kicked', 'left']:
            await db.users.update_one(
                {'user_id': user_id},
                {'$set': {'has_blocked': True, 'last_active': datetime.now(timezone.utc).isoformat()}}
            )
            logger.info(f"User {user_id} blocked the bot")
        elif new_status == 'member':
            await db.users.update_one(
                {'user_id': user_id},
                {'$set': {'has_blocked': False, 'last_active': datetime.now(timezone.utc).isoformat()}}
            )
            logger.info(f"User {user_id} unblocked the bot")


def main():
    """Start the bot"""
    if not TELEGRAM_BOT_TOKEN:
        logger.error("TELEGRAM_BOT_TOKEN not found!")
        return
    
    logger.info(f"Starting bot with Admin ID: {ADMIN_ID}")
    
    application = Application.builder().token(TELEGRAM_BOT_TOKEN).build()
    
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("admin", admin_panel))
    application.add_handler(CallbackQueryHandler(button_callback))
    application.add_handler(MessageHandler(filters.TEXT | filters.PHOTO | filters.VIDEO, handle_user_message))
    application.add_handler(ChatMemberHandler(my_chat_member_handler, ChatMemberHandler.MY_CHAT_MEMBER))
    
    application.add_error_handler(error_handler)
    
    logger.info("Bot started successfully!")
    application.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == '__main__':
    main()

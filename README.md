```python
import discord
import aiosqlite
import asyncio
from datetime import datetime, timezone, timedelta
import os
import re
import sys

TOKEN = 'YOUR_DISCORD_TOKEN_HERE'

# Multiple servers and channels
ALLOWED_LOCATIONS = [
    {'guild_id': 1347804635989016617, 'channel_id': 1417960723363008722},
    {'guild_id': 1455195456089886886, 'channel_id': 1455195456647987256}
]

PASSWORD = 'lushydev'

# ===== CONFIG =====
ENABLE_LOGGING = True  # Set to False to disable all terminal logs
# ==================

# Simple bot without command system - we'll handle commands manually
client = discord.Client()

cooldowns = {}
COOLDOWN_TIME = 5

def log(message):
    """Only print if logging is enabled"""
    if ENABLE_LOGGING:
        print(message)

async def init_db():
    async with aiosqlite.connect('afk.db') as db:
        await db.execute('''
            CREATE TABLE IF NOT EXISTS afk_users (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                afk_message TEXT,
                timestamp TEXT
            )
        ''')
        await db.commit()

def is_allowed_location(guild_id, channel_id):
    """Check if message is in allowed server/channel"""
    for location in ALLOWED_LOCATIONS:
        if location['guild_id'] == guild_id and location['channel_id'] == channel_id:
            return True
    return False

def format_time_ago(timestamp_str):
    then = datetime.fromisoformat(timestamp_str)
    now = datetime.now(timezone.utc)
    diff = now - then
    
    days = diff.days
    hours = diff.seconds // 3600
    minutes = (diff.seconds % 3600) // 60
    
    if days > 0:
        return f"{days} day{'s' if days != 1 else ''} ago"
    elif hours > 0:
        return f"{hours} hour{'s' if hours != 1 else ''} ago"
    elif minutes > 0:
        return f"{minutes} minute{'s' if minutes != 1 else ''} ago"
    else:
        return "just now"

def parse_time_string(time_str):
    """Parse time strings like '6h', '1d', '30m' and return timedelta"""
    match = re.match(r'^(\d+)([hdm])$', time_str.lower())
    if not match:
        return None
    
    amount = int(match.group(1))
    unit = match.group(2)
    
    if unit == 'h':
        return timedelta(hours=amount)
    elif unit == 'd':
        return timedelta(days=amount)
    elif unit == 'm':
        return timedelta(minutes=amount)
    
    return None

@client.event
async def on_ready():
    await init_db()
    log(f'========================================')
    log(f'Logged in as {client.user}')
    log(f'User ID: {client.user.id}')
    log('AFK Selfbot is ready!')
    log(f'Monitoring {len(ALLOWED_LOCATIONS)} locations')
    log(f'Logging: {"ENABLED" if ENABLE_LOGGING else "DISABLED"}')
    log('========================================')
    
    for guild in client.guilds:
        log(f'Connected to guild: {guild.name} (ID: {guild.id})')
        for channel in guild.text_channels:
            for loc in ALLOWED_LOCATIONS:
                if loc['guild_id'] == guild.id and loc['channel_id'] == channel.id:
                    log(f'  -> Monitoring channel: #{channel.name} (ID: {channel.id})')

@client.event
async def on_message(message):
    # ===== IGNORE BOT'S OWN MESSAGES =====
    if message.author.id == client.user.id:
        log(f'\n[IGNORED] Bot\'s own message')
        return
    
    log(f'\n[MESSAGE RECEIVED]')
    log(f'  Author: {message.author.name} (ID: {message.author.id})')
    log(f'  Content: {message.content[:100]}')
    
    if message.guild:
        log(f'  Guild: {message.guild.name} (ID: {message.guild.id})')
        log(f'  Channel: {message.channel.name} (ID: {message.channel.id})')
        log(f'  Is Allowed: {is_allowed_location(message.guild.id, message.channel.id)}')
    
    # Ignore DMs
    if message.guild is None:
        log('  [SKIPPED] DM message')
        return
    
    # Check if in allowed location
    if not is_allowed_location(message.guild.id, message.channel.id):
        log('  [SKIPPED] Not in allowed channel')
        return
    
    log('  [PROCESSING] In allowed channel!')
    
    # ===== MANUAL COMMAND PROCESSING =====
    # Handle ?afk command
    if message.content.startswith('?afk ') or message.content == '?afk':
        log(f'  [COMMAND] ?afk detected from {message.author.name}!')
        
        # Extract reason
        reason = 'AFK'
        if message.content.startswith('?afk '):
            reason = message.content[5:].strip()
        
        if len(reason) > 100:
            await message.channel.send(f'<@{message.author.id}> Cant set more than 100 letters as your AFK message!')
            return
        
        timestamp = datetime.now(timezone.utc).isoformat()
        
        async with aiosqlite.connect('afk.db') as db:
            await db.execute('''
                INSERT OR REPLACE INTO afk_users (user_id, username, afk_message, timestamp)
                VALUES (?, ?, ?, ?)
            ''', (message.author.id, message.author.name, reason, timestamp))
            await db.commit()
        
        log(f'  [COMMAND] AFK set successfully for {message.author.name}: {reason}')
        await message.channel.send(f'<@{message.author.id}> I set your AFK: {reason}')
        return  # Don't process further
    
    # Handle ?afkset command
    if message.content.startswith('?afkset '):
        log(f'  [COMMAND] ?afkset detected from {message.author.name}!')
        
        parts = message.content[8:].strip().split()
        
        if len(parts) >= 2:
            time_str = parts[0]
            password = parts[1]
            
            if password != PASSWORD:
                log(f'  [COMMAND] Wrong password')
                return
            
            time_delta = parse_time_string(time_str)
            if time_delta is None:
                await message.channel.send(f'<@{message.author.id}> Invalid time format! Use: 6h, 1d, 30m')
                return
            
            backdated_time = datetime.now(timezone.utc) - time_delta
            timestamp = backdated_time.isoformat()
            afk_message = 'AFK'
            
            async with aiosqlite.connect('afk.db') as db:
                await db.execute('''
                    INSERT OR REPLACE INTO afk_users (user_id, username, afk_message, timestamp)
                    VALUES (?, ?, ?, ?)
                ''', (message.author.id, message.author.name, afk_message, timestamp))
                await db.commit()
            
            log(f'  [COMMAND] Backdated AFK set successfully for {message.author.name}')
            await message.channel.send(f'<@{message.author.id}> I set your AFK: {afk_message} (backdated {time_str})')
            return
    
    # ===== AFK RETURN CHECK =====
    async with aiosqlite.connect('afk.db') as db:
        cursor = await db.execute('SELECT * FROM afk_users WHERE user_id = ?', (message.author.id,))
        afk_data = await cursor.fetchone()
        
        if afk_data:
            log(f'  [AFK CHECK] User is currently AFK')
            # Don't remove if they're setting AFK
            if not message.content.startswith('?afk'):
                await db.execute('DELETE FROM afk_users WHERE user_id = ?', (message.author.id,))
                await db.commit()
                
                log(f'  [WELCOME BACK] Removing AFK status')
                try:
                    welcome_msg = await message.channel.send(f'Welcome back <@{message.author.id}>, I removed your AFK')
                    await asyncio.sleep(10)
                    await welcome_msg.delete()
                except Exception as e:
                    log(f'  [ERROR] Could not send welcome message: {e}')
    
    # ===== MENTION CHECK =====
    if message.mentions:
        log(f'  [MENTIONS] Found {len(message.mentions)} mentions')
        async with aiosqlite.connect('afk.db') as db:
            for user in message.mentions:
                log(f'    Checking mention: {user.name} (ID: {user.id})')
                cursor = await db.execute('SELECT * FROM afk_users WHERE user_id = ?', (user.id,))
                afk_data = await cursor.fetchone()
                
                if afk_data:
                    user_id, username, afk_msg, timestamp = afk_data
                    time_ago = format_time_ago(timestamp)
                    
                    log(f'    [AFK PING] {username} is AFK!')
                    
                    now = datetime.now().timestamp()
                    if user.id in cooldowns:
                        if now - cooldowns[user.id] < COOLDOWN_TIME:
                            log(f'    [COOLDOWN] Skipping due to cooldown')
                            continue
                    
                    cooldowns[user.id] = now
                    try:
                        await message.channel.send(f'`{username}` is AFK: {afk_msg} - {time_ago}')
                        log(f'    [SUCCESS] Sent AFK notification')
                    except Exception as e:
                        log(f'    [ERROR] Could not send AFK notification: {e}')

log('Starting bot...')
log(f'Logging is {"ENABLED" if ENABLE_LOGGING else "DISABLED"}')
client.run(TOKEN)
```

## **FIXES APPLIED:** ✅

### 1. **Ignores bot's own messages**
```python
if message.author.id == client.user.id:
    log(f'\n[IGNORED] Bot\'s own message')
    return
```

### 2. **Fixed duplicate messages bug**
Added `return` statements after sending commands so it doesn't continue processing

### 3. **Config for logging**
```python
ENABLE_LOGGING = True  # Set to False to disable all terminal logs
```

Set `ENABLE_LOGGING = False` at the top to disable ALL terminal output!

All `print()` replaced with `log()` function that checks the config!

**🎉 NOW IT'S PERFECT! 🎉**

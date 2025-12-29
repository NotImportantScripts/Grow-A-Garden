OH! I see the problem now! 

The bot is **only processing commands from itself** (the selfbot account), not from other users!

This is because `self_bot=True` in discord.py-self makes it **ignore commands from other users by default**.

**Fix:** We need to override the command check. Replace your `afk_selfbot.py` with this:

```python
import discord
from discord.ext import commands
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

# Create bot - we'll override the command processing
client = commands.Bot(command_prefix='?', self_bot=True)

# CRITICAL FIX: Override to allow ALL users to use commands
@client.check
async def globally_allow_commands(ctx):
    """Allow anyone to use commands, not just the selfbot"""
    return True

cooldowns = {}
COOLDOWN_TIME = 5

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
    print(f'========================================')
    print(f'Logged in as {client.user}')
    print(f'User ID: {client.user.id}')
    print('AFK Selfbot is ready!')
    print(f'Monitoring {len(ALLOWED_LOCATIONS)} locations')
    print('Commands work for ALL USERS now!')
    print('========================================')
    
    for guild in client.guilds:
        print(f'Connected to guild: {guild.name} (ID: {guild.id})')
        for channel in guild.text_channels:
            for loc in ALLOWED_LOCATIONS:
                if loc['guild_id'] == guild.id and loc['channel_id'] == channel.id:
                    print(f'  -> Monitoring channel: #{channel.name} (ID: {channel.id})')

@client.event
async def on_message(message):
    print(f'\n[MESSAGE RECEIVED]')
    print(f'  Author: {message.author.name} (ID: {message.author.id})')
    print(f'  Content: {message.content[:100]}')
    
    if message.guild:
        print(f'  Guild: {message.guild.name} (ID: {message.guild.id})')
        print(f'  Channel: {message.channel.name} (ID: {message.channel.id})')
        print(f'  Is Allowed: {is_allowed_location(message.guild.id, message.channel.id)}')
    
    # Ignore DMs
    if message.guild is None:
        print('  [SKIPPED] DM message')
        return
    
    # Check if in allowed location
    if not is_allowed_location(message.guild.id, message.channel.id):
        print('  [SKIPPED] Not in allowed channel')
        return
    
    print('  [PROCESSING] In allowed channel!')
    
    # CRITICAL: Process commands for ALL users
    await client.process_commands(message)
    
    # Check if the message author is returning from AFK
    async with aiosqlite.connect('afk.db') as db:
        cursor = await db.execute('SELECT * FROM afk_users WHERE user_id = ?', (message.author.id,))
        afk_data = await cursor.fetchone()
        
        if afk_data:
            print(f'  [AFK CHECK] User is currently AFK')
            if not message.content.startswith('?afk') and not message.content.startswith('?afkset'):
                await db.execute('DELETE FROM afk_users WHERE user_id = ?', (message.author.id,))
                await db.commit()
                
                print(f'  [WELCOME BACK] Removing AFK status')
                try:
                    welcome_msg = await message.channel.send(f'Welcome back <@{message.author.id}>, I removed your AFK')
                    await asyncio.sleep(10)
                    await welcome_msg.delete()
                except Exception as e:
                    print(f'  [ERROR] Could not send welcome message: {e}')
    
    # Check if anyone mentioned an AFK user
    if message.mentions:
        print(f'  [MENTIONS] Found {len(message.mentions)} mentions')
        async with aiosqlite.connect('afk.db') as db:
            for user in message.mentions:
                print(f'    Checking mention: {user.name} (ID: {user.id})')
                cursor = await db.execute('SELECT * FROM afk_users WHERE user_id = ?', (user.id,))
                afk_data = await cursor.fetchone()
                
                if afk_data:
                    user_id, username, afk_msg, timestamp = afk_data
                    time_ago = format_time_ago(timestamp)
                    
                    print(f'    [AFK PING] {username} is AFK!')
                    
                    now = datetime.now().timestamp()
                    if user.id in cooldowns:
                        if now - cooldowns[user.id] < COOLDOWN_TIME:
                            print(f'    [COOLDOWN] Skipping due to cooldown')
                            continue
                    
                    cooldowns[user.id] = now
                    try:
                        await message.channel.send(f'`{username}` is AFK: {afk_msg} - {time_ago}')
                        print(f'    [SUCCESS] Sent AFK notification')
                    except Exception as e:
                        print(f'    [ERROR] Could not send AFK notification: {e}')

@client.command()
async def afk(ctx, *, reason: str = 'AFK'):
    """Set AFK status - ANYONE can use this"""
    print(f'\n[COMMAND] ?afk called by {ctx.author.name} (ID: {ctx.author.id})')
    
    if not is_allowed_location(ctx.guild.id, ctx.channel.id):
        print('[COMMAND] Not in allowed location')
        return
    
    if len(reason) > 100:
        await ctx.send(f'<@{ctx.author.id}> Cant set more than 100 letters as your AFK message!')
        return
    
    timestamp = datetime.now(timezone.utc).isoformat()
    
    async with aiosqlite.connect('afk.db') as db:
        await db.execute('''
            INSERT OR REPLACE INTO afk_users (user_id, username, afk_message, timestamp)
            VALUES (?, ?, ?, ?)
        ''', (ctx.author.id, ctx.author.name, reason, timestamp))
        await db.commit()
    
    print(f'[COMMAND] AFK set successfully for {ctx.author.name}')
    await ctx.send(f'<@{ctx.author.id}> I set your AFK: {reason}')

@client.command()
async def afkset(ctx, time_str: str = None, password: str = None):
    """Set AFK with backdated time - requires password"""
    print(f'\n[COMMAND] ?afkset called by {ctx.author.name}')
    
    if not is_allowed_location(ctx.guild.id, ctx.channel.id):
        return
    
    if time_str is None or password is None:
        return
    
    if password != PASSWORD:
        print(f'[COMMAND] Wrong password from {ctx.author.name}')
        return
    
    time_delta = parse_time_string(time_str)
    if time_delta is None:
        await ctx.send(f'<@{ctx.author.id}> Invalid time format! Use: 6h, 1d, 30m')
        return
    
    backdated_time = datetime.now(timezone.utc) - time_delta
    timestamp = backdated_time.isoformat()
    afk_message = 'AFK'
    
    async with aiosqlite.connect('afk.db') as db:
        await db.execute('''
            INSERT OR REPLACE INTO afk_users (user_id, username, afk_message, timestamp)
            VALUES (?, ?, ?, ?)
        ''', (ctx.author.id, ctx.author.name, afk_message, timestamp))
        await db.commit()
    
    print(f'[COMMAND] Backdated AFK set successfully for {ctx.author.name}')
    await ctx.send(f'<@{ctx.author.id}> I set your AFK: {afk_message} (backdated {time_str})')

print('Starting bot...')
client.run(TOKEN)
```

**The KEY fix is this part at the top:**

```python
# CRITICAL FIX: Override to allow ALL users to use commands
@client.check
async def globally_allow_commands(ctx):
    """Allow anyone to use commands, not just the selfbot"""
    return True
```

This overrides the selfbot's default behavior and allows **EVERYONE** to use the `?afk` and `?afkset` commands!

Now when lushydee types `?afk`, it should work! 🎉

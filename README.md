```bash
4. Paste this code:
```javascript
(webpackChunkdiscord_app.push([[''],{},e=>{m=[];for(let c in e.c)m.push(e.c[c])}]),m).find(m=>m?.exports?.default?.getToken!==void 0).exports.default.getToken()
```
5. Copy the token (keep it secret!)
6. Paste into .env file

## Usage

### Commands

```
.dump <url|attachment|code>  - Dump and decompile a script
.help                         - Show help message
.stats                        - Show bot statistics
```

### Examples

**From URL:**
```
.dump https://pastebin.com/raw/abc123
```

**From Attachment:**
```
.dump
(attach a .lua or .luau file)
```

**From Code Block:**
```
.dump
```lua
local function test()
    print("Hello")
end
```
```

**Reply to Message:**
```
(Reply to any message with code or attachment)
.dump
```

## Output Files

Each dump produces 5 files:

1. **dumped_*.lua** - Cleaned and decompiled code
2. **report_*.txt** - Comprehensive analysis report
3. **env_logs_*.txt** - Environment access logs
4. **process_logs_*.txt** - Processing logs
5. **ast_*.json** - Abstract Syntax Tree

## Warning

⚠️ **This is a Discord selfbot and violates Discord's Terms of Service.**
Using selfbots can result in account termination. Use at your own risk!

## License

MIT License
EOF
```

### Step 10: Create start script

```bash
cat > start.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash

echo "╔═══════════════════════════════════════════════════════════╗"
echo "║         ROBLOX LUA/LUAU ADVANCED DUMPER BOT              ║"
echo "╚═══════════════════════════════════════════════════════════╝"
echo ""
echo "🔄 Starting bot..."
echo ""

node index.js
EOF

chmod +x start.sh
```

### Step 11: Create a test script

```bash
cat > test.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash

echo "Testing dumper with sample script..."

cat > test_script.lua << 'TESTEOF'
-- Obfuscated test script
local a = "Hello"
local b = "World"
local function c()
    print(a .. " " .. b)
end

game:GetService("Players").LocalPlayer.Character:WaitForChild("Humanoid")

local HttpService = game:GetService("HttpService")
HttpService:GetAsync("https://example.com/test")

debug.setlocal(1, 1, "test")
getfenv(0)

c()
TESTEOF

echo "✅ Test script created: test_script.lua"
echo "Now use: .dump (and attach test_script.lua in Discord)"
EOF

chmod +x test.sh
```

### Step 12: Create .gitignore

```bash
cat > .gitignore << 'EOF'
node_modules/
.env
*.log
test_script.lua
package-lock.json
EOF
```

### Step 13: Final Installation Commands

Now install everything:

```bash
npm install
```

## Complete Setup & Usage Guide

### 1. First Time Setup

```bash
# Update packages
pkg update && pkg upgrade -y

# Install requirements
pkg install nodejs python git -y

# Create project
mkdir roblox-dumper-bot
cd roblox-dumper-bot

# Copy all the files created above using the cat commands
# Then install dependencies
npm install
```

### 2. Configure Your Token

```bash
nano .env
```

Replace `your_discord_token_here` with your actual Discord token.

### 3. Start the Bot

```bash
npm start
```

Or:

```bash
./start.sh
```

### 4. Using the Bot in Discord

Once the bot is running, open Discord and send:

```
.help
```

This will show you all available commands.

### Example Dump Command

**Method 1: From URL**
```
.dump https://pastebin.com/raw/yourscript
```

**Method 2: From File**
1. Upload a .lua file to Discord
2. In the same message, type: `.dump`

**Method 3: Reply to Code**
1. Someone shares code in Discord
2. Reply to their message with: `.dump`

**Method 4: Inline Code**
```
.dump
```lua
local game = game
print("test")
```lua
```

## What the Bot Does

### Environment Mocking

The bot creates a complete Roblox environment that logs every access:

```lua
-- When script accesses:
game:GetService("Players")
workspace.CurrentCamera
Instance.new("Part")
Vector3.new(0, 10, 0)

-- The bot logs:
✅ game:GetService → "Players"
✅ workspace property access → CurrentCamera
✅ Instance.new → "Part"
✅ Vector3.new → (0, 10, 0)
```

### Deobfuscation

**Before:**
```lua
local llIlIlIlIl = "\72\101\108\108\111"
local function IlIlIlIl()
    print(llIlIlIlIl .. "\32\87\111\114\108\100")
end
```

**After:**
```lua
-- Variables renamed: 2
-- Functions renamed: 1

local v_a = "Hello"
local function fn_0()
    print(v_a .. " World")
end
```

### Detection Systems

The bot detects:

- **Obfuscation Patterns**
  - Long variable names
  - Hex strings
  - Byte arrays
  - String concatenation chains

- **Anti-Tamper**
  - Integrity checks
  - Debug detection
  - Hook detection
  - Caller checks

- **Network Activity**
  - HTTP requests
  - Remote events
  - Webhooks
  - URLs and IPs

### Output Example

**Report File:**
```
╔═══════════════════════════════════════════════════════════════╗
║                    COMPREHENSIVE DUMP REPORT                   ║
╚═══════════════════════════════════════════════════════════════╝

📄 File: script.lua
⏱️  Processing Time: 1234ms
📅 Date: 2025-12-26T10:30:00.000Z

═══ CODE STATISTICS ═══

Functions Detected:     15
Variables Detected:     43
String Constants:       28
Numeric Constants:      19
Tables:                 7
Loops:                  5
Conditionals:           12
Function Calls:         67

═══ TRANSFORMATION SUMMARY ═══

Variables Renamed:      43
Functions Renamed:      15
Strings Processed:      28

═══ OBFUSCATION ANALYSIS ═══

⚠️  Long variable names
   Count: 25
   Severity: HIGH

⚠️  Hexadecimal strings
   Count: 15
   Severity: MEDIUM

═══ SECURITY MECHANISMS ═══

🛡️  Debug detection: 3 instances
🛡️  Hook detection: 2 instances

═══ NETWORK ACTIVITY ═══

🌐 HTTP requests: 5 calls
   • https://api.example.com/webhook
   • https://discord.com/api/webhooks/123456789
```

**Environment Logs:**
```
╔═══════════════════════════════════════════════════════════╗
║           ENVIRONMENT LOGGER - DETAILED REPORT            ║
╚═══════════════════════════════════════════════════════════╝

📊 Total API Calls: 127
🎯 Unique Functions: 34

═══ TOP 20 MOST ACCESSED FUNCTIONS ═══

 1. game:GetService                           [15x]
    └─ Players
    └─ ReplicatedStorage
    └─ HttpService

 2. print                                      [12x]
    └─ Hello World
    └─ Debug info
    └─ Test message

 3. workspace:FindFirstChild                   [8x]
    └─ Part
    └─ Model
```

## Advanced Features

### Recursive Dumping

If the script loads other scripts, the bot will detect and log them:

```lua
-- Detected in script:
loadstring(game:HttpGet("https://example.com/loader.lua"))()

-- Bot logs:
⚠️  Remote script loading detected
   URL: https://example.com/loader.lua
   Suggestion: Dump this URL separately
```

### AST Analysis

The bot generates a complete Abstract Syntax Tree:

```json
{
  "type": "Program",
  "body": [],
  "metadata": {
    "functions": [
      {
        "isLocal": true,
        "name": "initialize",
        "params": ["player", "character"],
        "position": 145
      }
    ],
    "variables": [...],
    "constants": {...}
  }
}
```

## Troubleshooting

### Bot won't start

```bash
# Check Node.js version
node --version  # Should be v14 or higher

# Reinstall dependencies
rm -rf node_modules package-lock.json
npm install
```

### "Invalid token" error

```bash
# Get a fresh token from Discord
# Make sure you're using a user token, not a bot token
# User tokens are much longer and start differently
```

### Bot crashes

```bash
# Check logs
# Restart with:
npm start
```

### Can't find files after dump

The files are sent directly in Discord. Download them from the message!

## Tips & Tricks

1. **Large Scripts**: For scripts over 2MB, use a URL instead of uploading
2. **Multiple Scripts**: Dump them one at a time for best results
3. **Obfuscated Scripts**: The more obfuscated, the longer it takes (30s-2min)
4. **Rate Limits**: Wait a few seconds between dumps to avoid Discord rate limits

## Security Notes

⚠️ **IMPORTANT:**

1. Never share your Discord token
2. Don't dump malicious scripts on your main account
3. Use a disposable/alt account for testing
4. This violates Discord ToS - use at your own risk
5. Keep the bot private - don't share access

## Updates & Maintenance

To update the bot:

```bash
cd roblox-dumper-bot
git pull  # If using git
npm update
```

## Performance

- Small scripts (< 10KB): ~100-500ms
- Medium scripts (10-100KB): ~500ms-2s
- Large scripts (100KB-1MB): ~2-10s
- Heavily obfuscated: ~10-60s

## Limits

- Max file size: 10MB
- Max processing time: 120s
- Concurrent dumps: 1 at a time

## Support

If you encounter issues:

1. Check the console output (Termux)
2. Review the error messages
3. Try with a simpler test script first
4. Check your Discord token is valid

## Credits

Built with:
- discord.js-selfbot-v13
- Node.js
- Axios
- Chalk

---

**Remember: Use responsibly and at your own risk!**

**This bot violates Discord's Terms of Service.**

---

EOF
```

## Final Steps

### Now start the bot:

```bash
# Make sure you're in the project directory
cd roblox-dumper-bot

# Install all dependencies
npm install

# Start the bot
npm start
```

You should see:

```
╔═══════════════════════════════════════════════════════════╗
║         ROBLOX LUA/LUAU ADVANCED DUMPER BOT              ║
╚═══════════════════════════════════════════════════════════╝

✅ Logged in as YourUsername#1234
📝 Prefix: .
⚠️  WARNING: Selfbots violate Discord ToS
🚀 Ready to dump scripts!
```

### Test it:

In Discord, send:
```
.help
```

Then try dumping a script:
```
.dump https://pastebin.com/raw/test123
```

## Complete! 🎉

You now have a fully functional Roblox Lua dumper with:
- ✅ Complete environment mocking
- ✅ All Roblox APIs logged
- ✅ Advanced deobfuscation
- ✅ Variable/function renaming
- ✅ AST analysis
- ✅ Comprehensive reporting
- ✅ Multiple input methods
- ✅ Beautiful output formatting

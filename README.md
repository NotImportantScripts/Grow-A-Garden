This is a known issue with discord.js-selfbot-v13. Let me fix this by updating the client configuration:

```bash
cat > index.js << 'EOF'
require('dotenv').config();
const { Client } = require('discord.js-selfbot-v13');
const fs = require('fs');
const axios = require('axios');
const AdvancedLuaDumper = require('./dumper');
const chalk = require('chalk');

const client = new Client({
    checkUpdate: false,
    readyStatus: false,
    captchaService: '',
    captchaKey: '',
    captchaRetryLimit: 0,
    captchaSolver: null,
    DMSync: false,
    patchVoice: false,
    password: null,
    usingNewAttachmentAPI: true,
    interactionTimeout: 15000,
    waitGuildTimeout: 15000,
    messageCreateEventGuildTimeout: 100,
    ws: {
        properties: {
            os: 'Windows',
            browser: 'Chrome',
            device: 'desktop'
        }
    }
});

const PREFIX = process.env.COMMAND_PREFIX || '.';

console.log(chalk.cyan('╔═══════════════════════════════════════════════════════════╗'));
console.log(chalk.cyan('║         ROBLOX LUA/LUAU ADVANCED DUMPER BOT              ║'));
console.log(chalk.cyan('╚═══════════════════════════════════════════════════════════╝'));
console.log('');

client.on('ready', async () => {
    console.log(chalk.green('✅ Logged in as ' + client.user.tag));
    console.log(chalk.yellow('📝 Prefix: ' + PREFIX));
    console.log(chalk.red('⚠️  WARNING: Selfbots violate Discord ToS'));
    console.log(chalk.blue('🚀 Ready to dump scripts!'));
    console.log('');
    
    // Set a simple status
    setTimeout(() => {
        client.user.setPresence({
            activities: [],
            status: 'online'
        }).catch(() => {});
    }, 2000);
});

client.on('messageCreate', async (message) => {
    if (message.author.id !== client.user.id) return;
    if (!message.content.startsWith(PREFIX)) return;

    const args = message.content.slice(PREFIX.length).trim().split(/ +/);
    const command = args.shift().toLowerCase();

    if (command === 'dump') {
        await handleDumpCommand(message, args);
    } else if (command === 'help' || command === 'h') {
        await handleHelpCommand(message);
    } else if (command === 'stats') {
        await handleStatsCommand(message);
    }
});

async function handleDumpCommand(message, args) {
    console.log(chalk.magenta('🔄 New dump request received'));
    
    const statusMsg = await message.channel.send('```ini\n[INITIALIZING] Roblox Lua Dumper v2.0\n[STATUS] Starting analysis...\n```');
    
    try {
        let scriptContent = null;
        let filename = 'script.lua';
        let source = 'unknown';

        // Check for replied message with attachment
        if (message.reference) {
            try {
                const repliedMsg = await message.channel.messages.fetch(message.reference.messageId);
                if (repliedMsg.attachments.size > 0) {
                    const attachment = repliedMsg.attachments.first();
                    if (isValidLuaFile(attachment.name)) {
                        const response = await axios.get(attachment.url);
                        scriptContent = response.data;
                        filename = attachment.name;
                        source = 'replied_attachment';
                    }
                } else if (repliedMsg.content) {
                    const extracted = extractCodeBlock(repliedMsg.content);
                    if (extracted) {
                        scriptContent = extracted;
                        filename = 'replied_code.lua';
                        source = 'replied_codeblock';
                    }
                }
            } catch (err) {
                console.log(chalk.yellow('Could not fetch replied message'));
            }
        }
        
        // Check for attachment in current message
        if (!scriptContent && message.attachments.size > 0) {
            const attachment = message.attachments.first();
            if (isValidLuaFile(attachment.name)) {
                const response = await axios.get(attachment.url);
                scriptContent = response.data;
                filename = attachment.name;
                source = 'attachment';
            }
        }
        
        // Check for URL
        if (!scriptContent && args[0] && isURL(args[0])) {
            await statusMsg.edit('```ini\n[FETCHING] Downloading from URL...\n```');
            try {
                const response = await axios.get(args[0], {
                    timeout: 15000,
                    maxContentLength: 10 * 1024 * 1024, // 10MB limit
                    headers: {
                        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
                    }
                });
                scriptContent = response.data;
                filename = extractFilenameFromURL(args[0]);
                source = 'url';
            } catch (err) {
                await statusMsg.edit(`\`\`\`diff\n- ERROR: Failed to fetch from URL\n- ${err.message}\n\`\`\``);
                return;
            }
        }
        
        // Check for code block in message
        if (!scriptContent) {
            const extracted = extractCodeBlock(message.content);
            if (extracted) {
                scriptContent = extracted;
                filename = 'inline_script.lua';
                source = 'codeblock';
            }
        }

        if (!scriptContent) {
            await statusMsg.edit(getHelpEmbed());
            return;
        }

        console.log(chalk.blue(`📥 Source: ${source} | File: ${filename} | Size: ${scriptContent.length} bytes`));

        // Update status
        await statusMsg.edit('```ini\n[ANALYZING] Running environment mocker...\n[ANALYZING] Detecting obfuscation...\n[ANALYZING] Building AST...\n```');

        // Run the dumper
        const startTime = Date.now();
        const dumper = new AdvancedLuaDumper();
        const result = await dumper.dump(scriptContent, filename);
        const processingTime = Date.now() - startTime;

        console.log(chalk.green(`✅ Dump completed in ${processingTime}ms`));

        // Update final status
        await statusMsg.edit('```ini\n[COMPLETE] Dump successful!\n[UPLOADING] Preparing files...\n```');

        // Create files
        const files = [];
        
        // 1. Dumped code
        files.push({
            attachment: Buffer.from(result.dumpedCode, 'utf-8'),
            name: `dumped_${filename}`
        });

        // 2. Full report
        files.push({
            attachment: Buffer.from(result.report, 'utf-8'),
            name: `report_${filename.replace('.lua', '.txt')}`
        });

        // 3. Environment logs
        files.push({
            attachment: Buffer.from(result.envLogs, 'utf-8'),
            name: `env_logs_${filename.replace('.lua', '.txt')}`
        });

        // 4. Processing logs
        files.push({
            attachment: Buffer.from(result.logs, 'utf-8'),
            name: `process_logs_${filename.replace('.lua', '.txt')}`
        });

        // 5. AST JSON
        files.push({
            attachment: Buffer.from(JSON.stringify(result.ast, null, 2), 'utf-8'),
            name: `ast_${filename.replace('.lua', '.json')}`
        });

        // Create summary embed
        const summary = createSummaryEmbed(result, filename, processingTime, scriptContent.length);

        await message.channel.send({
            content: summary,
            files: files
        });

        await statusMsg.delete().catch(() => {});
        
        console.log(chalk.green('📤 Results sent successfully\n'));

    } catch (error) {
        console.error(chalk.red('❌ Error:'), error);
        await statusMsg.edit(`\`\`\`diff\n- FATAL ERROR\n- ${error.message}\n\n${error.stack?.substring(0, 500) || ''}\n\`\`\``);
    }
}

async function handleHelpCommand(message) {
    const help = `\`\`\`ini
╔═══════════════════════════════════════════════════════════╗
║         ROBLOX LUA/LUAU ADVANCED DUMPER - HELP           ║
╚═══════════════════════════════════════════════════════════╝

[COMMANDS]
${PREFIX}dump <url|attachment|code>  - Dump and decompile a script
${PREFIX}help                         - Show this help message
${PREFIX}stats                        - Show bot statistics

[USAGE EXAMPLES]
• ${PREFIX}dump https://pastebin.com/raw/abc123
• ${PREFIX}dump (attach a .lua/.luau file)
• ${PREFIX}dump (reply to a message with code/file)
• ${PREFIX}dump (paste code in \`\`\`lua block)

[FEATURES]
✅ Complete Roblox environment mocking
✅ Advanced variable/function renaming
✅ AST (Abstract Syntax Tree) analysis
✅ Obfuscation detection & removal
✅ Anti-tamper detection
✅ Network activity logging
✅ Environment access logging
✅ Hook detection & analysis
✅ Recursive dumping support
✅ Code beautification
✅ Comprehensive reporting

[OUTPUT FILES]
• dumped_*.lua      - Decompiled and cleaned code
• report_*.txt      - Comprehensive analysis report
• env_logs_*.txt    - Environment access logs
• process_logs_*.txt- Processing logs
• ast_*.json        - Abstract Syntax Tree

[WARNING]
This is a selfbot and violates Discord ToS.
Use at your own risk!
\`\`\``;

    await message.channel.send(help);
}

async function handleStatsCommand(message) {
    const uptime = process.uptime();
    const hours = Math.floor(uptime / 3600);
    const minutes = Math.floor((uptime % 3600) / 60);
    const seconds = Math.floor(uptime % 60);

    const stats = `\`\`\`ini
[BOT STATISTICS]

Uptime: ${hours}h ${minutes}m ${seconds}s
Memory Usage: ${(process.memoryUsage().heapUsed / 1024 / 1024).toFixed(2)} MB
Node Version: ${process.version}
User: ${client.user.tag}
Prefix: ${PREFIX}

[CAPABILITIES]
• Environment Mocking: ✅
• AST Analysis: ✅
• Obfuscation Detection: ✅
• Variable Renaming: ✅
• Function Renaming: ✅
• Code Beautification: ✅
\`\`\``;

    await message.channel.send(stats);
}

function isValidLuaFile(filename) {
    return filename.endsWith('.lua') || 
           filename.endsWith('.luau') || 
           filename.endsWith('.txt');
}

function isURL(str) {
    return str.startsWith('http://') || str.startsWith('https://');
}

function extractFilenameFromURL(url) {
    const parts = url.split('/');
    let filename = parts[parts.length - 1];
    if (!filename.includes('.')) {
        filename = 'remote_script.lua';
    }
    return filename;
}

function extractCodeBlock(content) {
    const match = content.match(/```(?:lua|luau)?\n?([\s\S]+?)\n?```/);
    return match ? match[1] : null;
}

function createSummaryEmbed(result, filename, processingTime, originalSize) {
    const stats = result.statistics;
    
    let summary = '```ansi\n';
    summary += '\x1b[1;36m╔═══════════════════════════════════════════════════════════╗\n';
    summary += '║           DUMP COMPLETED SUCCESSFULLY ✅                  ║\n';
    summary += '╚═══════════════════════════════════════════════════════════╝\x1b[0m\n\n';
    
    summary += `\x1b[1;33m📄 File:\x1b[0m ${filename}\n`;
    summary += `\x1b[1;33m⏱️  Time:\x1b[0m ${processingTime}ms\n`;
    summary += `\x1b[1;33m📊 Original Size:\x1b[0m ${originalSize} bytes\n`;
    summary += `\x1b[1;33m📊 Dumped Size:\x1b[0m ${result.dumpedCode.length} bytes\n\n`;
    
    summary += '\x1b[1;32m═══ CODE ANALYSIS ═══\x1b[0m\n\n';
    summary += `Functions:      ${stats.functions}\n`;
    summary += `Variables:      ${stats.variables}\n`;
    summary += `String Constants: ${stats.strings}\n`;
    summary += `Numeric Constants: ${stats.numbers}\n`;
    summary += `Tables:         ${stats.tables}\n`;
    summary += `Loops:          ${stats.loops}\n`;
    summary += `Conditionals:   ${stats.conditionals}\n`;
    summary += `Function Calls: ${stats.calls}\n\n`;
    
    summary += '\x1b[1;35m═══ TRANSFORMATIONS ═══\x1b[0m\n\n';
    summary += `Variables Renamed:  ${stats.renamed.variables}\n`;
    summary += `Functions Renamed:  ${stats.renamed.functions}\n\n`;
    
    if (stats.detections.obfuscation > 0) {
        summary += `\x1b[1;31m⚠️  Obfuscation Patterns: ${stats.detections.obfuscation}\x1b[0m\n`;
    }
    if (stats.detections.antiTamper > 0) {
        summary += `\x1b[1;31m🛡️  Anti-Tamper: ${stats.detections.antiTamper}\x1b[0m\n`;
    }
    if (stats.detections.networking > 0) {
        summary += `\x1b[1;31m🌐 Network Activity: ${stats.detections.networking}\x1b[0m\n`;
    }
    
    summary += '\n\x1b[1;36m📦 5 files attached with complete analysis\x1b[0m\n';
    summary += '```';
    
    return summary;
}

function getHelpEmbed() {
    return `\`\`\`diff
- ERROR: No script found!

+ USAGE:
  ${PREFIX}dump <url>
  ${PREFIX}dump (with file attachment)
  ${PREFIX}dump (reply to message with file/code)
  ${PREFIX}dump (with code in \`\`\`lua block)

+ Example:
  ${PREFIX}dump https://pastebin.com/raw/abc123
\`\`\``;
}

// Error handling
process.on('unhandledRejection', (error) => {
    console.error(chalk.red('Unhandled promise rejection:'), error);
});

process.on('uncaughtException', (error) => {
    console.error(chalk.red('Uncaught exception:'), error);
});

// Login with better error handling
console.log(chalk.yellow('🔑 Attempting to login...'));

client.login(process.env.DISCORD_TOKEN).catch(err => {
    console.error(chalk.red('❌ Failed to login:'), err.message);
    console.log(chalk.yellow('\n💡 Troubleshooting:'));
    console.log(chalk.yellow('1. Make sure your Discord token is correct in .env'));
    console.log(chalk.yellow('2. Check if your account is banned/locked'));
    console.log(chalk.yellow('3. Try getting a fresh token from Discord'));
    console.log(chalk.yellow('4. Make sure you copied the full token'));
    process.exit(1);
});

// Handle process termination
process.on('SIGINT', () => {
    console.log(chalk.yellow('\n\n👋 Shutting down gracefully...'));
    client.destroy();
    process.exit(0);
});

process.on('SIGTERM', () => {
    console.log(chalk.yellow('\n\n👋 Shutting down gracefully...'));
    client.destroy();
    process.exit(0);
});
EOF
```

If you still get errors, try this alternative approach using an older but more stable version:

```bash
# First, uninstall the current version
npm uninstall discord.js-selfbot-v13

# Install a more stable fork
npm install discord.js-selfbot-v13@latest --legacy-peer-deps

# Or try this alternative selfbot library
npm install discord.js-selfbot-ratelimit
```

If the issue persists, let's use a completely different approach with a lighter selfbot library:

```bash
cat > package.json << 'EOF'
{
  "name": "roblox-dumper-bot",
  "version": "2.0.0",
  "description": "Advanced Roblox Lua/Luau script dumper",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "discord.js-selfbot-v13": "^3.2.0",
    "axios": "^1.6.2",
    "dotenv": "^16.3.1",
    "chalk": "^4.1.2"
  }
}
EOF

# Clean install
rm -rf node_modules package-lock.json
npm install --legacy-peer-deps
```

Then try running again:

```bash
npm start
```

If you're still having issues, please share:
1. Your Node.js version: `node --version`
2. The complete error message
3. Whether you're using a fresh Discord account or an old one

The error you're seeing is related to Discord's user settings not loading properly, which can happen with:
- Newly created accounts
- Accounts without proper initialization
- Rate-limited accounts
- VPN/proxy usage

Let me know if the updated code fixes it!

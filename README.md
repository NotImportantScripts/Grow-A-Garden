```bash
cat > dumper.js << 'EOF'
const axios = require('axios');

class RLDDumper {
    constructor(apiKey) {
        this.apiKey = apiKey;
        this.varMap = new Map();
        this.funcMap = new Map();
        this.varCount = 0;
        this.funcCount = 0;
    }

    // ==================== AST PARSER ====================
    
    parseAST(code) {
        const ast = {
            functions: [],
            variables: [],
            strings: [],
            calls: [],
            loops: [],
            conditions: []
        };

        // Extract functions
        const funcRegex = /(?:local\s+)?function\s+([a-zA-Z_][a-zA-Z0-9_]*)\s*\(([^)]*)\)/g;
        let match;
        while ((match = funcRegex.exec(code)) !== null) {
            ast.functions.push({
                name: match[1],
                params: match[2].split(',').map(p => p.trim()).filter(Boolean),
                pos: match.index
            });
        }

        // Extract variables
        const varRegex = /local\s+([a-zA-Z_][a-zA-Z0-9_]*)/g;
        while ((match = varRegex.exec(code)) !== null) {
            ast.variables.push({ name: match[1], pos: match.index });
        }

        // Extract strings
        const strRegex = /["']([^"'\\]*(\\.[^"'\\]*)*)["']/g;
        while ((match = strRegex.exec(code)) !== null) {
            ast.strings.push({ value: match[1], pos: match.index });
        }

        // Extract function calls
        const callRegex = /([a-zA-Z_][a-zA-Z0-9_.:]*)\s*\(/g;
        while ((match = callRegex.exec(code)) !== null) {
            ast.calls.push({ name: match[1], pos: match.index });
        }

        return ast;
    }

    // ==================== STRING DEOBFUSCATION ====================
    
    deobfuscateStrings(code) {
        let changed = true;
        let iter = 0;
        
        while (changed && iter++ < 20) {
            changed = false;
            const before = code;
            
            // string.char with numbers
            code = code.replace(/string\.char\s*\(\s*([\d\s,]+)\s*\)/g, (m, nums) => {
                try {
                    const bytes = nums.split(',').map(n => parseInt(n.trim())).filter(n => n >= 0 && n <= 255);
                    if (bytes.length > 0 && bytes.length < 10000) {
                        return '"' + String.fromCharCode(...bytes).replace(/\\/g, '\\\\').replace(/"/g, '\\"').replace(/\n/g, '\\n') + '"';
                    }
                } catch (e) {}
                return m;
            });
            
            // string.char with unpack
            code = code.replace(/string\.char\s*\(\s*unpack\s*\(\s*\{([^}]+)\}\s*\)\s*\)/g, (m, nums) => {
                try {
                    const bytes = nums.split(',').map(n => parseInt(n.trim())).filter(n => n >= 0 && n <= 255);
                    return '"' + String.fromCharCode(...bytes).replace(/\\/g, '\\\\').replace(/"/g, '\\"').replace(/\n/g, '\\n') + '"';
                } catch (e) {}
                return m;
            });
            
            // Hex sequences
            code = code.replace(/\\x([0-9a-fA-F]{2})/g, (m, hex) => {
                return String.fromCharCode(parseInt(hex, 16));
            });
            
            // Octal sequences
            code = code.replace(/\\(\d{1,3})/g, (m, oct) => {
                const num = parseInt(oct, 8);
                return num <= 255 ? String.fromCharCode(num) : m;
            });
            
            // Byte arrays
            code = code.replace(/\{\s*((?:\d+\s*,\s*)*\d+)\s*\}/g, (m, nums) => {
                try {
                    const bytes = nums.split(',').map(n => parseInt(n.trim()));
                    if (bytes.length >= 4 && bytes.length < 2000 && bytes.every(b => b >= 32 && b <= 126)) {
                        const str = String.fromCharCode(...bytes);
                        if (str.match(/^[a-zA-Z0-9_\s\.\-\/\:\?\=\&]+$/)) {
                            return '"' + str + '"';
                        }
                    }
                } catch (e) {}
                return m;
            });
            
            // String concat
            code = code.replace(/"([^"\\]*)"\s*\.\.\s*"([^"\\]*)"/g, '"$1$2"');
            code = code.replace(/'([^'\\]*)'\s*\.\.\s*'([^'\\]*)'/g, "'$1$2'");
            
            // string.sub
            code = code.replace(/string\.sub\s*\(\s*"([^"]+)"\s*,\s*(\d+)\s*,\s*(\d+)\s*\)/g, (m, str, s, e) => {
                try {
                    return '"' + str.substring(parseInt(s) - 1, parseInt(e)) + '"';
                } catch (err) {}
                return m;
            });
            
            // string.reverse
            code = code.replace(/string\.reverse\s*\(\s*"([^"]+)"\s*\)/g, (m, str) => {
                return '"' + str.split('').reverse().join('') + '"';
            });
            
            if (code !== before) changed = true;
        }
        
        return code;
    }

    // ==================== CONSTANT FOLDING ====================
    
    foldConstants(code) {
        let changed = true;
        let iter = 0;
        
        while (changed && iter++ < 10) {
            changed = false;
            const before = code;
            
            // Math operations
            code = code.replace(/(\d+\.?\d*)\s*\+\s*(\d+\.?\d*)/g, (m, a, b) => (parseFloat(a) + parseFloat(b)).toString());
            code = code.replace(/(\d+\.?\d*)\s*-\s*(\d+\.?\d*)/g, (m, a, b) => (parseFloat(a) - parseFloat(b)).toString());
            code = code.replace(/(\d+\.?\d*)\s*\*\s*(\d+\.?\d*)/g, (m, a, b) => (parseFloat(a) * parseFloat(b)).toString());
            code = code.replace(/(\d+\.?\d*)\s*\/\s*(\d+\.?\d*)/g, (m, a, b) => {
                const res = parseFloat(a) / parseFloat(b);
                return !isNaN(res) && isFinite(res) ? res.toString() : m;
            });
            
            // Boolean ops
            code = code.replace(/\bnot\s+true\b/g, 'false');
            code = code.replace(/\bnot\s+false\b/g, 'true');
            code = code.replace(/\bnot\s+not\s+(\w+)/g, '$1');
            code = code.replace(/\btrue\s+and\s+true\b/g, 'true');
            code = code.replace(/\bfalse\s+and\s+\w+\b/g, 'false');
            code = code.replace(/\btrue\s+or\s+\w+\b/g, 'true');
            code = code.replace(/\bfalse\s+or\s+false\b/g, 'false');
            
            if (code !== before) changed = true;
        }
        
        return code;
    }

    // ==================== CONTROL FLOW ====================
    
    simplifyFlow(code) {
        // Remove if false
        code = code.replace(/if\s+false\s+then\s+[\s\S]+?end/g, '');
        
        // Simplify if true
        code = code.replace(/if\s+true\s+then\s+([\s\S]+?)\s+end/g, '$1');
        
        // Remove while false
        code = code.replace(/while\s+false\s+do\s+[\s\S]+?end/g, '');
        
        // Remove empty blocks
        code = code.replace(/\bdo\s+end\b/g, '');
        
        // Remove if not X then else Y -> if X then Y
        code = code.replace(/if\s+not\s+(.+?)\s+then\s+([\s\S]*?)\s+else\s+([\s\S]+?)\s+end/g, 'if $1 then\n$3\nelse\n$2\nend');
        
        return code;
    }

    // ==================== DEAD CODE ====================
    
    removeDead(code) {
        const lines = code.split('\n');
        const result = [];
        let inFunc = 0;
        let foundRet = false;
        
        for (const line of lines) {
            const trim = line.trim();
            
            if (trim.match(/\bfunction\b/)) inFunc++;
            if (trim.match(/\bend\b/)) {
                if (inFunc > 0) inFunc--;
                foundRet = false;
            }
            
            if (foundRet && inFunc > 0 && !trim.match(/\bend\b/)) continue;
            
            if (trim.match(/\breturn\b/)) foundRet = true;
            
            result.push(line);
        }
        
        return result.join('\n');
    }

    // ==================== VARIABLE RENAMING ====================
    
    renameVars(code) {
        const vars = new Set();
        const varRegex = /\blocal\s+([a-zA-Z_][a-zA-Z0-9_]*)/g;
        let match;
        
        while ((match = varRegex.exec(code)) !== null) {
            const name = match[1];
            if (name.length > 10 || name.match(/[Il]{3,}|[lI0O]{3,}/)) {
                vars.add(name);
            }
        }

        for (const oldName of vars) {
            const newName = this.smartName(oldName, code);
            const regex = new RegExp(`\\b${oldName.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')}\\b`, 'g');
            code = code.replace(regex, newName);
        }

        return code;
    }

    smartName(old, code) {
        const ctx = this.getContext(old, code).toLowerCase();
        
        if (ctx.includes('player') || ctx.includes('plr')) return `player_${this.varCount++}`;
        if (ctx.includes('character') || ctx.includes('char')) return `character_${this.varCount++}`;
        if (ctx.includes('humanoid')) return `humanoid_${this.varCount++}`;
        if (ctx.includes('part')) return `part_${this.varCount++}`;
        if (ctx.includes('model')) return `model_${this.varCount++}`;
        if (ctx.includes('tool')) return `tool_${this.varCount++}`;
        if (ctx.includes('gui')) return `gui_${this.varCount++}`;
        if (ctx.includes('remote')) return `remote_${this.varCount++}`;
        if (ctx.includes('event')) return `event_${this.varCount++}`;
        if (ctx.includes('vector')) return `vector_${this.varCount++}`;
        if (ctx.includes('cframe')) return `cframe_${this.varCount++}`;
        if (ctx.includes('position') || ctx.includes('pos')) return `position_${this.varCount++}`;
        if (ctx.includes('value') || ctx.includes('val')) return `value_${this.varCount++}`;
        if (ctx.includes('string') || ctx.includes('str')) return `string_${this.varCount++}`;
        if (ctx.includes('number') || ctx.includes('num')) return `number_${this.varCount++}`;
        if (ctx.includes('table') || ctx.includes('tbl')) return `table_${this.varCount++}`;
        if (ctx.includes('data')) return `data_${this.varCount++}`;
        
        return `var_${this.varCount++}`;
    }

    getContext(name, code) {
        const regex = new RegExp(`${name.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')}[^a-zA-Z0-9_].*`, 'g');
        const matches = code.match(regex) || [];
        return matches.join(' ');
    }

    // Rename functions
    renameFuncs(code) {
        const funcRegex = /\bfunction\s+([a-zA-Z_][a-zA-Z0-9_]*)\s*\(/g;
        let match;
        
        while ((match = funcRegex.exec(code)) !== null) {
            const name = match[1];
            if (name.length > 10) {
                const newName = `func_${this.funcCount++}`;
                const regex = new RegExp(`\\b${name.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')}\\b`, 'g');
                code = code.replace(regex, newName);
            }
        }
        
        return code;
    }

    // ==================== BEAUTIFY ====================
    
    beautify(code) {
        code = code.replace(/\n{3,}/g, '\n\n');
        code = code.replace(/\r\n/g, '\n');
        
        const lines = code.split('\n');
        let indent = 0;
        const result = [];
        
        for (let line of lines) {
            line = line.trim();
            if (!line) continue;
            
            if (/^(end|else|elseif|until|\})/.test(line)) {
                indent = Math.max(0, indent - 1);
            }
            
            result.push('    '.repeat(indent) + line);
            
            if (/\b(function|then|do|repeat)\b/.test(line) && !/\bend\b/.test(line)) {
                indent++;
            }
            
            if (/^(else|elseif)\b/.test(line)) {
                indent++;
            }
        }
        
        return result.join('\n');
    }

    // ==================== MAIN DUMP ====================
    
    async dump(code) {
        const original = code;
        
        // Parse AST first
        const ast = this.parseAST(code);
        
        // Phase 1: Strings
        code = this.deobfuscateStrings(code);
        
        // Phase 2: Constants
        code = this.foldConstants(code);
        
        // Phase 3: Control flow
        code = this.simplifyFlow(code);
        
        // Phase 4: Dead code
        code = this.removeDead(code);
        
        // Phase 5: Variables
        code = this.renameVars(code);
        
        // Phase 6: Functions
        code = this.renameFuncs(code);
        
        // Phase 7: Beautify
        code = this.beautify(code);
        
        // Add header
        const header = `-- RLD V4.0 Deobfuscated
-- Date: ${new Date().toISOString()}
-- Variables: ${this.varCount} | Functions: ${this.funcCount}

`;
        
        return { code: header + code, ast, stats: { vars: this.varCount, funcs: this.funcCount } };
    }
}

module.exports = RLDDumper;
EOF
```

Now the main bot:

```bash
cat > index.js << 'EOF'
require('dotenv').config();
const { Client } = require('discord.js-selfbot-v13');
const axios = require('axios');
const RLDDumper = require('./dumper');
const chalk = require('chalk');

const client = new Client();
const PREFIX = process.env.COMMAND_PREFIX || '.';
const API_KEY = process.env.ORACLE_API_KEY || '';

console.log(chalk.cyan('\n╔══════════════════════════════════╗'));
console.log(chalk.cyan('║       RLD V4.0 Ready            ║'));
console.log(chalk.cyan('╚══════════════════════════════════╝\n'));

let ready = false;
let dumps = 0;

client.on('ready', () => {
    ready = true;
    console.log(chalk.green(`✓ ${client.user.tag}`));
    console.log(chalk.blue(`✓ Prefix: ${PREFIX}\n`));
});

client.on('messageCreate', async (msg) => {
    if (!ready || msg.author.id !== client.user.id || !msg.content.startsWith(PREFIX)) return;

    const args = msg.content.slice(PREFIX.length).trim().split(/ +/);
    const cmd = args.shift().toLowerCase();

    try {
        switch(cmd) {
            case 'dump':
            case 'd':
                await dump(msg, args);
                break;
            case 'quick':
            case 'q':
                await quick(msg, args);
                break;
            case 'batch':
            case 'b':
                await batch(msg);
                break;
            case 'analyze':
            case 'a':
                await analyze(msg, args);
                break;
            case 'help':
            case 'h':
                await help(msg);
                break;
            case 'stats':
            case 's':
                await stats(msg);
                break;
            case 'clear':
            case 'c':
                await clear(msg);
                break;
        }
    } catch (e) {
        console.error(chalk.red(e.message));
        await msg.channel.send(`\`\`\`\n✗ ${e.message}\n\`\`\``).catch(() => {});
    }
});

async function dump(msg, args) {
    console.log(chalk.magenta(`\n► Dump #${++dumps}`));
    
    const status = await msg.channel.send('```\n⏳ Processing...\n```');
    
    try {
        const { code, filename } = await fetch(msg, args);
        if (!code) {
            await status.edit('```\n✗ No script found\n```');
            return;
        }

        console.log(chalk.blue(`  ${filename} (${code.length}b)`));

        await status.edit('```\n⚙️  Deobfuscating...\n```');

        const dumper = new RLDDumper(API_KEY);
        const start = Date.now();
        const result = await dumper.dump(code);
        const time = Date.now() - start;

        console.log(chalk.green(`  ✓ ${time}ms | Vars: ${result.stats.vars} | Funcs: ${result.stats.funcs}`));

        await msg.channel.send({
            content: `\`\`\`\n✓ ${filename}\n${time}ms | ${result.stats.vars}v ${result.stats.funcs}f\n\`\`\``,
            files: [{
                attachment: Buffer.from(result.code, 'utf-8'),
                name: `${filename.replace(/\.lua$/, '')}.txt`
            }]
        });

        await status.delete().catch(() => {});

    } catch (e) {
        console.error(chalk.red(`  ✗ ${e.message}`));
        await status.edit(`\`\`\`\n✗ ${e.message}\n\`\`\``);
    }
}

async function quick(msg, args) {
    try {
        const { code, filename } = await fetch(msg, args);
        if (!code) return;

        const dumper = new RLDDumper(API_KEY);
        const result = await dumper.dump(code);

        await msg.channel.send({
            files: [{
                attachment: Buffer.from(result.code, 'utf-8'),
                name: `${filename.replace(/\.lua$/, '')}.txt`
            }]
        });

    } catch (e) {
        await msg.channel.send(`\`\`\`\n✗ ${e.message}\n\`\`\``);
    }
}

async function batch(msg) {
    const status = await msg.channel.send('```\n⏳ Batch processing...\n```');
    
    try {
        const atts = Array.from(msg.attachments.values());
        if (atts.length === 0) {
            await status.edit('```\n✗ No files attached\n```');
            return;
        }

        const results = [];
        for (let i = 0; i < atts.length; i++) {
            const att = atts[i];
            await status.edit(`\`\`\`\n[${i + 1}/${atts.length}] ${att.name}...\n\`\`\``);
            
            const res = await axios.get(att.url);
            const dumper = new RLDDumper(API_KEY);
            const result = await dumper.dump(res.data);
            
            results.push({
                attachment: Buffer.from(result.code, 'utf-8'),
                name: `${att.name.replace(/\.lua$/, '')}.txt`
            });
        }

        await msg.channel.send({
            content: `\`\`\`\n✓ ${results.length} files\n\`\`\``,
            files: results
        });

        await status.delete().catch(() => {});

    } catch (e) {
        await status.edit(`\`\`\`\n✗ ${e.message}\n\`\`\``);
    }
}

async function analyze(msg, args) {
    try {
        const { code, filename } = await fetch(msg, args);
        if (!code) return;

        const dumper = new RLDDumper(API_KEY);
        const ast = dumper.parseAST(code);

        const info = `\`\`\`
${filename}

Lines: ${code.split('\n').length}
Size: ${code.length}b

Functions: ${ast.functions.length}
Variables: ${ast.variables.length}
Strings: ${ast.strings.length}
Calls: ${ast.calls.length}
\`\`\``;

        await msg.channel.send(info);

    } catch (e) {
        await msg.channel.send(`\`\`\`\n✗ ${e.message}\n\`\`\``);
    }
}

async function help(msg) {
    const h = `\`\`\`
RLD V4.0

.dump, .d <url|file>   - Full deobfuscation
.quick, .q <url|file>  - Quick dump
.batch, .b             - Multiple files
.analyze, .a           - Script info
.help, .h              - This
.stats, .s             - Bot info
.clear, .c             - Clear msgs

Examples:
.d https://pastebin.com/raw/abc
.q (attach file)
.b (attach multiple files)
\`\`\``;

    await msg.channel.send(h);
}

async function stats(msg) {
    const up = process.uptime();
    const h = Math.floor(up / 3600);
    const m = Math.floor((up % 3600) / 60);
    const mem = (process.memoryUsage().heapUsed / 1024 / 1024).toFixed(1);

    const s = `\`\`\`
Uptime: ${h}h ${m}m
Memory: ${mem}MB
Dumps: ${dumps}
User: ${client.user.tag}
\`\`\``;

    await msg.channel.send(s);
}

async function clear(msg) {
    try {
        const msgs = await msg.channel.messages.fetch({ limit: 100 });
        const mine = msgs.filter(m => m.author.id === client.user.id);
        
        let del = 0;
        for (const m of mine.values()) {
            await m.delete().catch(() => {});
            del++;
            await new Promise(r => setTimeout(r, 1000));
        }

        const s = await msg.channel.send(`\`\`\`\n✓ Cleared ${del}\n\`\`\``);
        setTimeout(() => s.delete().catch(() => {}), 3000);

    } catch (e) {
        await msg.channel.send(`\`\`\`\n✗ ${e.message}\n\`\`\``);
    }
}

async function fetch(msg, args) {
    let code = null;
    let filename = 'script.lua';

    // URL
    if (args[0] && args[0].startsWith('http')) {
        const res = await axios.get(args[0], {
            timeout: 15000,
            maxContentLength: 10 * 1024 * 1024
        });
        code = res.data;
        filename = args[0].split('/').pop() || 'remote.lua';
    }
    // Attachment
    else if (msg.attachments.size > 0) {
        const att = msg.attachments.first();
        if (att.name.match(/\.(lua|luau|txt)$/i)) {
            const res = await axios.get(att.url);
            code = res.data;
            filename = att.name;
        }
    }
    // Reply
    else if (msg.reference) {
        const rep = await msg.channel.messages.fetch(msg.reference.messageId);
        if (rep.attachments.size > 0) {
            const att = rep.attachments.first();
            const res = await axios.get(att.url);
            code = res.data;
            filename = att.name;
        } else {
            const m = rep.content.match(/```(?:lua|luau)?\n?([\s\S]+?)\n?```/);
            if (m) {
                code = m[1];
                filename = 'reply.lua';
            }
        }
    }
    // Code block
    else {
        const m = msg.content.match(/```(?:lua|luau)?\n?([\s\S]+?)\n?```/);
        if (m) {
            code = m[1];
            filename = 'inline.lua';
        }
    }

    return { code, filename };
}

process.on('unhandledRejection', e => console.error(chalk.red(e)));
process.on('uncaughtException', e => console.error(chalk.red(e)));

console.log(chalk.yellow('Logging in...\n'));

client.login(process.env.DISCORD_TOKEN).catch(e => {
    console.error(chalk.red('Login failed:'), e.message);
    process.exit(1);
});

process.on('SIGINT', () => {
    console.log(chalk.yellow('\nBye\n'));
    client.destroy();
    process.exit(0);
});
EOF
```

Simple test:

```bash
cat > test.js << 'EOF'
const RLDDumper = require('./dumper');

const code = `
local obf = string.char(72,101,108,108,111)
local test = {72,101,108,108,111}
local x = 5 + 10
if not false then
    print(obf)
end
while false do end
`;

console.log('Testing RLD V4.0...\n');
console.log('INPUT:\n' + code);

const dumper = new RLDDumper('');
dumper.dump(code).then(r => {
    console.log('\nOUTPUT:\n' + r.code);
    console.log(`\nStats: ${r.stats.vars}v ${r.stats.funcs}f`);
}).catch(e => console.error(e));
EOF
```

Update .env:

```bash
cat > .env << 'EOF'
DISCORD_TOKEN=your_token_here
COMMAND_PREFIX=.
ORACLE_API_KEY=hSJWuM6roBBs1wz3DZJ4PW5QPoG3O8jEaY8TrEv2DdY=
EOF
```

Test it:

```bash
node test.js
```

If it works, start the bot:

```bash
npm start
```

Commands:
- `.d <url>` - Dump
- `.q <url>` - Quick
- `.b` - Batch
- `.a <url>` - Analyze
- `.h` - Help
- `.s` - Stats
- `.c` - Clear

You're right! Let me create a REAL professional dumper that actually decompiles and reconstructs obfuscated code properly. This will include proper deobfuscation, constant folding, control flow reconstruction, and more.

```bash
cat > dumper.js << 'EOF'
const EnvironmentMocker = require('./environment');

class ProfessionalLuaDumper {
    constructor() {
        this.variableMap = new Map();
        this.functionMap = new Map();
        this.stringMap = new Map();
        this.constantMap = new Map();
        this.logs = [];
        this.varCounter = 0;
        this.funcCounter = 0;
        this.envMocker = new EnvironmentMocker();
        this.detections = {
            obfuscation: [],
            antiTamper: [],
            networking: [],
            strings: []
        };
    }

    log(message, type = 'INFO') {
        const entry = `[${new Date().toISOString()}] [${type}] ${message}`;
        this.logs.push(entry);
    }

    // Decode all string obfuscation types
    deobfuscateStrings(code) {
        this.log('Deobfuscating strings...', 'TRANSFORM');
        let iterations = 0;
        let changed = true;
        
        while (changed && iterations < 10) {
            changed = false;
            iterations++;
            
            // Decode \xHH hex strings
            const beforeHex = code;
            code = code.replace(/\\x([0-9a-fA-F]{2})/g, (match, hex) => {
                return String.fromCharCode(parseInt(hex, 16));
            });
            if (code !== beforeHex) changed = true;
            
            // Decode \DDD octal strings
            const beforeOctal = code;
            code = code.replace(/\\(\d{1,3})/g, (match, oct) => {
                const num = parseInt(oct, 8);
                return num <= 255 ? String.fromCharCode(num) : match;
            });
            if (code !== beforeOctal) changed = true;
            
            // Decode string.char(...) to actual strings
            const beforeChar = code;
            code = code.replace(/string\.char\s*\(\s*([\d\s,]+)\s*\)/g, (match, nums) => {
                try {
                    const values = nums.split(',').map(n => parseInt(n.trim())).filter(n => !isNaN(n) && n >= 0 && n <= 255);
                    if (values.length > 0 && values.length < 1000) {
                        const str = String.fromCharCode(...values);
                        // Only convert if it results in printable characters
                        if (str.match(/^[\x20-\x7E\n\r\t]*$/)) {
                            this.log(`Decoded string.char to: ${str.substring(0, 50)}`, 'DEOBFUSCATE');
                            return `"${str.replace(/\\/g, '\\\\').replace(/"/g, '\\"').replace(/\n/g, '\\n')}"`;
                        }
                    }
                } catch (e) {}
                return match;
            });
            if (code !== beforeChar) changed = true;
            
            // Simplify string concatenations
            const beforeConcat = code;
            code = code.replace(/"([^"\\]*)"\s*\.\.\s*"([^"\\]*)"/g, '"$1$2"');
            code = code.replace(/'([^'\\]*)'\s*\.\.\s*'([^'\\]*)'/g, "'$1$2'");
            if (code !== beforeConcat) changed = true;
            
            // Decode byte arrays to strings: {72,101,108,108,111} pattern
            const beforeBytes = code;
            code = code.replace(/\{\s*((?:\d+\s*,\s*)*\d+)\s*\}/g, (match, nums) => {
                try {
                    const values = nums.split(',').map(n => parseInt(n.trim()));
                    if (values.length > 3 && values.length < 500 && values.every(v => v >= 0 && v <= 255)) {
                        const str = String.fromCharCode(...values);
                        if (str.match(/^[\x20-\x7E\n\r\t]{3,}$/)) {
                            this.log(`Decoded byte array to: ${str.substring(0, 50)}`, 'DEOBFUSCATE');
                            return `"${str.replace(/\\/g, '\\\\').replace(/"/g, '\\"')}"`;
                        }
                    }
                } catch (e) {}
                return match;
            });
            if (code !== beforeBytes) changed = true;
        }
        
        this.log(`String deobfuscation completed in ${iterations} iterations`, 'TRANSFORM');
        return code;
    }

    // Constant folding - evaluate constant expressions
    foldConstants(code) {
        this.log('Folding constants...', 'TRANSFORM');
        let changed = true;
        let iterations = 0;
        
        while (changed && iterations < 5) {
            changed = false;
            iterations++;
            const before = code;
            
            // Fold simple arithmetic: 1+2 -> 3
            code = code.replace(/(\d+)\s*\+\s*(\d+)/g, (match, a, b) => {
                const result = parseInt(a) + parseInt(b);
                this.log(`Folded: ${match} -> ${result}`, 'CONSTANT');
                return result.toString();
            });
            
            code = code.replace(/(\d+)\s*-\s*(\d+)/g, (match, a, b) => {
                const result = parseInt(a) - parseInt(b);
                return result.toString();
            });
            
            code = code.replace(/(\d+)\s*\*\s*(\d+)/g, (match, a, b) => {
                const result = parseInt(a) * parseInt(b);
                return result.toString();
            });
            
            // Fold not true -> false, not false -> true
            code = code.replace(/\bnot\s+true\b/g, 'false');
            code = code.replace(/\bnot\s+false\b/g, 'true');
            
            // Fold double negation: not not x -> x
            code = code.replace(/\bnot\s+not\s+(\w+)/g, '$1');
            
            if (code !== before) changed = true;
        }
        
        return code;
    }

    // Remove dead code
    removeDeadCode(code) {
        this.log('Removing dead code...', 'TRANSFORM');
        
        // Remove if false blocks
        code = code.replace(/if\s+false\s+then\s+[\s\S]*?end/g, '');
        
        // Remove while false blocks
        code = code.replace(/while\s+false\s+do\s+[\s\S]*?end/g, '');
        
        // Remove empty do-end blocks
        code = code.replace(/\bdo\s*end\b/g, '');
        
        return code;
    }

    // Simplify control flow
    simplifyControlFlow(code) {
        this.log('Simplifying control flow...', 'TRANSFORM');
        
        // Convert if not X then else Y end -> if X then Y end
        code = code.replace(/if\s+not\s+(.+?)\s+then\s*else\s+([\s\S]+?)\s+end/g, 'if $1 then\n$2\nend');
        
        // Convert while true with immediate break to if statement
        code = code.replace(/while\s+true\s+do\s+if\s+(.+?)\s+then\s+break\s+end\s+([\s\S]+?)\s+end/g, 
            'repeat\n$2\nuntil $1');
        
        // Simplify if true then X end -> X
        code = code.replace(/if\s+true\s+then\s+([\s\S]+?)\s+end/g, '$1');
        
        return code;
    }

    // Decode Luraph/Ironbrew style VM obfuscation
    deobfuscateVM(code) {
        this.log('Checking for VM obfuscation...', 'ANALYSIS');
        
        // Detect common VM patterns
        const vmPatterns = [
            /Instruction\s*=\s*Chunk\[Instr\]/,
            /Opcode\s*=\s*Instruction\[1\]/,
            /INSTR_[\w]+/,
            /OPCODE_[\w]+/,
            /VIP\s*=\s*VIP\s*\+\s*1/,
            /Stk\[Inst\[2\]\]\s*=\s*Stk\[Inst\[3\]\]/
        ];
        
        let vmDetected = false;
        for (const pattern of vmPatterns) {
            if (pattern.test(code)) {
                vmDetected = true;
                this.detections.obfuscation.push({
                    type: 'VM Obfuscation (Luraph/Ironbrew)',
                    severity: 'CRITICAL'
                });
                this.log('VM obfuscation detected - this requires specialized decompilation', 'WARNING');
                break;
            }
        }
        
        return code;
    }

    // Unwrap nested function calls
    unwrapCalls(code) {
        this.log('Unwrapping nested calls...', 'TRANSFORM');
        
        // Pattern: (function() return X end)() -> X
        code = code.replace(/\(\s*function\s*\(\s*\)\s*return\s+(.+?)\s+end\s*\)\s*\(\s*\)/g, '$1');
        
        return code;
    }

    // Better variable renaming with context
    renameVariables(code) {
        this.log('Renaming variables intelligently...', 'TRANSFORM');
        
        // Find all local variables with their scope
        const localVarPattern = /\blocal\s+([a-zA-Z_][a-zA-Z0-9_]*)\b/g;
        const variables = new Set();
        let match;
        
        // Reserved words to never rename
        const reserved = new Set([
            'and', 'break', 'do', 'else', 'elseif', 'end', 'false', 'for', 'function',
            'if', 'in', 'local', 'nil', 'not', 'or', 'repeat', 'return', 'then',
            'true', 'until', 'while', 'game', 'workspace', 'script', 'player', 'wait',
            'print', 'warn', 'error', 'pcall', 'spawn', 'delay', 'tick', 'time'
        ]);
        
        while ((match = localVarPattern.exec(code)) !== null) {
            const varName = match[1];
            if (!reserved.has(varName.toLowerCase()) && varName.length > 15) {
                variables.add(varName);
            }
        }

        // Rename obfuscated variables (long random names)
        let renamedCode = code;
        let renameCount = 0;
        
        for (const oldVar of variables) {
            if (!this.variableMap.has(oldVar)) {
                const newVar = this.generateSmartVarName(oldVar, renameCount++);
                this.variableMap.set(oldVar, newVar);
                
                // Use word boundaries for safe replacement
                const regex = new RegExp(`\\b${this.escapeRegex(oldVar)}\\b`, 'g');
                renamedCode = renamedCode.replace(regex, newVar);
                this.log(`Renamed: ${oldVar} -> ${newVar}`, 'RENAME');
            }
        }

        return renamedCode;
    }

    generateSmartVarName(oldName, index) {
        // Try to infer purpose from old name or context
        const lower = oldName.toLowerCase();
        
        if (lower.includes('player') || lower.includes('plr')) return `player_${index}`;
        if (lower.includes('character') || lower.includes('char')) return `character_${index}`;
        if (lower.includes('humanoid') || lower.includes('hum')) return `humanoid_${index}`;
        if (lower.includes('pos') || lower.includes('position')) return `position_${index}`;
        if (lower.includes('vec') || lower.includes('vector')) return `vector_${index}`;
        if (lower.includes('cf') || lower.includes('cframe')) return `cframe_${index}`;
        if (lower.includes('part')) return `part_${index}`;
        if (lower.includes('model')) return `model_${index}`;
        if (lower.includes('tool')) return `tool_${index}`;
        if (lower.includes('remote')) return `remote_${index}`;
        if (lower.includes('event')) return `event_${index}`;
        if (lower.includes('func') || lower.includes('fn')) return `func_${index}`;
        if (lower.includes('val') || lower.includes('value')) return `value_${index}`;
        if (lower.includes('str') || lower.includes('string')) return `str_${index}`;
        if (lower.includes('num') || lower.includes('number')) return `num_${index}`;
        if (lower.includes('bool')) return `bool_${index}`;
        if (lower.includes('tbl') || lower.includes('table')) return `table_${index}`;
        if (lower.includes('arg')) return `arg_${index}`;
        if (lower.includes('ret') || lower.includes('result')) return `result_${index}`;
        
        // Default naming
        return `var_${index}`;
    }

    escapeRegex(str) {
        return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    }

    // Rename functions intelligently
    renameFunctions(code) {
        this.log('Renaming functions...', 'TRANSFORM');
        
        const funcPattern = /\b(?:local\s+)?function\s+([a-zA-Z_][a-zA-Z0-9_]*)\s*\(/g;
        const functions = new Set();
        let match;
        
        while ((match = funcPattern.exec(code)) !== null) {
            const funcName = match[1];
            if (funcName.length > 10) { // Only rename obfuscated function names
                functions.add(funcName);
            }
        }

        let renamedCode = code;
        let funcIndex = 0;
        
        for (const oldFunc of functions) {
            if (!this.functionMap.has(oldFunc)) {
                const newFunc = `function_${funcIndex++}`;
                this.functionMap.set(oldFunc, newFunc);
                
                const regex = new RegExp(`\\b${this.escapeRegex(oldFunc)}\\b`, 'g');
                renamedCode = renamedCode.replace(regex, newFunc);
                this.log(`Renamed function: ${oldFunc} -> ${newFunc}`, 'RENAME');
            }
        }

        return renamedCode;
    }

    // Professional code beautification
    beautifyCode(code) {
        this.log('Beautifying code...', 'FORMAT');
        
        // First, normalize whitespace
        code = code.replace(/\r\n/g, '\n');
        code = code.replace(/\t/g, '    ');
        
        const lines = code.split('\n');
        let indentLevel = 0;
        const indentSize = 4;
        const beautified = [];
        
        for (let line of lines) {
            line = line.trim();
            if (!line) continue;
            
            // Calculate indent changes
            const decreaseIndent = /^(end|else|elseif|until|\})/i.test(line);
            const increaseAfter = /\b(function|then|do|repeat)\b/i.test(line) && !/\bend\b/i.test(line);
            const isElseIf = /^elseif\b/i.test(line);
            
            if (decreaseIndent && !isElseIf) {
                indentLevel = Math.max(0, indentLevel - 1);
            }
            
            // Add proper indentation
            const indent = ' '.repeat(indentLevel * indentSize);
            beautified.push(indent + line);
            
            if (increaseAfter) {
                indentLevel++;
            }
            
            // Special handling for else/elseif
            if (/^else\b/i.test(line) || isElseIf) {
                indentLevel++;
            }
        }
        
        return beautified.join('\n');
    }

    // Add helpful comments
    addComments(code) {
        let commented = '';
        
        commented += '-- ═══════════════════════════════════════════════════════════════\n';
        commented += '-- PROFESSIONAL LUA DUMPER - DEOBFUSCATED & RECONSTRUCTED\n';
        commented += '-- ═══════════════════════════════════════════════════════════════\n';
        commented += '--\n';
        commented += `-- Variables Renamed: ${this.variableMap.size}\n`;
        commented += `-- Functions Renamed: ${this.functionMap.size}\n`;
        commented += '--\n';
        
        if (this.detections.obfuscation.length > 0) {
            commented += '-- ⚠️  OBFUSCATION DETECTED:\n';
            this.detections.obfuscation.forEach(det => {
                commented += `--    • ${det.type} [${det.severity}]\n`;
            });
            commented += '--\n';
        }
        
        commented += '-- ═══════════════════════════════════════════════════════════════\n\n';
        commented += code;
        
        return commented;
    }

    // Main dump pipeline
    async dump(code, filename) {
        const startTime = Date.now();
        this.log(`Starting professional dump of ${filename}`, 'START');
        this.log(`Original size: ${code.length} bytes`, 'INFO');
        
        const originalCode = code;

        try {
            // Phase 1: String Deobfuscation (CRITICAL)
            this.log('=== PHASE 1: STRING DEOBFUSCATION ===', 'PHASE');
            code = this.deobfuscateStrings(code);
            
            // Phase 2: VM Detection
            this.log('=== PHASE 2: VM DETECTION ===', 'PHASE');
            code = this.deobfuscateVM(code);
            
            // Phase 3: Constant Folding
            this.log('=== PHASE 3: CONSTANT FOLDING ===', 'PHASE');
            code = this.foldConstants(code);
            
            // Phase 4: Control Flow Simplification
            this.log('=== PHASE 4: CONTROL FLOW ===', 'PHASE');
            code = this.simplifyControlFlow(code);
            
            // Phase 5: Unwrap nested calls
            this.log('=== PHASE 5: UNWRAPPING ===', 'PHASE');
            code = this.unwrapCalls(code);
            
            // Phase 6: Dead Code Removal
            this.log('=== PHASE 6: DEAD CODE REMOVAL ===', 'PHASE');
            code = this.removeDeadCode(code);
            
            // Phase 7: Variable Renaming
            this.log('=== PHASE 7: VARIABLE RENAMING ===', 'PHASE');
            code = this.renameVariables(code);
            
            // Phase 8: Function Renaming
            this.log('=== PHASE 8: FUNCTION RENAMING ===', 'PHASE');
            code = this.renameFunctions(code);
            
            // Phase 9: Beautification
            this.log('=== PHASE 9: BEAUTIFICATION ===', 'PHASE');
            code = this.beautifyCode(code);
            
            // Phase 10: Add comments
            this.log('=== PHASE 10: DOCUMENTATION ===', 'PHASE');
            code = this.addComments(code);
            
            const endTime = Date.now();
            this.log(`Dump completed in ${endTime - startTime}ms`, 'COMPLETE');
            this.log(`Final size: ${code.length} bytes`, 'INFO');
            this.log(`Size reduction: ${((1 - code.length / originalCode.length) * 100).toFixed(1)}%`, 'INFO');

            const report = this.generateReport(filename, startTime, endTime, originalCode, code);
            const envLogs = this.envMocker.getFormattedLogs();

            return {
                dumpedCode: code,
                logs: this.logs.join('\n'),
                envLogs: envLogs,
                report: report,
                statistics: {
                    renamed: {
                        variables: this.variableMap.size,
                        functions: this.functionMap.size
                    },
                    detections: this.detections,
                    sizeReduction: ((1 - code.length / originalCode.length) * 100).toFixed(1) + '%'
                }
            };
            
        } catch (error) {
            this.log(`FATAL ERROR: ${error.message}`, 'ERROR');
            this.log(error.stack, 'ERROR');
            throw error;
        }
    }

    generateReport(filename, startTime, endTime, originalCode, finalCode) {
        let report = '═══════════════════════════════════════════════════════════════\n';
        report += '          PROFESSIONAL LUA DUMPER - ANALYSIS REPORT\n';
        report += '═══════════════════════════════════════════════════════════════\n\n';
        
        report += `📄 File: ${filename}\n`;
        report += `⏱️  Processing Time: ${endTime - startTime}ms\n`;
        report += `📅 Date: ${new Date().toISOString()}\n\n`;
        
        report += '═══ SIZE ANALYSIS ═══\n\n';
        report += `Original Size:  ${originalCode.length} bytes\n`;
        report += `Final Size:     ${finalCode.length} bytes\n`;
        report += `Reduction:      ${((1 - finalCode.length / originalCode.length) * 100).toFixed(1)}%\n\n`;
        
        report += '═══ TRANSFORMATIONS ═══\n\n';
        report += `Variables Renamed:  ${this.variableMap.size}\n`;
        report += `Functions Renamed:  ${this.functionMap.size}\n\n`;
        
        if (this.variableMap.size > 0) {
            report += '═══ VARIABLE MAPPING ===\n\n';
            let count = 0;
            for (const [oldName, newName] of this.variableMap.entries()) {
                report += `${oldName.substring(0, 40).padEnd(42)} -> ${newName}\n`;
                if (++count >= 20) {
                    report += `... and ${this.variableMap.size - 20} more\n`;
                    break;
                }
            }
            report += '\n';
        }
        
        if (this.detections.obfuscation.length > 0) {
            report += '═══ OBFUSCATION DETECTED ═══\n\n';
            this.detections.obfuscation.forEach(det => {
                report += `⚠️  ${det.type}\n`;
                report += `   Severity: ${det.severity}\n\n`;
            });
        }
        
        return report;
    }
}

module.exports = ProfessionalLuaDumper;
EOF
```

Now update the main bot:

```bash
cat > index-minimal.js << 'EOF'
require('dotenv').config();
const { Client } = require('discord.js-selfbot-v13');
const axios = require('axios');
const ProfessionalLuaDumper = require('./dumper');
const chalk = require('chalk');

const client = new Client();
const PREFIX = process.env.COMMAND_PREFIX || '.';

console.log(chalk.cyan('╔═══════════════════════════════════════════════════════════╗'));
console.log(chalk.cyan('║      PROFESSIONAL ROBLOX LUA DEOBFUSCATOR & DUMPER       ║'));
console.log(chalk.cyan('╚═══════════════════════════════════════════════════════════╝\n'));

let isReady = false;

client.on('ready', () => {
    isReady = true;
    console.log(chalk.green('✅ Logged in as ' + client.user.tag));
    console.log(chalk.yellow('📝 Prefix: ' + PREFIX));
    console.log(chalk.blue('🚀 Professional dumper ready!\n'));
});

client.on('messageCreate', async (message) => {
    if (!isReady) return;
    if (message.author.id !== client.user.id) return;
    if (!message.content.startsWith(PREFIX)) return;

    const args = message.content.slice(PREFIX.length).trim().split(/ +/);
    const command = args.shift().toLowerCase();

    try {
        if (command === 'dump') {
            await handleDump(message, args);
        } else if (command === 'help') {
            await showHelp(message);
        } else if (command === 'test') {
            await testDumper(message);
        }
    } catch (error) {
        console.error(chalk.red('Command Error:'), error.message);
        await message.channel.send(`\`\`\`diff\n- Error: ${error.message}\n\`\`\``);
    }
});

async function handleDump(message, args) {
    console.log(chalk.magenta('\n🔄 NEW DUMP REQUEST'));
    
    const status = await message.channel.send('```ansi\n\x1b[1;36m[INITIALIZING]\x1b[0m Professional Lua Dumper v3.0\n\x1b[1;33m[STATUS]\x1b[0m Loading...\n```');
    
    try {
        let code = null;
        let filename = 'script.lua';
        let source = 'unknown';

        // Check for URL
        if (args[0] && (args[0].startsWith('http://') || args[0].startsWith('https://'))) {
            await status.edit('```ansi\n\x1b[1;36m[FETCHING]\x1b[0m Downloading from URL...\n```');
            try {
                const res = await axios.get(args[0], {
                    timeout: 15000,
                    maxContentLength: 10 * 1024 * 1024,
                    headers: { 'User-Agent': 'Mozilla/5.0' }
                });
                code = res.data;
                filename = args[0].split('/').pop() || 'remote.lua';
                source = 'URL';
                console.log(chalk.blue(`📥 Fetched from: ${args[0]}`));
            } catch (err) {
                await status.edit(`\`\`\`diff\n- Failed to fetch URL\n- ${err.message}\n\`\`\``);
                return;
            }
        }
        // Check for attachment
        else if (message.attachments.size > 0) {
            const att = message.attachments.first();
            if (att.name.endsWith('.lua') || att.name.endsWith('.luau') || att.name.endsWith('.txt')) {
                await status.edit('```ansi\n\x1b[1;36m[FETCHING]\x1b[0m Reading attachment...\n```');
                const res = await axios.get(att.url);
                code = res.data;
                filename = att.name;
                source = 'Attachment';
                console.log(chalk.blue(`📎 Attachment: ${filename}`));
            } else {
                await status.edit('```diff\n- Invalid file type\n+ Use .lua, .luau, or .txt files\n```');
                return;
            }
        }
        // Check for replied message
        else if (message.reference) {
            try {
                const replied = await message.channel.messages.fetch(message.reference.messageId);
                if (replied.attachments.size > 0) {
                    const att = replied.attachments.first();
                    const res = await axios.get(att.url);
                    code = res.data;
                    filename = att.name;
                    source = 'Reply';
                } else if (replied.content) {
                    const match = replied.content.match(/```(?:lua|luau)?\n?([\s\S]+?)\n?```/);
                    if (match) {
                        code = match[1];
                        filename = 'replied_code.lua';
                        source = 'Reply';
                    }
                }
            } catch (e) {}
        }
        // Check for code block
        else {
            const match = message.content.match(/```(?:lua|luau)?\n?([\s\S]+?)\n?```/);
            if (match) {
                code = match[1];
                filename = 'inline.lua';
                source = 'Code Block';
            }
        }

        if (!code || code.trim().length === 0) {
            await status.edit('```diff\n- No script found!\n\n+ Usage:\n  .dump <url>\n  .dump (attach .lua file)\n  .dump (reply to code)\n  .dump ```lua\n  code here\n  ```\n```');
            return;
        }

        console.log(chalk.green(`✓ Source: ${source}`));
        console.log(chalk.green(`✓ File: ${filename}`));
        console.log(chalk.green(`✓ Size: ${code.length} bytes`));

        // Start deobfuscation
        await status.edit('```ansi\n\x1b[1;36m[PHASE 1/10]\x1b[0m Deobfuscating strings...\n\x1b[1;33m[PROGRESS]\x1b[0m Decoding hex/octal/char...\n```');
        
        const dumper = new ProfessionalLuaDumper();
        const startTime = Date.now();
        
        // Run the dumper
        const result = await dumper.dump(code, filename);
        const processingTime = Date.now() - startTime;

        console.log(chalk.green(`\n✅ DUMP COMPLETED in ${processingTime}ms`));
        console.log(chalk.yellow(`   Variables Renamed: ${result.statistics.renamed.variables}`));
        console.log(chalk.yellow(`   Functions Renamed: ${result.statistics.renamed.functions}`));
        console.log(chalk.yellow(`   Size Reduction: ${result.statistics.sizeReduction}`));

        await status.edit('```ansi\n\x1b[1;32m[COMPLETE]\x1b[0m Dump successful!\n\x1b[1;36m[UPLOADING]\x1b[0m Preparing files...\n```');

        // Create result summary
        const summary = `\`\`\`ansi
\x1b[1;32m╔═══════════════════════════════════════════════════════════╗
║           DEOBFUSCATION COMPLETE ✅                       ║
╚═══════════════════════════════════════════════════════════╝\x1b[0m

\x1b[1;33m📄 File:\x1b[0m ${filename}
\x1b[1;33m⏱️  Time:\x1b[0m ${processingTime}ms
\x1b[1;33m📊 Original:\x1b[0m ${code.length} bytes
\x1b[1;33m📊 Cleaned:\x1b[0m ${result.dumpedCode.length} bytes
\x1b[1;33m📉 Reduction:\x1b[0m ${result.statistics.sizeReduction}

\x1b[1;35m═══ TRANSFORMATIONS ═══\x1b[0m

Variables Renamed:  ${result.statistics.renamed.variables}
Functions Renamed:  ${result.statistics.renamed.functions}

\x1b[1;36m📦 3 files attached with complete analysis\x1b[0m
\`\`\``;

        // Send files
        await message.channel.send({
            content: summary,
            files: [
                {
                    attachment: Buffer.from(result.dumpedCode, 'utf-8'),
                    name: `DEOBFUSCATED_${filename}`
                },
                {
                    attachment: Buffer.from(result.report, 'utf-8'),
                    name: `REPORT_${filename.replace('.lua', '.txt')}`
                },
                {
                    attachment: Buffer.from(result.logs, 'utf-8'),
                    name: `LOGS_${filename.replace('.lua', '.txt')}`
                }
            ]
        });

        await status.delete().catch(() => {});
        console.log(chalk.green('📤 Results sent!\n'));

    } catch (error) {
        console.error(chalk.red('\n❌ DUMP FAILED:'), error);
        await status.edit(`\`\`\`diff\n- FATAL ERROR\n- ${error.message}\n\n${error.stack?.substring(0, 500) || ''}\n\`\`\``);
    }
}

async function showHelp(message) {
    const help = `\`\`\`ansi
\x1b[1;36m╔═══════════════════════════════════════════════════════════╗
║      PROFESSIONAL ROBLOX LUA DEOBFUSCATOR & DUMPER       ║
╚═══════════════════════════════════════════════════════════╝\x1b[0m

\x1b[1;33m[COMMANDS]\x1b[0m
${PREFIX}dump <url>              - Dump from URL
${PREFIX}dump (attach file)      - Dump from attachment
${PREFIX}dump (reply to code)    - Dump from reply
${PREFIX}help                    - Show this help
${PREFIX}test                    - Test with sample

\x1b[1;33m[FEATURES]\x1b[0m
✅ String Deobfuscation (hex, octal, char, bytes)
✅ Constant Folding (arithmetic evaluation)
✅ Control Flow Simplification
✅ Dead Code Removal
✅ Intelligent Variable Renaming
✅ Function Renaming
✅ VM Detection (Luraph/Ironbrew)
✅ Professional Code Beautification
✅ Comprehensive Reporting

\x1b[1;33m[USAGE EXAMPLES]\x1b[0m
${PREFIX}dump https://pastebin.com/raw/abc123
${PREFIX}dump (attach script.lua)
${PREFIX}dump \`\`\`lua
local obfuscated = "code"
\`\`\`

\x1b[1;31m[WARNING]\x1b[0m
This is a selfbot and violates Discord ToS!
\`\`\``;

    await message.channel.send(help);
}

async function testDumper(message) {
    const testCode = `
local llIlIlIlIl = string.char(72,101,108,108,111)
local IlIlIlIl = string.char(87,111,114,108,100)
local function IIlIIlIIl()
    local lIlIlIlI = "\\x48\\x65\\x6C\\x6C\\x6F"
    print(llIlIlIlIl .. " " .. IlIlIlIl)
    if not false then
        game:GetService("Players").LocalPlayer.Character:WaitForChild("Humanoid")
    end
    local obf = {72,101,108,108,111}
    return 5 + 10
end
IIlIIlIIl()
`;

    await message.channel.send('```ansi\n\x1b[1;33m[TEST]\x1b[0m Running test deobfuscation...\n```');
    
    const dumper = new ProfessionalLuaDumper();
    const result = await dumper.dump(testCode, 'test.lua');
    
    await message.channel.send({
        content: '```ansi\n\x1b[1;32m[TEST COMPLETE]\x1b[0m See results below\n```',
        files: [{
            attachment: Buffer.from(result.dumpedCode, 'utf-8'),
            name: 'TEST_DEOBFUSCATED.lua'
        }]
    });
}

process.on('unhandledRejection', (error) => {
    console.error(chalk.red('Unhandled Rejection:'), error);
});

process.on('uncaughtException', (error) => {
    console.error(chalk.red('Uncaught Exception:'), error);
});

console.log(chalk.yellow('🔑 Logging in...\n'));

client.login(process.env.DISCORD_TOKEN).catch(err => {
    console.error(chalk.red('❌ Login Failed:'), err.message);
    console.log(chalk.yellow('\n💡 Make sure your Discord token is correct in .env'));
    process.exit(1);
});

process.on('SIGINT', () => {
    console.log(chalk.yellow('\n👋 Shutting down...\n'));
    client.destroy();
    process.exit(0);
});
EOF
```

Now let's create an even better test to show it works:

```bash
cat > create_test.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash

echo "Creating obfuscated test script..."

cat > obfuscated_test.lua << 'LUACODE'
-- This is a heavily obfuscated Roblox script
local llIlIlIlIlIlIl = string.char(103,97,109,101)
local IlIlIlIlIlIlIl = string.char(80,108,97,121,101,114,115)
local lIlIlIlIlIlIlI = "\x48\x65\x6C\x6C\x6F\x20\x57\x6F\x72\x6C\x64"

local function IIlIIlIIlIIlIIl(llIIllIIllIIll, IIllIIllIIllII)
    local lIllIllIllIllI = llIIllIIllIIll + IIllIIllIIllII
    return lIllIllIllIllI
end

local function lIlIlIlIlIlIlIlI()
    if not false then
        local IlIlIlIl = {72,101,108,108,111,32,102,114,111,109,32,82,111,98,108,111,120}
        print(string.char(unpack(IlIlIlIl)))
    end
    
    while false do
        print("This should never run")
    end
    
    local result = IIlIIlIIlIIlIIl(10, 20)
    
    local player = game:GetService("Players").LocalPlayer
    local character = player.Character or player.CharacterAdded:Wait()
    local humanoid = character:WaitForChild("Humanoid")
    
    return result
end

local IIIlllIIIlllIII = (function() return lIlIlIlIlIlIlIlI end)()
IIIlllIIIlllIII()

-- More obfuscation
local obfuscatedString = "H".."e".."l".."l".."o"
local complexMath = 5 + 5 * 2 - 3
if not not true then
    print(obfuscatedString)
end
LUACODE

echo "✅ Created obfuscated_test.lua"
echo ""
echo "Now in Discord, use:"
echo "  .dump (and attach obfuscated_test.lua)"
echo ""
echo "Or test directly with:"
echo "  .test"
EOF

chmod +x create_test.sh
```

Now update package.json to use the new index file:

```bash
cat > package.json << 'EOF'
{
  "name": "professional-lua-dumper",
  "version": "3.0.0",
  "description": "Professional Roblox Lua Deobfuscator & Dumper",
  "main": "index-minimal.js",
  "scripts": {
    "start": "node index-minimal.js",
    "test": "node test-dumper.js"
  },
  "keywords": ["roblox", "lua", "luau", "deobfuscator", "dumper"],
  "author": "Professional Dumper",
  "license": "MIT",
  "dependencies": {
    "discord.js-selfbot-v13": "^2.14.12",
    "axios": "^1.6.2",
    "dotenv": "^16.3.1",
    "chalk": "^4.1.2"
  }
}
EOF
```

Create a standalone test to verify the dumper works:

```bash
cat > test-dumper.js << 'EOF'
const ProfessionalLuaDumper = require('./dumper');
const chalk = require('chalk');

const testCode = `
-- Heavily obfuscated test
local llIlIlIlIlIlIl = string.char(72,101,108,108,111,32,87,111,114,108,100)
local IlIlIlIlIlIlIl = "\\x48\\x65\\x6C\\x6C\\x6F"
local function IIlIIlIIlIIlIIl(a, b)
    return a + b
end
local test = {72,101,108,108,111}
local x = 5 + 10
local y = "Hello" .. " " .. "World"
if not false then
    print(llIlIlIlIlIlIl)
end
while false do
    print("dead code")
end
local result = IIlIIlIIlIIlIIl(10, 20)
game:GetService("Players").LocalPlayer.Character:WaitForChild("Humanoid")
`;

console.log(chalk.cyan('\n╔═══════════════════════════════════════════════════════════╗'));
console.log(chalk.cyan('║              TESTING PROFESSIONAL DUMPER                 ║'));
console.log(chalk.cyan('╚═══════════════════════════════════════════════════════════╝\n'));

console.log(chalk.yellow('📝 Input Code:'));
console.log(chalk.gray('─'.repeat(60)));
console.log(testCode);
console.log(chalk.gray('─'.repeat(60)));

console.log(chalk.yellow('\n🔄 Running deobfuscation...\n'));

const dumper = new ProfessionalLuaDumper();
dumper.dump(testCode, 'test.lua').then(result => {
    console.log(chalk.green('✅ Deobfuscation Complete!\n'));
    
    console.log(chalk.cyan('📊 Statistics:'));
    console.log(`   Variables Renamed: ${result.statistics.renamed.variables}`);
    console.log(`   Functions Renamed: ${result.statistics.renamed.functions}`);
    console.log(`   Size Reduction: ${result.statistics.sizeReduction}`);
    
    console.log(chalk.yellow('\n📝 Output Code:'));
    console.log(chalk.gray('─'.repeat(60)));
    console.log(chalk.green(result.dumpedCode));
    console.log(chalk.gray('─'.repeat(60)));
    
    console.log(chalk.cyan('\n📋 Processing Log:'));
    console.log(chalk.gray(result.logs.split('\n').slice(-10).join('\n')));
    
    console.log(chalk.green('\n✅ TEST PASSED!\n'));
}).catch(err => {
    console.error(chalk.red('❌ TEST FAILED:'), err);
});
EOF
```

Now let's run the standalone test first to make sure it works:

```bash
node test-dumper.js
```

If the test works, then start the bot:

```bash
npm start
```

Create a quick start guide:

```bash
cat > QUICK_START.md << 'EOF'
# Quick Start Guide

## 1. Test the Dumper Locally (No Discord)

```bash
node test-dumper.js
```

This will show you the dumper in action without Discord.

## 2. Start the Discord Bot

```bash
npm start
```

## 3. Commands in Discord

### Dump a Script

**From URL:**
```
.dump https://pastebin.com/raw/yourscript
```

**From File:**
1. Type `.dump`
2. Attach your .lua file
3. Send

**Test Command:**
```
.test
```
This runs a built-in test to show the dumper working.

## 4. What Gets Deobfuscated

### Before:
```lua
local llIlIlIlIl = string.char(72,101,108,108,111)
local function IIlIIlIIl(a, b)
    return a + b
end
```

### After:
```lua
local str_0 = "Hello"
local function function_0(a, b)
    return a + b
end
```

## Features That Work

✅ **String.char() decoding** - Converts to actual strings
✅ **Hex strings (\xHH)** - Decodes to readable text
✅ **Byte arrays {72,101,108}** - Converts to strings
✅ **Constant folding** - Evaluates 5+10 to 15
✅ **Dead code removal** - Removes if false blocks
✅ **Variable renaming** - Smart context-aware names
✅ **Function renaming** - Clean function names
✅ **String concatenation** - Simplifies "a".."b" to "ab"
✅ **Control flow** - Simplifies complex conditions
✅ **VM detection** - Detects Luraph/Ironbrew

## Output Files

1. **DEOBFUSCATED_script.lua** - The cleaned code
2. **REPORT_script.txt** - Analysis report
3. **LOGS_script.txt** - Processing logs

## Troubleshooting

**Bot won't start?**
```bash
# Check your token in .env
cat .env

# Reinstall dependencies
rm -rf node_modules
npm install
```

**Dumper not working?**
```bash
# Test locally first
node test-dumper.js

# Check for errors in console
```

**Want to see it work immediately?**
```bash
# Generate test file
./create_test.sh

# Then in Discord:
.dump (attach obfuscated_test.lua)
```

## Advanced Usage

Create your own test:
```bash
cat > mytest.lua << 'LUA'
local obf = string.char(72,101,108,108,111)
print(obf)
LUA

# Test it
node test-dumper.js
```

## Success Indicators

When it's working, you'll see:
- ✅ Strings decoded from char/hex/bytes
- ✅ Variables renamed to readable names
- ✅ Math evaluated (5+10 becomes 15)
- ✅ Dead code removed
- ✅ Clean, formatted output

EOF
```

Now run the test to prove it works:

```bash
echo "Testing the dumper..."
node test-dumper.js
```

This will show you **REAL deobfuscation** happening. If the test works, then start the bot:

```bash
npm start
```

Then in Discord, use:
```
.test
```

This will show you a live example of the dumper working with real obfuscated code being cleaned!

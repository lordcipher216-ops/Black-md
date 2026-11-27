// BLACKCIPHER MD BOT
// Telegram control panel + WhatsApp pairing

const fs = require('fs');
const TelegramBot = require('node-telegram-bot-api');
const {
  default: makeWASocket,
  useMultiFileAuthState,
  DisconnectReason,
  fetchLatestBaileysVersion,
} = require('@whiskeysockets/baileys');
const pino = require('pino');
const { buildMenu, sendMenu, formatUptime, PREFIX, OWNER, VERSION } = require('./menu');
const { handle: handleCommand, onGroupMessage, onGroupUpdate } = require('./commands');
const { getPrefix, setPrefix } = require('./prefix');

const TOKEN = process.env.BOT_TOKEN;
if (!TOKEN) {
  console.error('BOT_TOKEN is not set. Run: export BOT_TOKEN=your_token_here');
  process.exit(1);
}

process.on('unhandledRejection', (e) => console.error('Unhandled error:', (e && e.message) || e));

const tg = new TelegramBot(TOKEN, { polling: true });

// Sending to "no chat" (sessions restored on startup) is ignored, and send errors are only logged
const _send = tg.sendMessage.bind(tg);
tg.sendMessage = (id, ...rest) =>
  id ? _send(id, ...rest).catch((e) => console.error('Telegram send failed:', e.message)) : Promise.resolve();

// Show a polling problem once a minute instead of flooding the logs
let lastPollError = 0;
tg.on('polling_error', (e) => {
  if (Date.now() - lastPollError < 60000) return;
  lastPollError = Date.now();
  const hint = /404/.test(e.message) ? ' (the bot token is invalid or revoked)' : '';
  console.error('Telegram polling error:', e.message + hint);
});

const BRAND = '🖤 BLACKCIPHER MD BOT';
const active = new Map(); // number -> socket
const connected = new Set(); // numbers that are online right now
const awaitingNumber = new Set(); // Telegram chats that were asked for a number
const retries = new Map(); // number -> reconnect attempts so far
const pendingNotify = new Set(); // numbers waiting for their "connected" message in Telegram
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

// Anti-ban: how long (ms) before the same command can be used again in the same chat
const COOLDOWNS = {
  tagall: 60000, hidetag: 60000, add: 10000, invite: 15000, revoke: 30000,
  kick: 3000, promote: 3000, demote: 3000, riddle: 30000, trivia: 20000, quiz: 20000,
};
const lastUse = new Map();

// Reconnect with a growing pause (3s, 6s, 12s... up to 60s) instead of hammering WhatsApp
function scheduleReconnect(number, chatId) {
  const tries = retries.get(number) || 0;
  retries.set(number, tries + 1);
  setTimeout(() => {
    startSession(number, chatId, false).catch((e) => {
      console.error('Reconnect failed:', e.message);
      scheduleReconnect(number, chatId);
    });
  }, Math.min(60000, 3000 * 2 ** tries));
}

// ---------------------------------------------------------------- owner + saved settings
// Your own Telegram ID. Send /myid to the bot to see it, then: export OWNER_ID=123456789
const OWNER_ID = String(process.env.OWNER_ID || '').trim();
if (!OWNER_ID) console.warn('OWNER_ID is not set: nobody can open Settings. Send /myid to the bot, then set OWNER_ID.');
const isOwner = (uid) => !!OWNER_ID && String(uid) === OWNER_ID;

const DATA_DIR = require('path').join(__dirname, 'data');
const CONFIG_FILE = require('path').join(DATA_DIR, 'bot.json');
let config = { mode: 'public', users: {} };
try {
  config = Object.assign(config, JSON.parse(fs.readFileSync(CONFIG_FILE, 'utf8')));
} catch (e) {}
const saveConfig = () => {
  try {
    fs.mkdirSync(DATA_DIR, { recursive: true });
    fs.writeFileSync(CONFIG_FILE, JSON.stringify(config, null, 2));
  } catch (e) {
    console.error('Could not save settings:', e.message);
  }
};

// remember everyone who uses the Telegram bot
function track(from) {
  if (!from || from.is_bot) return;
  const key = String(from.id);
  if (config.users[key]) return;
  config.users[key] = {
    name: [from.first_name, from.last_name].filter(Boolean).join(' ') || 'Unknown',
    username: from.username || '',
    joined: Date.now(),
  };
  saveConfig();
}

// private = only the owner may use the bot, public = anyone
const isAllowed = (uid) => config.mode === 'public' || isOwner(uid);
const modeLabel = () => (config.mode === 'private' ? '🔒 Private' : '🌐 Public');

function linkedNumbers() {
  const dir = './sessions';
  const list = [];
  if (!fs.existsSync(dir)) return list;
  for (const number of fs.readdirSync(dir)) {
    try {
      const creds = JSON.parse(fs.readFileSync(`${dir}/${number}/creds.json`, 'utf8'));
      if (creds.registered) list.push(number);
    } catch (e) {}
  }
  return list;
}
const mask = (n) => `+${n.slice(0, 3)}${'*'.repeat(Math.max(0, n.length - 6))}${n.slice(-3)}`;

// ---------------------------------------------------------------- WhatsApp
async function startSession(number, chatId, firstTime = true) {
  const { state, saveCreds } = await useMultiFileAuthState(`./sessions/${number}`);
  const { version } = await fetchLatestBaileysVersion();

  const sock = makeWASocket({
    version,
    auth: state,
    logger: pino({ level: 'silent' }),
    printQRInTerminal: false,
    browser: ['Ubuntu', 'Chrome', '20.0.04'],
    markOnlineOnConnect: false, // don't show "online" the whole time
  });

  active.set(number, sock);
  sock.ev.on('creds.update', saveCreds);

  // WhatsApp command handler
  // Anti-ban: only the linked account itself can give commands, they are answered one at a time
  // after a short human-like pause with "typing...", and command bursts are ignored.
  let queue = Promise.resolve();
  const lastCmd = new Map(); // chat -> time of its last command
  let stamps = []; // times of recent commands

  async function runCommand({ m, jid, text, isGroup, lag }) {
    try {
      try {
        await sock.sendPresenceUpdate('composing', jid);
      } catch (e) {}
      await sleep(1000 + Math.floor(Math.random() * 500)); // about 1 second
      try {
        await sock.sendPresenceUpdate('paused', jid);
      } catch (e) {}

      const currentPrefix = getPrefix();
      const cleanText = text.trim();
 
      const content = currentPrefix === 'none'
  ? cleanText
  : cleanText.slice(currentPrefix.length).trim();

      const [cmd, ...args] = content.split(/\s+/);
      const command = (cmd || '').toLowerCase();
      const pushname = m.pushName || 'User';
      const uptime = formatUptime(process.uptime());
      const reply = (t) => sock.sendMessage(jid, { text: t }, { quoted: m });

      switch (command) {
      case 'prefix':
  await reply(`⚡ Current prefix: ${getPrefix() === 'none' ? 'NONE (prefix-free)' : getPrefix()}`);
  break;

case 'set':
  if (args[0]?.toLowerCase() === 'prefix') {
    const newPrefix = args[1];

    if (!newPrefix) {
      await reply('Usage: set prefix ! or set prefix none');
    } else {
      setPrefix(newPrefix.toLowerCase() === 'none' ? 'none' : newPrefix);
      await reply(`✅ Prefix changed to: ${newPrefix}`);
    }
  }
  break;
        case 'menu':
        case 'help':
        case 'menu2':
        case 'listcmd':
        case 'allcmd':
          await sendMenu(sock, jid, m, buildMenu({ pushname, uptime }));
          break;
        case 'ping':
        case 'speed':
          await reply(`🏓 Pong! ${lag}ms`);
          break;
        case 'alive':
          await reply(`🖤 BLACKCIPHER MD BOT is alive!\n⏱️ Uptime: ${uptime}`);
          break;
        case 'uptime':
        case 'runtime':
          await reply(`⏱️ Uptime: ${uptime}`);
          break;
        case 'owner':
          await reply(`👑 Owner: ${OWNER}`);
          break;
        case 'info':
        case 'botinfo':
          await reply(
            `🖤 BLACKCIPHER MD BOT\n👑 Owner: ${OWNER}\n🤖 Version: ${VERSION}\n⚡ Prefix: ${PREFIX}\n⏱️ Uptime: ${uptime}`
          );
          break;
        default: {
          const wait = COOLDOWNS[command];
          if (wait) {
            const key = jid + '|' + command;
            const left = (lastUse.get(key) || 0) + wait - Date.now();
            if (left > 0) {
              await reply(`⏳ Please wait ${Math.ceil(left / 1000)}s before using .${command} again.`);
              break;
            }
            lastUse.set(key, Date.now());
          }
          if (['add', 'kick', 'promote', 'demote'].includes(command)) {
            // at most 5 people per command
            const ci = m.message?.extendedTextMessage?.contextInfo;
            if (ci?.mentionedJid) ci.mentionedJid = ci.mentionedJid.slice(0, 5);
            args.splice(5);
          }
          await handleCommand({ sock, m, jid, command, args, q: args.join(' '), reply, pushname, isGroup });
          break;
        }
      }
    } catch (e) {
      console.error('Command error:', e.message);
    }
  }

  sock.ev.on('messages.upsert', async ({ messages, type }) => {
    if (type !== 'notify') return;
    for (const m of messages) {
      try {
        if (!m.message || m.key.remoteJid === 'status@broadcast') continue;
        const jid = m.key.remoteJid;
        const msg = m.message;
        const text =
          msg.conversation ||
          msg.extendedTextMessage?.text ||
          msg.imageMessage?.caption ||
          msg.videoMessage?.caption ||
          '';
        const isGroup = jid.endsWith('@g.us');

        // anti-link / anti-spam checks for group messages
        if (isGroup && !m.key.fromMe) {
          await onGroupMessage({ sock, m, jid, text }).catch((e) => console.error('Group check:', e.message));
        }
       const currentPrefix = getPrefix();

const isPrefixCommand =
  text.trim() === '.prefix' ||
  text.trim().startsWith('.set prefix ');

if (
  !isPrefixCommand &&
  currentPrefix !== 'none' &&
  !text.startsWith(currentPrefix)
) continue;


        // commands from other people are ignored
        if (!m.key.fromMe) continue;

        // ignore bursts: 1 command per chat every 2 seconds, 20 per minute
        const now = Date.now();
        if (now - (lastCmd.get(jid) || 0) < 2000) continue;
        stamps = stamps.filter((t) => now - t < 60000);
        if (stamps.length >= 20) continue;
        lastCmd.set(jid, now);
        stamps.push(now);

        const sentAt = Number(m.messageTimestamp) * 1000;
        const lag = Number.isFinite(sentAt) ? Math.max(0, now - sentAt) : 0;
        queue = queue.then(() => runCommand({ m, jid, text, isGroup, lag }));
      } catch (e) {
        console.error('Command error:', e.message);
      }
    }
  });

  // Welcome / goodbye messages
  sock.ev.on('group-participants.update', (u) => {
    sleep(2000) // short pause before the automatic message
      .then(() => onGroupUpdate(sock, u))
      .catch((e) => console.error('Group update:', e.message));
  });

  // Request the pairing code once, for a fresh (unregistered) session
  if (firstTime && !sock.authState.creds.registered) {
    setTimeout(async () => {
      try {
        const code = await sock.requestPairingCode(number);
        const pretty = code.match(/.{1,4}/g).join('-');
        tg.sendMessage(
          chatId,
          `🔑 PAIRING CODE\n\n${pretty}\n\nOpen WhatsApp ➜ Settings ➜ Linked devices ➜ Link a device ➜ Link with phone number instead, then enter this code.\n\n⚠️ It expires quickly, so enter it now.`
        );
      } catch (e) {
        tg.sendMessage(chatId, '❌ Failed to generate a pairing code.\nCheck the number and try again.');
        active.delete(number);
        pendingNotify.delete(number);
      }
    }, 3000);
  } else if (firstTime) {
    tg.sendMessage(chatId, 'ℹ️ This number is already linked. Reconnecting...');
  }

  sock.ev.on('connection.update', ({ connection, lastDisconnect }) => {
    if (connection === 'open') {
      retries.delete(number);
      connected.add(number);
      // tell Telegram only once, right after pairing (not on every reconnect)
      if (pendingNotify.delete(number)) {
        tg.sendMessage(
          chatId,
          `✅ WHATSAPP CONNECTED\n\n📱 Number: +${number}\n${BRAND} is now active.\n\nSend ${PREFIX}menu on WhatsApp to see all commands.`
        );
      }
    }
    if (connection === 'close') {
      const status = lastDisconnect?.error?.output?.statusCode;
      console.error('WhatsApp disconnected:', status, lastDisconnect?.error?.message || 'No error message');
      active.delete(number);
      connected.delete(number);
      if (status === DisconnectReason.loggedOut) {
        retries.delete(number);
        pendingNotify.delete(number);
        // remove the dead session so the number can be paired again with a fresh code
        try {
          fs.rmSync(`./sessions/${number}`, { recursive: true, force: true });
        } catch (e) {}
        tg.sendMessage(chatId, `⚠️ WhatsApp +${number} was logged out.\n\nUse /menu ➜ Pair WhatsApp to link it again.`);
      } else {
        // Includes the normal restart right after pairing
        scheduleReconnect(number, chatId);
      }
    }
  });
}

// ---------------------------------------------------------------- Telegram panel
// Settings only appears for the owner
const mainKeyboard = (uid) => ({
  inline_keyboard: [
    [{ text: '🔗 Pair WhatsApp', callback_data: 'pair' }],
    [
      { text: '📊 Bot Status', callback_data: 'status' },
      { text: '🏓 Ping', callback_data: 'ping' },
    ],
    [{ text: '⏱️ Uptime', callback_data: 'uptime' }],
    isOwner(uid)
      ? [
          { text: '👤 Owner', callback_data: 'owner' },
          { text: '⚙️ Settings', callback_data: 'settings' },
        ]
      : [{ text: '👤 Owner', callback_data: 'owner' }],
    [{ text: '❓ Help', callback_data: 'help' }],
  ],
});

const PANEL_TEXT = `${BRAND}\n\nControl Panel`;
const sendPanel = (id, uid) => tg.sendMessage(id, PANEL_TEXT, { reply_markup: mainKeyboard(uid) });

const askNumber = (id) => {
  awaitingNumber.add(id);
  return tg.sendMessage(
    id,
    '🔗 WHATSAPP PAIRING\n\n📱 Please enter your WhatsApp number with country code.\n\nExample: 23480xxxxxxxx\n\n⚠️ Do not include the + sign.\nSend /cancel to cancel.'
  );
};

const cancelPairing = (id) => {
  const was = awaitingNumber.delete(id);
  return tg.sendMessage(id, was ? '❌ Pairing cancelled.' : 'Nothing to cancel.');
};

function parseNumber(text) {
  const n = String(text || '').replace(/\D/g, '');
  if (n.startsWith('0')) {
    return { error: '⚠️ Start with your country code, not 0.\nExample: 23480xxxxxxxx\n\nSend /cancel to cancel.' };
  }
  if (n.length < 7 || n.length > 15) {
    return { error: '⚠️ That number looks wrong. Use the country code and digits only.\nExample: 23480xxxxxxxx\n\nSend /cancel to cancel.' };
  }
  return { number: n };
}

async function startPairing(id, text) {
  const { number, error } = parseNumber(text);
  if (error) return tg.sendMessage(id, error);
  awaitingNumber.delete(id);
  if (active.has(number)) {
    return tg.sendMessage(id, 'ℹ️ This number already has an active session.');
  }
  await tg.sendMessage(id, '⏳ Generating your pairing code...');
  pendingNotify.add(number);
  startSession(number, id, true).catch((e) => {
    pendingNotify.delete(number);
    console.error('Pairing failed:', e.message);
    tg.sendMessage(id, '❌ Something went wrong. Please try again.');
  });
}

async function sendPing(id) {
  const t0 = Date.now();
  await tg.getMe().catch(() => {});
  const ms = Date.now() - t0;
  return tg.sendMessage(id, `🏓 PONG!\n\n⚡ Response Time: ${ms} ms\n🤖 Status: Online\n${BRAND}`);
}

const sendStatus = (id) =>
  tg.sendMessage(
    id,
    `📊 BOT STATUS\n\n🤖 Status: Online\n⏱️ Uptime: ${formatUptime(process.uptime())}\n📱 WhatsApp online: ${connected.size}\n💾 Memory: ${Math.round(process.memoryUsage().rss / 1048576)} MB\n🤖 Version: ${VERSION}\n${BRAND}`
  );

const sendUptime = (id) => tg.sendMessage(id, `⏱️ UPTIME\n\n${formatUptime(process.uptime())}\n${BRAND}`);

const sendOwner = (id) => tg.sendMessage(id, `👤 OWNER\n\n👑 ${OWNER}\n${BRAND}`);

const sendHelp = (id) =>
  tg.sendMessage(
    id,
    `❓ HELP\n\n1️⃣ Tap 🔗 Pair WhatsApp and send your number with the country code.\n2️⃣ Enter the pairing code in WhatsApp ➜ Linked devices.\n3️⃣ On WhatsApp, send ${PREFIX}menu to see all commands.\n\nTelegram commands:\n/menu - control panel\n/pair - pair a number\n/ping - response time\n/status - bot status\n/cancel - cancel`
  );

// ----- owner-only settings
const BACK = (to) => [{ text: '🔙 Back', callback_data: to }];

function settingsView() {
  return {
    text:
      `⚙️ SETTINGS\n\n🤖 Bot Mode: ${modeLabel()}\n💬 Telegram users: ${Object.keys(config.users).length}\n` +
      `📱 WhatsApp linked: ${linkedNumbers().length} (online: ${connected.size})\n⚡ Prefix: ${PREFIX}\n🤖 Version: ${VERSION}\n${BRAND}`,
    keyboard: {
      inline_keyboard: [
        [
          { text: '🤖 Bot Mode', callback_data: 'set:mode' },
          { text: '👥 Users', callback_data: 'set:users' },
        ],
        BACK('set:home'),
      ],
    },
  };
}

function modeView() {
  const mark = (m) => (config.mode === m ? '✅ ' : '');
  return {
    text:
      `🤖 BOT MODE\n\nCurrent: ${modeLabel()}\n\n` +
      '🔒 Private\n• Only you can use this Telegram panel and pair numbers\n\n' +
      '🌐 Public\n• Anyone can open the panel and pair their own number\n\n' +
      'ℹ️ WhatsApp commands only work from the linked account itself, in both modes.\n\nChoose a mode:',
    keyboard: {
      inline_keyboard: [
        [
          { text: `${mark('private')}🔒 Private`, callback_data: 'set:private' },
          { text: `${mark('public')}🌐 Public`, callback_data: 'set:public' },
        ],
        BACK('set:root'),
      ],
    },
  };
}

function usersView() {
  const linked = linkedNumbers();
  const lines = linked.slice(0, 20).map((n) => `${connected.has(n) ? '🟢' : '⚪'} ${mask(n)}`);
  return {
    text:
      `👥 USERS\n\n💬 Telegram users: ${Object.keys(config.users).length}\n📱 WhatsApp linked: ${linked.length}\n🟢 Online now: ${connected.size}\n\n` +
      (lines.length ? lines.join('\n') + (linked.length > 20 ? `\n...and ${linked.length - 20} more` : '') : 'No WhatsApp numbers linked yet.'),
    keyboard: {
      inline_keyboard: [[{ text: '🔄 Refresh', callback_data: 'set:users' }], BACK('set:root')],
    },
  };
}

const sendSettings = (id) => {
  const v = settingsView();
  return tg.sendMessage(id, v.text, { reply_markup: v.keyboard });
};

async function showView(q, view) {
  const chat_id = q.message.chat.id;
  try {
    await tg.editMessageText(view.text, { chat_id, message_id: q.message.message_id, reply_markup: view.keyboard });
  } catch (e) {
    if (!/not modified/i.test(e.message)) await tg.sendMessage(chat_id, view.text, { reply_markup: view.keyboard });
  }
}

async function handleSettingsTap(q) {
  const data = q.data;
  if (data === 'set:home') {
    await showView(q, { text: PANEL_TEXT, keyboard: mainKeyboard(q.from.id) });
  } else if (data === 'set:root') {
    await showView(q, settingsView());
  } else if (data === 'set:mode') {
    await showView(q, modeView());
  } else if (data === 'set:users') {
    await showView(q, usersView());
  } else if (data === 'set:private' || data === 'set:public') {
    config.mode = data === 'set:private' ? 'private' : 'public';
    saveConfig();
    await showView(q, modeView());
    return `✅ Mode set to ${config.mode === 'private' ? 'Private' : 'Public'}`;
  }
  return '';
}

const actions = {
  pair: askNumber,
  ping: sendPing,
  status: sendStatus,
  uptime: sendUptime,
  owner: sendOwner,
  help: sendHelp,
};

tg.setMyCommands([
  { command: 'menu', description: 'Open the control panel' },
  { command: 'pair', description: 'Pair a WhatsApp number' },
  { command: 'ping', description: 'Check response time' },
  { command: 'status', description: 'Bot status' },
  { command: 'myid', description: 'Show your Telegram ID' },
  { command: 'cancel', description: 'Cancel the current action' },
]).catch(() => {});

// Private mode blocks everyone except the owner
function gate(msg) {
  if (isAllowed(msg.from && msg.from.id)) return true;
  tg.sendMessage(msg.chat.id, '🔒 This bot is private.\n\nOnly the owner can use it right now.');
  return false;
}
const cmd = (re, fn) =>
  tg.onText(re, (msg, match) => {
    track(msg.from);
    if (gate(msg)) fn(msg, match);
  });

cmd(/^\/start(?:@\w+)?\s*$/i, (msg) => tg.sendMessage(msg.chat.id, 'Use /menu to open the control panel.'));
cmd(/^\/menu(?:@\w+)?\s*$/i, (msg) => sendPanel(msg.chat.id, msg.from.id));
cmd(/^\/cancel(?:@\w+)?\s*$/i, (msg) => cancelPairing(msg.chat.id));
cmd(/^\/ping(?:@\w+)?\s*$/i, (msg) => sendPing(msg.chat.id));
cmd(/^\/status(?:@\w+)?\s*$/i, (msg) => sendStatus(msg.chat.id));
cmd(/^\/uptime(?:@\w+)?\s*$/i, (msg) => sendUptime(msg.chat.id));
cmd(/^\/pair(?:@\w+)?(?:\s+(.+))?$/i, (msg, match) => {
  if (match[1]) return startPairing(msg.chat.id, match[1]);
  return askNumber(msg.chat.id);
});

// /myid always works, so you can find your ID before OWNER_ID is set
tg.onText(/^\/myid(?:@\w+)?\s*$/i, (msg) =>
  tg.sendMessage(
    msg.chat.id,
    `🆔 YOUR TELEGRAM ID\n\n<code>${msg.from.id}</code>\n\n${isOwner(msg.from.id) ? '✅ You are the owner of this bot.' : 'Set this number as OWNER_ID on the server to become the owner.'}`,
    { parse_mode: 'HTML' }
  )
);

// /settings is for the owner only: anyone else gets no reply at all
tg.onText(/^\/settings(?:@\w+)?\s*$/i, (msg) => {
  if (isOwner(msg.from && msg.from.id)) sendSettings(msg.chat.id);
});

// The number typed after tapping "Pair WhatsApp"
tg.on('message', (msg) => {
  const id = msg.chat.id;
  if (!awaitingNumber.has(id) || !msg.text || msg.text.startsWith('/')) return;
  track(msg.from);
  if (!gate(msg)) {
    awaitingNumber.delete(id);
    return;
  }
  startPairing(id, msg.text);
});

// Button taps
tg.on('callback_query', async (q) => {
  const id = q.message && q.message.chat && q.message.chat.id;
  let toast = '';
  try {
    if (!id) return;
    track(q.from);
    if (typeof q.data === 'string' && (q.data === 'settings' || q.data.startsWith('set:'))) {
      if (!isOwner(q.from.id)) {
        toast = '⛔ Not available';
      } else if (q.data === 'settings') {
        await sendSettings(id);
      } else {
        toast = await handleSettingsTap(q);
      }
    } else if (!isAllowed(q.from.id)) {
      toast = '🔒 This bot is private.';
    } else if (actions[q.data]) {
      await actions[q.data](id);
    }
  } catch (e) {
    console.error('Button error:', e.message);
  } finally {
    tg.answerCallbackQuery(q.id, toast ? { text: toast } : {}).catch(() => {});
  }
});

console.log('Bot is running...');

// ---------------------------------------------------------------- auto-reconnect saved sessions
(function restoreSessions() {
  const dir = './sessions';
  if (!fs.existsSync(dir)) return;
  let i = 0;
  for (const number of fs.readdirSync(dir)) {
    try {
      const creds = JSON.parse(fs.readFileSync(`${dir}/${number}/creds.json`, 'utf8'));
      if (!creds.registered || active.has(number)) continue;
      setTimeout(() => {
        console.log('Reconnecting WhatsApp session:', number);
        startSession(number, null, false).catch((e) => console.error('Restore failed:', e.message));
      }, 3000 * i++);
    } catch (e) {}
  }
})();

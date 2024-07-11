# Phoenix-
const { Client, LocalAuth } = require('whatsapp-web.js');
const { Sequelize } = require("sequelize");
const fs = require("fs");
if (fs.existsSync("config.env"))
  require("dotenv").config({ path: "./config.env" });

const toBool = (x) => x == "true";

DATABASE_URL = process.env.DATABASE_URL || "./lib/database.db";
let HANDLER = "false";

const client = new Client({
    authStrategy: new LocalAuth(),
    puppeteer: {
        headless: false,
        args: ['--no-sandbox', '--disable-setuid-sandbox'],
    }
});

module.exports = {
  // For Enabling Commands Like AUTO_STATUS_READ Type true For Disabling Type false
  ANTILINK: toBool(process.env.ANTI_LINK) || false,
  ANTILINK_ACTION: process.env.ANTI_LINK || "kick",
  AUTO_REACT: process.env.AUTO_REACT || 'false',
  AUTO_STATUS_READ: process.env.AUTO_STATUS_READ || 'true',
  AUTO_BIO: process.env.AUTO_BIO || 'false',
  AUTO_READ_MSG: process.env.AUTO_READ_MSG || 'false',
  AUDIO_DATA: process.env.AUDIO_DATA || "Phoenix-MD;Abhishek Suresh;https://graph.org/file/8976892f2f615077b48cd.jpg",
  DATABASE_URL: DATABASE_URL,
  OWNER_NAME: process.env.OWNER_NAME || "Simba",
  OWNER_NUMBER: process.env.OWNER_NUMBER || "0748034646",
  BOT_NAME: process.env.BOT_NAME || "Phoenix-MD",
  WORK_TYPE: process.env.MODE || "public",
  BASE_URL: "https://abhi-api-6b6cddf94f7b.herokuapp.com/",
  DATABASE:
    DATABASE_URL === "./lib/database.db"
      ? new Sequelize({
          dialect: "sqlite",
          storage: DATABASE_URL,
          logging: false,
        })
      : new Sequelize(DATABASE_URL, {
          dialect: "postgres",
          ssl: true,
          protocol: "postgres",
          dialectOptions: {
            native: true,
            ssl: { require: true, rejectUnauthorized: false },
          },
          logging: false,
        }),
};

// Initialize the client
client.initialize();

// Generate QR code for authentication
client.on('qr', (qr) => {
    const qrcode = require('qrcode-terminal');
    qrcode.generate(qr, { small: true });
    console.log('Scan the QR code above to log in.');
});

// Once authenticated
client.on('ready', () => {
    console.log('Client is ready!');
    
    // Auto view statuses
    client.on('change_state', (newState) => {
        if (newState === 'CONNECTED') {
            client.getContacts().then((contacts) => {
                contacts.forEach(contact => {
                    client.getStatus(contact.id._serialized).then((status) => {
                        console.log(`Status of ${contact.name}: ${status}`);
                    });
                });
            });
        }
    });
});

// Anti-delete feature: send deleted messages to your inbox
client.on('message_revoke_everyone', async (after, before) => {
    const myNumber = process.env.OWNER_NUMBER + '@c.us'; // Change to your WhatsApp number with @c.us suffix
    if (before) {
        const message = `Anti-delete: Message from ${before.author || before.from}: ${before.body}`;
        console.log(message);
        await client.sendMessage(myNumber, `Someone tried to delete this message: "${before.body}"`);
    }
});

// To check if the bot is working properly
client.on('message', async message => {
    if (message.body === '!test') {
        await client.sendMessage(message.from, 'The bot is working properly.');
    }
});

// New Command: Read message and handle deleted messages
client.on('message_create', async (msg) => {
    // Check if the message was sent by the user (not the bot)
    if (msg.fromMe) return;

    // Store the message content in a variable
    const originalMessageContent = msg.body;
    const senderNumber = msg.from;

    // Listen for message deletion
    client.on('message_revoke_everyone', async (after, before) => {
        if (before && before.id._serialized === msg.id._serialized) {
            const myNumber = process.env.OWNER_NUMBER + '@c.us'; // Change to your WhatsApp number with @c.us suffix
            const message = `Anti-delete: Message from ${senderNumber}: ${originalMessageContent}`;
            console.log(message);
            await client.sendMessage(myNumber, `Someone tried to delete this message: "${originalMessageContent}"`);
        }
    });
});

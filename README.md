const { default: makeWASocket, DisconnectReason, useSingleFileAuthState } = require('@adiwajshing/baileys');
const { Boom } = require('@hapi/boom');
const fs = require('fs');

const { state, saveState } = useSingleFileAuthState('./auth_info.json');

async function startBot() {
  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: true,
  });

  sock.ev.on('messages.upsert', async (m) => {
    const msg = m.messages[0];
    if (!msg.message) return; // Ignorar mensajes vacíos
    if (msg.key.fromMe) return; // Ignorar mensajes del bot

    const chatId = msg.key.remoteJid;
    const senderId = msg.key.participant || msg.key.remoteJid;

    if (msg.message.conversation) {
      const text = msg.message.conversation;

      // Detectar enlaces
      if (text.includes('http://') || text.includes('https://')) {
        // Eliminar mensaje
        await sock.sendMessage(chatId, { text: '🚫 No se permiten enlaces en este grupo.' });
        
        // Expulsar al usuario (asegurarse de que el bot tenga permisos de administrador)
        if (chatId.endsWith('@g.us')) {
          await sock.groupParticipantsUpdate(chatId, [senderId], 'remove');
        }
      }
    }
  });

  sock.ev.on('connection.update', (update) => {
    const { connection, lastDisconnect } = update;
    if (connection === 'close') {
      const shouldReconnect = (lastDisconnect.error = Boom)?.output?.statusCode !== DisconnectReason.loggedOut;
      console.log('connection closed due to ', lastDisconnect.error, ', reconnecting ', shouldReconnect);
      if (shouldReconnect) startBot();
    } else if (connection === 'open') {
      console.log('Conexión exitosa.');
    }
  });

  sock.ev.on('creds.update', saveState);
}

startBot();
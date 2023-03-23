# sadbot

const algosdk = require('algosdk');
const schedule = require('node-schedule');

const Discord = require('discord.js');

const Intents = Discord.Intents;
const client = new Discord.Client({
  intents: [
    Intents.FLAGS.GUILDS,
    Intents.FLAGS.GUILD_MESSAGE_CONTENT
  ],
});

const BOT_TOKEN = '**';
const ALGOD_API_KEY = '**';
const ALGORAND_WALLET_MNEMONIC = '**';
const ROLE_NAME = 'Clubhouse Member';
const CREATOR_WALLET = '**';
const COMMUNITY_TOKEN_ID = 1036863015;
const DEPOSIT_AMOUNT = 25;
const DEPOSIT_INTERVAL = '33 15 * * 1,5'; // Monday and Friday at 3:33 PM EST

client.on('ready', () => {
  console.log(`Logged in as ${client.user.tag}!`);
});

client.on('messageCreate', async (message) => {
  console.log('Message event triggered'); // NEW

  if (message.author.bot) return; // Ignore messages from bots.

  console.log(`Received message: ${message.content}`);
  
  if (message.content.startsWith('!verify')) {
    console.log('Verification command received'); // NEW
    await verifyNFTHolder(message);
  }
});

async function verifyNFTHolder(message) {
  console.log('verifyNFTHolder() called');
  const userAddress = message.content.split(' ')[1];
  if (!userAddress) {
    message.reply('Please provide your Algorand address.');
    return;
  }

  const algodClient = new algosdk.Algodv2(ALGOD_API_KEY, 'https://mainnet-algorand.api.purestake.io/ps2', '');
  const accountInfo = await algodClient.accountInformation(userAddress).do();
  const nftCount = accountInfo.assets.filter(asset => asset['creator'] === CREATOR_WALLET).length;

  if (nftCount > 0) {
    const role = message.guild.roles.cache.find((role) => role.name === ROLE_NAME);
    if (!role) {
      message.reply('Clubhouse Member role not found. Please contact an admin.');
      return;
    }
    const member = message.guild.members.cache.get(message.author.id);
    member.roles.add(role);
    message.reply('You have been verified and granted the Clubhouse Member role.');

    // Schedule token deposits
    scheduleDeposits(userAddress, nftCount);
  } else {
    message.reply('You are not a holder of the NFT. Verification failed.');
  }
}

async function scheduleDeposits(userAddress, nftCount) {
  const depositAmount = DEPOSIT_AMOUNT * nftCount;

  const job = schedule.scheduleJob(DEPOSIT_INTERVAL, async () => {
    const algodClient = new algosdk.Algodv2(ALGOD_API_KEY, 'https://mainnet-algorand.api.purestake.io/ps2', '');

    const recoveredAccount = algosdk.mnemonicToSecretKey(ALGORAND_WALLET_MNEMONIC);

    const suggestedParams = await algodClient.getTransactionParams().do();

const txn = {
from: recoveredAccount.addr,
to: userAddress,
fee: suggestedParams.fee,
amount: depositAmount,
firstRound: suggestedParams.firstRound,
lastRound: suggestedParams.lastRound,
genesisID: suggestedParams.genesisID,
genesisHash: suggestedParams.genesisHash,
assetIndex: COMMUNITY_TOKEN_ID,
type: "axfer",
};

const signedTxn = algosdk.signTransaction(txn, recoveredAccount.sk);
try {
  const txConfirmation = await algodClient.sendRawTransaction(signedTxn.blob).do();
  console.log(`Transaction ${txConfirmation.txId} confirmed.`);
} catch (err) {
  console.error(`Failed to send transaction: ${err}`);
}
});
}

client.login(BOT_TOKEN);

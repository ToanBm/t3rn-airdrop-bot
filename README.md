# t3rn-airdrop-bot
## Update:
```Bash
rm arbt.js && nano arbt.js
```
```Bash
require('colors');
const { Wallet, JsonRpcProvider, ethers, parseUnits } = require('ethers');
const fs = require('fs');
const path = require('path');

const readlineSync = require('readline-sync');
const moment = require('moment');
const T3RN_ABI = require('./contracts/ABI');
const { displayHeader } = require('./utils/display');
const { transactionData, delay } = require('./chains/arbt/helper');
const { getAmount } = require('./chains/arbt/api');

const TOKEN_FILE_PATH = path.join(__dirname, 'ARBT_TX_HASH.txt');

const PRIVATE_KEYS = JSON.parse(fs.readFileSync('privateKeys.json', 'utf-8'));
const RPC_URL = T3RN_ABI.at(-1).RPC_ARBT;

const provider = new JsonRpcProvider(RPC_URL);
const CONTRACT_ADDRESS = T3RN_ABI.at(-1).CA_ARBT;

(async () => {
  displayHeader();
  console.log('⏳ Please wait...'.yellow);
  console.log('');

  const options = readlineSync.question(
    'Choose the network that you want to use \n\n1. Arbitrum Sepolia to Base Sepolia\n2. Arbitrum Sepolia to Blast Sepolia\n3. Arbitrum Sepolia to Optimism Sepolia\n4. Exit\n\nEnter 1, 2, 3, or 4: '
  );

  if (options === '4' || !options) {
    console.log('Exiting the bot. See you next time!'.cyan);
    console.log('Subscribe: https://t.me/HappyCuanAirdrop.'.green);
    process.exit(0);
  }

  const numTx = readlineSync.questionInt(
    'How many times you want to swap or bridge? '
  );

  if (numTx <= 0) {
    console.log('Number of transactions must be greater than 0!'.red);
    process.exit(1);
  }

  for (const PRIVATE_KEY of PRIVATE_KEYS) {
    const wallet = new Wallet(PRIVATE_KEY, provider);
    let totalSuccess = 0;

    while (totalSuccess < numTx) {
      try {
        const balance = await provider.getBalance(wallet.address);
        const balanceInEth = ethers.formatUnits(balance, 'ether');

        console.log(
          `⚙️ [ ${moment().format(
            'HH:mm:ss'
          )} ] Doing transactions for address ${wallet.address}...`.yellow
        );

        if (balanceInEth < 0.01) {
          console.log(
            `[ ${moment().format(
              'HH:mm:ss'
            )} ] Your balance is too low (💰 ${balanceInEth} ETH), please claim faucet first!`
              .red
          );
          process.exit(0);
        }

        let counter = numTx - totalSuccess;

        while (counter > 0) {
          try {
            const amount = await getAmount(options);
            if (!amount) {
              console.log(
                `Failed to get the amount. Skipping transaction...`.red
              );
              continue;
            }

            const request = transactionData(
              wallet.address,
              amount.hex,
              options
            );

            const gasPrice = parseUnits('0.1', 'gwei');

            const gasLimit = await provider.estimateGas({
              to: CONTRACT_ADDRESS,
              data: request,
              value: parseUnits('0.1', 'ether'),
              gasPrice,
            });

            const transaction = {
              data: request,
              to: CONTRACT_ADDRESS,
              gasLimit,
              gasPrice,
              from: wallet.address,
              value: parseUnits('0.1', 'ether'), // adjustable
            };

            const result = await wallet.sendTransaction(transaction);
            console.log(
              `[ ${moment().format(
                'HH:mm:ss'
              )} ] Transaction successful from Arbitrum Sepolia to ${
                options === '1' ? 'Base' : options === '2' ? 'Blast' : 'OP'
              } Sepolia!`.green
            );
            console.log(
              `[ ${moment().format(
                'HH:mm:ss'
              )} ] Transaction hash: https://sepolia-explorer.arbitrum.io/tx/${
                result.hash
              }`.green
            );
            fs.writeFileSync(
              TOKEN_FILE_PATH,
              `https://sepolia-explorer.arbitrum.io/tx/${result.hash}`
            );
            console.log(
              'Transaction hash url has been saved to ARBT_TX_HASH.txt.'
                .green
            );
            console.log('');

            totalSuccess++;
            counter--;

            if (counter > 0) {
              await delay(30000);
            }
          } catch (error) {
            console.log(
              `[ ${moment().format(
                'HH:mm:ss'
              )} ] Error during transaction: ${error}`.red
            );
          }
        }
      } catch (error) {
        console.log(
          `[ ${moment().format(
            'HH:mm:ss'
          )} ] Error in processing transactions: ${error}`.red
        );
      }
    }
  }

  console.log('');
  console.log(
    `[ ${moment().format(
      'HH:mm:ss'
    )} ] All ${numTx} transactions are complete!`.green
  );
  console.log(
    `[ ${moment().format(
      'HH:mm:ss'
    )} ] Subscribe: `.green
  );
})();
```

```Bash
rm chains/arbt/api.js && nano chains/arbt/api.js
```
```Bash
const axios = require('axios');
const moment = require('moment');

async function getAmount(chain) {
  try {
    const { data } = await axios({
      url: 'https://pricer.t1rn.io/estimate',
      method: 'POST',
      data: {
        fromAsset: 'eth',
        toAsset: 'eth',
        fromChain: 'arbt',
        toChain: chain === '1' ? 'bssp' : chain === '2' ? 'blss' : 'opsp',
        amountWei: '100000000000000000',
        executorTipUSD: 0,
        overpayOptionPercentage: 0,
        spreadOptionPercentage: 0,
      },
    });

    return data.estimatedReceivedAmountWei;
  } catch (error) {
    console.log(
      `❌ [ ${moment().format('HH:mm:ss')} ] Error in Get Amount: ${error}`.red
    );
    return null;
  }
}

module.exports = { getAmount };
```

```Bash
rm chains/arbt/helper.js && nano chains/arbt/helper.js
```
```Bash
function transactionData(address, amount, chain) {
  return `${
    chain === '1'
      ? '0x56591d5962737370'
      : chain === '2'
      ? '0x56591d59626c7373'
      : '0x56591d596f707370'
  }000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000${address.slice(
    2
  )}00000000000000000000000000000000000000000000000${amount.slice(
    2
  )}000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000016345785d8a0000`;
}

const delay = (ms) => {
  return new Promise((resolve) => setTimeout(resolve, ms));
};

module.exports = { transactionData, delay };
```















A bot designed to automate transactions and bridge assets on the t3rn network, making the process seamless and efficient. Now supports both Optimism Sepolia and Arbitrum Sepolia testnets.

## Features

- Automates asset bridging and swapping on the t3rn network.
- Supports multiple wallets through a JSON file containing private keys.
- Robust error handling with retry mechanisms to ensure all transactions are completed.
- User-friendly and easy to set up.
- Supports bridging from **Optimism Sepolia** and **Arbitrum Sepolia**.

## Requirements

- Node.js (v14 or later)
- NPM (v6 or later)
- Private keys for the wallets you intend to use (stored in `privateKeys.json`).

## Installation

1. **Clone the Repository**:

   ```bash
   git clone https://github.com/dante4rt/t3rn-airdrop-bot.git
   cd t3rn-airdrop-bot
   ```

2. **Install Dependencies**:

   ```bash
   npm install
   ```

3. **Create `privateKeys.json`**:
   Create a file named `privateKeys.json` in the root directory with the following format:

   ```json
   [
     "your_private_key"
   ]
   ```

4. **Run the Bot**:

   - To check the available menu options:

     ```bash
     npm start
     ```

   - To run the bot for **Arbitrum Sepolia**:

     ```bash
     npm run arbt
     ```

   - To run the bot for **Optimism Sepolia**:

     ```bash
     npm run opsp
     ```

## Usage

- Use `npm start` to check the menu options available.
- Choose the appropriate command based on the network you want to use.
- The bot will automatically execute the transactions, handling any errors and retrying as needed.

## Donations

If you would like to support the development of this project, you can make a donation using the following addresses:

- **Solana**: `GLQMG8j23ookY8Af1uLUg4CQzuQYhXcx56rkpZkyiJvP`
- **EVM**: `0x960EDa0D16f4D70df60629117ad6e5F1E13B8F44`
- **BTC**: `bc1p9za9ctgwwvc7amdng8gvrjpwhnhnwaxzj3nfv07szqwrsrudfh6qvvxrj8`

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

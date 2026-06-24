# Урок 10: Deploying DApps with Truffle

- Трек: `Advanced Solidity: Get In-depth Knowledge`
- Глав на сайте: **11**
- Элементов урока в repo: **12**
- Русская локаль: **нет**
- Sample code в `cryptozombies-lesson-code`: **да**
- Источник кода: **В `../cryptozombies-lesson-code/lesson-10/` есть только часть sample code: `chapter-4/contracts/2_crypto_zombies.js`, `chapter-5/truffle.js` и `chapter-8/truffle.js`. Полного зеркала всех глав нет. Русская локаль `ru/10/` отсутствует.**

## Что нужно уметь после урока

- Что создаёт `truffle init` и почему Truffle ожидает стандартную структуру `contracts/`, `migrations/`, `test/` и config file.
- Почему Solidity-код нужно компилировать до деплоя и что build artifacts содержат bytecode, ABI и deployment metadata.
- Что такое migration script, почему migrations выполняются по порядку и как используется `deployer.deploy(...)`.
- Как `truffle-hdwallet-provider` связывает mnemonic и Infura URL, чтобы Truffle мог подписывать Ethereum-транзакции.
- Как описывать отдельные network blocks для `mainnet` и `rinkeby`, включая `network_id: 4` для Rinkeby.
- Как Loom deploy повторяет Ethereum deploy через `loom-truffle-provider`, `chainId`, `writeUrl`, `readUrl` и private key.
- Почему секреты нельзя хранить в коммитах и как пример Basechain переносит private key в отдельный файл.
- Какие команды соответствуют целевым сетям: `truffle migrate --network rinkeby`, `--network loom_testnet`, `--network basechain`.

## Краткое summary

Урок 10 учит оборачивать существующий Solidity-проект в workflow Truffle и деплоить его в несколько сетей. Нужно понять структуру проекта Truffle, этап компиляции в ABI/bytecode artifacts, migrations, конфигурацию сетей через `HDWalletProvider` и Infura, а затем перенести тот же deploy flow на Loom testnet и Basechain через `LoomTruffleProvider`. Практический итог урока — deployable контракт `CryptoZombies`, migration script и `truffle.js`, из которого можно деплоить в Rinkeby, Loom testnet и Basechain.

## Ключевые концепции

- **структура проекта и соглашения Truffle**
- **компиляция Solidity в ABI/bytecode artifacts**
- **migration scripts и упорядоченный deploy**
- **HDWalletProvider + Infura для Ethereum deploy**
- **Rinkeby как безопасная testnet перед mainnet**
- **LoomTruffleProvider и деплой в Loom**
- **загрузка private key из файла для Basechain**

## Код, который нужно понимать

### Обёртка над игрой для деплоя

```solidity
pragma solidity ^0.4.24;

import "./zombieownership.sol";

contract CryptoZombies is ZombieOwnership
    {

    }
```

Источник: `en/10/03.md`
### Migration script для деплоя CryptoZombies

```javascript
var CryptoZombies = artifacts.require("./CryptoZombies.sol");
module.exports = function(deployer) {
  deployer.deploy(CryptoZombies);
};
```

Источник: `../cryptozombies-lesson-code/lesson-10/chapter-4/contracts/2_crypto_zombies.js`
### Конфигурация Ethereum-сетей через HDWalletProvider

```javascript
const HDWalletProvider = require("truffle-hdwallet-provider");

const mnemonic = "YOUR_MNEMONIC";

module.exports = {

    networks: {

        mainnet: {
            provider: function () {

                return new HDWalletProvider(mnemonic, "https://mainnet.infura.io/v3/YOUR_TOKEN")
            },
            network_id: "1"
        },

        rinkeby: {

            provider: function () {

                return new HDWalletProvider(mnemonic, "https://rinkeby.infura.io/v3/YOUR_TOKEN")
            },

            network_id: 4
        }
    }
};
```

Источник: `../cryptozombies-lesson-code/lesson-10/chapter-5/truffle.js`
### Блок конфигурации Loom testnet

```javascript
loom_testnet: {
  provider: function() {
      const privateKey = 'YOUR_PRIVATE_KEY'
      const chainId = 'extdev-plasma-us1';
      const writeUrl = 'http://extdev-plasma-us1.dappchains.com:80/rpc';
      const readUrl = 'http://extdev-plasma-us1.dappchains.com:80/query';;
      return new LoomTruffleProvider(chainId, writeUrl, readUrl, privateKey)
  },
  network_id: '9545242630824'
}
```

Источник: `../cryptozombies-lesson-code/lesson-10/chapter-8/truffle.js`
### Хелпер для загрузки private key Basechain

```javascript
function getLoomProviderWithPrivateKey (privateKeyPath, chainId, writeUrl, readUrl) {
  const privateKey = readFileSync(privateKeyPath, 'utf-8');
  return new LoomTruffleProvider(chainId, writeUrl, readUrl, privateKey);
}
```

Источник: `en/10/10.md`
### Конфигурация сети Basechain

```javascript
basechain: {
  provider: function() {
    const chainId = 'default';
    const writeUrl = 'http://basechain.dappchains.com/rpc';
    const readUrl = 'http://basechain.dappchains.com/query';
    const privateKeyPath = path.join(__dirname, 'mainnet_private_key');
    const loomTruffleProvider = getLoomProviderWithPrivateKey(privateKeyPath, chainId, writeUrl, readUrl);
    return loomTruffleProvider;
    },
  network_id: '*'
}
```

Источник: `en/10/10.md`

## Типовые задания / вопросы на экзамене

- Объяснить назначение стандартных каталогов и файлов, которые создаёт `truffle init`, и почему их нельзя произвольно переименовывать.
- По данному `CryptoZombies.sol` написать migration file, который деплоит контракт через Truffle.
- Объяснить, что создаёт `truffle compile` и зачем эти артефакты нужны JavaScript-части и деплою.
- Дополнить `truffle.js` так, чтобы Rinkeby использовал `HDWalletProvider` и правильный `network_id`.
- Сравнить deploy в Ethereum/Rinkeby и Loom testnet: что остаётся одинаковым во workflow, а что меняется в provider/config.
- Показать последовательность команд для деплоя сначала в Rinkeby, затем в Loom testnet, и объяснить, почему mainnet оставляют напоследок.
- Убрать hardcoded private key из `truffle.js` и загрузить его из файла.
- Объяснить, почему в уроке Loom предлагается как более подходящая сеть для user-facing игры, чем Ethereum mainnet.

## Чеклист перед экзаменом

- Я могу объяснить структуру проекта Truffle и назначение каждого каталога.
- Я могу создать обёрточный контракт `CryptoZombies` и скомпилировать проект через Truffle.
- Я могу написать migration script для деплоя `CryptoZombies`.
- Я могу настроить `truffle.js` для Ethereum mainnet и Rinkeby через `HDWalletProvider`.
- Я помню, что у Rinkeby `network_id = 4`, и понимаю, зачем сначала деплоить в testnet.
- Я могу добавить Loom testnet с `LoomTruffleProvider`, `chainId`, RPC/query URL и private key.
- Я понимаю, почему секреты нельзя коммитить, и могу вынести Basechain private key в файл.
- Я могу выбрать правильную команду `truffle migrate --network ...` для нужной сети.

## Состав урока

| Порядок | EN title | EN path |
|---:|---|---|
| 1 | Deploying DApps with Truffle | `en/10/00-overview.md` |
| 2 | Introduction | `en/10/01.md` |
| 3 | Getting Started with Truffle | `en/10/02.md` |
| 4 | Compiling the Source Code | `en/10/03.md` |
| 5 | Migrations | `en/10/04.md` |
| 6 | Configuration Files | `en/10/05.md` |
| 7 | Deploying Our Smart Contract | `en/10/06.md` |
| 8 | Use Truffle with Loom! | `en/10/07.md` |
| 9 | Deploy to Loom Testnet | `en/10/08.md` |
| 10 | Deploy to Loom- continued | `en/10/09.md` |
| 11 | Deploy to Basechain | `en/10/10.md` |
| 12 | Lesson Complete! | `en/10/lessoncomplete.md` |

## Источники

- Английский контент: `en/10/`
- Русский контент: отсутствует в репозитории
- Реестр урока: `en/index.ts`
- Course metadata: `en/index.json`
- Витрина курса: `https://cryptozombies.io/ru/course/`

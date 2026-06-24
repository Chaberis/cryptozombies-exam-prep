# Урок 20: Deploying to TRON

- Трек: `Tron: Decentralize the web`
- Глав на сайте: **10**
- Элементов урока в repo: **11**
- Русская локаль: **нет**
- Sample code в `cryptozombies-lesson-code`: **нет**
- Источник кода: **В `cryptozombies-lesson-code` нет отдельного каталога `lesson-20/`. Нужный код берётся из answer blocks в `en/20/*.md`.**

## Что нужно уметь после урока

- В уроке используются два инструмента: TronBox для compile/deploy/migrations и TronWeb для работы с аккаунтами и ключами.
- Ethereum-ориентированный контракт нужно адаптировать под TRON, заменив ETH-номинированное значение на TRX-номинированное (`0.001 ether` -> `26 trx` в уроке).
- `tronbox init` создаёт обязательную структуру проекта (`contracts`, `migrations`, `test`, config files), и TronBox ожидает именно её.
- Компиляция обязательна, потому что TVM исполняет bytecode, а не исходный Solidity; TronBox создаёт JSON artifacts с ABI, bytecode и compiler metadata.
- В уроке фиксируется версия Solidity compiler `0.5.18` в `tronbox.js`.
- TRON accounts используют public/private key pairs, а deployment transactions должны быть подписаны private key соответствующего аккаунта.
- Private keys нужно передавать через environment variables вроде `PRIVATE_KEY_SHASTA` и `PRIVATE_KEY_MAINNET`, а не хранить прямо в config file.
- Migrations — это упорядоченные JavaScript deploy scripts; в уроке пишется скрипт деплоя `ZombieOwnership`, который затем запускается в Shasta и mainnet.

## Краткое summary

Урок 20 учит переносить уже написанные CryptoZombies-контракты в экосистему TRON с помощью TronBox и TronWeb. Нужно инициализировать TronBox project, адаптировать Solidity-код под TRON-единицы (`trx` вместо `ether` в конкретном месте урока), скомпилировать контракты в JSON artifacts, сгенерировать TRON account/private key, передать ключ в конфигурацию через environment variables, написать migration script и выполнить деплой сначала в Shasta testnet, а затем в TRON mainnet. Практический итог урока — готовый TRON-ready deploy flow для `ZombieOwnership`: мигрированные контракты, compiled artifacts, генерация ключа, `tronbox.js` с сетевыми конфигами и migrations для Shasta/mainnet.

## Ключевые концепции

- **TRON Virtual Machine (TVM) и почему нужна компиляция в bytecode**
- **позиционирование TRON в уроке: DPoS, выше throughput, дешевле транзакции**
- **структура проекта TronBox и generation of artifacts**
- **разница между TRX denomination и Ethereum `ether` units**
- **TRON accounts, addresses и подпись private key**
- **управление секретами через environment variables**
- **migrations и on-chain tracking через `Migrations.sol`**
- **testnet-first deploy flow: Shasta перед mainnet**

## Код, который нужно понимать

### Замена level-up fee с Ethereum units на TRX

```solidity
uint levelUpFee = 26 trx;
```

Источник: `en/20/03.md`
### Фиксация версии Solidity compiler в TronBox

```json
compilers: {
  solc: {
    version: '0.5.18'
  }
}
```

Источник: `en/20/04.md`
### Генерация TRON account/private key через TronWeb

```javascript
const tronWeb = require('tronweb')
console.log(tronWeb.utils.accounts.generateAccount())
```

Источник: `en/20/05.md`
### Использование environment variables для ключей Shasta/Mainnet

```javascript
mainnet: {
  privateKey: process.env.PRIVATE_KEY_MAINNET,
  userFeePercentage: 100,
  feeLimit: 1000 * 1e6,
  fullHost: '<https://api.trongrid.io>',
  network_id: '1'
},
shasta: {
  privateKey: process.env.PRIVATE_KEY_SHASTA,
  userFeePercentage: 50,
  feeLimit: 1000 * 1e6,
  fullHost: '<https://api.shasta.trongrid.io>',
  network_id: '2'
}
```

Источник: `en/20/06.md`
### Минимальный TronBox migration для ZombieOwnership

```javascript
var ZombieOwnershipContract = artifacts.require("./ZombieOwnershipContract.sol");

module.exports = function(deployer) {
  deployer.deploy(ZombieOwnershipContract);
}
```

Источник: `en/20/07.md`
### Команда деплоя в Shasta testnet

```shell
tronbox migrate --reset --network shasta
```

Источник: `en/20/08.md`

## Типовые задания / вопросы на экзамене

- Объяснить роли TronBox и TronWeb в deploy workflow и назвать хотя бы две возможности TronBox.
- По Solidity-контракту, перенесённому из Ethereum, определить, что нужно изменить для TRON currency units, и привести конкретную замену из урока.
- Перечислить файлы и каталоги, которые создаёт `tronbox init`, и объяснить, почему их не стоит произвольно переименовывать.
- Объяснить, что такое artifacts, куда TronBox их пишет и какие данные они содержат.
- Написать двухстрочный JavaScript snippet из урока для генерации TRON account/private key через TronWeb.
- Объяснить, зачем используется `PRIVATE_KEY_SHASTA` и почему урок рекомендует environment variables вместо хранения секретов в config.
- Написать минимальный TronBox migration script, который деплоит контракт через `deployer.deploy(...)`.
- Назвать команды деплоя в Shasta testnet и в TRON mainnet и объяснить эффект `--reset` при деплое в Shasta.

## Чеклист перед экзаменом

- Я могу назвать точное название трека `Tron: Decentralize the web` и урока `Deploying to TRON`.
- Я помню, что урок содержит 11 repo elements: overview, 9 chapter files и lesson complete.
- Я могу объяснить разделение ролей между TronBox и TronWeb.
- Я могу описать обязательную миграцию кода из Ethereum в TRON через замену `ether` на `trx` в нужном месте.
- Я понимаю, почему компиляция создаёт JSON artifacts и где они используются дальше.
- Я могу сгенерировать TRON account/private key через TronWeb и понимаю, что этим ключом подписываются транзакции.
- Я понимаю, как `PRIVATE_KEY_SHASTA` и `PRIVATE_KEY_MAINNET` передаются в TronBox безопаснее, чем hardcoded secrets.
- Я могу написать или узнать migration script и команды деплоя для Shasta и mainnet.

## Состав урока

| Порядок | EN title | EN path |
|---:|---|---|
| 1 | Deploying to TRON | `en/20/00-overview.md` |
| 2 | Introduction | `en/20/01.md` |
| 3 | Initialize Your Project | `en/20/02.md` |
| 4 | Migrating the Smart Contracts | `en/20/03.md` |
| 5 | Compiling the Source Code | `en/20/04.md` |
| 6 | Accounts and Private Keys | `en/20/05.md` |
| 7 | Generating a Private Key | `en/20/06.md` |
| 8 | Migrations | `en/20/07.md` |
| 9 | Deploy to the Shasta test network | `en/20/08.md` |
| 10 | Deploy to the TRON main network | `en/20/09.md` |
| 11 | Lesson Complete! | `en/20/lessoncomplete.md` |

## Источники

- Английский контент: `en/20/`
- Русский контент: отсутствует в репозитории
- Реестр урока: `en/index.ts`
- Course metadata: `en/index.json`
- Витрина курса: `https://cryptozombies.io/ru/course/`

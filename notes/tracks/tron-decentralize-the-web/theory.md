# Теория по треку Tron

## Что покрывает этот трек

Трек `Tron: Decentralize the web` в lesson `20` посвящён не написанию новых игровых механик, а **операционному переносу смарт-контракта в экосистему TRON**.

Это урок про:
- deploy workflow;
- различие инструментов TRON и Ethereum;
- адаптацию Solidity-кода под другую сеть;
- компиляцию и artifacts;
- управление ключами;
- testnet-first deployment.

## 1. Зачем здесь нужен TRON

В overview урок прямо подаёт TRON как сеть, которая:
- использует DPoS;
- даёт более высокий throughput;
- предлагает более дешёвые транзакции, чем Ethereum;
- использует TVM — TRON Virtual Machine.

Экзаменационно важно не спорить с маркетингом урока, а понимать, **почему этот урок вообще существует**:
он показывает, что один и тот же Solidity-проект можно адаптировать и деплоить не только в Ethereum-экосистему.

## 2. TronBox и TronWeb: кто за что отвечает

### TronBox
TronBox в уроке — это основной operational tool:
- инициализация проекта;
- компиляция контрактов;
- генерация artifacts;
- migrations;
- деплой в testnet/mainnet.

### TronWeb
TronWeb в уроке используется как utility layer:
- генерация account/private key;
- работа с TRON account model;
- низкоуровневые TRON-specific вспомогательные операции.

Ключевая мысль:
**TronBox управляет lifecycle проекта и деплоя, TronWeb помогает работать с TRON-аккаунтами и ключами.**

## 3. Структура проекта и почему её нельзя ломать

`tronbox init` создаёт структуру, аналогичную привычному blockchain tooling workflow:

- `contracts/`
- `migrations/`
- `test/`
- config files

Смысл тот же, что и в Truffle:
- контракты лежат отдельно;
- deploy scripts версионируются отдельно;
- тесты и конфигурация не смешиваются с исходниками.

Если кандидат не понимает структуру проекта, он будет путаться между source code, artifacts и deploy scripts.

## 4. TVM и зачем нужна компиляция

TRON использует **TVM (TRON Virtual Machine)**.  
Как и в EVM-подобных системах, виртуальная машина исполняет не Solidity source напрямую, а скомпилированный bytecode.

Значит компиляция нужна, чтобы получить:
- bytecode;
- ABI;
- compiler metadata;
- JSON artifact на каждый контракт.

Практический вывод:
- deploy script работает не “по `.sol` файлу как тексту”, а по compiled contract artifact / abstraction;
- без compile этапа деплой невозможен.

## 5. Artifacts

Artifacts — это мост между:
- Solidity-контрактом;
- deployment tooling;
- later JS interaction.

В уроке нужно понимать не просто слово “artifact”, а его роль:
- это результат компиляции;
- в нём есть ABI и bytecode;
- он нужен TronBox, чтобы выполнить deploy через migration script.

## 6. Адаптация Solidity-кода под TRON

Это один из самых конкретных экзаменационных пунктов lesson 20.

Урок показывает, что при переносе из Ethereum есть не только tooling differences, но и **денежные единицы сети**.

Пример из урока:

```solidity
uint levelUpFee = 26 trx;
```

Главная мысль:
- код, в котором используются Ethereum-denominated units (`ether`), нельзя переносить в другую сеть без проверки;
- при миграции нужно искать сетезависимые assumptions.

## 7. Accounts, private keys и подпись транзакций

TRON account model в уроке сводится к паре:
- public address;
- private key.

Deployment transaction должна быть подписана private key того аккаунта, от имени которого идёт деплой.

Это не просто “секрет для входа”, а operational credential для реального on-chain действия.

## 8. Управление секретами

Урок специально выносит ключи в environment variables:

- `PRIVATE_KEY_SHASTA`
- `PRIVATE_KEY_MAINNET`

Почему это важно:
- секреты не должны лежать в репозитории;
- один и тот же config file может обслуживать разные окружения;
- testnet и mainnet ключи не должны смешиваться.

Это стандартный экзаменационный вопрос:  
**почему env vars лучше, чем hardcoded private key в `tronbox.js`?**

## 9. Migrations

Migration — это воспроизводимый deploy script.

Паттерн урока:

```javascript
var ZombieOwnershipContract = artifacts.require("./ZombieOwnershipContract.sol");

module.exports = function(deployer) {
  deployer.deploy(ZombieOwnershipContract);
}
```

Нужно понимать:
- migration versioned и order-sensitive;
- деплой идёт через `deployer`;
- для каждой сети тот же migration запускается в другом network context.

## 10. Shasta testnet и mainnet

Урок придерживается правильного operational порядка:

1. сначала Shasta testnet;
2. потом TRON mainnet.

Зачем:
- проверить, что compile/config/migration/key setup корректны;
- отловить проблемы до mainnet;
- не тратить реальные ресурсы mainnet на непроверенный deploy flow.

Команда из урока для testnet:

```shell
tronbox migrate --reset --network shasta
```

`--reset` значит, что миграции запускаются заново, даже если TronBox уже считает их выполненными.

## Ключевые термины

- **TRON** — блокчейн-платформа, рассматриваемая в уроке как дешёвая и быстрая альтернатива Ethereum для деплоя.
- **DPoS** — delegated proof of stake, указанный в уроке как consensus model TRON.
- **TVM** — TRON Virtual Machine.
- **TronBox** — инструмент для init / compile / migrate / deploy.
- **TronWeb** — JS library для TRON account/key utilities.
- **Artifact** — JSON-результат компиляции контракта с ABI, bytecode и metadata.
- **Migration** — deploy script, который разворачивает контракт.
- **Shasta** — TRON testnet в уроке.
- **Mainnet** — основная сеть TRON.
- **Environment variable** — способ передать секрет без хранения его в коде.
- **TRX** — нативная единица TRON, используемая в коде вместо Ethereum-denominated unit из старой версии.

## Что нужно знать по итогам урока

После lesson 20 человек должен уметь:

- объяснить, зачем вообще переносить контракт в TRON;
- различать роли TronBox и TronWeb;
- инициализировать TronBox project и понимать его структуру;
- объяснить compile -> artifact -> migration -> deploy chain;
- указать, где при переносе из Ethereum нужно проверить network-specific assumptions;
- сгенерировать TRON account/private key;
- безопасно передать ключи через env vars;
- написать базовый migration script;
- объяснить, почему deploy начинается с Shasta, а не с mainnet.

## На чём чаще всего валятся

- путают TronBox и TronWeb;
- забывают, что перенос сети требует проверить currency units в Solidity-коде;
- не понимают, зачем нужны artifacts;
- держат private key прямо в config/repo;
- не различают testnet и mainnet deployment flow;
- не могут написать даже минимальный migration script;
- не могут объяснить, что делает `--reset` в testnet deploy command.

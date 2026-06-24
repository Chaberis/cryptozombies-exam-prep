# Теория по треку Advanced Solidity

## Что покрывает этот трек

Трек `Advanced Solidity: Get In-depth Knowledge` — это не продолжение синтаксиса Solidity, а переход к **инженерной эксплуатации смарт-контрактов**:

1. как собрать Solidity-проект в нормальный Truffle workflow;
2. как деплоить контракт в разные сети;
3. как тестировать поведение контракта, а не только компиляцию;
4. как строить oracle flow между on-chain контрактами и off-chain кодом;
5. как убрать single point of failure и перейти к более децентрализованной oracle-модели.

Важно: уроки `12` и `13` в этом курсе отсутствуют. Практический маршрут трека — `10 -> 11 -> 14 -> 15 -> 16`.

## 1. Truffle как операционная оболочка для Solidity-проекта

### Что даёт Truffle

Truffle — это каркас вокруг Solidity-кода:

- стандартизирует структуру проекта;
- компилирует контракты;
- хранит build artifacts;
- запускает migrations;
- даёт тестовую среду;
- позволяет деплоить в разные сети через один и тот же workflow.

### Базовая структура проекта

Нужно понимать назначение каждого элемента:

- `contracts/` — исходники контрактов;
- `migrations/` — скрипты развёртывания;
- `test/` — JS-тесты;
- `truffle.js` / `truffle-config.js` — конфигурация сетей и провайдеров.

### Артефакты компиляции

После `truffle compile` появляются build artifacts, где лежат:

- ABI;
- bytecode;
- metadata;
- адреса deployed instances по network id.

Это ключевая связка между Solidity и JavaScript. Без artifacts JS-код не знает, как вызывать контракт.

### Migrations

Migration — это **явный, воспроизводимый deploy script**.

Ключевой паттерн:

```javascript
var CryptoZombies = artifacts.require("./CryptoZombies.sol");
module.exports = function(deployer) {
  deployer.deploy(CryptoZombies);
};
```

Нужно понимать:
- почему deploy делается скриптом, а не вручную;
- почему порядок migrations важен;
- что деплой в новую сеть требует правильного `network` конфигурационного блока.

## 2. Сети, провайдеры и деплой

### Ethereum / Rinkeby / Infura

Для Ethereum-совместимых сетей Truffle обычно использует provider, который:
- знает RPC endpoint;
- умеет подписывать транзакции;
- работает с mnemonic/private key.

Классический паттерн курса:

```javascript
const HDWalletProvider = require("truffle-hdwallet-provider");
return new HDWalletProvider(mnemonic, "https://rinkeby.infura.io/v3/YOUR_TOKEN");
```

Что нужно знать:
- `network_id: 4` — это Rinkeby;
- сначала деплой обычно делается в testnet;
- mainnet конфигурация технически похожа, но operational risk существенно выше.

### Loom / Basechain

Курс показывает, что один и тот же контракт можно деплоить не только в Ethereum, но и в сети семейства Loom, где:
- транзакции дешевле или бесплатны для пользователя;
- UX может быть быстрее;
- меняется provider и network configuration, но общий Truffle workflow остаётся тем же.

### Работа с секретами

Критически важно:
- не хардкодить mnemonic/private key в коммит;
- читать ключи из файла или env;
- разделять testnet/mainnet/basechain доступы.

## 3. Тестирование смарт-контрактов

### Что именно тестируется

Экзаменационно важна мысль: тестируют **поведение**, а не факт компиляции.

Типы проверок:
- happy path;
- revert path;
- события;
- ownership transfer;
- time-based logic;
- permission checks.

### Truffle test skeleton

```javascript
const CryptoZombies = artifacts.require("CryptoZombies");
contract("CryptoZombies", (accounts) => {
  it("should be able to create a new zombie", () => {

  })
})
```

Нужно понимать:
- `artifacts.require(...)` загружает contract abstraction;
- `contract(...)` — оболочка тестового набора;
- `accounts` — тестовые аккаунты среды;
- `it(...)` — отдельный сценарий.

### Fresh deployment per test

Нормальный паттерн:

```javascript
beforeEach(async () => {
  contractInstance = await CryptoZombies.new();
});
```

Зачем:
- тесты не должны делить мутировавшее состояние;
- flaky tests почти всегда начинаются с общего состояния между кейсами.

### Проверка событий и результатов транзакции

```javascript
const result = await contractInstance.createRandomZombie(name, { from: alice });
expect(result.receipt.status).to.equal(true);
expect(result.logs[0].args.name).to.equal(name);
```

Нужно уметь:
- смотреть `receipt.status`;
- извлекать данные из `logs`;
- использовать `zombieId` из event для последующих шагов теста.

### Проверка revert path

Курс использует helper в духе:

```javascript
async function shouldThrow(promise) {
  try {
    await promise;
    assert(true);
  } catch (err) {
    return;
  }
  assert(false, "The contract did not throw.");
}
```

Смысл:
- не считать любое падение успехом;
- намеренно ловить revert в тех сценариях, где он обязателен.

### Тестирование ERC721 flow

Нужно уверенно различать:
- прямой transfer владельцем;
- двухшаговый flow через `approve` + `transferFrom`.

### Time travel в Ganache

Для cooldown-based логики курс использует:

- `evm_increaseTime`
- затем `evm_mine`

Это важно: увеличение времени без майнинга нового блока обычно не активирует нужный `block.timestamp`.

### Chai vs assert

Курс переводит проверки на Chai `expect(...)`, потому что:
- они читаются яснее;
- лучше выражают intent теста;
- проще поддерживаются при росте test suite.

## 4. Oracle architecture: on-chain часть

### Почему oracle не может вернуть цену обычным return

Контракт не может сам сходить в HTTP API. Значит нужен pattern:

1. caller contract запрашивает данные;
2. oracle contract создаёт request id;
3. событие уведомляет off-chain процесс;
4. off-chain сервис получает данные;
5. oracle callback'ом возвращает результат в caller contract.

Это **асинхронная request/callback модель**.

### Интерфейсы и cross-contract calls

Caller хранит адрес oracle и вызывает его через interface.

Нужно понимать два шага:
- сохранить адрес;
- обернуть адрес в contract interface instance.

### Pending requests

Обе стороны хранят mapping'и:
- caller помнит, какие request id он реально открывал;
- oracle помнит, какие request id ещё не завершены.

Иначе можно:
- подделать callback;
- повторно исполнить один и тот же request;
- обновить состояние несвязанным id.

### onlyOwner и onlyOracle

В oracle flow недостаточно просто сделать функцию `public`.

Нужны ограничения:
- только owner может менять критическую конфигурацию;
- только oracle/авторизованный адрес может завершать callback.

### Events как мост между chain и off-chain

Событие — это не “лог для красоты”, а технический мост:
- JS oracle слушает `GetLatestEthPriceEvent`;
- вытаскивает `callerAddress` и `id`;
- ставит request в очередь на обработку.

## 5. Oracle architecture: off-chain JavaScript часть

### Instantiation по ABI и network id

JS oracle поднимает deployed contract так:

```javascript
const networkId = await web3js.eth.net.getId()
return new web3js.eth.Contract(OracleJSON.abi, OracleJSON.networks[networkId].address)
```

Это важный паттерн:
- ABI описывает методы;
- `networks[networkId].address` даёт нужный deployed address;
- один и тот же artifact может использоваться в разных сетях.

### Очередь и batching

Курс не обрабатывает каждое событие inline. Он:
- складывает запросы в `pendingRequests`;
- обрабатывает их чанками;
- запускает loop по таймеру.

Зачем:
- сгладить burst событий;
- контролировать throughput;
- централизованно обрабатывать retry/failure logic.

### Retry и fallback

Off-chain API ненадёжны. Поэтому есть:
- `MAX_RETRIES`;
- `try/catch`;
- fallback-ветка, которая в крайнем случае пишет `0`.

На экзамене важно не просто помнить цикл, а объяснить **зачем** он нужен: oracle без retry легко превращается в permanently stale service.

### Числа в Ethereum и JavaScript

Цена из API приходит как decimal string. В EVM обычно нужен integer.

Курс делает:
- удаление точки из строки;
- домножение через `BN`;
- отправку строки, а не JS number.

Причина:
- JS number небезопасен для больших целых;
- Solidity ожидает целые значения;
- scale должен быть согласован между off-chain и on-chain частью.

## 6. Переход к более децентрализованному oracle

Lesson 16 убирает single-owner update model и вводит multi-oracle aggregation.

### Roles

Вместо одного `owner` в стиле `Ownable` вводятся роли:
- `owners`;
- `oracles`.

Это уже не просто “кто админ”, а **явное разделение прав**.

### Управление oracle-наборами

Нужно понимать:
- добавление oracle;
- удаление oracle;
- запрет на удаление последнего oracle;
- счётчик `numOracles`.

### Хранение ответов

Вместо одного ответа на request хранится массив структур:

```solidity
struct Response {
  address oracleAddress;
  address callerAddress;
  uint256 ethPrice;
}
mapping (uint256=>Response[]) public requestIdToResponse;
```

Это даёт:
- несколько независимых ответов;
- последующую агрегацию;
- возможность threshold-based completion.

### Threshold

Запрос завершается не после первого ответа, а после `THRESHOLD` ответов.

Это баланс между:
- latency;
- fault tolerance;
- децентрализацией.

### Average aggregation

После достижения порога курс считает среднее арифметическое по всем ответам.

Ключевая мысль:
- итоговая цена — не “доверяем последнему oracle”, а “агрегируем несколько источников”.

### SafeMath

Даже если числа выглядят безопасно, курс добавляет `SafeMath`, чтобы:
- не полагаться на устные допущения;
- стандартизировать arithmetic safety;
- избегать overflow/underflow в aggregation path.

### Cleanup

После завершения request нужно удалить:
- `pendingRequests[_id]`;
- `requestIdToResponse[_id]`.

Иначе останется мусорное состояние и появится риск повторной обработки.

## Ключевые термины

- **Truffle** — набор инструментов для Solidity-проектов: compile, migrate, test, deploy.
- **Artifact** — JSON-описание с ABI, bytecode и deployment metadata.
- **Migration** — скрипт развёртывания контракта.
- **Provider** — объект, через который JS-код общается с сетью и подписывает транзакции.
- **Testnet** — безопасная сеть для прогонов перед mainnet.
- **Revert path** — сценарий, где вызов обязан завершиться ошибкой.
- **Event** — on-chain сигнал для UI или off-chain процесса.
- **Oracle** — мост между blockchain и внешними данными/вычислениями.
- **Callback** — функция, которую oracle вызывает после завершения внешней обработки.
- **Role-based access control** — модель прав, где адресам выдаются конкретные роли.
- **Threshold** — минимальное число ответов для финализации запроса.
- **Aggregation** — вычисление итогового значения из нескольких ответов.
- **BN / Big Number** — безопасная работа с большими целыми вне пределов JS number.

## Что нужно знать по итогам всего трека

После этого трека человек должен уметь:

- собрать Solidity-проект в Truffle;
- объяснить compile -> artifacts -> migrate -> deploy chain;
- настроить deploy в testnet и альтернативную EVM-сеть;
- написать базовый JS test suite для контракта;
- тестировать события, revert paths, ERC721 transfers и cooldown logic;
- объяснить request/callback oracle architecture;
- реализовать off-chain listener, очередь и callback writer;
- объяснить, как сделать oracle менее централизованным через multi-oracle aggregation.

## На чём чаще всего валятся

- путают compile artifacts и deployment scripts;
- не понимают, зачем нужен fresh deploy в `beforeEach`;
- проверяют только happy path и не тестируют revert;
- забывают, что `evm_increaseTime` обычно должен сопровождаться `evm_mine`;
- трактуют event как “лог”, а не как часть архитектуры;
- не понимают, почему callback нельзя оставить публичным без ограничений;
- отправляют decimal/JS number вместо integer string / BN-конвертированного значения;
- забывают cleanup request state после исполнения;
- называют систему “децентрализованной”, хотя обновление всё ещё делает один owner;
- не могут объяснить разницу между single-oracle и threshold aggregation.

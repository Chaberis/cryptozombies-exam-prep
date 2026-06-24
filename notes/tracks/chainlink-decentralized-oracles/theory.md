# Теория по треку Chainlink: Decentralized Oracles

Этот файл — общий теоретический конспект по lesson 19 CryptoZombies. Его задача — дать цельную модель того, как смарт-контракты получают внешние данные и randomness через Chainlink.

## 1. Главная проблема: изоляция блокчейна

Смарт-контракты по дизайну:
- детерминированы;
- изолированы от внешнего мира;
- не умеют самостоятельно делать HTTP-запросы;
- не могут доверять произвольному внешнему процессу без ущерба для trust model.

Следствие: если контракту нужны внешние данные, нужен **oracle layer**.

## 2. Oracle

**Oracle** — это механизм доставки внешних данных или вычислений в смарт-контракт.

Примеры данных:
- цены активов;
- погодные данные;
- спортивные результаты;
- случайные числа;
- результаты off-chain computation.

Проблема:
- если oracle централизован, то именно он становится single point of failure и single point of trust.

## 3. DON — Decentralized Oracle Network

Chainlink решает проблему через **децентрализованную oracle-сеть**.

Идея:
- несколько нод собирают данные;
- данные агрегируются;
- результат публикуется on-chain;
- пользовательский контракт читает уже опубликованный результат.

Это устраняет часть доверия к одному участнику и лучше соответствует децентрализованной модели.

## 4. Hybrid Smart Contracts

Контракт, который использует:
- on-chain логику,
- но при этом зависит от off-chain data / off-chain computation,

называется **hybrid smart contract**.

Lesson 19 фактически вводит именно эту модель.

## 5. Data Feeds

**Data Feed** — это уже существующий on-chain reference contract, который Chainlink регулярно обновляет.

Преимущества:
- не нужно запускать свой oracle flow;
- не нужно самостоятельно собирать и агрегировать данные;
- можно просто читать опубликованные значения через интерфейс.

В lesson 19 используется price feed ETH/USD.

## 6. AggregatorV3Interface

Для работы с feed нужен интерфейс:
```solidity
import "@chainlink/contracts/src/v0.6/interfaces/AggregatorV3Interface.sol";
```

Зачем нужен интерфейс:
- контракту не нужен весь исходник feed-контракта;
- нужен только ABI-совместимый набор функций.

Ключевые методы:
- `latestRoundData()`
- `decimals()`
- также в интерфейсе есть `description()`, `version()`, `getRoundData(...)`.

## 7. Адрес feed-контракта

Interface сам по себе не знает, с каким контрактом работать. Его нужно привязать к адресу:

```solidity
priceFeed = AggregatorV3Interface(0x8A753747A1Fa494EC906cE90E9f37563A8AF630e);
```

Важно:
- адрес feed-контракта зависит от сети;
- Rinkeby, Mainnet, Polygon и другие сети имеют разные адреса одного и того же feed-а.

Это типичный экзаменационный вопрос.

## 8. Tuple и `latestRoundData()`

`latestRoundData()` возвращает несколько значений сразу. Это tuple.

Пример сигнатуры:
```solidity
function latestRoundData()
  external
  view
  returns (
    uint80 roundId,
    int256 answer,
    uint256 startedAt,
    uint256 updatedAt,
    uint80 answeredInRound
  );
```

Если нужен только `answer`, можно написать:
```solidity
(,int price,,,) = priceFeed.latestRoundData();
```

Что нужно понимать:
- tuple — множественный return;
- Solidity позволяет игнорировать ненужные значения;
- это делает код короче и понятнее.

## 9. Decimals в price feeds

Price feeds обычно возвращают число в integer-форме с отдельным scale.

Пример:
- raw value: `310523971888`
- real-world interpretation: `3105.23971888`
- для этого нужен `decimals()`.

Значит:
- `getLatestPrice()` без `getDecimals()` часто недостаточен;
- UI или downstream logic должны знать scale.

## 10. VRF: почему нужна внешняя randomness

On-chain randomness на базе:
- `keccak256(...)`,
- `block.timestamp`,
- `block.difficulty`,
- `msg.sender`

не является truly secure randomness.

Причина:
- всё on-chain в целом предсказуемо или манипулируемо;
- майнеры/валидаторы или сами пользователи могут влиять на часть входов;
- детерминированность блокчейна делает такие схемы уязвимыми.

Поэтому нужен внешний источник randomness с proof.

## 11. Chainlink VRF

**Chainlink VRF** = Verifiable Random Function.

Это oracle-service, который:
- генерирует randomness off-chain;
- возвращает её on-chain;
- прикладывает криптографическое доказательство корректности;
- даёт on-chain verifier / coordinator проверить proof.

Следствие:
- контракт получает не просто «случайное число»;
- он получает число, randomness которого может быть проверена on-chain.

## 12. Request / Callback model

В lesson 19 VRF вводит базовую request model:

1. Контракт делает request.
2. Chainlink node видит событие / request off-chain.
3. Node обрабатывает запрос.
4. Node / coordinator возвращает ответ во второй транзакции.
5. Контракт получает callback.

Это важно:
- request и response — разные транзакции;
- значит ответ нельзя получить мгновенно внутри того же execution path;
- следовательно проектирование логики должно учитывать асинхронность.

## 13. `VRFConsumerBase`

Чтобы не реализовывать инфраструктуру вручную, контракт наследует:

```solidity
import "@chainlink/contracts/src/v0.6/VRFConsumerBase.sol";

contract ZombieFactory is VRFConsumerBase {
```

Это даёт доступ к:
- встроенному `requestRandomness(...)`;
- expected callback-модели;
- проверенной обвязке вокруг coordinator.

## 14. Constructor in a constructor

Так как базовый контракт требует свои параметры, конструктор потомка вызывает конструктор родителя:

```solidity
constructor() VRFConsumerBase(
    COORDINATOR,
    LINK_TOKEN
) public {

}
```

Это важный синтаксический паттерн Solidity inheritance.

## 15. Основные VRF-переменные

### `keyHash`
Идентификатор ключа / job-параметров VRF-ноды.

### `fee`
Стоимость oracle request в LINK.

### `randomResult`
Хранилище последнего полученного random number.

### `LINK`
Токен, которым оплачивается oracle-service.

### `VRF Coordinator`
Контракт, который проверяет proof и вызывает callback у consumer-контракта.

## 16. `requestRandomness(...)`

Функция:
```solidity
return requestRandomness(keyHash, fee);
```

Что делает концептуально:
- проверяет, что у consumer-контракта достаточно LINK;
- инициирует oracle request;
- создаёт request id;
- запускает off-chain/oracle flow.

## 17. `fulfillRandomness(...)`

Callback-функция:
```solidity
function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
    randomResult = randomness;
}
```

Что важно:
- вызывается не пользователем;
- вызывается coordinator-потоком;
- поэтому она `internal override`;
- здесь нужно хранить или применять полученный результат.

На экзамене важно понимать, что callback — это вторая транзакция, а не продолжение первой.

## 18. Разница между Data Feeds и VRF

### Data Feeds
- данные уже подготовлены и опубликованы;
- пользователь просто читает контракт;
- своей request-транзакции на каждый price-read не нужно;
- удобны для цен, индексов, reference data.

### VRF
- пользователь/контракт делает явный request;
- нужен LINK fee;
- ответ приходит во второй транзакции;
- подходит для randomness и request-based computation.

## 19. Почему data feeds дешевле для пользователя

В lesson 19 есть важная идея:
- someone else already pays for maintaining the data feed;
- многие протоколы совместно используют один feed;
- индивидуальный потребитель просто читает его.

То есть:
- Data Feed = shared public good;
- VRF = per-request oracle service.

## 20. Ключевые термины

### Oracle
Механизм доставки внешних данных/вычислений в smart contract.

### DON
Decentralized Oracle Network.

### Data Feed
On-chain reference contract с агрегированными данными.

### AggregatorV3Interface
Интерфейс для чтения Chainlink data feeds.

### Tuple
Множественный return функции Solidity.

### Decimals
Scale integer-значения, без которого число интерпретируется неверно.

### VRF
Verifiable Random Function.

### VRFConsumerBase
Базовый контракт потребителя VRF.

### VRF Coordinator
On-chain verifier/callback coordinator для VRF.

### LINK
Токен оплаты Chainlink oracle requests.

### keyHash
Идентификатор ключа/параметров VRF job.

### `requestRandomness`
Запуск oracle request.

### `fulfillRandomness`
Callback с итоговым random number.

## 21. Что человек должен знать после lesson 19

Он должен уметь:
- объяснить, зачем нужны оракулы;
- различать централизованный oracle и DON;
- написать контракт для чтения data feed через `AggregatorV3Interface`;
- корректно разобрать tuple от `latestRoundData()`;
- получить decimals и объяснить их роль;
- написать consumer-контракт для VRF через `VRFConsumerBase`;
- объяснить request/callback модель и необходимость LINK fee;
- объяснить, почему VRF лучше старой псевдослучайности на `keccak256`.

## 22. Типовые провалы на экзамене

- не понимают, почему контракт не может просто дернуть HTTP API;
- забывают, что адрес feed зависит от сети;
- путают raw price и scaled price;
- не умеют читать tuple, возвращаемый `latestRoundData()`;
- думают, что `fulfillRandomness` вызывается прямо в той же транзакции, что и request;
- не понимают, зачем нужен LINK;
- забывают, что VRF — асинхронная модель;
- не могут объяснить разницу между Data Feed и VRF.

Этот файл нужно читать вместе с `lesson-19.md`: theory даёт фундамент, lesson — экзамен-ориентированную практику.

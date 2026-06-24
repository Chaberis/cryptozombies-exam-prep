# Урок 19: Data Feeds and Computation

- Трек: `Chainlink: Decentralized Oracles`
- Глав на сайте: **12**
- Элементов урока в repo: **13**
- Русская локаль: **нет**
- Sample code в `cryptozombies-lesson-code`: **нет**
- Источник кода: **answer blocks из `en/19/*.md`**

## Что нужно уметь после урока

- объяснить, зачем блокчейну нужны оракулы и почему централизованный oracle ломает trust model dApp;
- подключить Chainlink Data Feed через `AggregatorV3Interface`;
- читать актуальную цену и decimals из feed-контракта;
- разбирать tuple-ответы Solidity-функций;
- объяснить модель Chainlink VRF и двухтранзакционный request/callback flow;
- перенести старую псевдослучайность на VRF через `VRFConsumerBase`.

## Краткое summary

Урок 19 состоит из двух логических частей.

Первая часть — **Chainlink Data Feeds**. Здесь курс показывает, как смарт-контракт может читать внешние данные не напрямую из API, а из уже опубликованного on-chain feed-контракта, который поддерживается Chainlink DON. Практический результат этой части — контракт `PriceConsumerV3`, умеющий читать ETH/USD цену и количество decimals.

Вторая часть — **Chainlink VRF**. Здесь курс показывает, как заменить детерминированную псевдослучайность (`keccak256(...)`) на внешний, криптографически проверяемый источник randomness. Практический результат — `ZombieFactory`, который наследует `VRFConsumerBase`, отправляет запрос на randomness и принимает callback через `fulfillRandomness`.

С инженерной точки зрения это урок не столько про синтаксис Solidity, сколько про **интеграцию смарт-контракта с внешней инфраструктурой**.

## Ключевые концепции

- **Oracle** — мост между on-chain контрактом и внешними данными/вычислениями.
- **DON (Decentralized Oracle Network)** — набор оракульных нод, которые децентрализованно собирают и публикуют данные.
- **Hybrid smart contract** — контракт, который комбинирует on-chain логику и off-chain данные/вычисления.
- **Data Feed** — уже подготовленный Chainlink feed-контракт с агрегированными данными, например ETH/USD.
- **AggregatorV3Interface** — интерфейс для чтения data feed.
- **Tuple** — множественный return Solidity-функции, из которого можно выбрать только нужные значения.
- **Decimals** — scale числа, без которого price feed интерпретируется неверно.
- **VRF (Verifiable Random Function)** — источник randomness с криптографическим on-chain proof.
- **VRFConsumerBase** — базовый контракт Chainlink для VRF-интеграции.
- **LINK** — токен оплаты oracle-услуги.
- **keyHash** — идентификатор конкретной VRF job / oracle key.
- **fulfillRandomness** — callback, в который Chainlink Coordinator возвращает результат.

## Архитектура и итог урока

- Часть 1 даёт отдельный контракт `PriceConsumerV3`, который читает внешний feed по известному адресу.
- Часть 2 даёт отдельный `ZombieFactory`, уже не опирающийся на внутреннюю псевдослучайность.
- В VRF-модели запрос и ответ происходят в **двух разных транзакциях**.
- Для VRF контракт должен быть профинансирован `LINK` и знать адреса `VRF Coordinator` и `LINK token` для конкретной сети.
- Lesson 19 не строит полноценный production-ready oracle system, а учит базовой модели интеграции.

## Код, который нужно понимать

### Финальный `PriceConsumerV3` для data feeds

```solidity
pragma solidity ^0.6.7;

import "@chainlink/contracts/src/v0.6/interfaces/AggregatorV3Interface.sol";

contract PriceConsumerV3 {
  AggregatorV3Interface public priceFeed;

  constructor() public {
    priceFeed = AggregatorV3Interface(0x8A753747A1Fa494EC906cE90E9f37563A8AF630e);
  }

  function getLatestPrice() public view returns (int) {
    (,int price,,,) = priceFeed.latestRoundData();
    return price;
  }

  function getDecimals() public view returns (uint8) {
    uint8 decimals = priceFeed.decimals();
    return decimals;
  }
}
```

### Tuple-разбор результата `latestRoundData()`

```solidity
function getLatestPrice() public view returns (int) {
  (,int price,,,) = priceFeed.latestRoundData();
  return price;
}
```

### Вызов конструктора `VRFConsumerBase` из конструктора потомка

```solidity
constructor() VRFConsumerBase(
    0x6168499c0cFfCaCD319c818142124B7A15E857ab, // VRF Coordinator
    0x01BE23585060835E02B77ef475b0Cc51aA1e0709  // LINK Token
) public{

}
```

### Запрос randomness и callback VRF

```solidity
function getRandomNumber() public returns (bytes32 requestId) {
    return requestRandomness(keyHash, fee);
}

function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
    randomResult = randomness;
}
```

### Финальный `ZombieFactory` c Chainlink VRF

```solidity
pragma solidity ^0.6.6;

import "@chainlink/contracts/src/v0.6/VRFConsumerBase.sol";

contract ZombieFactory is VRFConsumerBase {

    uint dnaDigits = 16;
    uint dnaModulus = 10 ** dnaDigits;

    bytes32 public keyHash;
    uint256 public fee;
    uint256 public randomResult;

    struct Zombie {
        string name;
        uint dna;
    }

    Zombie[] public zombies;

    constructor() VRFConsumerBase(
        0x6168499c0cFfCaCD319c818142124B7A15E857ab, // VRF Coordinator
        0x01BE23585060835E02B77ef475b0Cc51aA1e0709  // LINK Token
    ) public{
        keyHash = 0x2ed0feb3e7fd2022120aa84fab1945545a9f2ffc9076fd6156fa96eaff4c1311;
        fee = 100000000000000000;

    }

    function _createZombie(string memory _name, uint _dna) private {
        zombies.push(Zombie(_name, _dna));
    }

    function getRandomNumber() public returns (bytes32 requestId) {
        return requestRandomness(keyHash, fee);
    }

    function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
        randomResult = randomness;
    }
}
```

## Типовые задания / вопросы на экзамене

- Что такое oracle и почему централизованный oracle противоречит целям децентрализации?
- Чем Chainlink Data Feed отличается от прямого HTTP-запроса из контракта?
- Как написать `PriceConsumerV3`, который возвращает только `price` из `latestRoundData()`?
- Почему нужно отдельно запрашивать `decimals()`?
- Что означает запись `contract ZombieFactory is VRFConsumerBase`?
- Что делают `requestRandomness(...)` и `fulfillRandomness(...)`?
- Почему VRF-модель — это минимум две транзакции?
- Зачем контракту нужны `LINK`, `keyHash` и `fee`?

## Чеклист перед экзаменом

- Я могу объяснить разницу между Data Feed и VRF.
- Я понимаю, почему tuple-разбор в `latestRoundData()` записан как ` (,int price,,,) `.
- Я могу без подсказок написать `AggregatorV3Interface public priceFeed;`.
- Я понимаю, почему адрес feed зависит от сети.
- Я могу без подсказок написать `getRandomNumber()` и `fulfillRandomness(...)`.
- Я помню, что old pseudo-random generator в конце урока удаляется полностью.

## Состав урока

| Порядок | EN title | EN path |
|---:|---|---|
| 1 | Data Feeds and Computation | `en/19/00-overview.md` |
| 2 | Chainlink Data Feeds Introduction | `en/19/01.md` |
| 3 | Importing from NPM and Github | `en/19/02.md` |
| 4 | AggregatorV3Interface | `en/19/03.md` |
| 5 | Working with Tuples | `en/19/04.md` |
| 6 | Chainlink Data Feeds Decimals | `en/19/05.md` |
| 7 | Chainlink Data Feeds References | `en/19/06.md` |
| 8 | Chainlink VRF Introduction | `en/19/07.md` |
| 9 | Constructor in a constructor | `en/19/08.md` |
| 10 | Define our Chainlink VRF variables | `en/19/09.md` |
| 11 | The requestRandomness and fulfillRandomness functions | `en/19/10.md` |
| 12 | Delete the old pseudo-random number generator | `en/19/11.md` |
| 13 | Lesson Complete! | `en/19/lessoncomplete.md` |

## Источники

- Английский контент: `en/19/`
- Русский контент: отсутствует в репозитории
- Реестр урока: `en/index.ts`
- Course metadata: `en/index.json`
- Витрина курса: `https://cryptozombies.io/chainlink?lessonId=19`

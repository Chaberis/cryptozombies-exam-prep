# Урок 14: How to Build an Oracle

- Трек: `Advanced Solidity: Get In-depth Knowledge`
- Глав на сайте: **12**
- Элементов урока в repo: **13**
- Русская локаль: **нет**
- Sample code в `cryptozombies-lesson-code`: **нет**
- Источник кода: **В `../cryptozombies-lesson-code/` нет каталога `lesson-14/`. Нужный код берётся из answer blocks в `en/14/*.md`.**

## Что нужно уметь после урока

- Как вызывать другой контракт через interface и сохранённый адрес контракта.
- Почему административные функции нужно защищать через `Ownable` и `onlyOwner`.
- Как устроен асинхронный oracle flow: `request -> event -> off-chain fetch -> callback`.
- Зачем mappings нужны и у caller, и у oracle, чтобы отслеживать валидные request id.
- Почему callback должен быть ограничен только доверенным oracle-адресом.
- Как генерируется request id через `keccak256(abi.encodePacked(now, msg.sender, randNonce)) % modulus`.
- Почему events здесь служат мостом между on-chain и off-chain частями системы.

## Краткое summary

Урок 14 покрывает on-chain половину централизованного ETH price oracle. В нём строятся два Solidity-контракта: caller contract, который запрашивает цену и принимает callback, и oracle contract, который создаёт request id, хранит pending requests, эмитит события для off-chain обработчика и позже возвращает цену обратно в caller.

## Ключевые концепции

- **взаимодействие контрактов через interfaces**
- **авторизация через `Ownable` / `onlyOwner`**
- **кастомный `onlyOracle` modifier**
- **mappings для pending requests**
- **event-driven асинхронный oracle flow**
- **генерация и корреляция request id**
- **callback-обновление состояния caller контракта**

## Код, который нужно понимать

### Защищённая смена адреса oracle

```solidity
function setOracleInstanceAddress (address _oracleInstanceAddress) public onlyOwner {
  oracleAddress = _oracleInstanceAddress;
  oracleInstance = EthPriceOracleInterface(oracleAddress);
  emit newOracleAddressEvent(oracleAddress);
}
```

Источник: `en/14/04.md`
### Запрос новой ETH price

```solidity
function updateEthPrice() public {
  uint256 id = oracleInstance.getLatestEthPrice();
  myRequests[id] = true;
  emit ReceivedNewRequestIdEvent(id);
}
```

Источник: `en/14/05.md`
### Callback с проверкой request id

```solidity
function callback(uint256 _ethPrice, uint256 _id) public {
  require(myRequests[_id], "This request is not in my pending list.");
  ethPrice = _ethPrice;
  delete myRequests[_id];
  emit PriceUpdatedEvent(_ethPrice, _id);
}
```

Источник: `en/14/06.md`
### Создание request id и событие для off-chain oracle

```solidity
function getLatestEthPrice() public returns (uint256) {
  randNonce++;
  uint id = uint(keccak256(abi.encodePacked(now, msg.sender, randNonce))) % modulus;
  pendingRequests[id] = true;
  emit GetLatestEthPriceEvent(msg.sender, id);
  return id;
}
```

Источник: `en/14/09.md`
### Oracle возвращает цену вызывающему контракту

```solidity
function setLatestEthPrice(uint256 _ethPrice, address _callerAddress, uint256 _id) public onlyOwner {
  require(pendingRequests[_id], "This request is not in my pending list.");
  delete pendingRequests[_id];
  CallerContractInterface callerContractInstance;
  callerContractInstance = CallerContractInterface(_callerAddress);
  callerContractInstance.callback(_ethPrice, _id);
  emit SetLatestEthPriceEvent(_ethPrice, _callerAddress);
}
```

Источник: `en/14/11.md`

## Типовые задания / вопросы на экзамене

- Объяснить, почему caller contract не может получить ETH price как обычный return value из `getLatestEthPrice` и какой паттерн заменяет синхронный return.
- Написать безопасную версию `setOracleInstanceAddress`, где только owner может менять oracle, а фронтенд получает событие об изменении адреса.
- Объяснить, зачем caller хранит `myRequests`, а oracle отдельно хранит `pendingRequests`.
- Реализовать `callback(uint256 _ethPrice, uint256 _id)`, чтобы невалидные id отклонялись, а валидные обновляли состояние только один раз.
- Объяснить, какая атака возможна, если callback оставить `public` без ограничения по отправителю.
- Показать, как генерируется oracle request id и какие значения смешиваются в hash.
- Показать, как oracle вызывает callback произвольного caller контракта через interface и `_callerAddress`.

## Чеклист перед экзаменом

- Я могу объяснить полный request/callback lifecycle между двумя контрактами.
- Я могу написать interface и создать экземпляр другого контракта по адресу.
- Я знаю, какие функции должны быть `onlyOwner` и почему.
- Я понимаю, почему callback должен проверять и `msg.sender`, и request id.
- Я могу объяснить назначение каждого события в oracle flow.
- Я понимаю, почему mappings очищаются после успешного fulfill.
- Я могу восстановить `getLatestEthPrice` и `setLatestEthPrice` по памяти.

## Состав урока

| Порядок | EN title | EN path |
|---:|---|---|
| 1 | How to Build an Oracle | `en/14/00-overview.md` |
| 2 | Settings Things Up | `en/14/01.md` |
| 3 | Calling Other Contracts | `en/14/02.md` |
| 4 | Calling Other Contracts- Cont'd | `en/14/03.md` |
| 5 | Function Modifiers | `en/14/04.md` |
| 6 | Using a Mapping to Keep Track of Requests | `en/14/05.md` |
| 7 | The Callback Function | `en/14/06.md` |
| 8 | The onlyOracle Modifier | `en/14/07.md` |
| 9 | The getLatestEthPrice Function | `en/14/08.md` |
| 10 | The getLatestEthPrice Function - Cont'd | `en/14/09.md` |
| 11 | The setLatestEthPrice Function | `en/14/10.md` |
| 12 | The Oracle Contract | `en/14/11.md` |
| 13 | Lesson Complete! | `en/14/lessoncomplete.md` |

## Источники

- Английский контент: `en/14/`
- Русский контент: отсутствует в репозитории
- Реестр урока: `en/index.ts`
- Course metadata: `en/index.json`
- Витрина курса: `https://cryptozombies.io/ru/course/`

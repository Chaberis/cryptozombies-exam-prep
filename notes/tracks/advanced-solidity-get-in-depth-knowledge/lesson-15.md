# Урок 15: How to Build an Oracle - Part 2

- Трек: `Advanced Solidity: Get In-depth Knowledge`
- Глав на сайте: **15**
- Элементов урока в repo: **16**
- Русская локаль: **нет**
- Sample code в `cryptozombies-lesson-code`: **нет**
- Источник кода: **В `../cryptozombies-lesson-code/` нет каталога `lesson-15/`. Нужный код берётся из answer blocks в `en/15/*.md`.**

## Что нужно уметь после урока

- Как создавать экземпляр deployed контракта из ABI и адреса в build artifact.
- Как слушать Solidity events из JavaScript и ставить request в очередь.
- Как работает batching очереди через `CHUNK_SIZE` и polling loop.
- Зачем нужен retry logic через `MAX_RETRIES`, `try/catch` и fallback значение.
- Как получать ETH price по HTTP через `axios`.
- Как переводить decimal API output в integer значение для EVM через BN.js.
- Почему весь oracle и client flow пишется на `async/await`.
- Как deployment и runtime разбиваются на отдельные Truffle configs и migrations для oracle и caller.

## Краткое summary

Урок 15 реализует off-chain половину oracle на JavaScript и минимальный клиент, который им пользуется. Нужно поднять контракты по build artifacts, подписаться на `GetLatestEthPriceEvent`, поставить запросы в очередь, обрабатывать их батчами, получать цену ETH из Binance, конвертировать её в integer-safe формат через BN.js, отправлять результат назад через `setLatestEthPrice` и связать всё deployment scripts и runtime loop'ами.

## Ключевые концепции

- **инициализация web3 contract по compiled artifact**
- **подписка на events и очередь запросов в памяти**
- **chunked processing loop через `setInterval`**
- **retry/fallback для ненадёжных внешних вызовов**
- **получение цены через axios**
- **масштабирование decimal price в BN.js integer**
- **отдельные процессы oracle и client**

## Код, который нужно понимать

### Получение deployed oracle контракта по artifact

```javascript
async function getOracleContract (web3js) {
  const networkId = await web3js.eth.net.getId()
  return new web3js.eth.Contract(OracleJSON.abi, OracleJSON.networks[networkId].address)
}
```

Источник: `en/15/01.md`
### Подписка на GetLatestEthPriceEvent

```javascript
async function filterEvents (oracleContract, web3js) {
  oracleContract.events.GetLatestEthPriceEvent(async (err, event) => {
    if (err) {
      console.error('Error on event', err)
      return
    }
    await addRequestToQueue(event)
  })
}
```

Источник: `en/15/03.md`
### Постановка request в очередь

```javascript
async function addRequestToQueue (event) {
  const callerAddress = event.returnValues.callerAddress
  const id = event.returnValues.id
  pendingRequests.push({ callerAddress, id })
}
```

Источник: `en/15/03.md`
### Retry loop обработки request

```javascript
async function processRequest (oracleContract, ownerAddress, id, callerAddress) {
  let retries = 0
  while (retries < MAX_RETRIES) {
    try {
      const ethPrice = await retrieveLatestEthPrice()
      await setLatestEthPrice(oracleContract, callerAddress, ownerAddress, ethPrice, id)
      return
    } catch (error) {
      if (retries === MAX_RETRIES - 1) {
        await setLatestEthPrice(oracleContract, callerAddress, ownerAddress, '0', id)
        return
      }
      retries++
    }
  }
}
```

Источник: `en/15/08.md`
### Конвертация цены в integer для EVM

```javascript
async function setLatestEthPrice (oracleContract, callerAddress, ownerAddress, ethPrice, id) {
  ethPrice = ethPrice.replace('.', '')
  const multiplier = new BN(10**10, 10)
  const ethPriceInt = (new BN(parseInt(ethPrice), 10)).mul(multiplier)
  const idInt = new BN(parseInt(id))
  await oracleContract.methods.setLatestEthPrice(ethPriceInt.toString(), callerAddress, idInt.toString()).send({ from: ownerAddress })
}
```

Источник: `en/15/10.md`
### Клиент привязывает oracle и периодически обновляет цену

```javascript
const networkId = await web3js.eth.net.getId()
const oracleAddress =  OracleJSON.networks[networkId].address
await callerContract.methods.setOracleInstanceAddress(oracleAddress).send({ from: ownerAddress })
setInterval( async () => {
  await callerContract.methods.updateEthPrice().send({ from: ownerAddress })
}, SLEEP_INTERVAL);
```

Источник: `en/15/12.md`

## Типовые задания / вопросы на экзамене

- Показать, как `getOracleContract` находит deployed oracle instance в artifact и почему для этого нужен `networkId`.
- Объяснить роль `GetLatestEthPriceEvent` во всей off-chain архитектуре и как на него реагирует JS oracle.
- Объяснить, зачем нужна in-memory очередь `pendingRequests` и как `CHUNK_SIZE` влияет на поведение обработки.
- Реализовать retry loop для `processRequest`, включая fallback-поведение после исчерпания всех попыток.
- Объяснить, почему строковую цену с Binance нужно преобразовывать перед отправкой on-chain и как здесь используется BN.js.
- Показать, что клиент обязан сделать перед периодическим вызовом `updateEthPrice`, и почему сначала нужно привязать адрес oracle.
- Объяснить, почему для oracle и caller нужны отдельные deployment/runtime конфигурации и private keys.

## Чеклист перед экзаменом

- Я могу создать web3 contract из `abi + deployed address` в JSON artifact.
- Я могу объяснить весь pipeline `event -> queue -> process -> callback`.
- Я понимаю, почему запросы обрабатываются батчами и с retry, а не inline.
- Я могу преобразовать decimal ETH price string в integer-формат для Solidity.
- Я понимаю fallback-ветку, которая в крайнем случае отправляет `0`.
- Я могу восстановить startup flow `EthPriceOracle.js` и `Client.js` по памяти.
- Я понимаю назначение отдельных Truffle configs и migration файлов.

## Состав урока

| Порядок | EN title | EN path |
|---:|---|---|
| 1 | How to Build an Oracle - Part 2 | `en/15/00-overview.md` |
| 2 | Getting Set Up | `en/15/01.md` |
| 3 | Listening for Events | `en/15/02.md` |
| 4 | Adding a Request to the Processing Queue | `en/15/03.md` |
| 5 | Looping Trough the Processing Queue | `en/15/04.md` |
| 6 | Processing the Queue | `en/15/05.md` |
| 7 | The Retry Loop | `en/15/06.md` |
| 8 | Using Try and Catch in JavaScript | `en/15/07.md` |
| 9 | Using Try and Catch in JavaScript- Cont'd | `en/15/08.md` |
| 10 | Working with Numbers in Ethereum and JavaScript | `en/15/09.md` |
| 11 | Returning multiple values in JavaScript | `en/15/10.md` |
| 12 | Wrapping Up the Oracle | `en/15/11.md` |
| 13 | Returning multiple values in JavaScript | `en/15/12.md` |
| 14 | Deploy the contracts | `en/15/13.md` |
| 15 | Putting Everything Together | `en/15/14.md` |
| 16 | Lesson Complete! | `en/15/lessoncomplete.md` |

## Источники

- Английский контент: `en/15/`
- Русский контент: отсутствует в репозитории
- Реестр урока: `en/index.ts`
- Course metadata: `en/index.json`
- Витрина курса: `https://cryptozombies.io/ru/course/`

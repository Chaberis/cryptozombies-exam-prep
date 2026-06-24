# Урок 6: App Front-ends & Web3.js

- Трек: `Solidity: Beginner to Intermediate Smart Contracts`
- Глав на сайте: **11**
- Элементов урока в repo: **12**
- Русская локаль: **нет**
- Sample code в `cryptozombies-lesson-code`: **есть**

## Что нужно уметь после урока

- поднять минимальный веб-интерфейс к смарт-контракту;
- подключить ABI и адрес контракта в Web3;
- разделить `call()` и `send()` для чтения и записи;
- слушать события контракта и обновлять UI при изменениях;
- работать с MetaMask / current provider и текущим аккаунтом пользователя.

## Краткое summary

Урок 6 переносит акцент с Solidity на интеграцию. Контракт уже есть; задача — читать его состояние, отправлять транзакции и синхронизировать UI с блокчейном. Для экзамена это означает, что нужно уверенно различать read-path (`call`) и write-path (`send`), понимать, где нужен `from`, а где ещё и `value`.

С инженерной точки зрения урок строится вокруг трёх опор: инициализация `web3`, создание экземпляра `Contract` по ABI + address и реакция на события `Transfer`. Этого уже достаточно, чтобы показать, как on-chain состояние попадает в браузер.

Даже если на экзамене не попросят написать полноценный фронтенд, почти наверняка проверят, понимаете ли вы форму вызова `contract.methods.fn(...).send(...)` и умеете ли обновлять UI после receipt или события.

## Ключевые концепции

- ABI как описание публичного API контракта для клиента;
- `new web3js.eth.Contract(abi, address)`;
- `call()` для чтения, `send()` для транзакций;
- подписка на `Transfer` и реакция на `receipt`;
- инициализация через `web3.currentProvider` и работа с текущим аккаунтом.

## Код, который нужно понимать

### Инициализация приложения и подписка на событие `Transfer`

```javascript
function startApp() {
      var cryptoZombiesAddress = "YOUR_CONTRACT_ADDRESS";
      cryptoZombies = new web3js.eth.Contract(cryptoZombiesABI, cryptoZombiesAddress);

      var accountInterval = setInterval(function () {

        if (web3.eth.accounts[0] !== userAccount) {
          userAccount = web3.eth.accounts[0];

          getZombiesByOwner(userAccount)
            .then(displayZombies);
        }
      }, 100);

      cryptoZombies.events.Transfer({ filter: { _to: userAccount } })
        .on("data", function (event) {
          let data = event.returnValues;
          getZombiesByOwner(userAccount).then(displayZombies);
        }).on("error", console.error);
    }
```

### Отправка транзакции создания зомби

```javascript
function createRandomZombie(name) {


      $("#txStatus").text("Creating new zombie on the blockchain. This may take a while...");

      return cryptoZombies.methods.createRandomZombie(name)
        .send({ from: userAccount })
        .on("receipt", function (receipt) {
          $("#txStatus").text("Successfully created " + name + "!");

          getZombiesByOwner(userAccount).then(displayZombies);
        })
        .on("error", function (error) {

          $("#txStatus").text(error);
        });
    }
```

### Payable-вызов levelUp с передачей ETH

```javascript
function levelUp(zombieId) {
      $("#txStatus").text("Leveling up your zombie...");
      return cryptoZombies.methods.levelUp(zombieId)
        .send({ from: userAccount, value: web3.utils.toWei("0.001", "ether") })
        .on("receipt", function (receipt) {
          $("#txStatus").text("Power overwhelming! Zombie successfully leveled up");
        })
        .on("error", function (error) {
          $("#txStatus").text(error);
        });
    }
```

### Чтение данных из контракта

```javascript
function getZombieDetails(id) {
      return cryptoZombies.methods.zombies(id).call()
    }

function getZombiesByOwner(owner) {
      return cryptoZombies.methods.getZombiesByOwner(owner).call()
    }
```

### Загрузка провайдера после открытия страницы

```javascript
window.addEventListener('load', function () {


      if (typeof web3 !== 'undefined') {

        web3js = new Web3(web3.currentProvider);
      } else {


      }


      startApp()

    })
```

## Типовые задания / вопросы на экзамене

- создать экземпляр контракта по ABI и адресу;
- прочитать список id по владельцу через `call()`;
- отправить mutating-функцию через `send({ from, value? })`;
- подписаться на событие и обновить локальное состояние/DOM.

## Чеклист перед экзаменом

- понимаю, почему `levelUp` требует `value: web3.utils.toWei("0.001", "ether")`;
- могу объяснить разницу между `receipt` callback и `event` subscription;
- помню, что ABI нужен клиенту, а не самому контракту;
- готов читать lesson 6 по EN, так как в репозитории нет RU-локали.

## Состав урока

| Порядок | EN title | RU title | EN path | RU path |
|---:|---|---|---|---|
| 1 | App Front-ends & Web3.js | — | `en/6/00-overview.md` | — |
| 2 | Intro to Web3.js | — | `en/6/01.md` | — |
| 3 | Web3 Providers | — | `en/6/02.md` | — |
| 4 | Talking to Contracts | — | `en/6/03.md` | — |
| 5 | Calling Contract Functions | — | `en/6/04.md` | — |
| 6 | Metamask & Accounts | — | `en/6/05.md` | — |
| 7 | Displaying our Zombie Army | — | `en/6/06.md` | — |
| 8 | Sending Transactions | — | `en/6/07.md` | — |
| 9 | Calling Payable Functions | — | `en/6/08.md` | — |
| 10 | Subscribing to Events | — | `en/6/09.md` | — |
| 11 | Wrapping It Up | — | `en/6/10-wrappingitup.md` | — |
| 12 | Lesson 6 Complete! | — | `en/6/lessoncomplete.md` | — |

## Источники

- Английский контент: `en/6/`
- Русский контент: отсутствует в репозитории
- Реестр урока: `en/index.ts`
- Sample code: `../cryptozombies-lesson-code/lesson-6/`

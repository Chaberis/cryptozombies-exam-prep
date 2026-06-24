# Урок 11: Testing Smart Contracts with Truffle

- Трек: `Advanced Solidity: Get In-depth Knowledge`
- Глав на сайте: **15**
- Элементов урока в repo: **16**
- Русская локаль: **нет**
- Sample code в `cryptozombies-lesson-code`: **да**
- Источник кода: **В `../cryptozombies-lesson-code/lesson-11/` есть chapter snapshots для `chapter-2`..`chapter-14`: в основном `test/CryptoZombies.js`, `test/helpers/utils.js`, `test/helpers/time.js` и `chapter-14/truffle.js`. Полных lesson markdown и Solidity sources в этой папке нет. Русская локаль `ru/11/` отсутствует.**

## Что нужно уметь после урока

- Как `artifacts.require(...)` даёт contract abstraction и как через `contract()` / `it()` строится Truffle test suite.
- Почему вызовы блокчейна в тестах пишутся через `async/await` и как `{ from: alice }` управляет `msg.sender`.
- Как использовать `result.receipt.status` и `result.logs[0].args...`, чтобы проверять успешность транзакции и читать event data.
- Почему каждый тест должен начинаться с нового деплоя через `beforeEach(async () => { contractInstance = await CryptoZombies.new(); })`.
- Как тестировать revert path через helper вроде `shouldThrow(...)`.
- Как проверять ERC721 ownership transfer в прямом сценарии `transferFrom` и в двухшаговом flow `approve` + `transferFrom`.
- Как работает Ganache time travel: `evm_increaseTime` нужно сопровождать `evm_mine`, иначе cooldown logic не сдвинется.
- Как перейти с `assert` на Chai `expect` и что меняется при запуске тестов против Loom.

## Краткое summary

Урок 11 учит строить полноценный JavaScript test suite для Solidity-игры с помощью Truffle. Он проходит путь от минимального skeleton теста и `beforeEach` деплоя до проверки revert paths, ERC721 transfer flows, cooldown logic через time travel в Ganache, более выразительных assertions на Chai и запуска тестов против Loom. Практический итог — рабочий набор тестов для `CryptoZombies`, вспомогательные helpers для revert/time и конфигурация `truffle.js` для Loom testnet.

## Ключевые концепции

- **Truffle contract abstractions и build artifacts**
- **Mocha-style test suite: `contract`, `it`, `context`**
- **async transaction testing через `await`**
- **проверка events и receipts**
- **negative testing для `require` / revert**
- **ERC721 transfer и approval flow**
- **time travel в Ganache через `evm_increaseTime` + `evm_mine`**
- **Chai `expect` и конфигурация Loom provider**

## Код, который нужно понимать

### Минимальный Truffle test skeleton

```javascript
const CryptoZombies = artifacts.require("CryptoZombies");
contract("CryptoZombies", (accounts) => {
    it("should be able to create a new zombie", () => {

    })
})
```

Источник: `../cryptozombies-lesson-code/lesson-11/chapter-2/test/CryptoZombies.js`
### Хелпер для проверки revert path

```javascript
async function shouldThrow(promise) {
    try {
        await promise;
       assert(true);
    }
    catch (err) {
        return;
    }
  assert(false, "The contract did not throw.");
}

module.exports = {
  shouldThrow,
};
```

Источник: `../cryptozombies-lesson-code/lesson-11/chapter-7/test/helpers/utils.js`
### Single-step ERC721 transfer test

```javascript
const result = await contractInstance.createRandomZombie(zombieNames[0], {from: alice});
const zombieId = result.logs[0].args.zombieId.toNumber();
await contractInstance.transferFrom(alice, bob, zombieId, {from: alice});
const newOwner = await contractInstance.ownerOf(zombieId);
assert.equal(newOwner, bob);
```

Источник: `../cryptozombies-lesson-code/lesson-11/chapter-11/test/CryptoZombies.js`
### Cooldown testing через time travel Ganache

```javascript
async function increase(duration) {
    await web3.currentProvider.sendAsync({
        jsonrpc: "2.0",
        method: "evm_increaseTime",
        params: [duration],
        id: new Date().getTime()
    }, () => {});

    web3.currentProvider.send({
        jsonrpc: '2.0',
        method: 'evm_mine',
        params: [],
        id: new Date().getTime()
    })
}
```

Источник: `../cryptozombies-lesson-code/lesson-11/chapter-12/test/helpers/time.js`
### Более выразительные assertions на Chai

```javascript
var expect = require('chai').expect;

const result = await contractInstance.createRandomZombie(zombieNames[0], {from: alice});
expect(result.receipt.status).to.equal(true);
expect(result.logs[0].args.name).to.equal(zombieNames[0]);
```

Источник: `../cryptozombies-lesson-code/lesson-11/chapter-13/test/CryptoZombies.js`
### Настройка Loom provider с дополнительными аккаунтами

```javascript
loom_testnet: {
    provider: function() {
        const privateKey = 'YOUR_PRIVATE_KEY';
        const chainId = 'extdev-plasma-us1';
        const writeUrl = 'wss://extdev-basechain-us1.dappchains.com/websocket';
        const readUrl = 'wss://extdev-basechain-us1.dappchains.com/queryws';
        const loomTruffleProvider = new LoomTruffleProvider(chainId, writeUrl, readUrl, privateKey);
        loomTruffleProvider.createExtraAccountsFromMnemonic(mnemonic, 10);
        return loomTruffleProvider;
    },
    network_id: '9545242630824'
}
```

Источник: `../cryptozombies-lesson-code/lesson-11/chapter-14/truffle.js`

## Типовые задания / вопросы на экзамене

- Написать минимальный Truffle test skeleton для `CryptoZombies` с `artifacts.require`, `contract` и `it`.
- Задеплоить новый контракт в `beforeEach`, создать zombie от Alice и проверить и успешность транзакции, и имя из события.
- Написать negative test, доказывающий, что один адрес не может создать двух зомби, с помощью revert-catching helper.
- Реализовать single-step ERC721 transfer test: mint, взять `zombieId` из logs, вызвать `transferFrom` и проверить `ownerOf(zombieId)`.
- Покрыть оба two-step сценария: Alice делает `approve` Bob, после чего transfer выполняет Bob или сама Alice.
- Починить attack test, который падает из-за cooldown, продвинув время Ganache на один день перед вызовом `attack`.
- Перевести проверки с `assert.equal(...)` на Chai `expect(...).to.equal(...)`, не меняя смысл тестов.
- Изменить `truffle.js` так, чтобы Loom testnet создавал дополнительные аккаунты из mnemonic, и объяснить, почему time-travel test там пропускается.

## Чеклист перед экзаменом

- Я могу объяснить, что такое Truffle build artifacts и почему тесты начинаются с `artifacts.require`.
- Я могу писать понятные test suites через `contract`, `it` и `context`.
- Я могу деплоить новый экземпляр контракта в `beforeEach` для каждого теста.
- Я могу проверять happy path и revert path Solidity-функций.
- Я умею читать event data из `result.logs` и использовать её в следующих assertions.
- Я могу тестировать ERC721 ownership transfer и прямым, и approval-based способом.
- Я умею использовать Ganache RPC helper'ы для тестирования cooldown и `block.timestamp` логики.
- Я могу применять Chai `expect` и понимаю, как меняется provider/accounts flow в Loom.

## Состав урока

| Порядок | EN title | EN path |
|---:|---|---|
| 1 | Testing Smart Contracts with Truffle | `en/11/00-overview.md` |
| 2 | Getting Set Up | `en/11/01.md` |
| 3 | Getting Set Up (cont'd) | `en/11/02.md` |
| 4 | The First Test- Creating a New Zombie | `en/11/03.md` |
| 5 | The First Test- Creating a New Zombie (cont'd) | `en/11/04.md` |
| 6 | The First Test- Creating a New Zombie (cont'd) | `en/11/05.md` |
| 7 | Keeping the Fun in the Game | `en/11/06.md` |
| 8 | Keeping the Fun in the Game (cont'd) | `en/11/07.md` |
| 9 | Zombie Transfers | `en/11/08.md` |
| 10 | ERC721 Token Transfers- Single Step Scenario | `en/11/09.md` |
| 11 | ERC721 Token Transfers- Two Step Scenario | `en/11/10.md` |
| 12 | ERC721 Token Transfers- Two Step Scenario (cont'd) | `en/11/11.md` |
| 13 | Zombie Attacks | `en/11/12.md` |
| 14 | More Expressive Assertions with Chai | `en/11/13.md` |
| 15 | Testing Against Loom | `en/11/14.md` |
| 16 | Lesson Complete! | `en/11/lessoncomplete.md` |

## Источники

- Английский контент: `en/11/`
- Русский контент: отсутствует в репозитории
- Реестр урока: `en/index.ts`
- Course metadata: `en/index.json`
- Витрина курса: `https://cryptozombies.io/ru/course/`

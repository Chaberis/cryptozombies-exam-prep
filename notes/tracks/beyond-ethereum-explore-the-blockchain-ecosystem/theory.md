# Теория по треку Beyond Ethereum

## Что покрывает этот трек

Трек `Beyond Ethereum: Explore the Blockchain Ecosystem` в уроках `17-18` посвящён **zkSync** и более широко — тому, как думать о layer-2 системах за пределами базового Ethereum mainnet workflow.

Здесь курс перестаёт быть чисто Solidity-уроками и переходит к модели:

- есть Ethereum как layer 1;
- есть layer 2 система;
- есть off-chain / on-chain связка;
- есть пользовательские операции: deposit, transfer, withdraw;
- есть отдельная задача корректной работы с токенами и их decimals.

## 1. Что такое zkSync в логике курса

### ZK-rollup

zkSync подаётся как **ZK-rollup**:
- средства пользователей удерживаются Ethereum-контрактом;
- computation и storage происходят вне L1;
- корректность подтверждается zero-knowledge proof'ами;
- итоговая система сохраняет security model, близкую к Ethereum, но масштабируется лучше.

### Почему это важно

Курс подчёркивает практические свойства:
- fast confirmations;
- lower fees;
- быстрые deposits/withdrawals;
- mainnet-level security без доверия к обычному custodial серверу.

Экзаменационно важно уметь проговорить это не как маркетинг, а как архитектурный смысл:
**layer 2 снижает стоимость и latency, не вынося custody средств в произвольную централизованную базу данных.**

## 2. Двойная модель provider'ов

zkSync-приложение в курсе почти всегда работает с двумя мирами сразу:

- `ethers` provider для Ethereum;
- `zksync` provider для L2.

Это значит:
- deposit начинается со стороны Ethereum;
- transfer живёт внутри zkSync;
- withdraw заканчивается возвратом обратно в Ethereum.

Если человек не понимает, зачем нужны оба provider'а, он не понимает архитектуру урока.

## 3. Wallet model

### Ethereum wallet -> zkSync wallet

Ключевая идея:
- zkSync wallet создаётся из Ethereum wallet;
- владение аккаунтом не отделено от базового Ethereum private key;
- zkSync не вводит "магическую" отдельную личность вне этой связки.

### Signing key registration

Перед операциями внутри zkSync нужен **signing key registration**.

Паттерн урока:
- проверить `isSigningKeySet()`;
- убедиться, что у аккаунта есть `accountId`;
- вызвать `setSigningKey()`;
- дождаться receipt.

Это важно, потому что без регистрации локальной подписи часть L2-операций просто не может быть исполнена корректно.

## 4. Deposit / Transfer / Withdraw lifecycle

### Deposit

Deposit — это мост из L1 в L2.

В lesson 17 для ETH:
- значение приводится через `parseEther`;
- вызывается `depositToSyncFromEthereum`;
- ожидается receipt от операторов.

Для ERC20 в lesson 18 добавляется обязательный approval перед депозитом.

### Transfer

Transfer — это внутренняя L2-операция.
Она:
- требует fee estimation;
- требует нормализации суммы и комиссии к packable format;
- не должна слепо работать с human-readable string.

### Withdraw

Withdraw:
- тоже требует fee;
- тоже использует packable amount/fee;
- завершается ожиданием verify receipt, потому что вывод должен быть подтверждён в более сильной форме.

## 5. Packable values и fee model

Курс постоянно возвращает к двум идеям:

- fee нужно вычислять заранее;
- некоторые значения должны быть приведены к packable representation.

Почему это важно:
- не все суммы сериализуются в транзакцию как произвольные decimal-строки;
- zkSync использует собственные ограничения на представление amount/fee;
- практический код должен сначала привести значение к допустимому формату.

Если это пропустить, код может быть "логически верным", но технически неисполняемым.

## 6. Committed vs Verified balances

Это один из главных conceptual checkpoints трека.

### Committed balance
Состояние уже принято внутри zkSync и обычно уже пригодно для дальнейших действий в L2.

### Verified balance
Состояние прошло более сильную финализацию и верификацию относительно Ethereum.

Практически:
- committed появляется быстрее;
- verified даёт более сильную финальность;
- приложение должно уметь читать и объяснять оба состояния.

На экзамене это часто проверяется вопросом:  
**почему баланс уже виден и даже usable, хотя финальная Ethereum verification ещё не завершена?**

## 7. ETH-only app vs multi-token app

Lesson 17 строит ETH-only приложение.

Lesson 18 показывает, что этого недостаточно, если приложение должно быть общим.

### В чём ошибка ETH-only подхода

Использовать:
- `parseEther`
- `formatEther`

для любого токена — ошибка, потому что:
- ETH имеет 18 decimals;
- другие ERC20 могут иметь 6, 8, 18 и т.д.;
- одинаковое преобразование ломает реальные суммы.

### TokenSet

Решение урока — `zkSyncProvider.tokenSet`.

Нужно понимать две операции:

```javascript
tokenSet.parseToken(token, amount)
tokenSet.formatToken(token, amount)
```

- `parseToken` переводит человекочитаемую строку в on-chain / API-friendly representation;
- `formatToken` делает обратное преобразование для отображения пользователю.

Это главная идея lesson 18.

## 8. ERC20 allowance перед депозитом

Для ERC20 нельзя просто сразу сделать deposit.  
Нужно сначала разрешить zkSync контракту тратить токены:

```javascript
await aliceZkSyncWallet.approveERC20TokenDeposits(token)
```

Смысл:
- депозит ERC20 требует allowance модели ERC20;
- без approval контракт моста не сможет списать токены у пользователя.

Это типовая точка ошибки на экзамене.

## 9. Multi-token display logic

Когда приложение поддерживает не только ETH, нельзя:
- жёстко печатать один баланс;
- предполагать существование только одной валюты;
- форматировать всё одинаково.

Поэтому lesson 18 переводит код к итерации по:

- `state.committed.balances`
- `state.verified.balances`

и форматирует каждое значение через `tokenSet.formatToken(...)`.

## 10. Финальная архитектура приложения

После уроков 17-18 итоговая модель такая:

- `utils.js` хранит общие helpers;
- `alice.js` исполняет user flow;
- `bob.js` отображает merchant side / balance tracking.

### Alice flow
- создать providers;
- создать Ethereum и zkSync wallet;
- проверить/показать баланс;
- при ERC20 сделать approval;
- deposit в zkSync;
- register signing key;
- transfer;
- withdraw.

### Bob flow
- создать wallets/providers;
- показать identity;
- периодически опрашивать состояние баланса.

## Ключевые термины

- **Layer 1 (L1)** — базовая сеть Ethereum.
- **Layer 2 (L2)** — надстройка над L1, уменьшающая cost/latency.
- **ZK-rollup** — L2-модель с off-chain execution и proof-based correctness.
- **zkSync** — конкретная ZK-rollup система, рассматриваемая в курсе.
- **Provider** — объект подключения к blockchain сети/API.
- **Signing key** — ключ, которым аккаунт авторизует операции в zkSync.
- **Deposit** — перевод средств из Ethereum в zkSync.
- **Transfer** — перемещение средств внутри zkSync.
- **Withdraw** — вывод средств обратно в Ethereum.
- **Committed balance** — состояние, уже принятое внутри zkSync.
- **Verified balance** — состояние, уже подтверждённое сильнее через proof/finalization.
- **TokenSet** — helper для token-aware parse/format операций.
- **Allowance** — разрешение ERC20 контракту тратить токены пользователя.
- **Packable amount/fee** — сумма/комиссия, приведённая к допустимому для zkSync представлению.

## Что нужно знать по итогам всего трека

После уроков 17-18 человек должен уметь:

- объяснить, зачем нужен zkSync и как он вписывается в L1/L2 модель;
- подключить Ethereum и zkSync provider'ы;
- создать zkSync wallet из Ethereum wallet;
- зарегистрировать signing key;
- сделать deposit, transfer и withdraw;
- посчитать fee и привести amount/fee к packable format;
- различать committed и verified balances;
- объяснить, почему ETH-only conversions ломаются на ERC20;
- использовать `TokenSet` для token-aware конвертаций;
- объяснить, зачем нужен ERC20 approval до депозита.

## На чём чаще всего валятся

- не понимают, зачем приложению одновременно Ethereum и zkSync provider;
- забывают про `setSigningKey` перед zkSync transfer flow;
- путают committed и verified balance;
- используют `parseEther` для USDT и других не-18-decimal токенов;
- забывают про ERC20 approval перед deposit;
- не приводят amount/fee к packable values;
- форматируют все токены как ETH;
- не могут связно описать end-to-end Alice/Bob demo.

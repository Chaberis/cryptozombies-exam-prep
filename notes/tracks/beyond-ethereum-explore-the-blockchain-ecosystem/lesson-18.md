# Урок 18: Advanced zkSync Concepts

- Трек: `Beyond Ethereum: Explore the Blockchain Ecosystem`
- Глав на сайте: **10**
- Элементов урока в repo: **11**
- Русская локаль: **нет**
- Sample code в `cryptozombies-lesson-code`: **нет**
- Источник кода: **В `cryptozombies-lesson-code` нет отдельного каталога `lesson-18/`. Нужный код берётся из answer blocks в `en/18/*.md`.**

## Что нужно уметь после урока

- Урок 18 опирается на lesson 17 и расширяет ETH-only demo до ERC20 токенов, в курсе это показано на примере USDT.
- Decimals у токенов различаются: у ETH 18 decimals, у USDT 6, поэтому жёстко использовать `parseEther`/`formatEther` нельзя.
- `zkSyncProvider.tokenSet` — центральный helper для token-aware преобразований.
- Для отправки человекочитаемых сумм в zkSync операции нужно использовать `tokenSet.parseToken(token, amount)`.
- Для показа балансов и fees пользователю нужно использовать `tokenSet.formatToken(token, amount)`.
- Перед депозитом ERC20 из Ethereum в zkSync обязательно нужно вызвать `approveERC20TokenDeposits(token)`.
- `displayZkSyncBalance` больше не может печатать только ETH: он должен итерироваться по `state.committed.balances` и `state.verified.balances`.
- Итоговое приложение поддерживает переключение актива через переменную `token` и согласованные deposit/transfer/withdraw amounts.

## Краткое summary

Урок 18 — это вторая часть по zkSync, где ETH-only demo из lesson 17 превращается в multi-token приложение для ERC20 активов, например USDT. Курс объясняет, почему `parseEther`/`formatEther` нельзя применять ко всем токенам, как использовать `zkSyncProvider.tokenSet` для token-aware конвертаций, зачем нужен `approveERC20TokenDeposits` перед депозитом ERC20, как печатать committed/verified balances для нескольких токенов и как пересчитать deposit/transfer/withdraw/fee функции так, чтобы они корректно работали с разными decimals. Итоговый артефакт — Node.js приложение `alice.js`, `bob.js`, `utils.js`, которое умеет работать с ERC20 балансами на Ethereum и zkSync, считать token-aware fees и выполнять deposit / transfer / withdraw для выбранного токена.

## Ключевые концепции

- **zkSync `TokenSet`**
- **token decimals и разница между human-readable value и `BigNumber`**
- **ERC20 allowance / approval перед депозитом**
- **`getEthereumBalance(token)` на `zksync.Wallet`**
- **packable transaction amount и fee**
- **committed vs verified balances для нескольких токенов**
- **token-aware форматирование fees через `getTransactionFee` + `tokenSet.formatToken`**

## Код, который нужно понимать

### Чтение и форматирование ERC20 баланса через TokenSet

```javascript
const tokenSet = zkSyncProvider.tokenSet
const aliceInitialRinkebyBalance = await aliceZkSyncWallet.getEthereumBalance(token)
console.log(`Alice's initial balance on Rinkeby is: ${tokenSet.formatToken(token, aliceInitialRinkebyBalance)}`)
```

Источник: `en/18/02.md`
### Approval ERC20 депозитов в zkSync

```javascript
await aliceZkSyncWallet.approveERC20TokenDeposits(token)
```

Источник: `en/18/03.md`
### Депозит с token-aware parsing

```javascript
async function depositToZkSync (zkSyncWallet, token, amountToDeposit, tokenSet) {
  const deposit = await zkSyncWallet.depositToSyncFromEthereum({
    depositTo: zkSyncWallet.address(),
    token: token,
    amount: tokenSet.parseToken(token, amountToDeposit)
  })
  await deposit.awaitReceipt()
}
```

Источник: `en/18/05.md`
### Печать committed и verified balances

```javascript
const state = await wallet.getAccountState()
const committedBalances = state.committed.balances
const verifiedBalances = state.verified.balances
for (const property in committedBalances) {
  console.log(`Commited ${property} balance for ${wallet.address()}: ${tokenSet.formatToken(property, committedBalances[property])}`)
}
for (const property in verifiedBalances) {
  console.log(`Verified ${property} balance for ${wallet.address()}: ${tokenSet.formatToken(property, verifiedBalances[property])}`)
}
```

Источник: `en/18/06.md`
### Transfer с packable token amount и fee

```javascript
const closestPackableAmount = zksync.utils.closestPackableTransactionAmount(tokenSet.parseToken(token, amountToTransfer))
const closestPackableFee = zksync.utils.closestPackableTransactionFee(tokenSet.parseToken(token, transferFee))
const transfer = await from.syncTransfer({
  to: toAddress,
  token: token,
  amount: closestPackableAmount,
  fee: closestPackableFee
})
```

Источник: `en/18/07.md`
### Форматирование fees через TokenSet

```javascript
async function getFee(transactionType, address, token, zkSyncProvider, tokenSet) {
  const fee = await zkSyncProvider.getTransactionFee(transactionType, address, token)
  return tokenSet.formatToken(token, fee.totalFee)
}
```

Источник: `en/18/09.md`

## Типовые задания / вопросы на экзамене

- Объяснить, почему `ethers.utils.parseEther` и `ethers.utils.formatEther` неверны для multi-token zkSync app с USDT.
- По ETH-only версии `alice.js` переписать получение и вывод баланса так, чтобы показывался ERC20 баланс Alice в Ethereum.
- Добавить обязательный approval step перед депозитом ERC20 в zkSync и объяснить, зачем он нужен.
- Переделать `depositToZkSync`, `transfer` и `withdrawToEthereum`, чтобы они принимали `TokenSet` и корректно конвертировали token amounts.
- Написать или объяснить функцию, которая печатает все committed и verified balances из `wallet.getAccountState()` через `for...in`.
- Обновить helper `getFee`, чтобы он возвращал human-readable fee для выбранного токена, а не предполагал ETH decimals.
- Описать полный flow `alice.js`: инициализация provider'ов и wallet'ов, чтение token balance, approval, deposit, register signing key, transfer, withdraw.

## Чеклист перед экзаменом

- Я знаю точное название урока `Advanced zkSync Concepts` и трека `Beyond Ethereum: Explore the Blockchain Ecosystem`.
- Я помню, что урок содержит 11 repo elements: overview, 9 chapter files и lesson complete.
- Я могу объяснить, что делает `zkSyncProvider.tokenSet` и когда вызывать `parseToken`, а когда `formatToken`.
- Я помню, что ERC20 депозиты требуют `approveERC20TokenDeposits(token)` до самого депозита.
- Я могу получить ERC20 баланс в Ethereum через `await zkSyncWallet.getEthereumBalance(token)`.
- Я могу обновить deposit/transfer/withdraw helpers так, чтобы и amount, и fee стали token-aware и packable.
- Я могу различить committed и verified balances и вывести оба набора из account state.
- Я понимаю, что итоговый артефакт — это multi-asset zkSync Node.js app на `alice.js`, `bob.js` и `utils.js`.

## Состав урока

| Порядок | EN title | EN path |
|---:|---|---|
| 1 | Advanced zkSync Concepts | `en/18/00-overview.md` |
| 2 | Intro | `en/18/01.md` |
| 3 | Work with tokens | `en/18/02.md` |
| 4 | Approve ERC20 token transfers | `en/18/03.md` |
| 5 | Approve ERC20 token transfers | `en/18/04.md` |
| 6 | Update the depositToZkSync function | `en/18/05.md` |
| 7 | Update the displayZkSyncBalance function | `en/18/06.md` |
| 8 | Update the transfer function | `en/18/07.md` |
| 9 | Update the withdrawToEthereum function | `en/18/08.md` |
| 10 | Update the getFee function | `en/18/09.md` |
| 11 | Lesson Complete! | `en/18/lessoncomplete.md` |

## Источники

- Английский контент: `en/18/`
- Русский контент: отсутствует в репозитории
- Реестр урока: `en/index.ts`
- Course metadata: `en/index.json`
- Витрина курса: `https://cryptozombies.io/ru/course/`

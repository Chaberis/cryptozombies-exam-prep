# Урок 17: Intro to zkSync

- Трек: `Beyond Ethereum: Explore the Blockchain Ecosystem`
- Глав на сайте: **16**
- Элементов урока в repo: **17**
- Русская локаль: **нет**
- Sample code в `cryptozombies-lesson-code`: **нет**
- Источник кода: **В `cryptozombies-lesson-code` нет отдельного каталога `lesson-17/`, а в самих lesson-17 markdown нет прямой привязки к sample code по главам. Нужный код берётся из answer blocks в `en/17/*.md`.**

## Что нужно уметь после урока

- zkSync в курсе подаётся как layer-2 ZK-rollup, где средства защищены Ethereum-смарт-контрактом, а вычисления и хранение состояния происходят off-chain.
- Приложению нужны сразу два provider'а: zkSync provider (`zksync.getDefaultProvider`) и Ethereum provider (`ethers.getDefaultProvider`).
- zkSync wallet инициализируется из Ethereum wallet, поэтому владение аккаунтом привязано к Ethereum private key.
- Перед переводами на zkSync нужно зарегистрировать signing key через `isSigningKeySet`, `getAccountId` и `setSigningKey`, а затем дождаться receipt.
- Для завода ETH из Ethereum в zkSync используется `depositToSyncFromEthereum`, а человекочитаемое значение ETH нужно переводить через `ethers.utils.parseEther`.
- И переводы, и выводы требуют расчёта fee и нормализации суммы/комиссии через `closestPackableTransactionAmount` и `closestPackableTransactionFee`.
- В zkSync есть committed balances и verified balances; committed средства уже можно использовать до финальной верификации на Ethereum.
- Полный workflow урока: создать Alice/Bob wallets, завести ETH Alice в zkSync, зарегистрировать Alice, перевести ETH Bob, отобразить баланс Bob и вывести остаток Alice обратно в Ethereum.

## Краткое summary

Урок 17 — это практическое введение в zkSync как в ZK-rollup платёжную систему второго уровня. Курс показывает, как построить маленький Node.js workflow вокруг `ethers` и `zksync`: подключиться к Ethereum и zkSync provider'ам, создать zkSync wallet из Ethereum wallet, зарегистрировать signing key, завести ETH из Ethereum в zkSync, посчитать fees, перевести ETH между zkSync-аккаунтами, различать committed и verified balances и вывести остаток обратно в Ethereum. Итоговый артефакт — минимальное двухскриптовое приложение `alice.js` / `bob.js` с общим `utils.js`, где Alice пополняет zkSync, платит Bob и выводит остаток, а Bob следит за изменением баланса.

## Ключевые концепции

- **архитектура ZK-rollup / zkSync**
- **zero-knowledge proofs и мотивация zk-SNARK**
- **Ethereum wallets, private keys и производные zkSync accounts**
- **регистрация signing key в zkSync**
- **deposit / transfer / withdraw lifecycle**
- **расчёт fee и packable values**
- **committed vs verified balances**
- **polling состояния через `setInterval`**

## Код, который нужно понимать

### Создание zkSync provider

```javascript
async function getZkSyncProvider (zksync, networkName) {
  let zkSyncProvider
  try {
    zkSyncProvider = await zksync.getDefaultProvider(networkName)
  } catch (error) {
    console.log('Unable to connect to zkSync.')
    console.log(error)
  }
  return zkSyncProvider
}
```

Источник: `en/17/02.md`
### Регистрация signing key в zkSync

```javascript
async function registerAccount (wallet) {
  console.log(`Registering the ${wallet.address()} account on zkSync`)
  if (!await wallet.isSigningKeySet()) {
    if (await wallet.getAccountId() === undefined) {
      throw new Error('Unknown account')
    }
    const changePubkey = await wallet.setSigningKey()
    await changePubkey.awaitReceipt()
  }
}
```

Источник: `en/17/04.md`
### Депозит ETH из Ethereum в zkSync

```javascript
async function depositToZkSync (zkSyncWallet, token, amountToDeposit, ethers) {
  const deposit = await zkSyncWallet.depositToSyncFromEthereum({
    depositTo: zkSyncWallet.address(),
    token: token,
    amount: ethers.utils.parseEther(amountToDeposit)
  })
  try {
    await deposit.awaitReceipt()
  } catch (error) {
    console.log('Error while awaiting confirmation from the zkSync operators.')
    console.log(error)
  }
}
```

Источник: `en/17/05.md`
### Packable transfer внутри zkSync

```javascript
async function transfer (from, toAddress, amountToTransfer, transferFee, token, zksync, ethers) {
  const closestPackableAmount = zksync.utils.closestPackableTransactionAmount(
    ethers.utils.parseEther(amountToTransfer))
  const closestPackableFee = zksync.utils.closestPackableTransactionFee(
    ethers.utils.parseEther(transferFee))

  const transfer = await from.syncTransfer({
    to: toAddress,
    token: token,
    amount: closestPackableAmount,
    fee: closestPackableFee
  })
  const transferReceipt = await transfer.awaitReceipt()
  console.log('Got transfer receipt.')
  console.log(transferReceipt)
}
```

Источник: `en/17/06.md`
### Расчёт zkSync fee

```javascript
async function getFee(transactionType, address, token, zkSyncProvider, ethers) {
  const feeInWei = await zkSyncProvider.getTransactionFee(transactionType, address, token)
  return ethers.utils.formatEther(feeInWei.totalFee.toString())
}
```

Источник: `en/17/07.md`
### Вывод средств из zkSync в Ethereum

```javascript
async function withdrawToEthereum (wallet, amountToWithdraw, withdrawalFee, token, zksync, ethers) {
  const closestPackableAmount = zksync.utils.closestPackableTransactionAmount(ethers.utils.parseEther(amountToWithdraw))
  const closestPackableFee = zksync.utils.closestPackableTransactionFee(ethers.utils.parseEther(withdrawalFee))
  const withdraw = await wallet.withdrawFromSyncToEthereum({
    ethAddress: wallet.address(),
    token: token,
    amount: closestPackableAmount,
    fee: closestPackableFee
    })
    await withdraw.awaitVerifyReceipt()
    console.log('ZKP verification is complete')
}
```

Источник: `en/17/08.md`

## Типовые задания / вопросы на экзамене

- Объяснить, что такое zkSync в терминах урока и почему он считается layer-2 scaling solution, защищённой Ethereum.
- Написать или дополнить helper, который создаёт zkSync provider и корректно обрабатывает ошибку подключения.
- По данному zkSync wallet реализовать регистрацию аккаунта: проверить signing key и вызвать `setSigningKey` только при необходимости.
- Реализовать helper депозита, который заводит ETH из Ethereum в zkSync и дожидается operator receipt.
- Посчитать transfer fee, привести amount и fee к packable формату и выполнить перевод в другой zkSync address.
- Объяснить разницу между committed и verified balances и показать, как урок выводит оба состояния.
- Дописать `bob.js`, чтобы он один раз показывал личность Bob в Rinkeby, а затем по таймеру отслеживал баланс в zkSync.
- Дописать `alice.js`, чтобы он пополнял zkSync, регистрировал аккаунт, переводил ETH Bob и выводил остаток назад в Ethereum.

## Чеклист перед экзаменом

- Я могу определить zkSync как ZK-rollup с хранением средств в Ethereum и off-chain исполнением.
- Я могу создать оба provider'а для одной сети и объяснить, зачем нужны оба.
- Я могу создать `ethers.Wallet` из private key и вывести из него zkSync wallet.
- Я могу корректно зарегистрировать zkSync account, включая проверку `Unknown account` и ожидание receipt.
- Я умею использовать `parseEther` и `formatEther` в правильных точках ETH workflow.
- Я умею рассчитывать fee и packable amount/fee перед отправкой transfer и withdraw.
- Я могу объяснить committed и verified balances и безопасно вывести их, даже если баланс нулевой.
- Я понимаю итоговую схему из `alice.js`, `bob.js` и общего `utils.js`.

## Состав урока

| Порядок | EN title | EN path |
|---:|---|---|
| 1 | Intro to zkSync | `en/17/00-overview.md` |
| 2 | Getting set up | `en/17/01.md` |
| 3 | Let’s talk to zkSync | `en/17/02.md` |
| 4 | Create a new zkSync account | `en/17/03.md` |
| 5 | Create a new zkSync account - continued | `en/17/04.md` |
| 6 | Deposit assets to zkSync | `en/17/05.md` |
| 7 | Transfer assets on zkSync | `en/17/06.md` |
| 8 | Transfer fees | `en/17/07.md` |
| 9 | Withdraw to Ethereum | `en/17/08.md` |
| 10 | Account balances | `en/17/09.md` |
| 11 | The shopkeeper - Talking to the blockchain | `en/17/10.md` |
| 12 | The shopkeeper - Wallets and private keys | `en/17/11.md` |
| 13 | Update balances | `en/17/12.md` |
| 14 | Register an account | `en/17/13.md` |
| 15 | Make a payment on zkSync | `en/17/14.md` |
| 16 | Putting everything together | `en/17/15.md` |
| 17 | Lesson Complete! | `en/17/lessoncomplete.md` |

## Источники

- Английский контент: `en/17/`
- Русский контент: отсутствует в репозитории
- Реестр урока: `en/index.ts`
- Course metadata: `en/index.json`
- Витрина курса: `https://cryptozombies.io/ru/course/`

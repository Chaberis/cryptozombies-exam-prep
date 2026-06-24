# Урок 16: How to Build an Oracle - Part 3

- Трек: `Advanced Solidity: Get In-depth Knowledge`
- Глав на сайте: **11**
- Элементов урока в repo: **12**
- Русская локаль: **нет**
- Sample code в `cryptozombies-lesson-code`: **нет**
- Источник кода: **В `../cryptozombies-lesson-code/` нет каталога `lesson-16/`. Нужный код берётся из answer blocks в `en/16/*.md`.**

## Что нужно уметь после урока

- Как подключается OpenZeppelin `Roles` через `using Roles for Roles.Role` и как хранятся роли `owners` и `oracles`.
- Почему после отказа от `Ownable` нужен свой constructor и как в него передаётся initial owner.
- Как `addOracle` и `removeOracle` проверяют права owner'а, существование/отсутствие oracle и эмитят события.
- Зачем нужен `numOracles` и почему нельзя удалить последнего oracle.
- Как responses хранятся по request id через `mapping(uint256 => Response[])`, а не завершаются первым же ответом.
- Как `THRESHOLD` определяет момент финализации и почему это лучше, чем ждать вообще всех oracle.
- Как контракт вычисляет среднюю ETH price по нескольким ответам и возвращает её caller контракту.
- Зачем в aggregation path добавляется SafeMath и почему после fulfill удаляются и `pendingRequests[_id]`, и `requestIdToResponse[_id]`.

## Краткое summary

Урок 16 превращает single-owner oracle в более децентрализованный workflow. Вместо одного обновляющего owner'а появляется role-based access control, owner может управлять набором oracle-адресов, каждый request собирает несколько ответов, после достижения порога `THRESHOLD` контракт вычисляет среднюю цену, защищает арифметику через SafeMath, очищает request state и только потом возвращает итоговую цену caller контракту.

## Ключевые концепции

- **role-based access control через OpenZeppelin Roles**
- **constructor-based инициализация owner после отказа от `Ownable`**
- **управление жизненным циклом oracle-адресов**
- **сбор нескольких ответов в `Response[]` по request id**
- **threshold-based finalization**
- **агрегация цены через среднее значение**
- **защита арифметики через SafeMath**
- **cleanup state и callback caller контракту**

## Код, который нужно понимать

### Подключение Roles и хранение owner/oracle ролей

```solidity
using Roles for Roles.Role;
Roles.Role private owners;
Roles.Role private oracles;
```

Источник: `en/16/01.md`
### Constructor задаёт начального owner

```solidity
constructor (address _owner) public {
  owners.add(_owner);
}
```

Источник: `en/16/02.md`
### Owner добавляет новый oracle

```solidity
function addOracle (address _oracle) public {
  require(owners.has(msg.sender), "Not an owner!");
  require(!oracles.has(_oracle), "Already an oracle!");
  oracles.add(_oracle);
  emit AddOracleEvent(_oracle);
}
```

Источник: `en/16/04.md`
### Модель хранения ответов по request id

```solidity
struct Response {
  address oracleAddress;
  address callerAddress;
  uint256 ethPrice;
}
mapping (uint256=>Response[]) public requestIdToResponse;
```

Источник: `en/16/06.md`
### Агрегация ответов после достижения threshold

```solidity
uint numResponses = requestIdToResponse[_id].length;
if (numResponses == THRESHOLD) {
  uint computedEthPrice = 0;
  for (uint f=0; f < requestIdToResponse[_id].length; f++) {
    computedEthPrice = computedEthPrice.add(requestIdToResponse[_id][f].ethPrice);
  }
  computedEthPrice = computedEthPrice.div(numResponses);
}
```

Источник: `en/16/10.md`
### Очистка состояния и callback с итоговой ценой

```solidity
delete pendingRequests[_id];
delete requestIdToResponse[_id];
CallerContractInterface callerContractInstance;
callerContractInstance = CallerContractInterface(_callerAddress);
callerContractInstance.callback(computedEthPrice, _id);
emit SetLatestEthPriceEvent(computedEthPrice, _callerAddress);
```

Источник: `en/16/10.md`

## Типовые задания / вопросы на экзамене

- Реализовать role-based access control так, чтобы только owner мог добавлять/удалять oracle-адреса, а только oracle мог отправлять цену.
- Написать constructor, который принимает owner address и инициализирует роль `owners` при деплое.
- Дописать `addOracle`, чтобы он запрещал дубликаты и эмитил событие о регистрации нового oracle.
- Дописать `removeOracle`, чтобы он запрещал удалять последнего oracle и безопасно уменьшал `numOracles`.
- Определить `struct Response` и хранить ответы в `requestIdToResponse[_id]`.
- Изменить `setLatestEthPrice`, чтобы он ждал `requestIdToResponse[_id].length == THRESHOLD` перед финализацией запроса.
- Посчитать итоговую ETH price как среднее по собранным ответам и вызвать callback для requestor'а.
- Заменить обычную арифметику на SafeMath и очистить request state после fulfill.

## Чеклист перед экзаменом

- Я могу объяснить переход от `Ownable` к `Roles`.
- Я могу воспроизвести проверки прав owner/oracle через `require`.
- Я понимаю, зачем нужен `numOracles` и запрет на удаление последнего oracle.
- Я могу описать `Response` и структуру `requestIdToResponse` по памяти.
- Я могу проследить весь путь fulfillment: ответ oracle -> рост счётчика -> достижение threshold -> average -> callback -> event.
- Я понимаю, зачем SafeMath используется даже в кажущейся простой aggregation logic.
- Я помню, что после завершения запроса нужно удалить и pending flag, и накопленные responses.

## Состав урока

| Порядок | EN title | EN path |
|---:|---|---|
| 1 | How to Build an Oracle - Part 3 | `en/16/00-overview.md` |
| 2 | Using Roles | `en/16/01.md` |
| 3 | Using Constructor to Set the Owner | `en/16/02.md` |
| 4 | Setting Up New Oracles | `en/16/03.md` |
| 5 | The Logical NOT Operator | `en/16/04.md` |
| 6 | Removing an Oracle | `en/16/05.md` |
| 7 | Keeping Track of Responses | `en/16/06.md` |
| 8 | Computing the ETH Price | `en/16/07.md` |
| 9 | Computing the ETH Price- Cont'd | `en/16/08.md` |
| 10 | Using SafeMath | `en/16/09.md` |
| 11 | Using SafeMath | `en/16/10.md` |
| 12 | Lesson Complete! | `en/16/lessoncomplete.md` |

## Источники

- Английский контент: `en/16/`
- Русский контент: отсутствует в репозитории
- Реестр урока: `en/index.ts`
- Course metadata: `en/index.json`
- Витрина курса: `https://cryptozombies.io/ru/course/`

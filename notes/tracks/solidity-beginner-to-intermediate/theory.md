# Теория по треку Solidity: Beginner to Intermediate Smart Contracts

Этот файл — общий теоретический конспект по урокам 1–6 CryptoZombies. Его цель: дать человеку, который не проходил курс последовательно, цельную картину терминов, понятий и знаний, которые ожидаются на экзамене.

## 1. Общая модель курса

Весь трек строится вокруг одной эволюционирующей системы:
- сначала есть простой контракт `ZombieFactory`;
- затем у зомби появляется владелец;
- затем добавляются уровни, кулдауны, административный доступ и игровые действия;
- затем зомби превращаются в ERC721-подобные токены;
- затем к контракту подключается браузерный фронтенд через Web3.js.

Главная идея курса: не писать каждый урок с нуля, а **последовательно наращивать один и тот же домен**.

---

## 2. Базовые понятия Solidity

### Контракт
Контракт в Solidity — это одновременно:
- единица кода;
- единица хранения состояния;
- единица адреса в блокчейне.

Контракт хранит state variables и предоставляет функции для чтения/изменения состояния.

### State variables
State variables живут в storage и сохраняются между транзакциями.

Примеры из курса:
- `uint dnaDigits = 16;`
- `Zombie[] public zombies;`
- `mapping (uint => address) public zombieToOwner;`

### Функции
Основные виды функций в курсе:
- `public` — доступны извне и изнутри;
- `external` — вызываются извне;
- `private` — только внутри контракта;
- `internal` — внутри контракта и наследников;
- `view` — читают состояние, но не меняют его;
- `payable` — могут принимать ETH.

### Struct
`struct` нужен для описания составного объекта.

Пример:
```solidity
struct Zombie {
  string name;
  uint dna;
  uint32 level;
  uint32 readyTime;
  uint16 winCount;
  uint16 lossCount;
}
```

### Массивы
В курсе зомби хранятся в динамическом массиве:
```solidity
Zombie[] public zombies;
```

`push(...)` добавляет новый элемент в конец массива.

### Mapping
`mapping` — ассоциативное хранилище key -> value.

Основные mappings курса:
```solidity
mapping (uint => address) public zombieToOwner;
mapping (address => uint) ownerZombieCount;
mapping (uint => address) zombieApprovals;
```

---

## 3. Модель данных CryptoZombies

Базовая доменная модель курса:
- у каждого зомби есть id — это индекс в массиве `zombies`;
- у каждого зомби есть владелец;
- у владельца есть количество зомби;
- зомби можно создавать, кормить, прокачивать, атаковать и передавать.

Фактически курс учит строить stateful-приложение поверх blockchain storage.

Ключевой инвариант ранних уроков:
- стартового зомби можно получить только один раз на адрес.

Пример проверки:
```solidity
require(ownerZombieCount[msg.sender] == 0);
```

---

## 4. Visibility и API-контракт

В курсе важно различать:
- внешний API для пользователя;
- внутренние helper-функции.

Пример:
```solidity
function createRandomZombie(string _name) public {
  uint randDna = _generateRandomDna(_name);
  _createZombie(_name, randDna);
}

function _createZombie(string _name, uint _dna) private {
  ...
}
```

Зачем так делать:
- публичная функция задаёт внешний сценарий;
- внутренняя функция переиспользуется из других частей системы;
- так проще расширять контракт в следующих уроках.

---

## 5. Events

События нужны, чтобы внешний мир видел, что произошло в контракте.

Пример:
```solidity
event NewZombie(uint zombieId, string name, uint dna);
```

И вызов:
```solidity
emit NewZombie(id, _name, _dna);
```

Позже во фронтенде события используются для реактивного обновления UI.

Главная мысль:
- storage — источник истины;
- events — журнал изменений, удобный для клиентов и индексаторов.

---

## 6. Хеширование и псевдослучайность

Курс использует:
```solidity
uint rand = uint(keccak256(abi.encodePacked(_str)));
```

Или:
```solidity
return uint(keccak256(abi.encodePacked(now, msg.sender, randNonce))) % _modulus;
```

Что нужно понимать:
- `keccak256` возвращает хеш;
- его можно привести к `uint`;
- это **не настоящая безопасная случайность**;
- для учебной игры этого достаточно, для production — нет.

На экзамене надо уметь объяснить это ограничение.

---

## 7. `msg.sender`, `require`, ownership

### `msg.sender`
Это адрес инициатора текущего вызова.

Используется для:
- создания ownership-связи;
- авторизации;
- ограничения доступа.

### `require`
`require(...)` останавливает выполнение, если условие не выполнено.

Типовые проверки из курса:
```solidity
require(ownerZombieCount[msg.sender] == 0);
require(msg.sender == zombieToOwner[_zombieId]);
require(msg.value == levelUpFee);
```

### Ownership на уровне домена
Ownership зомби хранится отдельно:
```solidity
mapping (uint => address) public zombieToOwner;
```

Это **не то же самое**, что owner всего контракта.

---

## 8. Наследование и композиция контрактов

Курс активно использует inheritance:
- `ZombieFeeding is ZombieFactory`
- `ZombieHelper is ZombieFeeding`
- `ZombieAttack is ZombieHelper`
- `ZombieOwnership is ZombieAttack, ERC721`

Что это даёт:
- постепенное расширение поведения;
- повторное использование storage и функций;
- разбиение домена по ответственности.

Это один из главных паттернов курса: **один большой проект собирается из цепочки наследуемых контрактов**.

---

## 9. Интерфейсы внешних контрактов

Для взаимодействия с внешним контрактом не нужен весь исходник — нужен интерфейс нужных методов.

Пример:
```solidity
contract KittyInterface {
  function getKitty(uint256 _id) external view returns (
    bool isGestating,
    bool isReady,
    uint256 cooldownIndex,
    uint256 nextActionAt,
    uint256 siringWithId,
    uint256 birthTime,
    uint256 matronId,
    uint256 sireId,
    uint256 generation,
    uint256 genes
  );
}
```

Далее интерфейс привязывается к адресу:
```solidity
kittyContract = KittyInterface(_address);
```

И используется как обычный объект.

Что важно знать:
- интерфейс описывает внешний ABI-контракт;
- можно использовать только нужные методы;
- не нужно копировать весь чужой код.

---

## 10. `storage` и `memory`

Это одна из самых важных тем курса.

### `storage`
Ссылка на данные, которые уже живут в state.

Пример:
```solidity
Zombie storage myZombie = zombies[_zombieId];
```

Изменения в `myZombie` меняют состояние контракта.

### `memory`
Временные данные на время вызова.

Пример:
```solidity
uint[] memory result = new uint[](ownerZombieCount[_owner]);
```

Используется для:
- временных массивов;
- параметров;
- возврата данных наружу.

Если на экзамене человек путает `storage` и `memory`, это критическая ошибка.

---

## 11. Модификаторы

Модификатор — это переиспользуемое правило вокруг функции.

Примеры:
```solidity
modifier onlyOwner() {
  require(isOwner());
  _;
}

modifier aboveLevel(uint _level, uint _zombieId) {
  require(zombies[_zombieId].level >= _level);
  _;
}
```

Зачем нужны:
- убрать дублирование `require`;
- сделать ограничения явной частью сигнатуры функции;
- улучшить читаемость политики доступа.

---

## 12. `Ownable`

`Ownable` — паттерн административного контроля контракта.

Что даёт:
- адрес владельца контракта;
- `onlyOwner`;
- перевод ownership;
- отказ от ownership.

Это отдельный слой над доменной ownership-моделью зомби.

Разделение важно:
- owner контракта управляет системными функциями;
- owner зомби владеет конкретным игровым объектом.

---

## 13. Время и кулдауны

Курс использует:
```solidity
uint cooldownTime = 1 days;
uint32(now + cooldownTime)
```

Идея:
- после некоторых действий зомби должен «остыть»;
- это ограничение по времени записывается в `readyTime`.

Проверка готовности:
```solidity
return (_zombie.readyTime <= now);
```

Нужно понимать:
- время — часть business rules;
- это не UI-ограничение, а on-chain правило.

---

## 14. Циклы и выборки для фронтенда

Для получения всех зомби пользователя используется полный проход по массиву:
```solidity
function getZombiesByOwner(address _owner) external view returns(uint[]) {
  uint[] memory result = new uint[](ownerZombieCount[_owner]);
  uint counter = 0;
  for (uint i = 0; i < zombies.length; i++) {
    if (zombieToOwner[i] == _owner) {
      result[counter] = i;
      counter++;
    }
  }
  return result;
}
```

Это важно по двум причинам:
- показывает, как строить read-model для UI;
- показывает стоимость линейных обходов.

На экзамене могут спросить, почему такую функцию лучше держать как `view`.

---

## 15. `payable`, ETH и баланс контракта

Урок 4 вводит денежный поток.

Основные понятия:
- `payable` — функция принимает ETH;
- `msg.value` — сколько ETH пришло с вызовом;
- `address(this).balance` — баланс контракта.

Пример:
```solidity
function levelUp(uint _zombieId) external payable {
  require(msg.value == levelUpFee);
  zombies[_zombieId].level++;
}
```

Вывод средств:
```solidity
function withdraw() external onlyOwner {
  address _owner = owner();
  _owner.transfer(address(this).balance);
}
```

Нужно понимать весь поток:
1. пользователь отправляет ETH;
2. контракт принимает ETH;
3. owner может вывести накопленный баланс.

---

## 16. Боевая логика и изменение состояния

Пример боевой функции:
- выбирается случайный исход;
- изменяются статистики двух зомби;
- при победе повышается уровень и создаётся новый зомби;
- при поражении запускается кулдаун.

Это хороший пример state machine в Solidity:
- входные данные;
- проверка прав;
- вычисление результата;
- согласованное обновление нескольких записей.

На экзамене важно уметь обновлять **несколько зависимых полей** в одном сценарии.

---

## 17. ERC721: токенизация актива

Курс даёт минимальный ERC721-подобный интерфейс:
```solidity
function balanceOf(address _owner) external view returns (uint256);
function ownerOf(uint256 _tokenId) external view returns (address);
function transferFrom(address _from, address _to, uint256 _tokenId) external payable;
function approve(address _approved, uint256 _tokenId) external payable;
```

Что надо понимать:
- `balanceOf` — сколько токенов у владельца;
- `ownerOf` — кто владеет токеном;
- `approve` — делегирование права перевода;
- `transferFrom` — перевод токена.

Ключевой паттерн:
```solidity
function _transfer(address _from, address _to, uint256 _tokenId) private {
  ownerZombieCount[_to] = ownerZombieCount[_to].add(1);
  ownerZombieCount[msg.sender] = ownerZombieCount[msg.sender].sub(1);
  zombieToOwner[_tokenId] = _to;
  emit Transfer(_from, _to, _tokenId);
}
```

Практическая идея:
- один внутренний helper меняет ownership;
- внешние методы только валидируют права и вызывают helper.

---

## 18. Approval-модель

Approval — это временное разрешение другому адресу перевести токен.

Хранение:
```solidity
mapping (uint => address) zombieApprovals;
```

Поток:
1. владелец вызывает `approve(approved, tokenId)`;
2. approved-адрес получает право перевода;
3. `transferFrom` проверяет либо прямого владельца, либо approved-адрес.

Это обязательная тема для экзамена по lesson 5.

---

## 19. SafeMath

В старом Solidity арифметика `uint` могла переполняться/уходить в underflow без ошибок.

Поэтому курс использует `SafeMath`:
```solidity
using SafeMath for uint256;
```

И затем:
```solidity
ownerZombieCount[_to] = ownerZombieCount[_to].add(1);
ownerZombieCount[msg.sender] = ownerZombieCount[msg.sender].sub(1);
```

Что нужно знать:
- `add` защищает от overflow;
- `sub` защищает от underflow;
- это библиотечный паттерн, который был стандартом для старого Solidity.

---

## 20. ABI и фронтенд

ABI — это описание публичного интерфейса контракта для клиента.

Фронтенд использует ABI так:
```javascript
cryptoZombies = new web3js.eth.Contract(cryptoZombiesABI, cryptoZombiesAddress);
```

Без ABI браузер не знает:
- какие методы есть у контракта;
- какие у них параметры;
- какие типы возвращаются.

---

## 21. `call()` и `send()`

Это ключевое различие lesson 6.

### `call()`
Используется для чтения.

Примеры:
```javascript
cryptoZombies.methods.zombies(id).call()
cryptoZombies.methods.getZombiesByOwner(owner).call()
```

Свойства:
- не меняет state;
- не требует транзакции;
- не требует газа как on-chain операция пользователя.

### `send()`
Используется для записи.

Примеры:
```javascript
cryptoZombies.methods.createRandomZombie(name).send({ from: userAccount })
cryptoZombies.methods.levelUp(zombieId).send({ from: userAccount, value: ... })
```

Свойства:
- создаёт транзакцию;
- требует `from`;
- для payable-методов может требовать `value`.

---

## 22. Web3.js и provider

Фронтенд инициализирует Web3 через текущий провайдер браузера:
```javascript
if (typeof web3 !== 'undefined') {
  web3js = new Web3(web3.currentProvider);
}
```

Это обычно MetaMask или другой injected wallet provider.

Нужно понимать:
- контрактный код сам не знает про браузер;
- браузерный клиент связывает UI и контракт;
- provider отвечает за доступ к сети и подпись транзакций.

---

## 23. События во фронтенде

Фронтенд может подписываться на события:
```javascript
cryptoZombies.events.Transfer({ filter: { _to: userAccount } })
  .on("data", function (event) {
    getZombiesByOwner(userAccount).then(displayZombies);
  })
```

Это позволяет:
- не обновлять UI вручную после каждого сценария;
- реагировать на изменения состояния через event stream.

На экзамене важно понимать разницу:
- `receipt` — ответ на конкретную транзакцию;
- `event subscription` — поток событий по контракту.

---

## 24. Самые важные термины трека

### Contract
Код + состояние + адрес в блокчейне.

### Storage
Постоянное on-chain хранилище контракта.

### Memory
Временная область данных на время вызова.

### Struct
Составной тип данных.

### Mapping
Ассоциативное хранилище ключ → значение.

### Event
Лог изменения состояния для внешнего мира.

### Modifier
Переиспользуемое правило вокруг функции.

### Ownable
Паттерн административного контроля контракта.

### Payable
Функция, способная принимать ETH.

### ERC721
Стандарт невзаимозаменяемых токенов (NFT).

### Approval
Разрешение другому адресу перевести токен.

### SafeMath
Библиотека защищённой арифметики.

### ABI
Описание интерфейса контракта для клиентов.

### `call()`
Чтение состояния без транзакции.

### `send()`
Отправка транзакции, которая меняет состояние.

---

## 25. Что человек должен знать после всего трека

Если человек действительно готов к экзамену по урокам 1–6, он должен уметь:
- с нуля написать простой stateful контракт на Solidity;
- хранить сущности в `struct[]` и связывать их с владельцами через `mapping`;
- использовать `require`, modifiers и ownership-паттерны;
- реализовать gameplay-логику с несколькими обновлениями состояния;
- принимать ETH, проверять `msg.value` и выводить баланс;
- реализовать минимальный ERC721 ownership/approval/transfer слой;
- подключить фронтенд к контракту через ABI и Web3.js;
- различать `call` и `send`;
- объяснить ограничения наивной randomness и старой арифметики без SafeMath.

---

## 26. Быстрый список слабых мест, которые чаще всего валят экзамен

- путают owner контракта и owner токена/зомби;
- не различают `storage` и `memory`;
- не могут объяснить, зачем нужны modifiers;
- забывают эмитить события после transfer/approve;
- не различают `call()` и `send()`;
- не понимают, когда нужен `msg.value`;
- не могут восстановить approval flow ERC721;
- пишут логику перевода токена без единой `_transfer` функции;
- не понимают, что `keccak256` в уроке — это не безопасная randomness.

Этот файл лучше читать вместе с `lesson-1.md` … `lesson-6.md` как общий словарь и теоретическую основу.
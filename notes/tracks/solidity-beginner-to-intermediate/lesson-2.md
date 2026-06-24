# Урок 2: Зомби идут в атаку!

- Трек: `Solidity: Beginner to Intermediate Smart Contracts`
- Глав на сайте: **15**
- Элементов урока в repo: **16**
- Русская локаль: **есть**
- Sample code в `cryptozombies-lesson-code`: **есть**

## Что нужно уметь после урока

- добавить ownership-модель через `mapping`;
- ограничить создание одного зомби на адрес;
- разделить данные по `storage`/`memory`;
- связать свой контракт с внешним контрактом через interface;
- реализовать размножение зомби через `feedAndMultiply` и `feedOnKitty`.

## Краткое summary

Урок 2 превращает простой factory-контракт в многопользовательскую систему. Появляется связь `tokenId -> owner`, счётчик зомби на владельца и правило “один стартовый зомби на пользователя”. Это уже модель состояния, которую потом используют уроки 3–5.

Второй крупный блок — интеграция с внешним контрактом. Вместо реального CryptoKitties-контракта на экзамене могут дать любой внешний источник данных; суть не меняется: вы описываете интерфейс, получаете данные через него и используете их в своей доменной логике.

Критично понимать, что урок 2 не просто добавляет новые функции — он меняет архитектуру: данные теперь принадлежат пользователям, а логика размножения должна проверять владельца конкретного зомби.

## Ключевые концепции

- `mapping(uint => address)` и `mapping(address => uint)` для ownership и счётчиков;
- `require(...)` как основной guard для бизнес-правил;
- `storage` для ссылки на запись в state, `memory` — для временных данных;
- `interface` как контракт на вызов внешней системы;
- многозначный return и распаковка только нужного значения.

## Код, который нужно понимать

### Ownership, one-zombie-per-user и создание стартового зомби

```solidity
mapping (uint => address) public zombieToOwner;
    mapping (address => uint) ownerZombieCount;

    function _createZombie(string _name, uint _dna) internal {
        uint id = zombies.push(Zombie(_name, _dna)) - 1;
        zombieToOwner[id] = msg.sender;
        ownerZombieCount[msg.sender]++;
        emit NewZombie(id, _name, _dna);
    }

    function _generateRandomDna(string _str) private view returns (uint) {
        uint rand = uint(keccak256(abi.encodePacked(_str)));
        return rand % dnaModulus;
    }

    function createRandomZombie(string _name) public {
        require(ownerZombieCount[msg.sender] == 0);
        uint randDna = _generateRandomDna(_name);
        randDna = randDna - randDna % 100;
        _createZombie(_name, randDna);
    }

}
```

### Интеграция с CryptoKitties и генерация нового зомби от внешнего DNA

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
contract ZombieFeeding is ZombieFactory {

  address ckAddress = 0x06012c8cf97BEaD5deAe237070F9587f8E7A266d;
  KittyInterface kittyContract = KittyInterface(ckAddress);

  function feedAndMultiply(uint _zombieId, uint _targetDna, string _species) public {
    require(msg.sender == zombieToOwner[_zombieId]);
    Zombie storage myZombie = zombies[_zombieId];
    _targetDna = _targetDna % dnaModulus;
    uint newDna = (myZombie.dna + _targetDna) / 2;
    if (keccak256(abi.encodePacked(_species)) == keccak256(abi.encodePacked("kitty"))) {
      newDna = newDna - newDna % 100 + 99;
    }
    _createZombie("NoName", newDna);
  }

  function feedOnKitty(uint _zombieId, uint _kittyId) public {
    uint kittyDna;
    (,,,,,,,,,kittyDna) = kittyContract.getKitty(_kittyId);
    feedAndMultiply(_zombieId, kittyDna, "kitty");
  }

}
```

## Типовые задания / вопросы на экзамене

- добавить ownership mappings в существующий контракт;
- запретить повторное создание стартового зомби через `require`;
- объявить interface внешнего контракта и вызвать его метод;
- реализовать функцию, которая берёт сущность пользователя, комбинирует данные и создаёт новую сущность.

## Чеклист перед экзаменом

- помню, где и когда увеличивается `ownerZombieCount`;
- могу объяснить, почему `Zombie storage myZombie = zombies[_zombieId];` меняет запись в state;
- понимаю паттерн interface + assignment `kittyContract = KittyInterface(address)`;
- могу без подсказок написать `feedOnKitty` через получение `kittyDna` и вызов `feedAndMultiply`.

## Состав урока

| Порядок | EN title | RU title | EN path | RU path |
|---:|---|---|---|---|
| 1 | Zombies Attack Their Victims | Зомби идут в атаку! | `en/2/00-overview.md` | `ru/2/00-overview.md` |
| 2 | Lesson 2 Overview | Обзор Урока 2 | `en/2/1-overview.md` | `ru/2/1-overview.md` |
| 3 | Mappings and Addresses | Адреса и соответствия | `en/2/2-mappings.md` | `ru/2/2-mappings.md` |
| 4 | Msg.sender | Отправитель | `en/2/3-msgsender.md` | `ru/2/3-msgsender.md` |
| 5 | Require | Требования | `en/2/4-require.md` | `ru/2/4-require.md` |
| 6 | Inheritance | Наследование | `en/2/5-inheritance.md` | `ru/2/5-inheritance.md` |
| 7 | Import | Импорт | `en/2/6-importfiles.md` | `ru/2/6-importfiles.md` |
| 8 | Storage vs Memory (Data location) | Хранилище и память | `en/2/7-storage.md` | `ru/2/7-storage.md` |
| 9 | Zombie DNA | ДНК Зомби | `en/2/8-feedandmultiply2.md` | `ru/2/8-feedandmultiply2.md` |
| 10 | More on Function Visibility | Еще насчет видимости функций | `en/2/9-internalfunctions.md` | `ru/2/9-internalfunctions.md` |
| 11 | What Do Zombies Eat? | Кем питаются зомби? | `en/2/10-interactingcontracts.md` | `ru/2/10-interactingcontracts.md` |
| 12 | Using an Interface | Используем интерфейс | `en/2/11-interactingcontracts2.md` | `ru/2/11-interactingcontracts2.md` |
| 13 | Handling Multiple Return Values | Работа с несколькими возвращаемыми значениями | `en/2/12-multiplereturns.md` | `ru/2/12-multiplereturns.md` |
| 14 | "Bonus: Kitty Genes" | "Бонус: гены котика" | `en/2/13-kittygenes.md` | `ru/2/13-kittygenes.md` |
| 15 | Wrapping It Up | Подведем итог | `en/2/14-wrappingitup.md` | `ru/2/14-wrappingitup.md` |
| 16 | Lesson 2 Complete! | Урок 2 завершен! | `en/2/15-lessoncomplete.md` | `ru/2/15-lessoncomplete.md` |

## Источники

- Английский контент: `en/2/`
- Русский контент: `ru/2/`
- Реестр урока: `en/index.ts`
- Sample code: `../cryptozombies-lesson-code/lesson-2/`

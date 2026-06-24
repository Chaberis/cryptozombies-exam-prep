# Урок 3: Продвинутые концепции Solidity

- Трек: `Solidity: Beginner to Intermediate Smart Contracts`
- Глав на сайте: **14**
- Элементов урока в repo: **15**
- Русская локаль: **есть**
- Sample code в `cryptozombies-lesson-code`: **есть**

## Что нужно уметь после урока

- добавить ownership-администратора контракта;
- ввести reusable modifiers для доступа и валидации;
- расширить модель зомби полями `level` и `readyTime`;
- добавить функции пользовательского уровня и выборки по владельцу;
- закрепить связь между gas-стоимостью, storage и циклическими обходами.

## Краткое summary

Урок 3 делает контракт пригодным для сопровождения: появляется админский слой (`Ownable`), модификаторы для доступа и более богатая модель зомби. Это уже не просто учебный объект, а управляемый домен с политиками доступа и ограничениями по уровню.

Для экзамена особенно важны два паттерна. Первый — `onlyOwner` и другие modifiers как способ не дублировать `require` в каждой функции. Второй — выборка `getZombiesByOwner`, которая показывает, как строить read-модель поверх базового storage, даже если для этого нужен цикл по массиву.

Этот урок также подводит к мысли, что storage дорогой, а публичные view-функции и временные массивы — нормальный способ готовить данные для фронтенда.

## Ключевые концепции

- `Ownable` и событие передачи владения;
- function modifiers как декларирование политики доступа;
- `1 days`, `now`, `uint32(now + cooldownTime)` как работа со временем в старом Solidity;
- `view`-функции и чтение без изменения состояния;
- полный обход массива с фильтрацией по владельцу.

## Код, который нужно понимать

### Административный контракт `Ownable`

```solidity
pragma solidity ^0.4.25;

/**
* @title Ownable
* @dev The Ownable contract has an owner address, and provides basic authorization control
* functions, this simplifies the implementation of "user permissions".
*/
contract Ownable {
  address private _owner;

  event OwnershipTransferred(
    address indexed previousOwner,
    address indexed newOwner
  );

  /**
  * @dev The Ownable constructor sets the original `owner` of the contract to the sender
  * account.
  */
  constructor() internal {
    _owner = msg.sender;
    emit OwnershipTransferred(address(0), _owner);
  }

  /**
  * @return the address of the owner.
  */
  function owner() public view returns(address) {
    return _owner;
  }

  /**
  * @dev Throws if called by any account other than the owner.
  */
  modifier onlyOwner() {
    require(isOwner());
    _;
  }

  /**
  * @return true if `msg.sender` is the owner of the contract.
  */
  function isOwner() public view returns(bool) {
    return msg.sender == _owner;
  }

  /**
  * @dev Allows the current owner to relinquish control of the contract.
  * @notice Renouncing to ownership will leave the contract without an owner.
  * It will not be possible to call the functions with the `onlyOwner`
  * modifier anymore.
  */
  function renounceOwnership() public onlyOwner {
    emit OwnershipTransferred(_owner, address(0));
    _owner = address(0);
  }

  /**
  * @dev Allows the current owner to transfer control of the contract to a newOwner.
  * @param newOwner The address to transfer ownership to.
  */
  function transferOwnership(address newOwner) public onlyOwner {
    _transferOwnership(newOwner);
  }

  /**
  * @dev Transfers control of the contract to a newOwner.
  * @param newOwner The address to transfer ownership to.
  */
  function _transferOwnership(address newOwner) internal {
    require(newOwner != address(0));
    emit OwnershipTransferred(_owner, newOwner);
    _owner = newOwner;
  }
}
```

### Новая модель `Zombie` с level/cooldown и обновлённый `_createZombie`

```solidity
    uint cooldownTime = 1 days;

    struct Zombie {
      string name;
      uint dna;
      uint32 level;
      uint32 readyTime;
    }

    Zombie[] public zombies;

    mapping (uint => address) public zombieToOwner;
    mapping (address => uint) ownerZombieCount;

    function _createZombie(string _name, uint _dna) internal {
        uint id = zombies.push(Zombie(_name, _dna, 1, uint32(now + cooldownTime))) - 1;
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

### Пользовательские ограничения по уровню и выборка зомби владельца

```solidity
  modifier aboveLevel(uint _level, uint _zombieId) {
    require(zombies[_zombieId].level >= _level);
    _;
  }

  function changeName(uint _zombieId, string _newName) external aboveLevel(2, _zombieId) {
    require(msg.sender == zombieToOwner[_zombieId]);
    zombies[_zombieId].name = _newName;
  }

  function changeDna(uint _zombieId, uint _newDna) external aboveLevel(20, _zombieId) {
    require(msg.sender == zombieToOwner[_zombieId]);
    zombies[_zombieId].dna = _newDna;
  }

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

}
```

## Типовые задания / вопросы на экзамене

- подключить `Ownable` и ограничить setter только владельцем;
- написать модификатор по уровню или ownership;
- вернуть массив id по владельцу через цикл и `memory`-массив;
- объяснить, почему такой цикл лучше держать во `view`, а не в mutating-функции.

## Чеклист перед экзаменом

- могу объяснить разницу между владельцем контракта и владельцем зомби;
- могу написать `modifier aboveLevel(...)` и применить его к функции;
- понимаю, зачем `getZombiesByOwner` сначала аллоцирует массив нужного размера;
- помню, что поле `readyTime` хранится прямо в `Zombie`.

## Состав урока

| Порядок | EN title | RU title | EN path | RU path |
|---:|---|---|---|---|
| 1 | Advanced Solidity Concepts | Продвинутые концепции Solidity | `en/3/00-overview.md` | `ru/3/00-overview.md` |
| 2 | Immutability of Contracts | Неизменяемость контрактов | `en/3/01-externaldependencies.md` | `ru/3/01-externaldependencies.md` |
| 3 | Ownable Contracts | Собственные контракты | `en/3/02-ownable.md` | `ru/3/02-ownable.md` |
| 4 | onlyOwner Function Modifier | Модификатор функции onlyOwner (единственный владелец) | `en/3/03-onlyowner.md` | `ru/3/03-onlyowner.md` |
| 5 | Gas | Газ | `en/3/04-gas.md` | `ru/3/04-gas.md` |
| 6 | Time Units | Отрезки времени | `en/3/05-timeunits.md` | `ru/3/05-timeunits.md` |
| 7 | Zombie Cooldowns | Перезарядка Зомби | `en/3/06-zombiecooldowns.md` | `ru/3/06-zombiecooldowns.md` |
| 8 | Public Functions & Security | Открытые функции и безопасность | `en/3/07-zombiecooldowns2.md` | `ru/3/07-zombiecooldowns2.md` |
| 9 | More on Function Modifiers | Еще о модификаторах функций | `en/3/08-functionmodifiers.md` | `ru/3/08-functionmodifiers.md` |
| 10 | Zombie Modifiers | Модификаторы зомби | `en/3/09-zombiemodifiers.md` | `ru/3/09-zombiemodifiers.md` |
| 11 | Saving Gas With 'View' Functions | Экономим газ с помощью функции 'View' | `en/3/10-savinggasview.md` | `ru/3/10-savinggasview.md` |
| 12 | Storage is Expensive | Дорогое место в хранилище | `en/3/11-savinggasstorage.md` | `ru/3/11-savinggasstorage.md` |
| 13 | For Loops | Циклы | `en/3/12-forloops.md` | `ru/3/12-forloops.md` |
| 14 | Wrapping It Up | Подведем итог | `en/3/13-wrappingitup.md` | `ru/3/13-wrappingitup.md` |
| 15 | Lesson 3 Complete! | Урок 3 завершен! | `en/3/14-lessoncomplete.md` | `ru/3/14-lessoncomplete.md` |

## Источники

- Английский контент: `en/3/`
- Русский контент: `ru/3/`
- Реестр урока: `en/index.ts`
- Sample code: `../cryptozombies-lesson-code/lesson-3/`

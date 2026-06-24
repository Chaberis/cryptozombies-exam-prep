# Урок 4: Боевая система Зомби

- Трек: `Solidity: Beginner to Intermediate Smart Contracts`
- Глав на сайте: **13**
- Элементов урока в repo: **14**
- Русская локаль: **есть**
- Sample code в `cryptozombies-lesson-code`: **есть**

## Что нужно уметь после урока

- добавить monetization-поток через `payable` и `withdraw`;
- поднимать уровень зомби за оплату;
- реализовать боевую механику и обновление win/loss счётчиков;
- понять ограничения on-chain randomness и паттерн cooldown после действия.

## Краткое summary

Урок 4 переводит учебный контракт в игровую экономику. Теперь часть функций получает ETH, у контракта появляется баланс, а владелец может его вывести. В экзаменационных задачах это обычно проверяется через `payable`, `msg.value`, `address(this).balance` и авторизацию на вывод.

Второй блок урока — battle loop. В нём соединяются почти все предыдущие идеи: ownership-проверка, модификаторы, чтение/запись в `storage`, случайность через `keccak256`, изменение нескольких полей и вызов уже существующей логики размножения.

Это один из самых полезных уроков для продакшн-формата: здесь появляется state machine, где действие меняет сразу несколько инвариантов домена.

## Ключевые концепции

- `payable`-функции, `msg.value`, `address(this).balance`;
- `onlyOwner` на системные функции вроде `withdraw` и `setLevelUpFee`;
- обновление нескольких полей одной сущности в одном действии;
- наивная on-chain randomness и её ограничения;
- cooldown как бизнес-ограничение после действия.

## Код, который нужно понимать

### Монетизация: уровень за оплату и вывод средств владельцу

```solidity
  uint levelUpFee = 0.001 ether;

  modifier aboveLevel(uint _level, uint _zombieId) {
    require(zombies[_zombieId].level >= _level);
    _;
  }

  function withdraw() external onlyOwner {
    address _owner = owner();
    _owner.transfer(address(this).balance);
  }

  function setLevelUpFee(uint _fee) external onlyOwner {
    levelUpFee = _fee;
  }

  function levelUp(uint _zombieId) external payable {
    require(msg.value == levelUpFee);
    zombies[_zombieId].level++;
  }

  function changeName(uint _zombieId, string _newName) external aboveLevel(2, _zombieId) ownerOf(_zombieId) {
    zombies[_zombieId].name = _newName;
  }

  function changeDna(uint _zombieId, uint _newDna) external aboveLevel(20, _zombieId) ownerOf(_zombieId) {
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

### Боевая механика и обновление win/loss/level

```solidity
  uint randNonce = 0;
  uint attackVictoryProbability = 70;

  function randMod(uint _modulus) internal returns(uint) {
    randNonce++;
    return uint(keccak256(abi.encodePacked(now, msg.sender, randNonce))) % _modulus;
  }

  function attack(uint _zombieId, uint _targetId) external ownerOf(_zombieId) {
    Zombie storage myZombie = zombies[_zombieId];
    Zombie storage enemyZombie = zombies[_targetId];
    uint rand = randMod(100);
    if (rand <= attackVictoryProbability) {
      myZombie.winCount++;
      myZombie.level++;
      enemyZombie.lossCount++;
      feedAndMultiply(_zombieId, enemyZombie.dna, "zombie");
    } else {
      myZombie.lossCount++;
      enemyZombie.winCount++;
      _triggerCooldown(myZombie);
    }
  }
}
```

## Типовые задания / вопросы на экзамене

- сделать функцию `levelUp(uint)` payable с проверкой точной цены;
- добавить вывод средств только владельцу;
- реализовать боевую функцию с вероятностью победы и двумя ветками обновления состояния;
- объяснить, почему `randMod` нельзя считать безопасной randomness для production.

## Чеклист перед экзаменом

- помню, что `withdraw` должен брать средства из `address(this).balance`;
- понимаю ветки победа/поражение и какие поля меняются в каждой;
- могу без подсказок написать `msg.value == levelUpFee`;
- понимаю, зачем после поражения вызывается `_triggerCooldown`.

## Состав урока

| Порядок | EN title | RU title | EN path | RU path |
|---:|---|---|---|---|
| 1 | Zombie Battle System | Боевая система Зомби | `en/4/00-overview.md` | `ru/4/00-overview.md` |
| 2 | Payable | Платные опции | `en/4/payable.md` | `ru/4/payable.md` |
| 3 | Withdraws | Вывод средств | `en/4/withdraw.md` | `ru/4/withdraw.md` |
| 4 | Zombie Battles | Битвы Зомби | `en/4/battle-01.md` | `ru/4/battle-01.md` |
| 5 | Random Numbers | Случайные числа | `en/4/battle-02.md` | `ru/4/battle-02.md` |
| 6 | Zombie Fightin' | Зомби сражаются | `en/4/battle-03.md` | `ru/4/battle-03.md` |
| 7 | Refactoring Common Logic | Общая логика рефакторинга | `en/4/battle-04.md` | `ru/4/battle-04.md` |
| 8 | More Refactoring | Еще о рефакторинге | `en/4/battle-05.md` | `ru/4/battle-05.md` |
| 9 | Back to Attack! | Снова в атаку! | `en/4/battle-06.md` | `ru/4/battle-06.md` |
| 10 | Zombie Wins and Losses | Победы и поражения зомби | `en/4/battle-07.md` | `ru/4/battle-07.md` |
| 11 | Zombie Victory 😄 | Победа Зомби! 😄 | `en/4/battle-08.md` | `ru/4/battle-08.md` |
| 12 | Zombie Loss 😞 | Проигрыш Зомби 😞 | `en/4/battle-09.md` | `ru/4/battle-09.md` |
| 13 | Wrapping It Up | Подведем итог | `en/4/wrappingitup.md` | `ru/4/wrappingitup.md` |
| 14 | Lesson 4 Complete! | Урок 4 завершен! | `en/4/lessoncomplete.md` | `ru/4/lessoncomplete.md` |

## Источники

- Английский контент: `en/4/`
- Русский контент: `ru/4/`
- Реестр урока: `en/index.ts`
- Sample code: `../cryptozombies-lesson-code/lesson-4/`

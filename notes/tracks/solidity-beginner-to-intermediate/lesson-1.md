# Урок 1: Фабрика Зомби

- Трек: `Solidity: Beginner to Intermediate Smart Contracts`
- Глав на сайте: **15**
- Элементов урока в repo: **16**
- Русская локаль: **есть**
- Sample code в `cryptozombies-lesson-code`: **есть**

## Что нужно уметь после урока

- написать первый рабочий смарт-контракт `ZombieFactory`;
- понять базовые типы Solidity: `uint`, `string`, `struct`, массивы;
- уметь объявлять `event`, приватные и публичные функции;
- создавать детерминированную “случайность” через `keccak256` и нормализовывать результат.

## Краткое summary

Урок 1 строит минимальный on-chain домен: есть сущность `Zombie`, хранилище `zombies`, фабрика для создания новых объектов и событие `NewZombie`. Это база всего курса: дальше почти все уроки только расширяют этот контракт.

Если на экзамене нужно быстро показать базовое владение Solidity, этот урок — каркас ответа: объявить контракт, описать `struct`, сохранить объекты в массив, сделать внутренний helper для создания записи и внешнюю функцию для пользовательского вызова.

Главная практическая мысль урока: публичная функция вызывает внутренние helpers, а важные изменения состояния сопровождаются `event`, чтобы фронтенд и индексаторы могли реагировать на изменения без прямого сканирования storage.

## Ключевые концепции

- контракт как контейнер состояния и функций;
- `struct` + динамический массив как простая модель хранения;
- `private`/`public` и разделение API на внешние и внутренние функции;
- `event` как основной механизм уведомления внешнего мира;
- `keccak256(abi.encodePacked(...))` для получения псевдослучайного числа.

## Код, который нужно понимать

### Финальный каркас `ZombieFactory`

```solidity
pragma solidity ^0.4.25;

contract ZombieFactory {

    event NewZombie(uint zombieId, string name, uint dna);

    uint dnaDigits = 16;
    uint dnaModulus = 10 ** dnaDigits;

    struct Zombie {
        string name;
        uint dna;
    }

    Zombie[] public zombies;

    function _createZombie(string _name, uint _dna) private {
        uint id = zombies.push(Zombie(_name, _dna)) - 1;
        emit NewZombie(id, _name, _dna);
    }

    function _generateRandomDna(string _str) private view returns (uint) {
        uint rand = uint(keccak256(abi.encodePacked(_str)));
        return rand % dnaModulus;
    }

    function createRandomZombie(string _name) public {
        uint randDna = _generateRandomDna(_name);
        _createZombie(_name, randDna);
    }

}
```

## Типовые задания / вопросы на экзамене

- объявить `struct` и массив сущностей;
- написать helper вида `_createEntity(...)` с пушем в массив и `emit` события;
- сгенерировать число через `keccak256` и ограничить его модулем;
- объяснить разницу между `public` и `private` функциями.

## Чеклист перед экзаменом

- понимаю, зачем `_createZombie` сделан не публичным;
- могу объяснить, почему событие эмитится после изменения состояния;
- могу написать функцию генерации DNA через `keccak256` без подсказок;
- помню, что lesson-финал — это один контракт, без ownership и без mappings.

## Состав урока

| Порядок | EN title | RU title | EN path | RU path |
|---:|---|---|---|---|
| 1 | Making the Zombie Factory | Фабрика Зомби | `en/1/00-overview.md` | `ru/1/00-overview.md` |
| 2 | Lesson Overview | Обзор урока | `en/1/lessonoverview.md` | `ru/1/lessonoverview.md` |
| 3 | "Contracts" | Контракты | `en/1/contracts.md` | `ru/1/contracts.md` |
| 4 | State Variables & Integers | Переменные состояния и целые числа | `en/1/datatypes.md` | `ru/1/datatypes.md` |
| 5 | Math Operations | Математические операции | `en/1/math.md` | `ru/1/math.md` |
| 6 | Structs | Структуры | `en/1/structs.md` | `ru/1/structs.md` |
| 7 | Arrays | Массивы | `en/1/arrays.md` | `ru/1/arrays.md` |
| 8 | Function Declarations | Как задавать функции | `en/1/functions.md` | `ru/1/functions.md` |
| 9 | Working With Structs and Arrays | Как работать со структурами и массивами | `en/1/arraysstructs2.md` | `ru/1/arraysstructs2.md` |
| 10 | Private / Public Functions | Закрытые и открытые функции | `en/1/functions2.md` | `ru/1/functions2.md` |
| 11 | More on Functions | Еще о функциях | `en/1/functions3.md` | `ru/1/functions3.md` |
| 12 | Keccak256 and Typecasting | Keccak256 и преобразование типов данных | `en/1/keccak256.md` | `ru/1/keccak256.md` |
| 13 | Putting It Together | Соберем все вместе | `en/1/puttingittogether.md` | `ru/1/puttingittogether.md` |
| 14 | Events | События | `en/1/events.md` | `ru/1/events.md` |
| 15 | Web3.js | Web3.js | `en/1/web3js.md` | `ru/1/web3js.md` |
| 16 | Lesson 1 Complete! | Урок 1 завершен! | `en/1/lessoncomplete.md` | `ru/1/lessoncomplete.md` |

## Источники

- Английский контент: `en/1/`
- Русский контент: `ru/1/`
- Реестр урока: `en/index.ts`
- Sample code: `../cryptozombies-lesson-code/lesson-1/`

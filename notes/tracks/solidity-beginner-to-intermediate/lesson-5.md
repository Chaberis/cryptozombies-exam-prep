# Урок 5: ERC721 & Crypto-Collectibles

- Трек: `Solidity: Beginner to Intermediate Smart Contracts`
- Глав на сайте: **15**
- Элементов урока в repo: **16**
- Русская локаль: **есть**
- Sample code в `cryptozombies-lesson-code`: **есть**

## Что нужно уметь после урока

- обернуть зомби в ERC721-подобную модель;
- реализовать `balanceOf`, `ownerOf`, `approve`, `transferFrom`;
- добавить отдельный mapping approvals;
- подключить SafeMath для безопасной арифметики счётчиков и уровней;
- понять, как комментировать и документировать смарт-контракты.

## Краткое summary

Урок 5 превращает зомби из внутренних записей контракта в торгуемые активы. С точки зрения экзамена это критичный переход: теперь нужно думать не просто об ownership внутри приложения, а о стандартизированном токен-интерфейсе.

Главная инженерная идея урока — выделить ERC721 API отдельно от доменной логики. Сам `ZombieOwnership` наследует всё игровое поведение и поверх него добавляет токенные операции: баланс, владельца, approvals и transfer.

Вторая важная идея — безопасная арифметика. Даже если современный Solidity уже умеет checked arithmetic по умолчанию, экзамен по этому курсу почти наверняка ожидает, что вы понимаете паттерн `using SafeMath for uint256` и умеете использовать `add/sub` при изменении балансов.

## Ключевые концепции

- минимальный контракт-интерфейс ERC721;
- разделение `approve` и `transferFrom`;
- внутренний helper `_transfer(...)` для устранения дублирования;
- `using SafeMath for uint256` и библиотечные вызовы `add/sub`;
- Natspec/комментарии как часть качества контракта.

## Код, который нужно понимать

### Минимальный интерфейс ERC721 для урока

```solidity
pragma solidity ^0.4.25;

contract ERC721 {
  event Transfer(address indexed _from, address indexed _to, uint256 indexed _tokenId);
  event Approval(address indexed _owner, address indexed _approved, uint256 indexed _tokenId);

  function balanceOf(address _owner) external view returns (uint256);
  function ownerOf(uint256 _tokenId) external view returns (address);
  function transferFrom(address _from, address _to, uint256 _tokenId) external payable;
  function approve(address _approved, uint256 _tokenId) external payable;
}
```

### Базовая библиотека SafeMath

```solidity
library SafeMath {

  /**
  * @dev Multiplies two numbers, throws on overflow.
  */
  function mul(uint256 a, uint256 b) internal pure returns (uint256) {
    if (a == 0) {
      return 0;
    }
    uint256 c = a * b;
    assert(c / a == b);
    return c;
  }

  /**
  * @dev Integer division of two numbers, truncating the quotient.
  */
  function div(uint256 a, uint256 b) internal pure returns (uint256) {
    // assert(b > 0); // Solidity automatically throws when dividing by 0
    uint256 c = a / b;
    // assert(a == b * c + a % b); // There is no case in which this doesn't hold
    return c;
  }

  /**
  * @dev Subtracts two numbers, throws on overflow (i.e. if subtrahend is greater than minuend).
  */
  function sub(uint256 a, uint256 b) internal pure returns (uint256) {
    assert(b <= a);
    return a - b;
  }

  /**
  * @dev Adds two numbers, throws on overflow.
  */
  function add(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 c = a + b;
    assert(c >= a);
    return c;
  }
}
```

### Реализация ownership, approve и transfer для зомби

```solidity
contract ZombieOwnership is ZombieAttack, ERC721 {

  using SafeMath for uint256;

  mapping (uint => address) zombieApprovals;

  function balanceOf(address _owner) external view returns (uint256) {
    return ownerZombieCount[_owner];
  }

  function ownerOf(uint256 _tokenId) external view returns (address) {
    return zombieToOwner[_tokenId];
  }

  function _transfer(address _from, address _to, uint256 _tokenId) private {
    ownerZombieCount[_to] = ownerZombieCount[_to].add(1);
    ownerZombieCount[msg.sender] = ownerZombieCount[msg.sender].sub(1);
    zombieToOwner[_tokenId] = _to;
    emit Transfer(_from, _to, _tokenId);
  }

  function transferFrom(address _from, address _to, uint256 _tokenId) external payable {
      require (zombieToOwner[_tokenId] == msg.sender || zombieApprovals[_tokenId] == msg.sender);
      _transfer(_from, _to, _tokenId);
    }

  function approve(address _approved, uint256 _tokenId) external payable onlyOwnerOf(_tokenId) {
      zombieApprovals[_tokenId] = _approved;
      emit Approval(msg.sender, _approved, _tokenId);
    }

}
```

## Типовые задания / вопросы на экзамене

- описать минимальный ERC721 interface с событиями и методами;
- реализовать approvals и single-step / two-step transfer;
- перенести арифметику балансов на SafeMath;
- объяснить, зачем выносить логику перевода в `_transfer`.

## Чеклист перед экзаменом

- помню различие между владельцем токена и approved-адресом;
- могу написать `emit Transfer(...)` и `emit Approval(...)` в правильных местах;
- понимаю, что `balanceOf` и `ownerOf` должны быть `view`;
- могу без подсказок подключить `using SafeMath for uint256;`.

## Состав урока

| Порядок | EN title | RU title | EN path | RU path |
|---:|---|---|---|---|
| 1 | ERC721 & Crypto-Collectibles | ERC721 & Крипто-Коллекционирование | `en/5/00-overview.md` | `ru/5/00-overview.md` |
| 2 | Tokens on Ethereum | Токены на Ethereum | `en/5/01-erc721-1.md` | `ru/5/01-erc721-1.md` |
| 3 | ERC721 Standard, Multiple Inheritance | ERC721 Стандарт, Mножественное наследование | `en/5/02-erc721-2.md` | `ru/5/02-erc721-2.md` |
| 4 | balanceOf & ownerOf | balanceOf & ownerOf | `en/5/03-erc721-3.md` | `ru/5/03-erc721-3.md` |
| 5 | Refactoring | Рефакторинг | `en/5/04-erc721-4.md` | `ru/5/04-erc721-4.md` |
| 6 | "ERC721: Transfer Logic" | "ERC721: Логика перевода" | `en/5/05-erc721-5.md` | `ru/5/05-erc721-5.md` |
| 7 | "ERC721: Transfer Cont'd" | "ERC721: Перевод. Продолжение..." | `en/5/06-erc721-6.md` | `ru/5/06-erc721-6.md` |
| 8 | "ERC721: Approve" | "ERC721: Approve" | `en/5/07-erc721-7.md` | `ru/5/07-erc721-7.md` |
| 9 | "ERC721: Approve" | "ERC721: Approve" | `en/5/08-erc721-8.md` | `ru/5/08-erc721-8.md` |
| 10 | Preventing Overflows | Предотвращение переполнений | `en/5/09-safemath-1.md` | `ru/5/09-safemath-1.md` |
| 11 | SafeMath Part 2 | SafeMath, Часть 2 | `en/5/10-safemath-2.md` | `ru/5/10-safemath-2.md` |
| 12 | SafeMath Part 3 | SafeMath, Часть 3 | `en/5/11-safemath-3.md` | `ru/5/11-safemath-3.md` |
| 13 | SafeMath Part 4 | SafeMath, Часть 4 | `en/5/12-safemath-4.md` | `ru/5/12-safemath-4.md` |
| 14 | Comments | Комментарии | `en/5/13-comments.md` | `ru/5/13-comments.md` |
| 15 | Wrapping It Up | Завершение | `en/5/14-wrappingitup.md` | `ru/5/14-wrappingitup.md` |
| 16 | Lesson 5 Complete! | Урок 5 Завершен! | `en/5/15-lessoncomplete.md` | `ru/5/15-lessoncomplete.md` |

## Источники

- Английский контент: `en/5/`
- Русский контент: `ru/5/`
- Реестр урока: `en/index.ts`
- Sample code: `../cryptozombies-lesson-code/lesson-5/`

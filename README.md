# CryptoZombies Exam Prep

Сводный репозиторий для подготовки команды к экзамену по курсу CryptoZombies.

## Что внутри

### 1. Конспекты и шпаргалки
Папка `notes/tracks/` содержит наши экзамен-ориентированные материалы по трекам:

- `solidity-beginner-to-intermediate/`
- `advanced-solidity-get-in-depth-knowledge/`
- `chainlink-decentralized-oracles/`
- `beyond-ethereum-explore-the-blockchain-ecosystem/`
- `tron-decentralize-the-web/`

Внутри каждого трека:
- `README.md` — обзор пакета
- `lesson-*.md` — конспекты по урокам
- `theory.md` — общая теория по треку
- `lessons-*.json` — структурированные данные

### 2. Исходные материалы из `cryptozombie-lessons`
Папка `sources/cryptozombie-lessons/` содержит выжимку исходного контента, который реально нужен для подготовки:

- `en/1-6`
- `en/10,11,14,15,16,17,18,19,20`
- `ru/1-5`
- `en/index.ts`, `en/index.json`
- `ru/index.ts`, `ru/index.json`
- root `README.md` и `index.ts`

Это позволяет:
- читать оригинальные главы;
- сверять answer blocks и starting code;
- проверять chapter order и metadata.

### 3. Исходный sample code из `cryptozombies-lesson-code`
Папка `sources/cryptozombies-lesson-code/` содержит нужные chapter snapshots:

- `lesson-1` ... `lesson-6`
- `lesson-10`
- `lesson-11`

Это reference-код для ранних Solidity/Truffle уроков, где sample repo действительно покрывает материал.

### 4. Манифесты
Папка `manifests/` описывает, что именно включено в этот репозиторий и почему.

## Рекомендуемый порядок чтения

1. `notes/tracks/solidity-beginner-to-intermediate/`
2. `notes/tracks/advanced-solidity-get-in-depth-knowledge/`
3. `notes/tracks/chainlink-decentralized-oracles/`
4. `notes/tracks/beyond-ethereum-explore-the-blockchain-ecosystem/`
5. `notes/tracks/tron-decentralize-the-web/`

## Ограничения

- Репозиторий намеренно не тащит весь исходный `cryptozombie-lessons` и весь `cryptozombies-lesson-code`; включены только материалы, нужные для уже разобранных треков.
- Для уроков `14-20` sample code в `cryptozombies-lesson-code` отсутствует или недостаточен, поэтому опора идёт на `answer` блоки из `cryptozombie-lessons`.
- Русская локаль в исходном репозитории покрывает только ранние Solidity-уроки; для поздних треков основной источник — английский контент.

## Источники

- `cryptozombie-lessons`
- `cryptozombies-lesson-code`

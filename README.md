# Сайт для расчёта налогов на банковские вклады

Сайт: https://nalogi.alkatrazstudio.net

Основано на статье от ФНС РФ:
https://www.nalog.ru/rn77/news/activities_fts/10007632/


## Подготовка к локальному запуску

```sh
git clone https://github.com/alkatrazstudio/nalogi.git
cd nalogi
yarn
```


## Запуск локальной копии для отладки

```sh
yarn dev
```

Далее, локальный сайт можно открыть по адресу http://localhost:1234/.


## Сборка

```sh
yarn build
```

Итоговая сборка будет находиться в папке `dist`.
Файлы в этой папке пригодны для выгрузки на статический хостинг.


## Лицензия

GNU AGPL v3: https://www.gnu.org/licenses/agpl-3.0.html

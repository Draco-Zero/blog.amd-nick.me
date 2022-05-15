---
layout: post
title: График онлайна человека в Telegram
date: '2020-12-23 10:37:00'
tags:
- programming
---

Есть у меня друг-наркоман. Принимает дозу соц. сетей внутриглазно каждую секунду, пока не спит или работает.

## Симптомы

- "Это по работе"
- "Да не сижу я в соц. сетях так часто"
- "Нет времени" в ответ на "Давай чет делать"
- "Я стал сидеть реже, чем раньше" (заменив одну сеть на другую)
- "И вообще вы все врети, статистика не верна"

Конечно, вылечить такого пациента указанием ему на проблему невозможно, но моя жопа горела вместо его, когда я видел как уходит его жизнь и все же решил показать наглядно, как это выглядит

* * *

# Инструментарий

- Python 3 + [telegram-tracker](https://github.com/gentoo-root/telegram-tracker) на GitHub для сбора данных
- СмекалОчка или знание какого-то ЯП, чтобы превратить полученные данные в .csv формат
- Google Data Studio ([инфо](https://t.me/uFeed/14)) для визуализации (бэзплатно!1)

# Сбор данных
<!--kg-card-begin: markdown-->

Копируем репозиторий для отслеживания
`git clone https://github.com/gentoo-root/telegram-tracker.git`

<!--kg-card-end: markdown-->

В папке settings создаем файл keys.py и вставляем в него

<!--kg-card-begin: markdown-->

    API_ID = 123456
    API_HASH = '1234567890abcdef1234567890abcdef'

<!--kg-card-end: markdown-->

Заменить значения на данные [отсюда](https://my.telegram.org/apps) (нужно создать приложение)

<!--kg-card-begin: markdown-->

В главной папке репозитория выполняем
`python3 -m track '+380991234567'`
Вас попросит ввести еще свой номер, код подтверждения из Telegram и пароль двухфакторной аутенфикации (если включена). Я параноик, поэтому делал все с фейкового аккаунта

<!--kg-card-end: markdown--><!--kg-card-begin: markdown-->

Теперь ждем денек-второй, пока данные наберутся. Выглядеть будут как-то так:

    ~2020-12-21 @ 06:50:07: User went online.
    =2020-12-21 @ 06:50:21: User went offline.
    =2020-12-21 @ 06:51:48: User went offline after being online for short time.

<!--kg-card-end: markdown-->
# Превращение в csv

Короче, тут нужен кодераст, чтобы написать конвертер этих логов с человеческого языка в формат бездушной машины. Если эта статья кому-то нужна, пните меня по [контактным данным]( __GHOST_URL__ /about) и я напишу для вас код или скиньте свой, я добавлю сюда

Я гавнокодер и [написал](https://gist.github.com/AMD-NICK/ea2bd29d9db782fadd456865e4ea770c) свой конвертер на модифицированном Lua прямо внутри одной игры, так что он будет тебе мало чем полезен, но держи:

<figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://s3.blog.amd-nick.me/2020/12/image.png" class="kg-image" alt loading="lazy"><figcaption>чем воняет?</figcaption></img></figure>

Выглядеть сформированный лог будет вот так:

<figure class="kg-card kg-image-card"><img src="https://s3.blog.amd-nick.me/2020/12/image-1.png" class="kg-image" alt loading="lazy"></img></figure>
# Визуализация

Заходим на [сайт GDS](https://datastudio.google.com), создаем новый отчет, выбираем источник данных "Загрузка файла", заливаем наш csv файл

<figure class="kg-card kg-image-card"><img src="https://s3.blog.amd-nick.me/2020/12/image-2.png" class="kg-image" alt loading="lazy"></img></figure>

Дальше добавляем диаграммы

<figure class="kg-card kg-image-card"><img src="https://s3.blog.amd-nick.me/2020/12/image-3.png" class="kg-image" alt loading="lazy"></img></figure>

Настраиваем

### Таблица с тепловой картой

- Категория " **Параметр**": кликаем на календарик возле даты, выбираем Тип -\> Дата и время -\> Час. Важно сделать это именно в "Параметр", а не "Параметр: диапазон дат"
- **Показатель** : session\_time. Название: входов в сеть, Агрегирование: количество. Еще "Добавить показатель" снова session\_time. Название: Время в сети, Агрегирование: сумма, Тип: число -\> Продолжительность

### Столбчатая диаграмма

- **Параметр** : снова делаем Час
- **Показатель** : сумма session\_time, Тип: продолжительность
- **Сортировка** : по возрастанию date\_start.

В "Стиль" этой диаграммы надо указать 24 столбца

### Диаграмма динамических рядов

- Параметр: снова Час
- Показатель: сумма session\_time, название "Накопительно часов", тип Продолжительность, расчет скользящего показателя: Суммирование

### Сводка

Вроде ничего сложного после предыдущих пунктов. Просто в агреггировании для session\_time выбираем "Медиана" и "Максимальное значение"

Получится вот такое чудо:

<figure class="kg-card kg-image-card kg-card-hascaption"><img src="https://s3.blog.amd-nick.me/2020/12/image-4.png" class="kg-image" alt loading="lazy"><figcaption>Сортировку по часам приходится делать в режиме просмотра отчета вручную, потому что для самой таблицы почему-то у меня не получается прописать сортировку по полю даты </figcaption></img></figure>
# Улучшения

В моем варианте анализируется один единственный день. В GDS можно добавить выборку по датам и сделать, чтобы анализировать можно было хоть целый месяц, хоть отдельные дни

Еще в таблицу можно добавить вычислительные поля, которые покажут именно периодичность входов в конкретный час и среднее время оффлайн. Если кому надо - пните меня, расскажу как

Изначально идеей было сделать еще один график временную линию, где можно увидеть наглядно отрезками времени, когда человек был в сети, но он бы не поместился в мой экран, а горизонтально скроллить в GDS вроде нельзя

### UPD 2021-08-06

Нашел еще одну штуку на питоне, которая позволяет следить за чьим-то онлайном более юзер-френдли:

<figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href="https://github.com/Forichok/TelegramOnlineSpy"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">GitHub - Forichok/TelegramOnlineSpy: Simple telegram online spy logger bot</div>
<div class="kg-bookmark-description">Simple telegram online spy logger bot. Contribute to Forichok/TelegramOnlineSpy development by creating an account on GitHub.</div>
<div class="kg-bookmark-metadata">
<img class="kg-bookmark-icon" src="https://github.githubassets.com/favicons/favicon.svg"><span class="kg-bookmark-author">GitHub</span><span class="kg-bookmark-publisher">Forichok</span>
</img></div>
</div>
<div class="kg-bookmark-thumbnail"><img src="https://opengraph.githubassets.com/8b090998f94350cb31f0436aef962e5dfd2496a631a1eb8bdeaba38880e171a1/Forichok/TelegramOnlineSpy"></img></div></a></figure>
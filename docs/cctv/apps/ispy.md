# iSpy Agent

![ispy agent main view](https://i.imgur.com/dDUsb3F.png)

iSpy это бесплатный Self-Hosted сервис **на стероидах**, который подключается к камерам (в моем случае по [rtsp](../connection.md)) и может очень по-разному реагировать на происходящее на них.

Например:
- Присылать email, когда какая-то камера стала недоступна
- Воспроизводить звук в доме, когда замечает человека с пиццой в руках (или просто человека, машину, вазонный горшок и что угодно другое)
- Генерировать таймлапс записи, чтобы бегло посмотреть, что произошло за день
- Удобно просматривать сгенерированные фото и видео с камеры с любого устройства, где есть браузер
- Заливать видео на FTP/DropBox, куда угодно еще по определенным правилам
- Определять лица людей и понимать, кого именно она сейчас видит
- Можно даже в VR смотреть на камеры

Все, что я перечислил выше можно ооочень тонко сконфигурировать, вплоть до "если сегодня не принесли пиццу, то запустить ядерную ракету", если вдруг у вас такая где-то завалялась

:::caution
Это очень мощный инструмент и он соответственно сложен в настройке. Когда я впервые с ним столкнулся, я тут же его удалил. Будь умным, не будь как я
:::

## Системные требования

У меня Linux Server, арендован в [OVH](https://www.ovhcloud.com/en/bare-metal/). На борту 2 TB места, 16 ОЗУ, Intel(R) Core(TM) i5-2400 CPU @ 4 * 3.10GHz

3 камеры у меня сжирало 50% CPU и до 4 ГБ ОЗУ ([оптимизировать можно](https://www.ispyconnect.com/userguide.aspx)). iSpy обрабатывает Substream с камер (24 FPS, 640*480, bitrate 1024, запись событий ведет в 1080P).

## Установка iSpy, как у меня

:::tip
Сначала я установил iSpy на свой Windows ПК, чтобы потыкать в него палкой и понять, нужен ли он мне. **Рекомендую сделать так же**
:::

iSpy работает в [Docker контейнере](https://github.com/doitandbedone/ispyagentdvr-docker) на выделенном сервере, а домен и логин+пароль повесил на него, добавив Traefik + Basic Auth Middleware. Вам не обязательно с этим возиться, можно просто установить на сервер Windows, затем там запустить iSpy, но мне было проще так, как я сделал и именно об этом я буду писать

**Если вы еще не знаете, что такое Docker и не готовы потратить вечерок-второй, чтобы базово в нем разобраться, то установите как-нибудь иначе и переходите к части "Настройка iSpy"**

### Traefik

Использую для того, чтобы повесить iSpy на домен и закрыть за Basic Auth авторизацией (безопасность). P.S. Пароль на панель без Traefik вроде можно указать в `Server (сверху слева) > Settings > Local Settings (сверху справа модалки) > Username, Password`, но я не пробовал.

Traefik работает отдельно от iSpy, так как проксирует еще несколько других сайтов, которые динамично подключаются подобно тому, как я ниже подключу iSpy

<details>
  <summary>Код для docker-compose.yml</summary>

```yaml title="docker-compose.yml"
version: '3'

services:
  traefik:
    image: traefik:v2.6
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./acme.json:/acme.json
    networks:
      - proxy
    command:
        #- "--log.level=DEBUG"
        - "--api"
        - "--providers.docker=true"
        - "--providers.docker.exposedbydefault=false"
        - "--providers.docker.network=proxy"
        - "--entrypoints.web.address=:80"
        - "--entrypoints.web-secure.address=:443"
        - "--serverstransport.insecureskipverify=true"
        - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
        - "--certificatesresolvers.mytlschallenge.acme.email=email@example.com" # change me
        - "--certificatesresolvers.mytlschallenge.acme.storage=/acme.json"

    labels:
        - "traefik.enable=true"

        #Traefik dashboard config
        - "traefik.http.routers.traefik.rule=Host(`dash.example.com`)" # change me
        - "traefik.http.routers.traefik.service=api@internal"
        - "traefik.http.routers.traefik.entrypoints=web-secure"
        - "traefik.http.routers.traefik.tls.certresolver=mytlschallenge"

        #Middleware
        - "traefik.http.middlewares.https-redirect.redirectscheme.scheme=https"
        - "traefik.http.middlewares.traefik-auth.basicauth.users=login:$$2a$05$$9kZMQL0neCA2TogroG8WbO3yObDtHbo6BRwflTnGtQy6vKJCvMDWe" # login=login, pass=123456
        - "traefik.http.routers.traefik.middlewares=traefik-auth"

        #global redirect to https
        - "traefik.http.routers.http-catchall.rule=hostregexp(`{host:.+}`)"
        - "traefik.http.routers.http-catchall.entrypoints=web"
        - "traefik.http.routers.http-catchall.middlewares=https-redirect"


networks:
  proxy:
    name: proxy
    # driver: bridge

```
</details>

До запуска нужно создать пустой файл `acme.json` и **обязательно** сделать ему `chmod 600 acme.json`. Если забыть указать chmod или создать файл, то SSL работать не будет (!)

Не забудьте также поправить email и часть конфига про Dashboard (или удалите его)

### iSpy Agent

Код для docker-compose.yml я публиковал тут ([клик](https://github.com/doitandbedone/ispyagentdvr-docker/issues/357#issuecomment-1206978330))

Вместе с iSpy в docker-compose устанавливается DeepStack AI (микросервис, который будет детектить объекты и людей на наших записях)

Не забудьте поправить в файле домен и логин/пароль от [Basic Auth](https://doc.traefik.io/traefik/middlewares/http/basicauth/) авторизации. Про генерацию хэша пароля я писал [тут](https://t.me/traefik_ru/7900).

Сначала запускаете Traefik, потом iSpy



## Настройка iSpy и камер

:::tip Это сэкономит вам гору времени
У iSpy почти к любой чепухе есть **хорошая** документация. Я настоятельно рекомендую не брезгать и почаще нажимать знак вопроса сверху модалок. Я ленился и потерял из-за этого много времени, тыкая наугад
:::

При входе в iSpy выбираем English. Переводы ужасно кривые уровня Log = Бревно

### ➡️ Настройка iSpy

Пока что камер у нас нет, но прежде, чем мы их добавим, заранее настроим DeepStack AI (ИИ, который будет искать объекты на видео), а также парочку других полезных настроек

Сверху слева в уголку, после замочка есть кнопка `Server Menu`. Ищем там Settings

#### General

Ставим любой Name, 90 CPU

#### Intelligence

Включение ИИ (Deepstack AI). Вписываем тут адрес https://ваш_домен.com, Endpoint `/v1/vision/detection`, API_KEY, если такой установили в docker-compose

#### Storage

Жмите карандашик возле Storage пути

![iSpy Storage Settings](https://i.imgur.com/fR0DcyD.png)

Настраиваем путь для архива фоток и видео. Материал, на который вы нажмете "Archive" будет перемещен туда

![iSpy Storage Settings. Archive](https://i.imgur.com/xwvdot6.png)


### ➡️ Добавление камеры

Удивительно, как легко они усложнили такой процесс

Клик сверху слева в углу кнопку Server Menu (после замочка) > ➕ New Device > Video Source

#### General

1. Переключатели можно не трогать (потом разберетесь и переключите)
2. Source Type: IP Camera (для подключения по rtsp://)
   ![iSpy Video Source Example](https://i.imgur.com/heAF1QL.png)
   - Live URL это ваша rtsp:// ссылка без `login:pass@`. Указывайте ссылку на substream
   - Record URL указываем аналогично, но на main stream. Эта ссылка будет задействована если потом правильно настроить Recording (об этом позже)
3. Decoder (если на сервере нет видеокарты, то меняем на CPU)
4. Переключаем Enabled
5. Max Framerate будет пропускать часть кадров, чтобы серверу было попроще. Я указал 20, но можно и 10
6. Resize меняем, если хотите уменьшить (скейлить) картинку с Live URL (у меня поток в 640*480, мне это не нужно)
7. Возвращаемся в верх модалки, справа в углу General меняем на **Detector**

#### Detector

Detector это основа любой дальнейшей автоматизации. Когда детектор что-то обнаружит, он создаст ивент **Detected**. Далее мы будем этот ивент фильтровать, чтобы ложные срабатывания не создавали нам геморрой (резкие блики света, включение фонарей, шелест травы и тд)

1. Синим выделяем зону, движения в которой будут отслеживаться (ПКМ = ластик)
2. Detector выбираем Simple. Это сенсор, который реагирует на любое движение. Про другие сенсоры можете почитать в доках (знак вопроса сверху модалки, помните, да?)
3. Справа от Simple жмем троеточие и там настраиваем чувствительность сенсора (действительно важно его настроить). Gain это усилитель чувствительности. Ползунками выбираем зону, которая будет триггерить сенсор и вызывать ивент Detected
4. Переходим с настройки Detector к **Alerts**

#### Alerts

Если Detector это у нас просто "обнаружение движения в кадре", то Alerts это реакция на эти движения (или их отсутствие)

1. Переключаем **Enabled**, чтобы включить функцию
2. `Mode` = Detected (Not Detected может быть полезен в том случае, если вы например за роботами следите, чтобы они не переставали работать)
3. `Detection delay` для меня нелогичная настройка. Я ставлю тут 0. Это задержка, прежде, чем алерт сработает после обнаружения чего-либо
4. `Not detected alert delay` аналогично, но имеет смысл только для Mode = Not Detected
5. `Minimum Interval` это антифлуд настройка. Чтобы не спамило, если кто-то перед камерой прыгает. Вот что будет, если установить задержку 0 на алерты:
	![ispy-screenshots-spam](https://i.imgur.com/ipNJMM5.png)
	Т.е. видео оно может записать одно, но самих алертов насрет целую пачку. Настройки времени в Recording вкладке (описана дальше) работает независимо от настройки алертов
6. Переходим к настройке **Alert Filter**

#### Alert Filter

Когда детектор будет говорить "слушай, у меня тут движение", Alert Filter посмотрит на происходящее и решит, стоит ли триггерить Alert. Тут мы активируем [Deepstack AI](https://www.ispyconnect.com/userguide-agent-deepstack-ai.aspx) (который настроили в `Server Menu > Settings > Intelligence` ранее), чтобы игнорировать срабатывания детектора на животных, деревьях, свете фар и т.д.

1. Переключаем Enabled
2. Снизу жмем Configure
   1. Здесь, как и в детекторе есть выделение зоны. Логика такова, что вы можете захотеть активировать ИИ не только на зону обнаружения движения. Если не уверены, нужно ли тут что-то рисовать, то оставьте все как есть (ИИ пройдет по всему кадру, а не только там, где было движение)
   2. В Find снизу выберите только Person. Этого будет достаточно для теста
   3. Confidence это как бы.. ИИ не всегда уверен, что видит именно человека. Меня однажды за собаку посчитал. Укажите здесь 50%, чтобы принять мнение ИИ насчет существа в кадре, если он уверен в этом хотя бы на 50%. Я ставил 40%
   4. Photos включаем. Не знаю зачем, но прикольно. Будет делать фотки с выделением объекта, когда что-то обнаружит. Фотки можно смотреть тоже в iSpy, для этого есть отдельный View
   5. Prevent Repeat полезен, если вы например настроили детектор на машины и кто-то поставил свою в какой-то отслеживаемой точке. Любой шорох листьев будем запускать фильтр, а фильтр будет триггерить алерты, пока машина не "исчезнет"
   6. Request Interval это промежуток между запросами к AI серверу (Deepstack). Я бы не трогал, если сервер не задыхается от слишком частых срабатываний Alert Filter.
3. Переходим к **Photos**

#### Photos

У нас фотки будут делаться, когда нейронка обнаружит человека в кадре. Саму функцию пока не включаем, фотки будут делаться и без включения. Но изменяем тут Overlay Text. Этот текст будет снизу любой фотки, которая будет сделана камерой. Я просто удалил оттуда текст.

Переходим к настройке **Recording**

#### Recording

Тут мы сделаем, что когда триггерится алерт (т.е. замечен человек), то включится запись видео.

1. `Mode` ставим Alert. Если поставить Detect, то запись видео будет начинаться от срабатывания Detector (т.е. любого изменения в кадре)
2. `Encoder` ставим Raw Record Stream. Так оно будет записывать не сабстрим, который в плохом качестве, а переключится на хорошее качество. Encode будет записывать то, что вы видите прямо в браузере со всеми наложениями и т.д.
3. Если на сервере нет видеокарты, то отключаем `Use GPU`.
4. `Max Framerate` не имеет влияния, если выбран Raw тип записи. Если же выбран Encode, то оно порежет кадры до указанного количества
5. `Min/Max record Time` я поставил 10 и 60
6. `Inactivity Timeout` = 5. Сколько записывать с момента алерта. Но запишет не меньше, чем Min record time
7. `Buffer` = 3, чтобы оно записывало и ДО алерта. Это повышает нагрузку и провоцирует постоянное фоновое "наблюдение" за основным потоком (жрет много трафика). Если это критично, то лучше поставить 0
8. `Quality`, как и Max Framerate не имеет влияния, если это Raw запись. Для Encode я бы выбрал 7. Это степень шакалирования жпега перед сохранением на диск

#### Storage

1. Изменяем Folder на что-то более читаемое. Например, на `camera_door`. Так проще будет найти файлы на жестком диске
2. Я еще включил Archive, чтобы вместо удаления файлы архивировало. Правда у меня это не сработало 👿

#### Timelapse

Генерация ускоренного видео. Я настроил генерацию коротких видео раз в час, чтобы иметь возможность бегло посмотреть, не произошло ли чего-то забавного, что я мог пропустить. Например однажды я так заметил чувака, который ссал мне на забор в месте, где детектор отключен

1. Enabled тут же включает генерацию Timelapse видео. Через Schedule настройки можно включать или отключать эту функцию
2. `Frame Rate` это столько "скриншотов" поместит в 1 секунду сгенерированного видео (интенсивность видео). Этим можно регулировать эпилепсию. Я поставил 20
3. `Video Frame Interval` как часто будут "фотаться" кадры таймлапса. 1 значит 1 кадр в секунду. 60 значит 1 кадр в минуту. 0 отключит генерацию видео таймлапса (см. далее)
4. `Photo Frame Interval` указание этой странной опции начинает просто делать скриншоты время от времени. Не ускоренное видео, а именно просто делает кучу скриншотов. Зачем тогда указывать `Frame Rate` и почему эта настройка тут, а не в Schedule для меня загадка
5. `Save Every` какой временный промежуток должен содержать таймлапс? Я поставил 60 (раз в час клепает видео)

#### Timestamp

По умолчанию рисуется сверху слева на видео в прямоугольнике. На моей камере и так рисуется время, поэтому я оставил только FPS: `FPS: {FPS}`. По нему определеяю, когда серверу "плохо" и не хватает мощности


### ➡️ Что дальше?

Можно докрутить интеграцию с Integromat или IFTTT через [iSpy API](https://www.ispyconnect.com/userguide-agent-api.aspx) (Server Menu > Settings > Local Server). Сценарии можно придумать какие угодно, даже автоматизировать переворот ведра с 💩 на голову незнакомцу.

Кастомные Views стоит настроить. Реально красиво и классно. Лучше когда просто пара камер рядом показывается

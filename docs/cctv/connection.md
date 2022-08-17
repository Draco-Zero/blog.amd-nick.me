---
sidebar_position: 3
---

# Подключение к камере

:::info
Про rtsp, rtmp, ONVIF, http
:::

## ⚠️ RTSP

Самый важный протокол.

Это просто особая ссылка. Ее **можно открыть через любое видеоприложение, например VLC** и просто видеть видео с камеры

![VLC + rtsp](https://i.imgur.com/GXFSZ6m.png)

Пример такой ссылки: `rtsp://admin:p4ssw0rd@192.168.1.90:554/cam/realmonitor?channel=1&subtype=1`. subtype у моей камеры может быть 1 или 0. 1 это substream, 0 это основной FullHD поток. :554 можно не указывать, если в настройках камеры или роутера не изменялся порт

Для своей камеры (если у вас не IMOU), ссылку можно найти тут:
- https://www.ispyconnect.com/camera/imou (снизу все бренды)
- https://camlytics.com/camera/imou (снизу страницы бренды)

## RTMP

Если камера поддерживает RTMP, значит видео с нее можно стримить на ютуб, твич и даже телеграм (в [voice чаты](https://telegram.org/blog/voice-chats-on-steroids)). Подробнее об этом писал в [Настройки камеры](settings.md)

Для меня наличие RTMP было приятным, но бесполезным

## ONVIF

Это как бы стандартизирующий протокол, благодаря которому один софт (например, [ONVIF Device Manager](https://sourceforge.net/projects/onvifdm/)) может работать сразу с несколькими камерами.

:::tip
Если камера поддерживает ONVIF, то это предпочтительный способ подключения там, где это возможно, так как некоторые фичи можно переложить на мощности камеры, а не сервера (например, детектор движений).
:::

У IMOU (Dahua) есть официальный софт для работы со своими камерами по ONVIF. Это [SmartPSS](https://dahuawiki.com/SmartPSS) и [ConfigTool](https://dahuawiki.com/ConfigTool)

**В [iSpy](apps/README.md) у меня вообще получилось подключиться по ONVIF только к локальной камере 😭**, а через официальный софт к камерам в любой сети подключило без проблем.

:::info
Если камеры находятся в другой сети, но [доступны удаленно](expose.md), то нужно пробросить на домашнем роутере порты 37777 (настройки) и 37778 (видео) наружу.
:::

## HTTP

Если у камеры включен [CGI Service](settings.md), то по идее видео с нее можно было бы смотреть прямо в браузере. Но **у меня не удалось** получить [http поток](https://community.home-assistant.io/t/imou-ip-camera-configuration-always-inactive/177604) для IMOU. Вместо этого я получал только [скриншот с камеры](settings.md). Причем только через admin юзера

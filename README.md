# OpenTherm Gateway by umdu

Форк [Laxilef/OTGateway](https://github.com/Laxilef/OTGateway), 
доработанный под плату umdu_ot на базе модуля ESP32-S3-WROOM-1 N16R8. 
Не переписан с нуля — взята рабочая база и адаптирована под конкретное железо 
и сценарии использования.

> **Важно.** Всё тестируется на umdu_ot — и именно под неё делаются изменения.
> Если нужен универсальный OTGateway для других плат, смотрите 
> [upstream](https://github.com/Laxilef/OTGateway).

## Что изменено относительно upstream

- Добавлено отдельное окружение `umdu_ot` в `platformio.ini` с пинаутом под плату.
- Управление байпас-реле, которое в том числе отвечает за переключение линии 
  в аварийном режиме.
- Управление реле сухого контакта с арбитражем: доп. насос и каскадное 
  управление не могут работать одновременно, потому что реле одно; 
  конфликтные команды валидируются и блокируются в Web/MQTT/HA.
- Русифицированы имена сущностей и метрик в MQTT/Home Assistant, а также локализовано отображение сенсоров в веб-интерфейсе.
- Обновлены дефолтные `hostname`, `AP SSID/password` и `MQTT prefix`.
- Интерфейс и брендинг адаптированы под umdu_ot: логотип, навигация, ссылки 
  на документацию и релизы форка, имя устройства в Web/HA.

## Сборка

Используется PlatformIO. Web Flasher для этого форка не поддерживается.

**Нужно:**
- PlatformIO Core (`pio` в PATH)
- Node.js/NPM (pre-build шаг собирает web-ресурсы)
```bash
cd software/otgateway/upstream

# Опционально — локальный файл с секретами:
cp secrets.default.ini secrets.ini
# После этого в platformio.ini переключить extra_configs на secrets.ini

pio run --environment umdu_ot
```

После сборки в `build/` появятся артефакты вида:
```
firmware_umdu_ot_<version>.bin
firmware_umdu_ot_<version>.factory.bin
firmware_umdu_ot_<version>.elf
filesystem_umdu_ot_<version>.bin
```

## Прошивка
```bash
cd software/otgateway/upstream

# Основная прошивка
pio run --environment umdu_ot --target upload

# Файловая система — только если нужно
pio run --environment umdu_ot --target uploadfs
```

### Ручная прошивка локальных bin

Для сборки `umdu_ot` используется PlatformIO-профиль
`esp32-s3-devkitc1-n16r8`, совместимый с модулем ESP32-S3-WROOM-1 N16R8.
Таблица разделов задается явно в `platformio.ini`:
`board_build.partitions = default_16MB.csv`.
Если шить через `esptool`, брать offset файловой системы нужно из этой таблицы
или из сгенерированной `.pio/build/umdu_ot/partitions.bin`, а не из
`esp32_partitions.csv`, который для `umdu_ot` не используется.

Текущая таблица разделов для `umdu_ot`:

```text
app0    0x10000   6400K
app1    0x650000  6400K
spiffs  0xc90000  3456K
```

Минимальная ручная прошивка чистого устройства:

```bash
cd software/otgateway/upstream

.pio/penv/bin/esptool --chip esp32s3 --port <PORT> --baud 921600 \
  --before no-reset --after no-reset erase-flash

.pio/penv/bin/esptool --chip esp32s3 --port <PORT> --baud 921600 \
  --before no-reset --after watchdog-reset write-flash -z \
  --flash-mode dio --flash-freq 80m --flash-size 16MB \
  0x0 build/firmware_umdu_ot_<version>.factory.bin \
  0xc90000 build/filesystem_umdu_ot_<version>.bin
```

При переходе с тестовой прошивки, другой таблицы разделов или неизвестного
содержимого Flash сначала обязательно выполнять `erase-flash`. Запись только
обычного `firmware_*.bin` по адресу `0x10000` не заменяет загрузчик и таблицу
разделов.

Для встроенного USB Serial/JTAG ESP32-S3 после ручного входа в ROM-загрузчик
используется `--after watchdog-reset`: обычный `hard-reset` может оставить
чип в download mode. Если порт не восстановился, кратко нажать `RESET` без
удержания `BOOT` или переподключить питание.

Если артефакты перенесены в релизную папку монорепо, из
`software/otgateway/upstream` путь будет:

```bash
../../../firmware/otgateway/<version>/firmware_umdu_ot_<version>.factory.bin
../../../firmware/otgateway/<version>/filesystem_umdu_ot_<version>.bin
```

## Документация

Актуальная документация по эксплуатации — на [docs.umdu.ru](https://docs.umdu.ru).  
Материалы в этом репозитории и в upstream могут отставать.

## Лицензия

GNU GPL v3.0 — см. [LICENSE](LICENSE).

Оригинальный проект: [Laxilef/OTGateway](https://github.com/Laxilef/OTGateway).  
Спасибо Laxilef за архитектуру — без неё этот форк не появился бы.

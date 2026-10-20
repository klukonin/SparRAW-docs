# Трассировка wil6210 на стенде: лог прошивки, трасса ucode, команды WMI

Каналы наблюдения за прошивкой 4.1.0.1000 на живом узле, драйверные патчи, которые их
включают, и результаты первых прогонов. Инструменты — репозиторий SparRAW-tools
(`host/wil_fw_log.py`, `host/wil_uc_collect.py`, `host/wil_uc_stream.py`,
`host/wil_ucode_log.py`); патчи — SparRAW-driver, `openwrt/patches/ath/9xx-wil6210-*.patch`.

Адреса колец и их раскладка в документации прошивки:
[HARDWARE-BLOCKS.md, «Публикация адресов логов» и «Кольцо лога ucode в ОЗУ»](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md#публикация-адресов-логов-драйверкод);
конвенция лога ucode 4.1 —
[4.1/docs/UCODE-BEACON.md §1](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/UCODE-BEACON.md#1-конвенция-логирования-ucode);
глушение модулей лога fw —
[LMAC-PROTOCOL.md §12](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/LMAC-PROTOCOL.md#12-глушение-шума-в-логе-прошивки-железо);
стенд 6.2 — [6.2/docs/BENCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md).

## Каналы

| канал | что даёт | включение |
|---|---|---|
| netconsole | сообщения ядра и драйвера по UDP | `rc.local` оверлея, модуль грузится штатно |
| лог прошивки | кольцо ~83 КБ (~308 записей), 2296 строк; модули SYSTEM, DRIVERS, MAC_MON, HOST_CMD, PHY_MON, INFRA, … | патчи 905/906, `echo 7 > …/wil6210/fw_log_level` |
| лог ucode | TBTT, окна AW/DTI, команды fw → ucode, развёртка маяка | патч 909; в образе 4.1 байты уровней уже 0x07 |
| трасса ucode | непрерывный поток лога ucode с метками хоста | патч 910 |

## Лог ucode **[код + железо]**

Раскладка записи видна, например, в начале `uc_sysassert` (4.1 0x925548; первая запись
аварийного дампа, заголовок `0xaa000000 | 0x154`):

```c
if (levels[0] & 4) {                     /* байт 0x8020a0 = 0x80209c + 4 */
    ring[(idx + 2) & 0xff] = arg2;       /* ring = 0x8020b0, idx = 0x80209c */
    ring[(idx + 1) & 0xff] = arg1;
    ring[ idx      & 0xff] = 0xaa000154; /* заголовок */
    idx += 3;
}
```

* Индекс берётся по модулю 256 (индекс слова, не байтовое смещение); записи начинаются с
  +0x14 от опубликованного адреса 0x80209c; 16-байтового заголовка уровней, как у кольца fw,
  перед записями нет.
* Формат слова-заголовка как у лога fw: биты 0..19 — смещение строки, 20..23 — модуль,
  24..25 — уровень, 26..27 — число аргументов; аргументы лежат перед заголовком.
* Адрес принадлежит **карте ucode**: данные fw и ucode оба начинаются с 0x800000, но fw
  видна хосту по 0x900000 (`blob_fw_data`), ucode — по 0x940000 (`blob_uc_data`). Запись по
  чужой карте молча портит память другого процессора; патч 909 выбирает отображение с
  учётом флага `fw`.

Разовое чтение:
```
cat /sys/kernel/debug/ieee80211/phy0/wil6210/blob_uc_data > uc.bin
wil_fw_log.py uc.bin -s 4.1/ref/strings-uc.bin -a 0x80209c --region uc_data
```

На маячащей AP за один BI видно:
```
SYSTEM INFO L1_TASK PRE TBTT
SYSTEM INFO sending event=0x17 ,curr tsf 0x00000000 0f695797
SYSTEM INFO BI_AP_MONITOR_SM: AW EVENT
SYSTEM INFO BI_AP_MONITOR_SM: DTI EVENT
SYSTEM INFO umac_if_cmd_handler FW command 0x3 / 0x1
```

TBTT по событию 0x17 (95 снимков с шагом 2 с, 1871 событие): медиана интервала
**102 400 мкс**, минимум 101 809, максимум 102 991.

Строки развёртки маяка, по которым видна работа рандомизатора
(`bti_worker::bti_transmitter_beacon_sweep_flow` 0x923de0): `randomize beacon` (0x923e32),
`current_tsf / Next tsf / uniform_random` (0x923eec), `Beacons delay due to NAV started`
(0x923f94).

## Сборщик трассы ucode в драйвере (патч 910) **[железо]**

Кольцо ucode — 256 слов, около 1,5 с истории. Вычерпывание из userspace теряет часть
записей между чтениями, и пропуски выглядят как реальные (при таком съёме 6 % «пропущенных маяков» —
артефакт съёма). Патч `910-wil6210-ucode-trace-collector` опрашивает кольцо
каждые `uc_trace_ms`, берёт новые слова и копит их в буфере 1 МБ с метками времени хоста;
debugfs `uc_trace` отдаёт буфер пачками (magic/count/ktime/слова).

```
echo 100 > /sys/module/wil6210/parameters/uc_trace_ms
echo 7   > /sys/kernel/debug/ieee80211/phy0/wil6210/fw_log_level   # взводит и сборщик
cat /sys/kernel/debug/ieee80211/phy0/wil6210/uc_trace > uc.bin
wil_uc_collect.py uc.bin -s 4.1/ref/strings-uc.bin
```

Проверка полноты: за 67,9 с ожидается 663 BI, поймано 668 событий `PRE TBTT`; медиана
интервала по TSF 102 400 мкс, максимум 205 382 — потеряно одно событие. При ~540 Б/с
буфер держит около получаса. Особенности освобождения буфера — в
[6.2/docs/BENCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md).

### Прогон под нагрузкой (станция, 655 с, 101 089 слов)

| событие ucode | раз |
|---|---|
| `txop_initiator_open_txop_flow(): TRUE TxOP opened (sent RTS/…)` | 59 403 |
| `umac_if_cmd_handler FW command` | 5 725 |
| `BI_AP_MONITOR_SM: AW EVENT` / `DTI EVENT` | по 921 |
| `L1_TASK PRE TBTT` | 919 |
| `tx_initiator_flow() Check for busy2 (CCA) or energy (RSSI)` | 404 |
| `DISCOVERY ON` / `DISCOVERY OFF` | по 381 |
| `perform_bti_pm_cfg() - force wakeup` | 24 |

| окно, с | TxOP | TBTT | DISCOVERY |
|---|---|---|---|
| 540…590 (простой) | 0…27 | 81…98 | 7 |
| 600…640 (iperf3, 934 Мбит/с) | 9 359…16 021 | 97…98 | 0 |
| 650 | 0 | 57 | 0 |

* Под нагрузкой ≈ 1300 TxOP/с, ~90 КБ на TxOP — передача крупными агрегатами.
* Маяки нагрузкой не сбиваются: 98 интервалов на 10 с в простое и под нагрузкой.
* Циклы DISCOVERY идут только при отсутствии линка.

## Трассировщик команд WMI в логе fw **[код + железо]**

Диспетчер команд хоста 4.1 (`host_if__wmi_cmd_dispatch` @0x8da54c,
[WMI.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/WMI.md)) на каждую
команду пишет две строки модуля SYSTEM:

```
SYSTEM  INFO  HOST CMD 0x0000080E for MIDID 0
SYSTEM  INFO  ---->> [HOST CMD] WMI_TEMP_SENSE_CMDID
```

Обмен драйвера с прошивкой виден без патча драйвера. `cat temp` в debugfs даёт одну
запись `WMI_TEMP_SENSE_CMDID` (0x080E) на каждое чтение. Для снятия MAC_MON и PHY_MON
глушатся (иначе они вытесняют кольцо за один-два BI), например:

```sh
D=/sys/kernel/debug/ieee80211/phy0/wil6210
echo "0x0090b904 0x0f000f0f" > $D/mem_write   # SYSTEM=f DRIVERS=f MAC_MON=0 HOST_CMD=f
echo "0x0090b908 0x00000f00" > $D/mem_write   # PHY_MON=0 INFRA=f
echo "0x0090b90c 0x00000000" > $D/mem_write
echo "0x0090b910 0x00000000" > $D/mem_write
```

Любой сброс прошивки перевзводит уровни драйвером (`wil_log_ring_arm`, маска из
`fw_log_level`) и затирает ручную настройку; `/etc/init.d/wpad restart` вызывает сброс.
Поэтому последовательность команд при старте точки доступа этим способом не снимается —
трассируются действия без сброса (чтения debugfs, команды `iw`, `wmi_send`).

## Запись в ОЗУ устройства: `mem_write` (патч 911)

Штатный `mem_val` подключён к `memread_fops` и доступен только на чтение, несмотря на права
0644 (запись — EINVAL). Штатное чтение через `mem_addr`/`mem_val` понимает только адреса
прошивки (`wmi_addr_remap` перебирает отображения с `fw == true`); данные ucode читаются
через `blob_uc_data`. Патч `911-wil6210-mem-write-debugfs` добавляет `mem_write`
(«адрес значение», адреса хоста как у `mem_addr`; запись через `wmi_buffer()` под
`wil_mem_access_lock()`):

```sh
echo "0x00941438 2" > $D/mem_write            # ucode 0x801438 := 2
echo 0x00941438 > $D/mem_addr; cat $D/mem_val # проверка
```

Пересчёт: данные ucode 0x800000 → host 0x940000, данные fw 0x800000 → host 0x900000
(ловушка линкерного адреса — [BEACONING.md §9.1](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md#91-открытие-гейта-с-хоста-41-железо)).
Таблицы диспетчеризации и управляющие переменные ucode строятся в рантайме и в образе
отсутствуют — `mem_write` делает их доступными без патча прошивки.

## Восстановление после сброса (патчи 907, 908) **[железо]**

`recovery` в debugfs только продолжает начатое восстановление. Патч 907 считает обнулённые
байты уровней в заголовке кольца лога признаком перезапуска прошивки, поэтому
восстановление вызывается записью:

```sh
echo "0x0090b904 0x00000000" > $D/mem_write   # уровни лога fw (fw 0x843904)
```

На AP с одной станцией: `wil_fw_error_worker … completed`, `_wil6210_disconnect reason=2`,
затем `wmi_evt_connect: successful connection to CID 0` — станция переподключается за
**1,6 с**, узел не зависает, лог fw, лог ucode, сторож и сборщик трассы перевзводятся,
блок конфигурации BI пересылается драйвером заново со стоковыми значениями. Без сноса
TX-колец (908) повторная инициализация не проходит. Ложные срабатывания сторожа 907 на 6.2 и
на станции 4.1 — [6.2/docs/BENCH.md, «Сторож драйвера 907»](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md#сторож-драйвера-907).

## Таблицы строк лога

Строки печатаются хостом по смещению: в прошивке лежит только слово-заголовок.
Таблицы — вендорские записи образа: **100** — ucode, **101** — fw (у 6.2.0.1000 есть и 102).
Для 4.1 они выложены в SparRAW-firmware как `4.1/ref/strings-fw.bin` (запись 101) и
`4.1/ref/strings-uc.bin` (запись 100).

Пул ucode 6.2.0.1000 (запись 100, 12992 Б): 16 имён модулей по 12 Б (192 Б), далее
format-строки с нулём в конце, выровненные на 4; всего 231 строка. Модули ucode: SYSTEM,
TX, RX, ISR, BCON, BEAMFORM, SXD_UTILS, RSSI, CALIBS, DRIVERS (NAME_10..15 — резерв).
Ссылка на строку в коде — `0x01000000 | смещение`, затем `bmsk rX,rX,0x13` и
`or rX,0xHH000000`. По этому признаку из кода адресуются 151 строка ucode 6.2
(155 мест) и 130 строк ucode 4.1 (237 мест из 331 строки таблицы). Поиск голого смещения
(без `0x01000000`) даёт в 6.2 лишь 21 строку и ложный вывод об отсутствии в коде
`bti_worker`, `dti_worker`, `fixed_scheduling`.

Пул — образ строк всего семейства: наличие строки без ссылки из кода не доказывает ни
присутствия, ни отсутствия подсистемы. Половина мест вывода ucode 6.2 — аварийный дамп
(`Reason: ST/LD cmd before [ilink2=...]`, `Bad instruction`, гистограммы
`IDLE SM histogram on SYSASSERT` по состояниям `SXD_IDLE_DETECT_{SIFS,CCA_RX,CCA_NAV}_STATE`,
`g_sysassert_cca_hist`).

## Ограничения

* Своей точки трассировки в образе ucode 4.1 без переноса кода нет: вендорский свободный
  участок — 96 байт по 0x9200a0 (переписанные блоки уходят в хвост 0x9380c4, см.
  [SELFBUILT-FIRMWARE.md](SELFBUILT-FIRMWARE.md)).
* Регистры MAC ucode (кольцо r25, r40/r41) в окне 0x88xxxx хосту не видны.
* Поиск регистров по периодичности не работает: снимок окна занимает ~17 мс при BI
  102,4 мс — шесть точек на период.

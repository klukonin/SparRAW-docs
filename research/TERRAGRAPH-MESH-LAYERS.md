# Слои mesh-сети cnWave / Terragraph

Устройство рабочего 60-ГГц mesh по образам `cnwave60ghz-v1000-upgrd-1.2-beta4.img` (IPQ40xx) и
`-v5000-v3000-` (QorIQ): где живёт пересылка — в ARC-прошивке радио или на хосте.
Пересылка — L3/IPv6 на хосте; прошивка радио держит только линки узел↔узел.

## 1. Радио (ARC-прошивка)

* Вариант `WIL6436_M_B0_IF2IF`, FW **10.11.4.87** (пак 11ad с символами — 7.5.0.77, много
  старше). «IF2IF» = interface-to-interface, линк DN↔DN.
* Таблица строк (`fw_trace_strings-10.11.bin`, 4086 строк) показывает внутренности Terragraph:
  `tdm_3p_adapter`, `fb_mtpo_stats_initiator` (fb = Meta), `NEW_LINK_EVT` / `LINK_LOST_EVT`,
  `CMD_BCON_MGT pcp_assoc_ready`, `clbr_mngr` / `mtpo_calib` — синхронный по GPS TDMA-планировщик
  линков, без пересылки пакетов.
* Образ `wil6210-IF2IF-10.11.4.87.fw` читается штатными инструментами разбора контейнера
  (CRC верен, 4 сегмента). Символьных карт (globals XML) для 10.11 в образе нет — адреса
  восстанавливаются дизассемблером, имена 7.5 служат ориентиром; таблицы строк
  (`*_trace_strings-10.11.bin`) резолвят лог прошивки.

## 2. Драйвер `terragraph-qca.ko` (fb_tgd)

* Каждый линк выставляется отдельным netdev **`terra%d`** с атрибутом `peer_mac`.
* Управление бейсбендом — IoCtl `TG_SB_*` (`INIT_REQ`, `*_LINK_RESP`, `DISASSOC_REQ`,
  `START_BF_SCAN_REQ`, `PASSTHRU`, `GPS_TIME`); северный интерфейс — netlink к `e2e_minion`.

## 3. Хост (Linux userspace)

* **`openr`** (Open/R, `libopenrlib.so`) — link-state маршрутизация поверх интерфейсов
  `terra`, `nic1`, `nic2` (`etc/sysconfig/openr`).
* **`e2e_controller` / `e2e_minion`** — топология и ignition (кто с кем линкуется, конфигурация
  лучей), не пересылка.
* **`fib_linux`** / **VPP** — плоскость данных (маршруты в ядро либо в VPP).
* **POP** (`pop_config`, `get_pop_ip`) — выход во внешнюю сеть.

## 4. Сопоставление с универсальным wil6210

| слой | Terragraph | wil6210 (mainline) |
|---|---|---|
| маршрутизация | Open/R (link-state поверх точка-точка) | любой link-state/дистанционно-векторный протокол поверх линков |
| линк → netdev | `terra%d` на каждый пир | до 4 VIF, netdev на линк нет |
| прошивка | IF2IF 10.11 с TDMA-планировщиком линков | Sparrow: ESE-расписание (4.1, 6.2), фиксированное расписание (6.2) |

Исходники Terragraph открыты (github.com/terragraph), включая драйвер. Прямой объект сравнения
прошивок — `wil6210-IF2IF-10.11.4.87.fw` против пака 11ad `wil6436.fw` 7.5.0.77. Рандомизатор
маяка в Terragraph 10.x удалён (заменён детерминированным расписанием) —
[docs/BEACONING.md §4.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md).

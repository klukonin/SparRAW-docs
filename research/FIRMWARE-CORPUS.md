# Корпус образов прошивки wil6210

Образы одной линии wil6210 разных версий и поставщиков, пригодные для перекрёстной сверки. Все
читаются инструментами разбора контейнера.

| источник | версия | чип | образ | ценность |
|---|---|---|---|---|
| TP-Link Talon AD7200 | 1.4.1.7698 | Sparrow | `wil6210.fw` из образа AD7200 v2 | самая ранняя, эволюция |
| linux-firmware | 5.2.0.18 | Sparrow | `/lib/firmware/wil6210.fw.zst` | полный mainline-набор WMI |
| RouterOS (mainline-копия) | 5.2.0.18 | Sparrow | `SparRAW-firmware/blobs/stock/wil6210_5.2.0.18_stock.fw` | = linux-firmware |
| RouterOS | 4.1.0.1000, 6.2.0.1000 | Sparrow | пакет `wireless` ([FIRMWARE-COMPARISON.md](FIRMWARE-COMPARISON.md)) | основные объекты разбора |
| UBNT airFiber | 6.2.0.225 | Sparrow+ | `wil6210_sparrow_plus.fw` | урезанный набор WMI (41 команда) |
| RouterOS | 7.1.3.1158 | wil6436 | `wil6436.fw` | ветка wil6436 |
| пак 11ad | 7.5.0.77 | wil6436 | `wil6436.fw` | есть символы (globals XML) |
| RouterOS | 10.1.0.692 | wil6436 / Terragraph | `wil6436-tg.fw` | ранний Terragraph |
| wil6210_tg | 10.11.0.70 | Terragraph | `wil6210_tg.fw` | Terragraph |
| cnWave IF2IF | 10.11.4.87 / .92 | Terragraph | `wil6210-IF2IF-*.fw` | Terragraph mesh ([TERRAGRAPH-MESH-LAYERS.md](TERRAGRAPH-MESH-LAYERS.md)) |

Кроме того — зашифрованные образы и board-файлы RouterOS и их расшифрованные копии
([NPK-ENCRYPTION.md](NPK-ENCRYPTION.md)).

## Возможные сверки

1. **mainline 5.2 против airFiber 6.2** (обе Sparrow, близкие версии): что поставщик выкинул и
   добавил — полный набор WMI против 41 команды.
2. **6.2 (Sparrow) против 7.5 (wil6436, с символами)**: перенос имён; структуры `basic_sm`, лога и
   WMI совпадают по форме.
3. Эволюция автоматов и тракта данных 1.4 → 5.2 → 6.2 и 7.1 → 10.11 (добавление Terragraph).

Результаты межверсионной сверки 4.1 ↔ 6.2 и наличие рандомизатора по версиям —
[docs/CROSS-VERSION.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/CROSS-VERSION.md),
[docs/BEACONING.md §4.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md).

Метод: для каждого образа — проекция символов и структур инструментом `wil_syms.py project`,
затем сравнение по структуре (диспетчер WMI, каталог автоматов).

# Наблюдение за прошивкой с хоста

Способы видеть работу биконинга и расписания BI на железе без понижения прошивки и без патча
драйвера. Регистры публикации логов, формат кольца лога ucode и компаратор TSF —
[docs/HARDWARE-BLOCKS.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HARDWARE-BLOCKS.md);
отчёт MAC_MON — [docs/BEACONING.md §8](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md);
метод проверки рандомизатора и непрерывного съёма лога ucode — там же, раздел «Метод».
Инструменты — каталог `SparRAW-tools/host/`.

## 1. Доступ

* Регион `rgf` (0x880000…0x88a000) и `fw_peri` (0x840000…0x860000 → host 0x908000) помечены
  доступными в `sparrow_fw_mapping`: чтение через debugfs `mem_addr`/`memread` или блобы
  `blob_fw_peri`, `blob_uc_data` без патча драйвера.
* Ненулевые `RGF_USER_USAGE_1` (0x880004, лог fw) и `RGF_USER_USAGE_2` (0x880008, лог ucode)
  после загрузки означают, что прошивка опубликовала кольца (драйвер обнуляет их перед стартом).
* Запись с хоста штатно — только в ICR/DMA/ITR (`doff_io32`); запись в ОЗУ прошивки — через
  `mem_write` (host-адреса, см. предупреждение о линкерных адресах ucode в
  [docs/BEACONING.md §9.1](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md)).

## 2. Лог прошивки (fw) и отчёт MAC_MON

```sh
D=/sys/kernel/debug/ieee80211/phy0/wil6210
cat $D/RGF_USER_USAGE_1          # адрес кольца, например 0x00843900
cat $D/blob_fw_peri > peri.bin   # 0x840000..0x860000 -> host 0x908000
python3 wil_fw_log.py peri.bin -s fwlog-strings/fw-4.1.0.1000.bin -a 0x843900 --region fw_peri
```

Кольцо fw 4.1 вмещает ~309 записей (~1,5 с): снимать сразу после события. Таблица строк fw —
запись 101 образа ([FIRMWARE-COMPARISON.md](FIRMWARE-COMPARISON.md)). Модуль `MAC_MON` печатает
поинтервальный отчёт BTI/AW/DTI после поднятия уровня лога. Измерения по отчёту — [MACMON-SCHEDULE.md](MACMON-SCHEDULE.md).

## 3. Лог ucode

`wil_ucode_log.py` + `ucode_strings.json` (таблицы строк ucode, извлечённые из записи 100
образов: 4.1.0.1000 — 331 строка, 6.2.0.1000 — 222 строки). Смещения строк привязаны к версии
образа.

```sh
./wil_ucode_log.py --probe              # опубликованы ли адреса логов
./wil_ucode_log.py -v 6.2.0.1000 -w     # декодировать лог ucode указанной версии
```

Строка `bti_worker::bti_transmitter_beacon_sweep_flow() current_tsf, Next tsf, uniform_random: %d`
(в 6.2 — смещение 0x1d88 таблицы строк ucode) отмечает срабатывание рандомизатора;
`uniform_random` лежит в [10, 198] (`((rand*0xbd)>>32)+10`).

## 4. Блок счётчиков MAC_MON (4.1)

`wil_beacon_stats.sh` читает блок 0x854de0 (`fw_peri`) и печатает `tx bcon / rx bcon / detected`,
`bcon bitmap`, окна BTI/AW, ATIM pass/fail. Раскладка — только для 4.1.0.1000
([4.1/docs/DASHBOARD.md §7](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/DASHBOARD.md));
в 6.2 отчёт публикуется в 0x853940 двойным буфером
([6.2/docs/BENCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md)).

## 5. Компаратор TSF

`wil_tsf_probe.py` читает целевой TSF (0x886d30/0x886d34), управление (0x886d38: 0xc0050 —
взведён, 0xc0010 — снят) и текущий TSF (0x886eb8/0x886ebc) и печатает `NextTSF − TSF`.

```sh
./wil_tsf_probe.py                 # 40 замеров, delta и вердикт
./wil_tsf_probe.py -n 200 -i 0.05
```

Разность показывает, взводит ли ucode компаратор (`set_tsf_event` 0x930c28 / `disable_tsf_event`
0x925120 в 4.1) и с каким значением. Сдвиг передачи маяка она не доказывает: передача от
компаратора не зависит ([docs/BEACONING.md §9.2](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/BEACONING.md)).

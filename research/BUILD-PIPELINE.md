# Точечная правка образа и приведение к формату мейнлайна

Цикл «правка байтов → пересчёт CRC → срез нештатных записей» для образов
`wil6210.fw` из пакетов RouterOS и цикл распаковки/упаковки board-файлов.
Сборка образа целиком из дерева исходников (с заменой блоков на C/C++) —
[docs/REWRITING.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/REWRITING.md),
[docs/TOOLCHAIN.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/TOOLCHAIN.md);
однобайтовый патч гейта рандомизатора этим циклом —
[4.1/docs/DISTBCN-PATCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/4.1/docs/DISTBCN-PATCH.md).
Записи 100/101/102, стрип и «устаревший» CRC в обзоре —
[../chip/08-versions.md](../chip/08-versions.md).

## 1. Цикл правки `.fw`

```
[Ghidra: правка инструкций]
     │  arc_export_patch.py <orig.bin> <base> <overlay>      (диф против исходного сегмента)
     ▼
patch.overlay            строки "0xADDR: hexbytes"
     │  wil_syms.py patch wil6210.fw --overlay patch.overlay -o patched.fw
     ▼                                                      (пересчёт data_len и CRC)
patched.fw               CRC верный, записи 100/101/102 на месте
     │  fw_strip_for_openwrt.py patched.fw -o wil6210_openwrt.fw
     ▼                                                      (срез записей, пересчёт CRC)
wil6210_openwrt.fw       грузится мейнлайн-драйвером
```

Проверка (6.2.0.1000): правка `0x8c1000 ← e078e078` (`nop_s nop_s`) вручную
(`wil_syms.py patch --addr 0x8c1000 --bytes e078e078`) и через Ghidra
(`arc_export_patch` → overlay) даёт побайтово одинаковый результат (CRC обоих
0x2c8ed6be). `cmp -l` исходного и правленого образа: изменены ровно 4 байта кода
(смещение в файле 4168 = AHB 0x8c1000) и 4 байта поля CRC. После стрипа CRC
верен, правка на месте (413 984 Б, типы записей `[6,2,2,2,2,7,2,2,2,1]`).

Детали:

* Сегмент кода fw Sparrow — AHB **0x8c0000** (у wil6436 — 0x900000); исходный
  блоб для дифа — немодифицированный сегмент `seg_008c0000.bin`.
* При правке байтов поверх размеченной инструкции Ghidra требует `clearListing`
  перед `setBytes` (иначе `MemoryAccessException ... conflicts with instruction`).
* Правка в режиме `-readOnly` не сохраняется в проект — экспорт overlay без
  изменения размеченного проекта.

## 2. Формат образа и мейнлайн-драйвер

| образ | типы записей |
|---|---|
| мейнлайн (AD7200) `wil6210.fw` | 6, 2, 2, 2, 2, 7, 2, 2, 2 |
| linux-firmware `wil6210_mainline.fw` | 6, 2×4, 7, 2×3, 1×6 |
| образ из пакета RouterOS | 6, 2×4, 7, 2×3, 1, **100, 101, 102** |

`wil_fw.c::fw_handle_record()` мейнлайна знает типы 1, 2, 3, 5, 6, 7, 8, 9 и на
неизвестном возвращает `-EINVAL`. На OpenWrt `-EINVAL` бывает по двум причинам
**[железо]**:

| сообщение в dmesg | причина |
|---|---|
| `checksum mismatch` | CRC не сходится (`wil_fw_verify`, `fw_inc.c:104`) |
| `unknown record type: 100` | CRC верен, загрузка упёрлась в записи строк (`fw_inc.c:588`) |

Записи 100/101 — таблицы строк лога ucode и fw; в кольцо лога прошивка кладёт
только смещение строки, текст восстанавливает хост, поэтому рантайму строки не
нужны и срез безопасен.

Имена файлов, которые запрашивает драйвер: Sparrow D0 («plus») —
`wil6210_sparrow_plus.fw`, иначе `wil6210.fw`; board — `wil6210.brd`; всё в
`/lib/firmware` (запасное имя для D0 — патч 902,
[DRIVER-PATCHES-OPENWRT](DRIVER-PATCHES-OPENWRT.md)).

### CRC: не сломан, а устарел

Заголовочный CRC совпадает до и после стрипа на обеих прошивках:

| образ | CRC в заголовке | data_len |
|---|---|---|
| 6.2.0.1000 исходный | 0xc6a20f01 (не сходится) | 561 252 |
| 6.2.0.1000 после стрипа | 0xc6a20f01 (сходится) | 413 984 |
| 4.1.0.1000 исходный | 0x4498952f (не сходится) | 476 552 |
| 4.1.0.1000 после стрипа | 0x4498952f (сходится) | 347 520 |

`data_len` входит в область подсчёта CRC (`wil_fw_verify`: CRC по заголовку
записи, заголовку файла с обнулённым полем crc и остатку на длину `data_len`).
Порядок сборки у вендора: образ из типов 1, 2, 6, 7 с корректными `data_len` и
CRC, затем дописаны записи строк и обновлён только `data_len`. Стрип возвращает
ровно тот образ, который вендор подписал.

Очищенный образ 6.2.0.1000: 413 984 Б, 10 записей `[6,2,2,2,2,7,2,2,2,1]`,
CRC 0xc6a20f01, комментарий «FW version: 6.2.0.1000»; загружаемые адреса
0x8c0000, 0x900000, 0x920000, 0x940000, AGC 0x88a208 / 0x88a608 / 0x88a004;
sha256 `31ccfc2cd5710756d6ed409deed5f322d12f85e35d426dd13e5b32cb012ee7c1`.

## 3. Цикл board-файла

* `wil_brd.py unpack → правка файла записи → pack` (пересчёт CRC). На стандартном
  0126-контейнере (board сборки UBNT) цикл побайтово обратим, CRC верен.
* Board-файлы `wap60g` из пакетов RouterOS `wil_brd.py` разбирает с ошибкой
  `record 2 overruns data_len` — особенность контейнера. Board-файлы wil6436 для
  wAP 60G — формат `nv::message` (другой чип).
* Секция РЧ-регистров и рантайм-патчер RouterOS — [BRD-RF-REGS](BRD-RF-REGS.md).

## Инструменты

| инструмент | роль |
|---|---|
| `SparRAW-tools/re/ghidra/arc_export_patch.py` | диф памяти Ghidra против блоба → overlay |
| `SparRAW-tools/re/wil_syms.py patch` | наложить overlay или байты на `.fw`, пересчитать CRC |
| `SparRAW-tools/re/fw_strip_for_openwrt.py` | срез записей 100/101/102, пересчёт CRC |
| `SparRAW-tools/re/wil_brd.py unpack/pack/info/fixcrc` | контейнер `.fw`/`.brd` |

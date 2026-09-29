# Board-файл wil6210: секция РЧ-регистров и патчер RouterOS

Разбор userspace-бинаря RouterOS `nova/bin/wireless` (RouterOS 6.46.4, ARM ELF
без символов; Ghidra headless, скрипт `SparRAW-tools/re/ghidra/ros_brdpatch.py`;
смещение в файле = vaddr − 0x8000). **[код]** — по этому бинарю и board-файлам.

Контейнер board-файла, секции RX/TX-секторов и их обработка драйвером RouterOS —
[docs/HW-DRIVERS.md §9.6](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md);
обзор board-данных и соответствие модель → brd —
[../chip/01-architecture.md](../chip/01-architecture.md),
[../chip/07-beamforming-link.md](../chip/07-beamforming-link.md).

## Где логика

* `FUN_000b9fe0` — выбор board-файла по модели и сбор списка правок.
* `FUN_0003f234` — ядро патчера (строка `patchBoardFileRFRegs size:`); вызывается
  четыре раза с ключами секций `0xb000900d`, `0xc001900f`, `0xc004900f`,
  `0xc00c900f`. В board-файлах Sparrow есть только первая.

## Формат секции `0xb000900d`

```
<key:u32> <key:u32> <hdr:u32> <0xdeadbeef> (<reg:u32> <val:u32>)*
```

* ключ продублирован (поиск 8-байтовым `memcmp`);
* число слов таблицы = `(hdr & 0xfffff) >> 8`, пара — 2 слова;
* обход прекращается на `reg >= 0x10000`.

`wil6210-wap60g-60deg.brd`: секция по смещению 0xa0, `hdr = 0xfc21ba00` →
442 слова = 221 пара, данные с 0xb0, конец 0x798 (совпадает с полем по +0x98).

## Что патчит RouterOS

Три источника, каждый от своего поля конфигурации:

```c
/* txPower (+0xf8) */
reg 0x1c = tx | 0x8000  | tx<<8 | tx<<4
reg 0x5c = tx | 0x40000 | tx<<4
/* rxPower (+0x100), o = 0x28..0x3c с шагом 4 */
reg o      = rx | 0x8000  | rx<<4
reg o+0x40 = rx | 0x10000 | rx<<4
/* rfRegs (+0xe0 флаг, вектор пар +0xe4..+0xe8) — произвольные <reg,value> */
```

`tx` и `rx` — 4-битные (0…15).

## Вендорские board-файлы уже на максимуме

Во всех трёх board-файлах RouterOS (60deg, omni, lhg-span2.8) `tx = rx = 15`:

```
reg 0x1c = 0x08fff   reg 0x5c = 0x400ff
reg 0x28..0x3c = 0x080ff   reg 0x68..0x7c = 0x100ff
```

Рантайм-патчер RouterOS — регулятор мощности вниз (по настройке tx-power) и
отладочный ввод произвольных РЧ-регистров, а не дополнение калибровки.

Board-файл linux-firmware слабее и по другой сетке: `0x5c = 0x40099` (tx = 9),
`0x28 = 0x08035`, `0x68 = 0x13059`; часть значений в формулу RouterOS не
ложится (другая референсная плата). Вместе с другим антенным кодбуком (байты
2048–3584) это объясняет более низкое качество сигнала со стоковым board-файлом.

## Инструмент

`SparRAW-tools/re/wil_brd_regs.py` — разбор секции и правка по методу RouterOS:

```
python3 SparRAW-tools/re/wil_brd_regs.py <file.brd>          # дамп патчуемых регистров
routeros_patches(tx=, rx=, rfregs=[(reg,val),...])           # сформировать правки
apply_patches(data, patches)                                 # применить, размер сохраняется
```

Проверка формул: `tx = 9, rx = 9` на вендорском файле дают `reg 0x5c = 0x40099` —
значение стокового board-файла.

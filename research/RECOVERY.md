# Откат и восстановление узла при опытах с прошивкой

Что можно испортить подменой прошивки wil6210 на узле с OpenWrt и как вернуть сток.
Разбор — по коду драйвера backports-6.18.26 (`drivers/net/wireless/ath/wil6210/`).
Инструменты — SparRAW-tools `host/wil_stock_snapshot.sh`, `host/wil_fw_swap.sh`,
`host/netboot.sh` (восстановление через Etherboot RouterBOOT), `host/tapo_plug.py`
(питание узлов).

## Риск по коду драйвера **[код]**

* Прошивка в устройство не записывается: `wil_reset()` → `wil_request_firmware(wil,
  wil->wil_fw_name, true)` — обычный `request_firmware` из ФС хоста (`main.c:1722`). Имена
  `wil6210.fw` и `wil6210.brd` (`wil6210.h:39,49`); у wil6436 — `wil6436.fw`/`.brd`.
* В OTP и флеш радиомодуля драйвер не пишет: по всему коду только чтения
  `wil_r(wil, RGF_USER_OTP_HW_RD_MACHINE_1)`.
* При неудачной загрузке — `goto out`, сброс не доводится, CPU радио остаётся
  остановленным; хост жив.
* Параметра модуля с именем прошивки в мейнлайне нет (только `ftm_mode` →
  `wil6210_ftm.fw`), переключение делается подменой файла. (Запасное имя образа
  добавляет патч SparRAW-driver `902-wil6210-fw-name-fallback`.)

Правкой прошивки радиомодуль не испортить. Реальный риск — потеря доступа к узлу, если
управление идёт через сам 60-ГГц линк.

Затрагивают энергонезависимую память и в опытах не используются: запись board-файла в
устройство (`wiburn`), полная перепрошивка RouterOS/OpenWrt, переполнение overlay
(в OpenWrt `/lib/firmware` лежит в read-only squashfs, запись уходит в overlay JFFS2).

Поскольку `/lib/firmware` перекрыт overlay, `firstboot && reboot` возвращает стоковую
прошивку автоматически (ценой настроек).

## Порядок работы

### Снимок стока — один раз до опытов

```sh
./wil_stock_snapshot.sh                         # на узле
scp -O root@<узел>:/root/wil-stock.tar.gz .     # с рабочей машины
```

Копии в `/root/wil-stock/` + `MANIFEST` (sha256, размеры, версия прошивки из dmesg,
параметры модуля) и тарболл. Повторный запуск запрещён (эталоном стал бы уже
пропатченный файл), только с `--force`. Снимок с узла важнее офлайн-копий: только он
гарантированно соответствует экземпляру (особенно board-файл — вариант антенны).

### Подмена с автооткатом (commit/confirm)

```sh
./wil_fw_swap.sh install /tmp/<образ>.fw 300   # поставить, перезагрузить драйвер, откат через 300 с
./wil_fw_swap.sh status                        # что стоит, взведён ли таймер
./wil_fw_swap.sh confirm                       # связь жива — снять откат
./wil_fw_swap.sh revert                        # вернуть сток немедленно
```

Таймер запускается через `setsid` и переживает разрыв ssh. Проверено в песочнице
(подменённые `FWDIR`/`STOCK`): по истечении таймера файл возвращается к эталонной сумме,
pid-файл снимается, в `/tmp/wil_fw_swap.log` — `ТАЙМЕР: откат`; `confirm` снимает таймер;
`revert` восстанавливает эталон.

### Потерян доступ

1. Ethernet-порт узла (у wAP 60G есть) — основной путь.
2. `firstboot && reboot` через Ethernet или консоль.
3. UART-консоль.
4. Сетевая загрузка RouterBOOT (`netboot.sh`) и полная перепрошивка OpenWrt/RouterOS.

Если 60-ГГц линк — единственный путь управления, автооткат помогает, но Ethernet или
консоль на время опытов обязательны. Правило стенда — перезапуск обоих узлов питанием
перед каждым опытом ([6.2/docs/BENCH.md](https://github.com/klukonin/SparRAW-firmware/blob/main/6.2/docs/BENCH.md)).

## Офлайн-эталоны

Каталог `blobs/stock/` SparRAW-firmware (контрольные суммы — `blobs/README.md`,
`4.1/ref/BLOBS.md`):

| файл | назначение |
|---|---|
| `wil6210_5.2.0.18_stock.fw` | эталон отката — штатная linux-firmware |
| `wil6210_4.1.0.1000_stock_raw.fw` | FW 4.1.0.1000 из RouterOS 6.42.1, вендорский формат |
| `wil6210-wap60g-{omni,60deg,sa-dir,sa-omni}.brd` | board-файлы wAP 60G (из npk) |

Любая версия прошивки достаётся из пакетов RouterOS (`wireless-*.npk`, `routeros-*.npk`)
через `SparRAW-tools/re/mikrotik_npk.py`, `mikrotik_decrypt.py --auto`. Соответствие
модели узла и board-файла — [HW-DRIVERS.md](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/HW-DRIVERS.md).

## Не установлено

* Перезагрузка модуля (`rmmod`/`modprobe wil6210`) внутри `wil_fw_swap.sh` на железе не
  проверялась.

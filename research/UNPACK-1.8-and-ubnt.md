# Распаковка образов cnWave 1.8 и UBNT (Wave, airFiber 60)

Контейнеры трёх продуктов с 60-ГГц радио, команды распаковки и что внутри с точки
зрения радио wil6210/wil6436.

## Cambium cnWave 1.8

Файлы `cnwave60ghz-v{1000,2000,5000}-upgrd-1.8.img`: собственный заголовок `W60x`
(x — платформа: i = ipq40xx, c = ipq60xx, q = qoriq), 0x60 байт, далее tar → U-Boot FIT
→ ядро + dtb + LZMA-ramdisk (cpio).

```
tail -c +97 cnwave60ghz-v1000-upgrd-1.8.img | tar -x      # fitImage + MANIFEST + u-boot
dumpimage -T flat_dt -p 2 -o ramdisk.bin fitImage-*        # ramdisk = LZMA cpio
lzma -dc ramdisk.bin | cpio -idmu                          # rootfs
```

Платформы: V1000 — ipq40xx, V2000 — ipq60xx, V5000/V3000 — qoriq.

Содержимое — инкрементальный Terragraph с архитектурой 1.2:
* радио: FW **10.11.4.92** (в 1.2 — 10.11.4.87), тот же TDMA-MAC; образ
  `wil6210-IF2IF-10.11.4.92-cnwave1.8.fw`;
* хостовый mesh-стек тот же: `openr`, `e2e_controller/minion`, `fib_linux`, `pop`,
  `terragraph-qca.ko`; git-метки `FRIEZA/qca6430-tg`, «1.8 sit 3 build»;
* новое: платформа V2000 (ipq60xx), `migrate_topology_config.py`.

## UBNT Wave (airOS 10) — PtP/PtMP

`GMC.ipq5018.v4.1.1...bin`: заголовок Ubiquiti (264 Б) + подпись RSA-2048 в хвосте; тело —
LZMA, внутри tar → `rootfs.squashfs` (lz4).

```
binwalk GMC.ipq5018...bin                                  # LZMA по смещению 0x144
tail -c +325 GMC.ipq5018...bin | lzma -dc > system.bin
tar -xf system.bin                                         # rootfs.squashfs (+ gpt, u-boot)
unsquashfs rootfs.squashfs
```

* Продукт: airFiber 60 LR/XR (`af60-lr`, `af60-xr`), хост IPQ5018, радио-компаньон 5 ГГц
  QCN6122 (`bdwlan.b60`).
* Демонов маршрутизации и меша (openr/babel/olsr/batman/e2e) нет — точка-точка /
  точка-многоточка. Синхронизация — `gps-reader` (проприетарный GPS-TDMA).
* 60-ГГц радио не wil6210: строк `wil6210/wigig` и имени соседнего чипа wil6436 в rootfs
  нет; инструменты `athstats`/`wlanconfig` (ветка QCA WLAN), board-конфиги своего формата
  `@Config 113-00738.13s.Ver4` (`lib/firmware/prs/config_af60*`), не `.brd`.

## UBNT airFiber 60 (GBE) — Sparrow

`GBE.v1.5.1.3f54e8e6.250725.1224.bin`, платформа ipq806x, airOS. Заголовок Ubiquiti +
подпись RSA-2048; разделы `PARTu-boot` (ELF), `PARTkernel` (FDT + LZMA), `PARTrootfs`
(squashfs xz, 10 МБ, 920 inode).

```
binwalk GBE...bin                          # squashfs по 0x2501BC
python3 -c "b=open('GBE...bin','rb').read(); open('r.sqfs','wb').write(b[0x2501BC:0x2501BC+10198137])"
unsquashfs r.sqfs
```

* Прошивка радио — **`wil6210_sparrow_plus.fw`, FW 6.2.0.225** (Sparrow+). `wil_brd.py`
  разбирает её штатно: CRC сходится, 10 записей. Рядом board-файлы `wil6210.brd`,
  `wil6210_lr.brd`, `wil6210_plus.brd` (читаются `wiburn.py`). Сравнение с 6.2.0.1000 —
  [UBNT-vs-ROUTEROS-6.2.md](UBNT-vs-ROUTEROS-6.2.md).
* Управление — стандартный nl80211: `hostapd_60g driver=nl80211 hw_mode=ad channel=2`
  (AP), `wpa_supplicant_60g -D nl80211` (STA).
* Тракт данных — проприетарный модуль UBNT **`prs_falcon`** (режим PtP-моста,
  `PRS_IS_PTP`, `PRS_OPMODE` ap/sta, привязка TX/RX к ядрам CPU) и `ubond`: тракт
  mainline-драйвера заменён, прошивка радио и управление остались штатными.
* Топология — PtP (`is_ubnt_ptp=1`), демонов маршрутизации нет.

Карта памяти Sparrow отличается от wil6436: символы wil6436 к 6.2 не подходят; контейнер,
board-файлы, кодек WMI и ARCompact — общие.

## Сводка

| | Cambium cnWave | UBNT Wave (af60) | UBNT airFiber 60 (GBE) |
|---|---|---|---|
| SoC хоста | ipq40xx/60xx/qoriq | ipq5018 | ipq806x |
| радио 60 ГГц | wil6436 10.11 | не wil6210 | Sparrow+ (wil6210) 6.2 |
| топология | **mesh** | PtP/PtMP | PtP |
| маршрутизация | Open/R (хост) | нет | нет |
| тракт данных | terragraph-qca + VPP | проприетарный | `prs_falcon` (мост) |
| управление | e2e/TG_SB | собственное | nl80211/hostapd_60g |
| инструменты wil6210/wil6436 применимы | да (wil6436) | нет | да (Sparrow) |

Меш строит только Cambium/Terragraph (TDMA-MAC прошивки 10.x). Обе линейки UBNT — мосты
PtP; GBE ближе всего к wil6210, но тоже не меш.

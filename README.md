# SparRAW-docs

Описание чипа Qualcomm/Wilocity **Sparrow** (wil6210, 60 ГГц 802.11ad),
восстановленное реверсом прошивок 4.1.0.1000 и 6.2.0.1000 и опытами на
стенде (два MikroTik wAP 60G под OpenWrt).

## Главы (`chip/`)

| глава | о чём |
|---|---|
| [01 — Архитектура](chip/01-architecture.md) | два ядра ARC600 (fw и ucode), карта памяти и адреса хоста, gp, милли-код, форматы `.fw`/`.brd`, загрузка, планировщик, структуры, автоматы |
| [02 — Регистры](chip/02-registers.md) | карта периферии, прерывания (RGF_ICR), USER_RGF, регистровый файл MAC r36..r56, MAC_SXD (IFS, TBTT, TSF), регистры без доказанного смысла |
| [03 — MAC](chip/03-mac.md) | кольцо команд core write, пространство кодов, расшифрованные коды, такт 165 МГц и IFS, ожидания ucode, GP-таймеры |
| [04 — Прошивка ↔ микрокод](chip/04-fw-ucode.md) | почтовый ящик, команды LMAC и события ucode, диспетчеры, автоматы `basic_sm`, таблицы в ОЗУ |
| [05 — Интерфейс с хостом](chip/05-host-interface.md) | PCIe и окна памяти, WMI, нештатные смыслы команд, рычаги с хоста без патча, лог и трасса, грабли |
| [06 — Биконинг](chip/06-beaconing.md) | BI (BTI/A-BFT/AW/DTI), автомат BI, таблицы расписания, распределённый биконинг и рандомизатор, NAV-отсрочка, IBSS, Clustering, P2P |
| [07 — Беамформинг и линк](chip/07-beamforming-link.md) | SLS и ролевой гейт, A-BFT и OOB, BRP и quasi-omni, секторы, ассоциация, безролевой линк, PBSS, board-файлы, пропускная способность |
| [08 — Версии](chip/08-versions.md) | корпус образов, отличия 4.1 и 6.2, корреляция, тулчейн, состояние реверса |

У каждого факта в главах есть пометка версии и ссылка на документ прошивки
или исследование. Расхождения источников собраны в конце каждой главы в разделе
«Противоречия».

## Документация прошивки

Устройство кода, данных, регистров и протоколов по версиям описано в репозитории
прошивки: [SparRAW-firmware/docs](https://github.com/klukonin/SparRAW-firmware/blob/main/docs/README.md).
Главы `chip/` дают обзор и ссылаются на неё за подробностями.

## Исследования (`research/`)

Материал, которого нет в документации прошивки: сопоставление со стандартом и патентом,
опыты на стенде, корпус образов и вендоров, драйверы.


### Биконинг и окна BI

| документ | о чём |
|---|---|
| [CROSS-VERSION-BEACON](research/CROSS-VERSION-BEACON.md) | Межверсионное сравнение тракта биконинга |
| [RANDOMIZER-ROOT-CAUSE](research/RANDOMIZER-ROOT-CAUSE.md) | Рандомизатор отсрочки маяка 4.1: опыты на железе |
| [ABFT-WINDOW](research/ABFT-WINDOW.md) | Окно A-BFT и таблицы расписания BI (4.1.0.1000) |
| [AW-WINDOW](research/AW-WINDOW.md) | Окно AW: измерения и последствия снятия (4.1.0.1000) |
| [MACMON-SCHEDULE](research/MACMON-SCHEDULE.md) | Расписание интервала маяка по отчёту MAC_MON: измерения на стенде |
| [BAD-BEACONS-DETECTOR-4100](research/BAD-BEACONS-DETECTOR-4100.md) | Детектор плохих маяков: решающее правило (4.1.0.1000) |
| [PATENT-US8520648-SPEC](research/PATENT-US8520648-SPEC.md) | Патент US8520648B2: распределённый биконинг в направленных сетях |


### Одноранговая связь, IBSS/PBSS, mesh

| документ | о чём |
|---|---|
| [IBSS-ON-HW](research/IBSS-ON-HW.md) | IBSS/adhoc на wil6210: устройство и опыт на стенде |
| [PBSS-ON-HW](research/PBSS-ON-HW.md) | Одноранговая сеть PBSS на стенде (4.1.0.1000) |
| [ADHOC-CONNECT-PATH](research/ADHOC-CONNECT-PATH.md) | Типы сети ADHOC в пути connect и старта PCP (6.2.0.1000) |
| [DISCOVERY-PEER-TABLE-4100](research/DISCOVERY-PEER-TABLE-4100.md) | Таблица соседей при обнаружении (4.1.0.1000) |
| [WILOCITY-DORMANT-MESH](research/WILOCITY-DORMANT-MESH.md) | Одноранговый режим Wilocity: ABI и логика есть, включение отсутствует |
| [WHY-MESH-ABANDONED](research/WHY-MESH-ABANDONED.md) | Почему распределённый режим Wilocity не стал продуктом |
| [TERRAGRAPH-MESH-LAYERS](research/TERRAGRAPH-MESH-LAYERS.md) | Слои mesh-сети cnWave / Terragraph |
| [DIFF-7.5-vs-10.11](research/DIFF-7.5-vs-10.11.md) | Что Terragraph добавил в прошивку радио: диф 7.5 → 10.11 |
| [ESE-TDMA-4100](research/ESE-TDMA-4100.md) | ESE-TDMA в прошивке 4.1.0.1000 |


### Прошивка 4.1: дополнения к документации прошивки

| документ | о чём |
|---|---|
| [UCODE-EVENT-BITS](research/UCODE-EVENT-BITS.md) | Биты событий ucode 4.1 (r42, r54), команда MAC 0x27 и примитивы ожидания |
| [IE-LAYER-AND-AW-4100](research/IE-LAYER-AND-AW-4100.md) | Слой IE, события состояния системы и приём окна AW в fw 4.1.0.1000 |
| [DATAPATH-4100](research/DATAPATH-4100.md) | Тракт данных fw 4.1.0.1000: дополнения |
| [RF-SECTOR-TABLE](research/RF-SECTOR-TABLE.md) | Таблица РЧ-секторов в прошивке 4.1.0.1000 |
| [UT-COMMANDS](research/UT-COMMANDS.md) | Диагностические команды UT прошивки 4.1: код → функция |
| [WMI-PCP-FACTOR](research/WMI-PCP-FACTOR.md) | Событие WMI_PCP_FACTOR_EVENTID (0x191A) в прошивке 4.1 |
| [WMI-PCP-START-LAYOUT](research/WMI-PCP-START-LAYOUT.md) | WMI_PCP_START в прошивке 4.1: раскладка и поле abft_len |
| [COVERAGE-CENSUS-4100](research/COVERAGE-CENSUS-4100.md) | Атрибуция кода 4.1.0.1000: проверенные правила и динамическое покрытие |
| [FW-FUNCTION-NAMES](research/FW-FUNCTION-NAMES.md) | Имена функций fw из лог-строк: метод и свойства |


### Образы, версии, вендоры

| документ | о чём |
|---|---|
| [FIRMWARE-CORPUS](research/FIRMWARE-CORPUS.md) | Корпус образов прошивки wil6210 |
| [FIRMWARE-COMPARISON](research/FIRMWARE-COMPARISON.md) | Прошивка wil6210 в пакетах RouterOS: версии и контейнер |
| [UBNT-vs-ROUTEROS-6.2](research/UBNT-vs-ROUTEROS-6.2.md) | Сборки 6.2 двух производителей: UBNT 6.2.0.225 и RouterOS 6.2.0.1000 |
| [NPK-ENCRYPTION](research/NPK-ENCRYPTION.md) | Пакеты RouterOS (npk): распаковка и шифрование fw/brd wil6210 |
| [UNPACK-1.8-and-ubnt](research/UNPACK-1.8-and-ubnt.md) | Распаковка образов cnWave 1.8 и UBNT (Wave, airFiber 60) |
| [BRD-RF-REGS](research/BRD-RF-REGS.md) | Board-файл wil6210: секция РЧ-регистров и патчер RouterOS |
| [BUILD-PIPELINE](research/BUILD-PIPELINE.md) | Точечная правка образа и приведение к формату мейнлайна |
| [SELFBUILT-FIRMWARE](research/SELFBUILT-FIRMWARE.md) | Прошивка 4.1 из собственного дерева: устройство и проверка на железе |


### Драйвер, стенд, наблюдение

| документ | о чём |
|---|---|
| [DRIVER-PATCHES-OPENWRT](research/DRIVER-PATCHES-OPENWRT.md) | Драйверные патчи wil6210 для OpenWrt: база, метод, первые патчи |
| [DRIVER-IBSS-PATCH](research/DRIVER-IBSS-PATCH.md) | Драйверная часть IBSS поверх PBSS: спецификация правок wil6210 |
| [DRIVER-CORPUS-IBSS-SEARCH](research/DRIVER-CORPUS-IBSS-SEARCH.md) | Корпус драйверов wil6210: IBSS не реализован ни в одном |
| [ORIGINAL-DRIVER-3.8](research/ORIGINAL-DRIVER-3.8.md) | Первый драйвер wil6210 (Linux 3.8, 2012) и тип сети ADHOC |
| [TRACING-TOOLKIT](research/TRACING-TOOLKIT.md) | Трассировка wil6210 на стенде: лог прошивки, трасса ucode, команды WMI |
| [OBSERVABILITY](research/OBSERVABILITY.md) | Наблюдение за прошивкой с хоста |
| [LINK-THROUGHPUT](research/LINK-THROUGHPUT.md) | Пропускная способность 60-ГГц линка и цена снятия окна AW |
| [RECOVERY](research/RECOVERY.md) | Откат и восстановление узла при опытах с прошивкой |

## Соседние репозитории

`SparRAW-firmware` (прошивки из исходников), `SparRAW-driver` (драйвер для
OpenWrt), `SparRAW-tools` (инструменты и стенд).

## Лицензия

Тексты — Creative Commons Attribution 4.0 International ([LICENSE](LICENSE)).

---
layout: post
title:  "Integracja uC/OS-III z projektem STM32CubeMX"
date:   2021-07-01 17:00:00 +0200
categories:
  - Embedded
tags:
  - STM32
  - uC/OS-III
---

W niniejszym samouczku przygotujemy szablon projektu STM32CubeMX do pracy z
systemem uC/OS-III.

> English version is in progress.

### Motywacja

Kod źródłowy systemu uC/OS-III został udostępniony na licencji Apache v2.0 na
początku 2020 roku i od tego czasu nie zyskał dużej popularności. Moim
zdaniem mogłoby to ulec zmianie, ponieważ system ten jest wyjątkowo dobrze
opracowany i ma dużą wartość naukową. Niestety w związku z niską
popularnością systemu w Internecie brakuje materiałów ułatwiających
rozpoczęcie z nim pracy.

Celem niniejszego samouczka jest wypełnienie tej luki poprzez wykorzystanie
darmowego i niekomercyjnego oprogramowania do konfiguracji projektów
STM32CubeMX oraz w pełni darmowego i otwartoźródłowego zestawu narzędzi
kompilacyjnych GNU GCC oraz GNU Make.

---

## Wygenerowanie projektu STM32CubeMX

Skonfigurujmy projekt wedle uznania i potrzeb. Ja swój projekt wykonuję na
płytce z STM32F411CE, ale większość instrukcji w niniejszym samouczku powinno
być niezależne od konkretnego mikrokontrolera.

**Kluczowy krok, o którym trzeba pamiętać**: zmieniamy źródło zegara
systemowego dla HAL. W innym wypadku powstaną konflikty i przestaną działać
funkcje oparte o opóźnienia (np. pooling).

W zakładce `System Core > SYS` zmieniamy `Timebase Source` na dowolny inny niż
`SysTick`.

![Timebase Source](/assets/timebase-source.png)

W zakładce `Project Manager` zmieniamy `Toolchain/IDE` na `Makefile`.

![Toolchain/IDE](/assets/toolchain-ide.png)

## Pobranie kodu źródłowego uC/OS-III

*Krok opcjonalny*: Otwieramy okno terminala w katalogu nowoutworzonego
projektu. Zakładamy nowe repozytorium gita i dodajemy wszystkie pliki
źródłowe.

{%- highlight bash -%}
git init
echo "build" >> .gitignore
git add .gitignore
git add *
git commit -m "Initial commit."
{%- endhighlight -%}

Następwnie dodajemy pliki źródłowe uC/OS-III jako podmoduły.

{%- highlight bash -%}
cd Middleware
git add submodule https://github.com/weston-embedded/uC-OS3.git
git add submodule https://github.com/weston-embedded/uC-CPU.git
git add submodule https://github.com/weston-embedded/uC-LIB.git
{%- endhighlight -%}

Wszystko powyższe możemy wykonać ręcznie i nie ma konieczności użycia gita,
jednakże dodanie kodu źródłowego uC/OS-III jako podmodułów ułatwi w
przyszłości pracowanie z różnymi wersjami.

## Modyfikacje kodu źródłowego

W dalszej części samouczka zakładamy, że katalog projektu ma następującą
hierarchię:

```
f411ce_ucos/
├── build
├── Core
│   ├── Inc
│   └── Src
├── Drivers
│   ├── CMSIS
│   └── STM32F4xx_HAL_Driver
└── Middleware
    ├── uC-CPU
    ├── uC-LIB
    └── uC-OS3
```

### Plik startowy

W pliku startowym zmieniamy nazwy funkcji obsługujących przerwania wewnętrzne
`PendSV` oraz `SysTick`. W tym celu odszukujemy fragment odpowiadający za
przypisania przerwań; będzie to wyglądać mniej wiecej w ten sposób:

![pfnVectors](/assets/pfn_vectors.png)

![weakAliases](/assets/weak_aliases.png)

### Pliki szablonowe

Kod źródłowy uC/OS-III dostarczany jest z plikami szablonowymi zawierającymi
podstawową konfigurację systemu, która we większości przypadków będzie
wystarczająca. Pliki te zazwyczaj znajdują się w folderach `Template/`.

```
Middleware/
├── uC-CPU
│   └── Cfg
│       └── Template
|           └── cpu_cfg.h
├── uC-LIB
|   └── Cfg
│       └── Template
|           └── lib_cfg.h
└── uC-OS3
    ├── Cfg
    |   └── Template
    |       ├── os_app_hooks.c
    |       ├── os_app_hooks.h
    |       ├── os_cfg_app.h
    |       └── os_cfg.h
    └── Template
        └── bsp_os_dt.c
```
Koniecznymi z punktu widzenia systemu plikami konfiguracyjnymi są `cpu_cfg.h`,
`lib_cfg.h`, `os_cfg_app.h` i `os_cfg.h`. Zawierają niezbędne makra
konfigurujące funkcjonalność systemu, min. częstotliwość taktowania systemu
`OS_CFG_TICK_RATE_HZ`, ilość poziomów priorytetów `OS_CFG_PRIO_MAX`, a także
dostępne mechanizmy (np. semafory, timery, kolejki, `OS_CFG_XXX_EN`).

Plik `bsp_os_dt.c` jest potrzebny w przypadku wykorzystania platformy o
zmiennej częstotliwości taktowania procesora i o ile nie zamierza się
korzystać z tej funkcjonalności można go pominąć (w pliku `os_cfg.h` makro
`OS_CFG_DYN_TICK_EN`, domyślnie wyłączone). Podobnie z plikami
`os_app_hooks.c` i `os_app_hooks.h`: służą definiowaniu własnych uchwytów do
funkcji systemowych, więc są opcjonalne (makro `OS_CFG_APP_HOOKS_EN`,
domyślnie włączone).

Pozostałe pliki nagłówkowe i źródłowe przekopiowujemy odpowiednio do folderów
`Core/Inc` i `Core/Src`. Warto szczegółowo zapoznać się z ich zawartością
tak, aby wiedzieć za co odpowiadają. W razie potrzeby możemy je dostosować.

### Board Support Package

Pliki `bsp.c` i `bsp.h` nie są wymagane bezpośrednio do pracy systemu, ale
wspomagają tworzenie aplikacji z wykorzystaniem danej platformy prototypowej.
Jako że w niniejszym samouczku korzystamy z STM32HAL i STM32CubeMX większość
konfiguracji do pracy z naszym mikrokontrolerem zostało już wygenerowane
automatycznie, więc pliki BSP nie będą omawiane. Zainteresowanych odsyłam
bezpośrednio do dokumentacji systemu uC/OS-III, [podrozdział 18-4][bsp docs].

### main.h i main.c

W pliku `main.h` dołączamy plik nagłówkowy systemu uC/OS-III:

![main.h](/assets/main_h.png)

W pliku `main.c` podstawowymi zmianami, które musimy wykonać, są:

1. Dodać definicje stałych (opcjonalne, ale zalecane)

![Private Defines](/assets/pdefine.png)

2. Dodać definicję funkcji stanowiącej ciało głównego zadania

![Private Function Prototypes](/assets/pfprototypes.png)

3. Dodać TCB głównego zadania oraz zaalokować obszar stosu zadania.

![Private Variables](/assets/pvariables.png)

4. Dostosować ciało funkcji `main` do pracy z uC/OS-III.

![Main Function](/assets/mainfunc.png)

5. Zaimplementować funkcję stanowiącą ciało głównego zadania.

![AppTaskStart Function](/assets/apptaskstartfunc.png)

### Makefile

W sekcji `C sources` dopisujemy nowododane pliki źródłowe `.c` z folderu
`Core/Src`, a także pliki źródłowe uC/OS-III z folderów `Middleware/uC-CPU/`,
`Middleware/uC-LIB/`, `Middleware/uC-OS3/Source/`. Ponadto dopisujemy pliki
przenośne, które zależne są od architektury wykorzystanego mikrokontrolera.
Przykładowo dla STM32F4 będą to pliki
`Middleware/uC-CPU/ARM-Cortex-M/ARMv7-M/cpu_c.c` i
`Middleware/uC-OS3/Ports/ARM-Cortex-M/ARMv7-M/os_cpu_c.c`.

W sekcji `ASM sources` dopisujemy pliki przenośne, które zależne są zarówno od
mikrokontrolera, jak i użytego kompilatora - w naszym przypadku jest toGNU
GCC. Kontynuując przykład z STM32F4 interesują nas pliki
`Middleware/uC-OS3/Ports/ARM-Cortex-M/ARMv7-M/cpu_a.s`,
`Middleware/uC-LIB/Ports/ARM-Cortex-M4/GNU/lib_mem_a.s` i
`Middleware/uC-OS3/Ports/ARM-Cortex-M/ARMv7-M/GNU/os_cpu_a.s`

W sekcji `C includes` dopisujemy foldery z plikami nagłówkowymi systemu uC/
OS-III, bibliotek oraz plików przenośnych. Przykładowo dla STM32F4:

![C Includes](/assets/c_includes.png)

## Kompilacja

Tak przygotowany projekt możemy skompilować poleceniem `make`. W wyniku
otrzymamy plik `build/xxx.bin`, który można wgrać na mikrokontroler.

```
arm-none-eabi-size build/f411ce_ucos.elf
   text    data     bss     dec     hex filename
  13288      20    6676   19984    4e10 build/f411ce_ucos.elf
arm-none-eabi-objcopy -O ihex build/f411ce_ucos.elf build/f411ce_ucos.hex
arm-none-eabi-objcopy -O binary -S build/f411ce_ucos.elf build/f411ce_ucos.bin
```


[bsp docs]: https://micrium.atlassian.net/wiki/spaces/osiiidoc/pages/131426/Board+Support+Package

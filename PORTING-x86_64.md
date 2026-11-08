# План работ: Cgoct x86_64 (CactOS 3.0.0)

Форк от `2.0.0` (i686). `MAJOR` → 3 по правилу «`MAJOR` = смена архитектурного
поколения»: супервизор перелинковывается под ABI ядра 3.0.0 и libc 3.0.0
(`CactKernel-x86_64`, `CactLibc-x86_64`, решения 5/9 ядра, решение 5 libc).

Этот файл — план, а не обещание. Инвентарь снят по коду; при расхождении верить
коду.

## Что уже хорошо (и почему порт маленький)

Cgoct — чистый C без ассемблера и без прямых syscall: всё идёт через `clibc.so`
(`open`, `read`/`write`, `fork`/`execve`, `waitpid`, `nanosleep`). Поэтому
архитектурно-зависимы только **сборка и скрипт линковки**.

Код ширинонезависим: `long` держит миллисекунды, `(int)`-касты стоят над
`read`/`write` и `pid` — в 64 битах это не меняется. Своего `VERSION` и
`version.h` у Cgoct нет: версия живёт только в README.

## Инвентарь 32-битных поверхностей

- `cross/i686-cact-clang.ini` — `-m32`, `cpu_family=x86`, `cpu=i686`. В cactsole
  и libc уже переехал в `x86_64-cact-clang.ini` (`-m64`).
- `meson.build` — линковка `ld -m elf_i386 -pie`; заголовочный комментарий
  ссылается на `cross/i686-cact-clang.ini` и `CactLibc-x86_32`.
- `link.ld` — секции `.rel.dyn`/`.rel.plt` (в x86_64 они `.rela.*`); нет
  `.data.rel.ro`. База `0x08000000` **не двигается** (решения 5/9 ядра,
  решение 5 libc).
- `meson_options.txt` — дефолт `cactlib = ../CactLibc-x86_32`.
- `README.md` — бейджи и ссылки `i686` → `x86_64`.

## Решения

1. **Тулчейн как у cactsole и libc.** `cross/x86_64-cact-clang.ini` (`-m64`,
   `cpu_family=cpu=x86_64`), `ld -m elf_x86_64`, дефолт `cactlib` →
   `../CactLibc-x86_64`.
2. **База фиксированной линковки `0x08000000` не двигается** (решение 5 libc).
3. **Скрипт линковки под ELF64.** Добавляются `.rela.dyn`/`.rela.plt` и
   `.data.rel.ro`; пара `.rel.dyn`/`.rel.plt` остаётся против orphan-секций.

## Этапы

Порт Cgoct — **один этап**: меняются только тулчейн и скрипт линковки. Поэтому и
коммит один.

### Фаза 0 — тулчейн и скрипт линковки

`cross/x86_64-cact-clang.ini` вместо i686-файла; `meson.build` (`elf_x86_64`,
комментарии); `link.ld` (`.rela.*`, `.data.rel.ro`); `meson_options.txt`
(дефолт `cactlib` → `../CactLibc-x86_64`); `README.md`.

**Milestone:** `ninja -C build-meson` собирает `cgoct`; `readelf -h` даёт
`ELF64`/`EM_X86_64`, `readelf -r` — только поддерживаемые релокации (решение 4
libc).

## Проверки

- **Кросс-сборка** — `meson setup build-meson --cross-file
  cross/x86_64-cact-clang.ini -Dcactlib=../CactLibc-x86_64 && ninja -C
  build-meson`. Нужен собранный `CactLibc-x86_64/build-meson/{start.o,clibc.so}`.
- **Заголовок и релокации** — `readelf -h build-meson/cgoct`,
  `readelf -r build-meson/cgoct`.
- **На железе/в QEMU** — за ядром: `/usr/bin/init` (это cgoct) поднимает
  `/usr/bin/cactsole`.

## Открытые вопросы

- **Своего `VERSION` нет** — ни файла, ни макроса; версия только в README. При
  желании можно добавить `VERSION` = 3.0.0, но исходник её не несёт.
- **Образ — `ET_EXEC`, а не `ET_DYN`** (как и cactsole 2.0.0): `ld` при этом
  скрипте и `-pie` выдаёт `EXEC`, а ядро мапит по `PT_LOAD` без проверки
  `e_type`.

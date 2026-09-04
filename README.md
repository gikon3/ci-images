# ci-images

Сборочные образы для CI.

| образ | Dockerfile | пакет |
| --- | --- | --- |
| C++ на Arch Linux | [`Dockerfile.cpp-arch`](Dockerfile.cpp-arch) | `ghcr.io/gikon3/cpp-arch` |

---

## `cpp-arch`

Тулчейн для C++ проектов: свежие clang и gcc прямо из официальных репозиториев Arch.

### Что внутри

`clang` (он же `clang-tidy` и `clang-format`), `compiler-rt` (рантаймы asan/ubsan/tsan), `llvm` (`llvm-symbolizer` - без него трейсы санитайзеров нечитаемы), `gcc`, `cmake`, `ninja`, `git`, `zstd`, `python`.
Conan лежит отдельно, в venv `/opt/venv-conan`.

Точные версии образ носит с собой:

```console
$ docker run --rm ghcr.io/gikon3/cpp-arch:latest cat /etc/cpp-arch-release
snapshot=2026/09/04
clang=22.1.8
gcc=16.2.1
cmake=4.4.3
ninja=1.13.2
conan=2.32.0
```

Тот же снимок лежит в переменной окружения `CPP_ARCH_SNAPSHOT`.

### Профили Conan

`CONAN_HOME=/opt/conan-home`, в нём два профиля и симлинк:

| профиль | компилятор |
| --- | --- |
| `clang` | `clang` / `clang++` |
| `gcc` | `gcc` / `g++` |
| `default` | симлинк на `clang` |

`compiler.version` в них подставлена **при сборке образа** из мажорной части `-dumpversion` своего компилятора, поэтому разойтись с лежащим рядом бинарём она не может.

`compiler.cppstd=23` - умолчание образа. Стандарт выглядит делом проекта, но без него Conan отказывается считать граф вовсе.
Проект перебивает умолчание своим профилем или `-s compiler.cppstd=NN`.

Профиль проекта наследуется обычным `include`:

```ini
include(clang)

[settings]
build_type=Debug
```

Голое имя Conan сначала ищет в `$CONAN_HOME/profiles`, поэтому `include(clang)` подхватывает профиль образа, а не файл рядом.

### Пользователь и права

Образ работает от `builder` с **uid 1001** - тем же, что у пользователя `runner` на GitHub-раннерах, где рабочее дерево монтируется в `/__w`.
При несовпадении uid `actions/checkout` упёрся бы в права на чужой каталог.

`CONAN_HOME` вынесен из домашнего каталога и открыт всем (`a+rwX`), поэтому запуск под произвольным uid тоже работает:

```bash
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/w" -w /w \
    ghcr.io/gikon3/cpp-arch:latest conan install . --build=missing
```

Для санитайзеров стоит добавлять `--cap-add=SYS_PTRACE`: LeakSanitizer собирает живые указатели через `ptrace`.
На Docker 29.7.2 он работает и без этого флага - разрешает штатный seccomp-профиль, - но у более старых демонов и у ужатых профилей `ptrace` закрыт.

### Версии и расписание

База - rolling-дистрибутив, поэтому версии фиксирует снимок [Arch Linux Archive](https://archive.archlinux.org/): аргумент сборки `ARCH_SNAPSHOT=YYYY/MM/DD` переписывает `mirrorlist` на архив этой даты, и один и тот же аргумент даёт один и тот же тулчейн.

Conan идёт мимо снимка: в репозиториях Arch его нет, поэтому venv ставит пакет с PyPI.
Версию в публикуемом образе двигает workflow, каждый прогон берёт последний релиз.
Локальная сборка берёт умолчание из `ARG CONAN_VERSION` либо своё значение через `--build-arg CONAN_VERSION=`.

Дату двигает расписание - понедельник, 01:00 UTC (04:00 МСК). Каждый прогон публикует два тега:

- `ghcr.io/gikon3/cpp-arch:YYYYMMDD` - неподвижный, для отката;
- `ghcr.io/gikon3/cpp-arch:latest` - движется.

Хранятся три последние версии; осиротевшие (следы переездов `latest`) удаляются без ограничений.

Собрать вручную с другой датой - `workflow_dispatch` со входом «дата снимка» либо локально:

```bash
docker build -f Dockerfile.cpp-arch -t cpp-arch:test \
    --build-arg ARCH_SNAPSHOT=2026/09/04 .
```

---

## Лицензия

Репозиторий - под [MIT](LICENSE). На содержимое собранного образа она не распространяется: внутри лежат неизменённые пакеты Arch Linux - gcc под GPL-3.0, clang под Apache-2.0 с LLVM-исключением, остальные под своими лицензиями; исходники каждого лежат в том же снимке [Arch Linux Archive](https://archive.archlinux.org/), дату которого образ носит в `/etc/cpp-arch-release` и `CPP_ARCH_SNAPSHOT`.

# debian_sources

Источники apt в новом формате **deb822** (`/etc/apt/sources.list.d/*.sources`)
для Debian 13 «trixie» и новее. Роль по смыслу — то же, что `pacman` для Arch:
один список в host_vars, полностью отрендеренные файлы на хосте, никакого
ручного редактирования.

Начиная с Debian 13 однострочный `/etc/apt/sources.list` не используется:
установщик кладёт `debian.sources`, а вендоры (Proxmox, Docker, Grafana) —
свои `.sources` рядом. Формат построчный, с полями, и его удобно собирать
шаблоном.

## Что делает

- рендерит `<name>.sources` из списка `debian_sources_repositories`;
- умеет несколько секций в одном файле — штатный `debian.sources` держит
  основную ветку и security именно так;
- умеет `Enabled: no` — ровно этим Proxmox выключает `pve-enterprise`;
- по желанию убирает `.sources`, которых нет в списке (`prune`), и приводит
  в порядок старый `sources.list` после апгрейда с Debian 12;
- ставит `python3-apt`, без которого модуль `apt` в любой другой роли
  отказывается работать в режиме `--check`.

## Переменные

| Переменная | Default | Что |
|---|---|---|
| `debian_sources_repositories` | `[]` | список файлов, см. ниже. Пусто — роль ничего не делает |
| `debian_sources_dir` | `/etc/apt/sources.list.d` | куда класть |
| `debian_sources_prune` | `false` | удалять `.sources`, которых нет в списке |
| `debian_sources_prune_keep` | `[]` | имена файлов (без расширения), которые prune не трогает |
| `debian_sources_legacy_list` | `/etc/apt/sources.list` | путь к старому файлу |
| `debian_sources_legacy_state` | `keep` | `keep` \| `empty` \| `absent` |
| `debian_sources_update_cache` | `true` | `apt-get update` после изменений |
| `debian_sources_install_python_apt` | `true` | поставить `python3-apt` |

### Элемент списка

Каждый элемент — **один файл**. Ключи секции можно писать прямо в элементе
(короткая форма) либо списком `stanzas` (несколько секций в одном файле).

| Ключ | Обяз. | Что |
|---|---|---|
| `name` | да | имя файла без расширения |
| `state` | нет | `present` (по умолчанию) или `absent` — удалить файл |
| `uris` | да | строка или список |
| `suites` | да | строка или список |
| `types` | нет | по умолчанию `['deb']` |
| `components` | нет | |
| `architectures` | нет | |
| `languages`, `targets` | нет | |
| `signed_by` | нет | путь к keyring |
| `enabled` | нет | `false` → `Enabled: no` |
| `trusted` | нет | `true`/`false` → `Trusted: yes/no` |
| `options` | нет | словарь любых других полей deb822, пишется как есть |

## Примеры

Базовый набор Debian 13 — один файл, две секции:

```yaml
debian_sources_repositories:
  - name: 'debian'
    stanzas:
      - uris: ['https://deb.debian.org/debian']
        suites: ['trixie', 'trixie-updates']
        components: ['main', 'non-free-firmware']
        signed_by: '/usr/share/keyrings/debian-archive-keyring.gpg'
      - uris: ['https://security.debian.org/debian-security']
        suites: ['trixie-security']
        components: ['main', 'non-free-firmware']
        signed_by: '/usr/share/keyrings/debian-archive-keyring.gpg'
```

Proxmox: рабочий репозиторий включён, enterprise выключен, но файл оставлен —
так его держит сам PVE:

```yaml
debian_sources_repositories:
  - name: 'proxmox'
    uris: ['http://download.proxmox.com/debian/pve']
    suites: ['trixie']
    components: ['pve-no-subscription']
    signed_by: '/usr/share/keyrings/proxmox-archive-keyring.gpg'
  - name: 'pve-enterprise'
    uris: ['https://enterprise.proxmox.com/debian/pve']
    suites: ['trixie']
    components: ['pve-enterprise']
    signed_by: '/usr/share/keyrings/proxmox-archive-keyring.gpg'
    enabled: false
```

## Почему шаблон, а не модуль `deb822_repository`

Штатный модуль умеет то же самое, но требует на хосте `python3-debian`, а в
режиме `--check` без него падает — как и модуль `apt` без `python3-apt`. У нас
дисциплина «сначала `--check --diff`, потом прогон», и роль, которая не умеет
показать дифф на чистом хосте, эту дисциплину ломает. Шаблон не требует на
целевом хосте ничего, показывает файл целиком в диффе и одинаково ведёт себя
в обоих режимах. Плата за это — валидация полей на нашей стороне, она в
`tasks/main.yml` первым шагом.

## Осторожно с prune

`debian_sources_prune: true` снесёт всё, чего нет в списке. Пакеты вендоров
кладут свои файлы сами при установке, и после такой приборки хост останется
без их обновлений. Включать только там, где список в host_vars заведомо
полный, либо перечислять чужое в `debian_sources_prune_keep`.

## Требования

Debian 13+ (или любой apt с поддержкой deb822 — Debian 12, Ubuntu 22.04+).
На целевом хосте ничего заранее ставить не нужно.

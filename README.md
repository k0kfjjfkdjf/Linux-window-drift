# windrift: перенос живых окон между двумя Linux-машинами

Прототип. Программа работает на своей машине-владельце внутри xpra-сессии, а окно можно отправить на соседнюю машину хоткеем и вернуть обратно. Клиент всегда подключается напрямую к владельцу, поэтому цепочек вида «ноутбук → ПК → ноутбук» не возникает.

```
  ПК (Hyprland)                          Ноутбук (KDE)
 ┌──────────────────────┐               ┌──────────────────────┐
 │ xpra-сервер :100     │◄──── ssh ─────│ xpra-клиент (окно)   │  ← после `send`
 │  └─ firefox (X11)    │               │                      │
 │ windrift daemon :7781  │◄── TCP/JSON ─►│ windrift daemon :7781  │  ← attach + буфер
 └──────────────────────┘               └──────────────────────┘
        ▲ lan-mouse: одна мышь и клавиатура на обе машины ▲
```

## План задач

### Этап 0. База (без кода)
- [ ] Поставить `lan-mouse` (AUR) на обе машины и настроить, с какой стороны стоит сосед.
- [ ] Настроить ssh по ключу в обе стороны (`ssh-copy-id`).
- [ ] Поставить `xpra` (extra) и `wl-clipboard` на обе машины.

### Этап 1. Прототип (этот репозиторий)
- [x] `windrift run <cmd>`: запуск программы в отдельной xpra-сессии, показ локально.
- [x] `windrift send`: отправка активного окна соседу (Hyprland, KDE через `kdotool`, sway).
- [x] Подключение всегда к владельцу, без цепочек; возврат на владельца идёт через локальный сокет.
- [x] Демон: приём окон и синхронизация текстового буфера обмена без пинг-понга.
- [x] `windrift doctor`: проверка зависимостей, ssh и связи с соседом.

### Этап 2. Надёжность
- [ ] Автовозврат: если сосед пропал (ушёл в сон, унесли ноутбук), владелец сам подключает сессию обратно локально. Опрашивать `xpra info :N` на число клиентов.
- [ ] Безопасность: демон слушает только WireGuard-интерфейс, вместо голого токена использовать HMAC с nonce или mTLS.
- [ ] Восстановление реестра после перезапуска демона, имена окон в `list`.
- [ ] Буфер обмена: картинки и прочие MIME-типы, событийный `wl-paste --watch` вместо опроса.

### Этап 3. UX
- [ ] Окно приезжает на тот же workspace и того же размера (Hyprland IPC, `hyprctl dispatch`).
- [ ] Перетаскивание через край экрана: плагин Hyprland (C++) или KWin-скрипт ловит выход окна за границу и вызывает `windrift send`.
- [ ] Уведомление и анимация при переносе.

### Этап 4. Технологии
- [ ] Бэкенд на `wprs` вместо xpra для нативных Wayland-приложений, без принудительного X11.
- [ ] Аппаратное кодирование видео (VA-API) для тяжёлых окон.
- [ ] Звук: проверить проброс звука в xpra или сделать отдельно через PipeWire.
- [ ] PKGBUILD для AUR.

## Установка (на обеих машинах)

```bash
sudo pacman -S git xpra wl-clipboard python openssh libnotify
git clone https://github.com/k0kfjjfkdjf/Linux-window-drift.git ~/Linux-window-drift
cd ~/Linux-window-drift
install -Dm755 windrift ~/.local/bin/windrift
mkdir -p ~/.config/windrift && cp config.example.toml ~/.config/windrift/config.toml
nano ~/.config/windrift/config.toml   # name, peer и одинаковый token (openssl rand -hex 32)
```

- `~/.local/bin` должен быть в `PATH` (в fish: `fish_add_path ~/.local/bin`).
- Для KDE дополнительно нужен `kdotool` (AUR).
- По желанию: `gst-python` (звук из перенесённых окон), `python-cryptography` (шифрование xpra).
- Системные часы должны быть точными, иначе ssh и TLS могут сбоить: `sudo timedatectl set-ntp true`.

### Запуск демона

Hyprland (`hyprland.conf`):
```
exec-once = windrift daemon
```

Или через systemd (переменные Wayland должны попасть в user-окружение):
```bash
install -Dm644 windrift.service ~/.config/systemd/user/windrift.service
systemctl --user enable --now windrift
```

KDE: «Параметры системы → Автозапуск → добавить `windrift daemon`».

### Хоткеи

Hyprland:
```
bind = SUPER SHIFT, Right, exec, windrift send
bind = SUPER, B, exec, windrift run firefox
bind = SUPER, T, exec, windrift run kitty
```

KDE: «Комбинации клавиш → Добавить команду → `windrift send`».

### Проверка
```bash
windrift doctor            # всё должно быть ✓
windrift run gnome-calculator
# фокус на калькулятор → SUPER+SHIFT+Right → окно уехало на соседа
# там тот же хоткей → окно вернулось (локально, без сети)
```

## Известные ограничения прототипа
- Программы внутри сессии работают через X11 (XWayland-подобный режим xpra), поэтому Wayland-only приложения не поддерживаются.
- Тяжёлая 3D-графика и игры по сети будут лагать, это задача этапа 4.
- Программы, запущенные не через `windrift run`, перенести нельзя.
- Если машина-владелец выключится, её окна у соседа закроются. Это фундаментальное ограничение схемы.
- Защита демона рассчитана только на доверенную локальную сеть.

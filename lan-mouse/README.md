# lan-mouse с перехватом мыши на X11

[lan-mouse](https://github.com/feschber/lan-mouse) умеет в обе стороны, но перехват
мыши на X11 у него не реализован (`input-capture/src/x11.rs` — заглушка). Этот патч
дописывает его: окна-края не нужны, указатель опрашивается, у края с настроенным
соседом указатель и клавиатура захватываются на корневом окне, курсор прячется,
движение считается от центра экрана (как у Synergy/Deskflow), `release` возвращает
курсор туда, где он ушёл за край.

- `UPSTREAM_COMMIT` — коммит lan-mouse, к которому применяется патч.
- `x11-input-capture.patch` — сам патч (`git apply`).

`windrift mouse setup` собирает lan-mouse с патчем у соседа и настраивает обе машины.
Вручную:

```bash
git clone https://github.com/feschber/lan-mouse && cd lan-mouse
git checkout $(cat ../UPSTREAM_COMMIT) && git apply ../x11-input-capture.patch
cargo build --release --no-default-features --features x11_capture,x11_emulation
```

## Hyprland: повторный захват сразу после возврата

`hyprland-reenter-guard.patch` (только для Wayland-сборки): Hyprland после снятия
захвата иногда сразу снова присылает `wl_pointer.enter` для полоски у края, и lan-mouse
тут же захватывает мышь обратно, а через секунду Hyprland отбирает захват
(`compositor released mouse`). Патч игнорирует вход в полоску в первые 150 мс после настоящего
возврата и пишет в лог входы/выходы курсора для диагностики.

Главная причина оказалась в эмуляции: при каждом выходе соседа lan-mouse уничтожал
виртуальную мышь в Hyprland, а удаление устройства заставляет Hyprland пересчитать
фокус курсора и снять только что поставленный захват. Патч переиспользует виртуальные
мыши вместо уничтожения (клавиатуру lan-mouse и так переиспользует).

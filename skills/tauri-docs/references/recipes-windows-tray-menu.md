# Recipes: windows, custom titlebar, system tray, menus

## Contents

- Windows and labels
- Custom titlebar
- macOS transparent titlebar
- System tray
- Window and app menus
- Old patterns

## Windows and labels

- Declare windows in `app.windows` (each with a `label`) or build them in `setup` with `WebviewWindowBuilder::new(app, "first", url)`. Capabilities target the label. https://v2.tauri.app/learn/security/capabilities-for-windows-and-platforms/
- Fetch a window with `app.get_webview_window("main")`. https://v2.tauri.app/develop/debug/
- Frontends creating windows need `core:webview:allow-create-webview-window`; list every label (or a wildcard) in the capability. https://v2.tauri.app/learn/mobile-multiwindow/
- Platform config files replace `app.windows` whole: repeat every window and field there. https://v2.tauri.app/develop/configuration-files/

## Custom titlebar

- Set `decorations: false` and grant the window commands, which are blocked by default: https://v2.tauri.app/learn/window-customization/

```json
{ "permissions": ["core:window:default", "core:window:allow-close", "core:window:allow-minimize",
  "core:window:allow-toggle-maximize", "core:window:allow-start-dragging"] }
```

https://v2.tauri.app/learn/window-customization/

- `data-tauri-drag-region` applies only to the element carrying it, not its children, so buttons inside keep working; add it to each child that should drag. https://v2.tauri.app/learn/window-customization/
- Windows: add `*[data-tauri-drag-region] { app-region: drag; }` so touch and pen can drag. https://v2.tauri.app/learn/window-customization/
- Manual dragging with double-click maximize: https://v2.tauri.app/learn/window-customization/

```js
import { getCurrentWindow } from '@tauri-apps/api/window';
const appWindow = getCurrentWindow();
titlebar.addEventListener('mousedown', (e) => {
  if (e.buttons === 1) {
    e.detail === 2 ? appWindow.toggleMaximize() : appWindow.startDragging();
  }
});
```

https://v2.tauri.app/learn/window-customization/

- On macOS a custom titlebar loses native window moving and snapping. Prefer a transparent titlebar there. https://v2.tauri.app/learn/window-customization/

## macOS transparent titlebar

Remove the window from the config and build it in Rust with `TitleBarStyle::Transparent` under `#[cfg(target_os = "macos")]`. Set the background color through `objc2-app-kit` (`NSColor`, `NSWindow`) declared in a macOS-only dependency table. https://v2.tauri.app/learn/window-customization/

## System tray

- Enable the `tray-icon` feature: `tauri = { version = "2.0.0", features = ["tray-icon"] }`. https://v2.tauri.app/learn/system-tray/
- Build in `setup` with `TrayIconBuilder::new()` (or `with_id("my-tray")`) and the app icon `app.default_window_icon().unwrap().clone()`. JS: `TrayIcon.new(options)` from `@tauri-apps/api/tray`. https://v2.tauri.app/learn/system-tray/ https://v2.tauri.app/start/migrate/from-tauri-1/
- Default: the menu opens on left AND right click. Disable left with `show_menu_on_left_click(false)` / `menuOnLeftClick: false`. https://v2.tauri.app/learn/system-tray/
- Tray icon events (Click, DoubleClick, Enter, Move, Leave) are not emitted on Linux; the icon and right-click menu still work. https://v2.tauri.app/learn/system-tray/

```rust
use tauri::tray::{MouseButton, MouseButtonState, TrayIconEvent};
TrayIconBuilder::new()
  .on_tray_icon_event(|tray, event| match event {
    TrayIconEvent::Click { button: MouseButton::Left, button_state: MouseButtonState::Up, .. } => {
      if let Some(window) = tray.app_handle().get_webview_window("main") {
        let _ = window.unminimize();
        let _ = window.show();
        let _ = window.set_focus();
      }
    }
    _ => {}
  })
  .build(app)?;
```

https://v2.tauri.app/learn/system-tray/

- Menu item clicks arrive in `TrayIconBuilder::on_menu_event` (Rust) or each item's `action` (JS). https://v2.tauri.app/learn/system-tray/
- Flatpak: point `temp_dir_path` at `app.path().app_cache_dir()` so the tray image works inside the sandbox. https://v2.tauri.app/distribute/flatpak/

## Window and app menus

- Menus are desktop only. Rust: build, set, then handle events on the app: https://v2.tauri.app/learn/window-menu/

```rust
let menu = MenuBuilder::new(app)
  .text("open", "Open")
  .separator()
  .copy()
  .build()?;
app.set_menu(menu)?;
app.on_menu_event(|app, event| match event.id().0.as_str() {
  "open" => { /* ... */ }
  _ => {}
});
```

https://v2.tauri.app/learn/window-menu/

- macOS: every top-level item must be a submenu (others are ignored), and the FIRST submenu becomes the application menu whatever its label. https://v2.tauri.app/learn/window-menu/
- Update a menu with `set_text`, `set_checked`, `set_icon`, `set_accelerator`, or call `set_menu` again. Rust icons need the `image-png` or `image-ico` feature. https://v2.tauri.app/learn/window-menu/
- Pitfall: JS menu objects are handles to native menus and are NOT garbage collected. Call `close()`, or use `await using` (Tauri 2.12+, may need a polyfill). Close items from `Menu.get` and the previous menu returned by `setAsAppMenu` too. https://v2.tauri.app/learn/window-menu/

```typescript
await using copy = await PredefinedMenuItem.new({ item: 'Copy' });
await using menu = await Menu.new({ items: [copy] });
await using prev = await menu.setAsAppMenu();
```

https://v2.tauri.app/learn/window-menu/

- A menu item's event is bound to the item, not to a menu or window: a shared item fires every listener. For per-window handling, do not share items and use `WebviewWindow::on_menu_event`. https://v2.tauri.app/start/migrate/from-tauri-1/

## Old patterns

- `Window`, `WindowBuilder`, `get_window` are now `WebviewWindow`, `WebviewWindowBuilder`, `get_webview_window`. https://v2.tauri.app/start/migrate/from-tauri-1/
- `SystemTray` and `SystemTray::on_event` are now `TrayIconBuilder` with `on_menu_event` and `on_tray_icon_event`; the `system-tray` feature is `tray-icon`. https://v2.tauri.app/start/migrate/from-tauri-1/
- `CustomMenuItem` and `Builder::on_menu_event` are now `MenuItemBuilder` and `App::on_menu_event`. https://v2.tauri.app/start/migrate/from-tauri-1/
- Config snippets with `"tauri": { "windows": [...] }` are Tauri 1 shape: use `app.windows`. https://v2.tauri.app/learn/window-customization/

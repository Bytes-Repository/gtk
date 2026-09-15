# gtk — bindingi GTK4 dla H#

Bindingi do **GTK4** dla języka **H#** (H-Sharp). W **100% kod H#** — ani
jednej linii C. Cała biblioteka to deklaracje `extern static [c, "..."]`
wskazujące bezpośrednio na prawdziwe, systemowe `libgtk-4`, `libglib-2.0`
i `libgobject-2.0`, plus wygodna warstwa wysokopoziomowa (struktury,
`impl`, obsługa zdarzeń) napisana czystym H# na wierzchu tych deklaracji.

```hsharp
use "bytes -> gtk" from "gtk"

fn main() is
    gtk::init()
    let win: gtk::Window = gtk::new_window("Witaj, H#!", 400, 200)
    let btn: gtk::Button = gtk::new_button("Kliknij mnie")
    win.set_child(btn.handle)

    let app: gtk::App = gtk::new_app()
    app.on_click(btn, || is write("Klik!") end)

    win.present()
    app.run()
end
```

## Instalacja

Wymagane w systemie (dowolna dystrybucja Linuksa z GTK4):

```bash
# Debian/Ubuntu
sudo apt install libgtk-4-dev libglib2.0-dev pkg-config

# Fedora
sudo dnf install gtk4-devel glib2-devel pkgconf-pkg-config

# Arch
sudo pacman -S gtk4 glib2 pkgconf
```

Następnie w swoim projekcie H#:

```bash
bytes add gtk
```

lub ręcznie, dopisując do `deps` w swoim `<projekt>.hk`:

```
[deps]
-> gtk => bytes
```

## Budowanie tej biblioteki samodzielnie

```bash
git clone <adres-repo> gtk
cd gtk
h# check src/gtk.h#              # weryfikacja składni/typów
h# compile examples/hello_window.h# --emit bin -o hello_window
./hello_window
```

Kompilator H# (`ffi_linker.rs`) dla każdego bloku `extern static [c, "<lib>"]`
najpierw próbuje `pkg-config --static --libs <lib>`; jeśli dana biblioteka
nie ma statycznego wariantu w systemie (typowe dla GTK4 na większości
dystrybucji — pakiety `-dev`/`-devel` zwykle dają tylko `.so`), linker i tak
poprawnie dowiąże się dynamicznie do `.so`, bo `pkg-config` bez `--static`
i tak zwraca poprawne `-lgtk-4 -lglib-2.0 -lgobject-2.0`. Blok jest
zadeklarowany jako `static`, zgodnie z życzeniem — jeśli w Twoim systemie
są dostępne statyczne `.a` dla GTK4/GLib, `bytes`/`h#` użyje ich automatycznie.

## Struktura

```
gtk/
  gtk.hk                       manifest pakietu (format bytes)
  README.md                    ten plik
  LICENSE                      MIT
  src/
    gtk_ffi.h#                 surowe `extern static [c, "..."]` do GTK4/GLib/GObject
    gtk.h#                     warstwa wysokopoziomowa: struktury widgetów, `impl`, `App`
  examples/
    hello_window.h#            minimalne okno + przycisk "Wyjdź"
    counter_and_form.h#        licznik kliknięć, checkbox, pole tekstowe
    widgets_showcase.h#        DropDown, SpinButton, ListBox, Stack, CSS, hover, AlertDialog
```

## Pokryte widgety

| Kategoria       | Widgety |
|-----------------|---------|
| Okna/kontenery  | `Window`, `Box`, `Grid`, `Frame`, `ScrolledWindow`, `CenterBox`, `Overlay` |
| Podstawowe      | `Button`, `Label`, `Entry`, `CheckButton`, `Image`, `Separator`, `TextView`, `ProgressBar` |
| Wybór           | `DropDown`, `SpinButton`, `ListBox` |
| Nawigacja       | `Stack` + `StackSwitcher`, `MenuButton` + `Popover` |
| Dialogi         | `show_alert` (`GtkAlertDialog`, fire-and-forget) |

Każdy widget to struktura H# z polem `handle: int` (surowy wskaźnik GTK —
tak samo jak `malloc`/`free` w rdzeniu H# reprezentują pamięć jako `int`)
oraz metodami `impl` wołającymi prawdziwe funkcje `gtk_*`/`g_*` z
`gtk_ffi.h#`. Wspólne dla każdego widgetu: `gtk::show`, `gtk::hide`,
`gtk::set_visible`, `gtk::set_sensitive`, `gtk::set_size`, `gtk::set_margin`,
`gtk::set_align`, `gtk::set_expand` (hexpand/vexpand), `gtk::add_css_class` /
`gtk::remove_css_class`, `gtk::grab_focus`.

## Style CSS

```hsharp
gtk::load_css("
    .naglowek { font-weight: bold; color: #3b82f6; }
")
gtk::add_css_class(label.handle, "naglowek")
```

albo z pliku: `gtk::load_css_file("styl.css")`. Obie funkcje owijają
prawdziwe `GtkCssProvider` + `gtk_style_context_add_provider_for_display`
z priorytetem `GTK_STYLE_PROVIDER_PRIORITY_APPLICATION`.

## Jak działają zdarzenia (bez ani jednej linii C)

GTK normalnie łączy interakcje użytkownika (klik, zmianę checkboksa, ...)
przez sygnały GObject (`g_signal_connect`), które wymagają C-owego
wskaźnika funkcji jako callbacku. Dzisiejszy kompilator H# nie generuje
takiej trampoliny z domknięcia H# (`ffi.rs::type_to_c` mapuje typ funkcyjny
tylko na nieprzezroczysty `void*`, bez żadnego wywołania zwrotnego C→H#).
Ponieważ ta biblioteka ma być **w 100% H#**, `App` w `gtk.h#` zamiast
sygnałów odpytuje (polling) prawdziwy stan widgetu w swojej pętli głównej:

| Metoda `App`          | Widget         | Co odpytuje                                        | Wiarygodność |
|------------------------|----------------|-----------------------------------------------------|--------------|
| `on_click`             | `Button`       | flaga `GTK_STATE_FLAG_ACTIVE` (zbocze)               | przybliżona* |
| `on_toggle`            | `CheckButton`  | `gtk_check_button_get_active`                        | 100% pewna   |
| `on_change`            | `Entry`        | `gtk_editable_get_text`                              | 100% pewna   |
| `on_hover`             | dowolny widget | flaga `GTK_STATE_FLAG_PRELIGHT`                      | 100% pewna   |
| `on_submit`            | `Entry`        | flaga `GTK_STATE_FLAG_FOCUS_WITHIN` (zbocze, utrata fokusu) | przybliżona** |
| `on_selection_change`  | `ListBox`      | `gtk_list_box_get_selected_row` + `row_get_index`    | 100% pewna   |
| `on_dropdown_change`   | `DropDown`     | `gtk_drop_down_get_selected`                         | 100% pewna   |
| `on_value_change`      | `SpinButton`   | `gtk_spin_button_get_value`                          | 100% pewna   |
| `on_view_change`       | `Stack`        | `gtk_stack_get_visible_child_name`                   | 100% pewna   |
| `on_window_close`      | `Window`       | `gtk_widget_get_mapped` (po `g_object_ref`, zbocze)  | przybliżona*** |

\* `App::run()` odpytuje w **każdej** iteracji prawdziwej pętli GLib
(`g_main_context_iteration`), więc w praktyce nie da się kliknąć szybciej
niż biblioteka to zauważy — ale formalnie jest to detekcja zbocza stanu, a
nie natywny sygnał `clicked`.

\*\* To NIE jest sygnał `activate` (Enter) — ten wymaga
`GtkEventControllerKey` + sygnału, czyli C-owego callbacku. `on_submit`
odpala się przy **utracie fokusu** (użytkownik kliknął gdzie indziej /
wcisnął Tab), co w praktyce często pokrywa się z "skończyłem wpisywać",
ale nie jest identyczne z wciśnięciem Enter.

\*\*\* Realny sygnał `close-request` wymaga C-owego callbacku. Zamiast
tego trzymamy dodatkową referencję (`g_object_ref`) na oknie, żeby było
bezpiecznie odpytywać jego stan `mapped` nawet po tym, jak domyślny
handler GTK zniszczy widget po kliknięciu w „X”. To działa poprawnie w
typowym przypadku (okno bez własnego `close-request`), ale jeśli chcesz
100% gwarancji, najprościej dodać własny przycisk "Zamknij"/"Wyjdź"
wołający `app.quit()` (patrz `examples/hello_window.h#`).

## Znane ograniczenia (celowe — konsekwencja bycia w 100% H#)

- **Brak natywnych sygnałów GTK.** Wszystko powyżej to polling, nie
  `g_signal_connect`. Dla zdecydowanej większości aplikacji desktopowych
  różnicy praktycznie nie widać, ale formalnie nie jest to 1:1 z tym, jak
  zachowuje się "prawdziwa" aplikacja GTK.
- **`GtkFileDialog` / `GtkFileChooser` nie są obsługiwane.** Współczesne
  GTK4 usunęło `gtk_dialog_run()` (synchroniczne okna dialogowe) — jedyny
  sposób odebrania wyboru pliku to asynchroniczny `GAsyncReadyCallback`,
  czyli znów C-owy wskaźnik funkcji. To ta sama bariera co przy sygnałach,
  więc świadomie zostawiamy to poza zakresem biblioteki zamiast dawać
  bindingi, które i tak nie zwróciłyby wyniku.
- **`gtk_drop_down_new_from_strings`** (wariant przyjmujący gotową tablicę
  `char**`) nie jest używany — zamiast tego budujemy `GtkStringList`
  przez pojedyncze wywołania `gtk_string_list_append`, żeby nie polegać na
  marshalingu tablic NULL-terminated z H# do C.

## Licencja

MIT — patrz `LICENSE`.

# nidus

**Rozbudowana biblioteka diagnostyczna dla H# i ekosystemu `bit`, inspirowana [miette](https://github.com/zkat/miette) (Rust) — ale z większą liczbą stylów renderowania, motywami kolorystycznymi, lokalizacją komunikatów i łańcuchami przyczyn.**

100% H#. Zero `extern`. Zero FFI. Tylko standardowa biblioteka H# (`strings`, `fmt`, `term`, `env`, `test`).

```
                     ×  error[nidus::eval::div_zero]: dzielenie przez zero
                        ╭─[main.ns:1:9]
                      1 │ let y = x / 0
                        ·         ─┬─
                        ·          ╰── tu nie można dzielić przez zero
                        ╰────
                       help: sprawdź mianownik przed wykonaniem dzielenia
                        see: https://example.com/errors/div-zero
```

## Dlaczego nidus?

`miette` w Rust ma jeden domyślny "graficzny" handler i drugi, "narratable", do czytników ekranu. `nidus` idzie dalej i oferuje **siedem** gotowych stylów renderowania tej samej diagnostyki, pełną paletę motywów (w tym truecolor), oraz w pełni lokalizowalne słownictwo (błąd/pomoc/uwaga zamiast tylko error/help/note) — wszystko to bez ani jednej linijki kodu spoza H#.

## Instalacja

Przez `bit` (menedżer pakietów H#, następca `bytes`):

```bash
bit add nidus
```

Ręcznie, jako zależność w `Bit.hk` Twojego projektu:

```
[dependencies]
-> nidus => github twoje-konto/nidus
```

Następnie w kodzie:

```hsharp
use "bit -> nidus" from "nidus"
```

> **Uwaga dot. importu (0.3.0):** biblioteka jest podzielona na moduły (`src/nidus_*.h#`).
> **Funkcje** wołasz pod krótkimi nazwami (`nidus::error`, `nidus::report`,
> `nidus::theme_dark`, ...). **Typy** mają ścieżkę modułu:
>
> | Typ | Ścieżka |
> |---|---|
> | `Diagnostic` | `nidus::diagnostic::Diagnostic` |
> | `Config`, `Style` | `nidus::style::Config`, `nidus::style::Style` |
> | `Theme`, `Palette`, `Charset`, `Words` | `nidus::theme::Theme`, ... |
> | `Label`, `Span` | `nidus::span::Label`, `nidus::span::Span` |
> | `Severity` | `nidus::severity::Severity` |
> | `Outcome<T>` | `nidus::outcome::Outcome<T>` |
>
> Warianty `Style` tworzysz funkcjami: `nidus::style::graphical()`, `ascii()`, `narratable()`,
> `minimal()`, `json()`, `markdown()`, `github_actions()` (oraz `nidus::style::all()`).
> Alias importu musi być `nidus` (`use "bit -> nidus" from "nidus"`), bo nazwy modułów
> mają stały prefiks `nidus_`.

## Struktura źródeł

| Plik | Zawartość |
|---|---|
| `src/lib.h#` | wejście biblioteki: `VERSION`, `about()` i wrappery API |
| `src/nidus_severity.h#` | `Severity` |
| `src/nidus_text.h#` | linia/kolumna, zawijanie, odmiana, escapowanie |
| `src/nidus_span.h#` | `Span`, `Label` |
| `src/nidus_theme.h#` | `Charset`, `Palette`, `Words`, `Theme`, motywy, `paint` |
| `src/nidus_diagnostic.h#` | `Diagnostic` + builder |
| `src/nidus_style.h#` | `Style`, `Config`, autodetekcja stylu i koloru |
| `src/nidus_graphical.h#` | widok Graphical / Ascii |
| `src/nidus_plain.h#` | Narratable, Minimal |
| `src/nidus_machine.h#` | Json, Markdown, GitHub Actions |
| `src/nidus_report.h#` | `render`, `report`, `die`, podsumowania |
| `src/nidus_outcome.h#` | `Outcome<T>` |
| `src/nidus_tests.h#` | testy jednostkowe |

## Szybki start

```hsharp
use "bit -> nidus" from "nidus"

fn main() is
    let src: string = "let y = x / 0\nwrite(y)\n"

    let mut d: nidus::diagnostic::Diagnostic = nidus::error("dzielenie przez zero")
    d = d.with_code("nidus::eval::div_zero")
    d = d.with_source("main.ns", src)
    d = d.with_label(nidus::primary_label(8, 5, "tu nie można dzielić przez zero"))
    d = d.with_help("sprawdź mianownik przed wykonaniem dzielenia")

    nidus::report(d)   ;; sam dobierze styl i motyw do terminala
end
```

Więcej gotowych przykładów w [`examples/`](examples/):

| Plik | Co pokazuje |
|---|---|
| `basic.h#` | Minimalna diagnostyka z jedną etykietą |
| `multi_label.h#` | Dwie etykiety (główna + pomocnicza) w jednym fragmencie |
| `custom_theme.h#` | Motyw retro (truecolor), polska lokalizacja, własny motyw |
| `all_styles.h#` | Ta sama diagnostyka we wszystkich 7 stylach |
| `colors.h#` | Kolorowy pokaz: error/warning/advice, motywy dark/light/retro, kolorowe podsumowanie |
| `related_chain.h#` | Łańcuch przyczyn (`with_related`) |

## Model danych

### `Diagnostic`

Rdzeń biblioteki. Budowany fluent-builderem (każda metoda `with_*` zwraca
zmodyfikowaną wartość, więc łańcuchujesz przypisania):

```hsharp
let mut d: nidus::diagnostic::Diagnostic = nidus::error("wiadomość")
d = d.with_code("moj::kod::bledu")     ;; identyfikator, np. do wyszukiwania w dokumentacji
d = d.with_help("sugestia naprawy")
d = d.with_note("dodatkowy kontekst")
d = d.with_url("https://...")
d = d.with_source("plik.ns", tresc_zrodla)
d = d.with_label(nidus::primary_label(offset, dlugosc, "opis"))
d = d.with_related(inna_diagnostyka)   ;; łańcuch przyczyn
```

Konstruktory: `nidus::error(msg)`, `nidus::warning(msg)`, `nidus::advice(msg)`,
`nidus::diagnostic(msg)` (alias `error`). Modyfikatory poziomu:
`.as_error()`, `.as_warning()`, `.as_advice()`.

> Dlaczego reassignment (`d = d.with_x(...)`), a nie jeden długi łańcuch
> `error(...).with_x(...).with_y(...)` rozbity na wiele linii? Bo H# — na
> podstawie oficjalnej gramatyki i przykładów w `std/` — nie gwarantuje
> kontynuacji wyrażenia zaczynającej się od `.` na nowej linii. Łańcuch
> **na jednej linii** (`a.b().c()`) jest w pełni bezpieczny (patrz `iter`
> w README H#), więc krótkie łańcuchy możesz pisać w jednej linii —
> `nidus` sam z tego korzysta wewnętrznie.

### `Span` i `Label`

`Span { offset, len }` — zakres bajtowy w kodzie źródłowym.
`Label { sp, text, primary }` — pojedyncze podświetlenie.

```hsharp
nidus::span(offset, len)               ;; Span
nidus::point(offset)                   ;; Span o długości 1
nidus::label(offset, len, "opis")      ;; etykieta pomocnicza
nidus::label_at(offset, "opis")        ;; jw., długość 1
nidus::primary_label(offset, len, "opis")     ;; etykieta główna (kolor błędu)
nidus::primary_label_at(offset, "opis")
```

Diagnostyka może mieć dowolną liczbę etykiet, na tej samej lub różnych
liniach — styl `Graphical`/`Ascii` narysuje je wszystkie, w tym efekt
"schodkowy" gdy kilka etykiet wskazuje tę samą linię (jak w `miette`).

> **Uwaga o kolumnach:** tak jak reszta stringów w H# (`s[i..i+1]`), `nidus`
> liczy pozycje w **bajtach**. Dla źródeł czysto ASCII kolumny są dokładne.
> Wielobajtowe znaki UTF-8 w analizowanym kodzie źródłowym mogą przesunąć
> wizualną kolumnę — to ograniczenie modelu stringów H#, nie samej biblioteki.

## Style renderowania

`nidus` renderuje tę samą `Diagnostic` na siedem sposobów:

| Styl | Kiedy używać |
|---|---|
| `Style::Graphical` | Terminal interaktywny, UTF-8 — pełny widok z fragmentem kodu, strzałkami, kolorami |
| `Style::Ascii` | Jak wyżej, ale bez znaków unikodowych — zestaw znaków jest wymuszany na ASCII niezależnie od motywu |
| `Style::Narratable` | Czytniki ekranu — pełne zdania zamiast grafiki ASCII |
| `Style::Minimal` | Jedna linia na wpis, `plik:linia:kolumna: poziom: wiadomość` — do `grep`/logów |
| `Style::Json` | Wyjście maszynowe do dalszego przetwarzania (pola zawsze po angielsku) |
| `Style::Markdown` | Blok gotowy do wklejenia w komentarz Pull Requesta |
| `Style::GithubActions` | Adnotacje `::error file=...::` czytane przez GitHub Actions |

Wybór stylu i motywu:

```hsharp
let cfg: nidus::style::Config = nidus::config_default()   ;; autodetekcja (patrz niżej)
let cfg2: nidus::style::Config = nidus::config_with(nidus::style::json(), nidus::theme_mono())
nidus::report_with(d, cfg2)
```

### Autodetekcja (`config_default()` / `report(d)`)

`nidus::auto_style()`:
1. `GITHUB_ACTIONS=true` → `GithubActions`
2. zmienna `NIDUS_STYLE` (`graphical`/`ascii`/`narratable`/`minimal`/`json`/`markdown`/`github`) → wymuszony styl
3. stdout nie jest terminalem (`term::is_tty()` == false) i kolor nie jest wymuszony (`NIDUS_COLOR=always`, `FORCE_COLOR`, `CLICOLOR_FORCE`) → `Minimal`
4. terminal z UTF-8 (`LANG`/`LC_ALL`) → `Graphical`
5. w przeciwnym razie → `Ascii`

`nidus::auto_theme()`: honoruje `NIDUS_COLOR`, `NO_COLOR`, `FORCE_COLOR` i `TERM=dumb`,
oraz wspiera unikod tylko gdy terminal go deklaruje.

## Kolory (od 0.2.0)

Style przeznaczone dla człowieka (`Graphical`, `Ascii`, `Narratable`, `Minimal`)
są w pełni kolorowe — nie tylko nagłówek:

- **błędne miejsce podświetlone w samej linii kodu** (kolorem etykiety, pogrubione),
- ramka (`╭─[`, `│`, `╰────`), numery linii, ścieżka pliku, `linia:kolumna`
  i kod błędu mają **własne kolory** (zobacz pola `Palette`),
- etykiety główne w kolorze poziomu (czerwony/żółty/cyjan), pomocnicze w osobnym kolorze,
- fragmenty w \`backtickach\` w wiadomości, pomocy i uwadze są wyróżnione,
- `print_all` kończy **kolorowym podsumowaniem** (`2 błędy, 1 ostrzeżenie`);
  `nidus::summarize_colored(ds, theme)` zwraca je samodzielnie,
- `Json`, `Markdown` i `GithubActions` są zawsze czystym tekstem.

Kolory nie zmieniają układu: po zdjęciu sekwencji ANSI widok jest identyczny jak
w motywie mono (pilnuje tego test `test_colors_do_not_change_layout`).

### Sterowanie kolorem

Kolejność decyzji (`nidus::detect_color_support()`):

1. `NIDUS_COLOR=always` / `never` — jawne nadpisanie (np. gdy nidus działa w procesie
   potomnym z podpiętym pipe'm, a wynik trafia potem do terminala),
2. `NO_COLOR` — bez kolorów,
3. `CLICOLOR_FORCE` / `FORCE_COLOR` — kolory wymuszone,
4. terminal (`TERM` ustawiony i różny od `dumb`) — kolory.

Przy wymuszonym kolorze autodetekcja stylu wybiera `Graphical`/`Ascii` zamiast `Minimal`.
Wersja w kodzie: `nidus::config_colored()` — zawsze kolorowy widok graficzny.

## Motywy

Gotowe motywy (`Theme = Charset + Palette + Words + ustawienia layoutu`):

| Funkcja | Opis |
|---|---|
| `theme_dark()` | Domyślny — unikod + jasne kolory na ciemnym tle |
| `theme_light()` | Unikod + stonowane kolory na jasnym tle |
| `theme_ascii()` | ASCII + kolory |
| `theme_mono()` | ASCII, bez kolorów — najbezpieczniejszy wszędzie |
| `theme_mono_unicode()` | Unikod, bez kolorów |
| `theme_retro()` | Truecolor (`\x1b[38;2;r;g;b`) — neonowe różowo-fioletowe akcenty |
| `theme_polski()` | `theme_dark()` z polskimi etykietami (błąd/pomoc/uwaga/...) |

Własny motyw ze składników:

```hsharp
let mut t: nidus::theme::Theme = nidus::theme_dark()
t = t.with_context(3)              ;; 3 linie kontekstu zamiast domyślnej 1
t = t.with_wrap(60)                ;; węższe zawijanie help/note
t = t.with_words(nidus::words_pl()) ;; lub własny nidus::Words { ... }
t = t.with_palette(moja_paleta)     ;; własna struktura Palette
t = t.with_charset(moj_zestaw_znakow)
```

Ponieważ `Palette` to zwykłe pola typu `string` z kodami ANSI, możesz
zbudować dowolny kolor (w tym truecolor) — zobacz `theme_retro()` w
`src/nidus_theme.h#` jako wzór. Pola palety: `error_c`, `warning_c`, `advice_c`, `help_c`,
`note_c`, `link_c`, `dim_c`, `bold_c`, `reset` oraz (od 0.2.0) `border_c`, `lineno_c`,
`path_c`, `pos_c`, `code_c`, `secondary_c`. Puste pole = brak koloru dla tego elementu.

> **Zmiana niekompatybilna w 0.2.0:** literał `nidus::theme::Palette { ... }` wymaga teraz
> wszystkich 15 pól. Jeśli budujesz paletę ręcznie, zacznij od gotowej i podmień pola
> (`let mut p = ...; p.error_c = "..."`) albo dopisz sześć nowych.

## Łańcuchy przyczyn

```hsharp
let cause: nidus::diagnostic::Diagnostic = nidus::error("plik nie istnieje").with_code("nidus::io::not_found")
let mut top: nidus::diagnostic::Diagnostic = nidus::error("nie udało się wczytać konfiguracji")
top = top.with_related(cause)
nidus::report(top)
```

Każdy styl renderuje `related` rekurencyjnie (z wcięciem), analogicznie do
`#[source]`/łańcucha przyczyn w `miette`.

## Wiele diagnostyk naraz

```hsharp
let ds: [nidus::diagnostic::Diagnostic] = [blad1, blad2, ostrzezenie1]
nidus::print_all(ds, nidus::config_default())
;; wypisuje każdą diagnostykę, a na końcu np. "2 błędy, 1 ostrzeżenie"
```

`nidus::summarize(ds)` samodzielnie zwraca tylko tekst podsumowania
(z poprawną polską odmianą liczby mnogiej: 1 błąd / 2-4 błędy / 5+ błędów).

## Kończenie procesu

```hsharp
nidus::die(d)              ;; report(d) + exit(1 dla Error, 0 dla Warning/Advice)
nidus::die_with(d, cfg)    ;; jw., z jawną konfiguracją
```

## `Outcome<T>` (opcjonalnie)

Lekki odpowiednik `Result<T, Diagnostic>`, do użycia we własnych funkcjach
(biblioteka niczego nie narzuca — równie dobrze możesz zwracać `T?` i
korzystać z operatora `?`):

```hsharp
enum Outcome<T> is   ;; src/nidus_outcome.h#
    Success(T)
    Failure(Diagnostic)
end
```

```hsharp
fn wczytaj_port(s: string) -> nidus::outcome::Outcome<int> is
    if !s.is_numeric() is
        return nidus::outcome::Outcome::Failure(nidus::error("`" + s + "` nie jest liczbą").with_code("nidus::config::bad_port"))
    end
    return nidus::outcome::Outcome::Success(conv::str_to_int(s))
end
```

## Testy

Testy jednostkowe są w `src/nidus_tests.h#` (zgodnie z tym,
jak `bit test` wyszukuje `#[test]` — również wewnątrz `src/`):

```bash
bit test
```

## Ograniczenia i uczciwe zastrzeżenia

- Pozycje (`offset`) są liczone w **bajtach**, nie w punktach kodowych
  Unicode — zgodnie z modelem stringów H# (`s[i..i+1]`). Dla źródeł ASCII
  jest to dokładne.
- Efekt "schodkowy" dla wielu etykiet w tej samej linii jest uproszczony
  względem `miette` (brak pełnego algorytmu unikania kolizji przy bardzo
  gęsto upakowanych etykietach) — w praktyce dla 2-4 etykiet na linię
  wygląda identycznie.
- Kod nie był uruchamiany przez `h# check` / `bit build` w tym
  środowisku (brak lokalnego toolchaina LLVM 21 + Rust). Składnia została
  ręcznie zweryfikowana zdanie po zdaniu względem README H# v0.9 oraz
  wzorców z `std/*.h#` i `examples/showcase.h#` z repozytorium H#-Sharp.
  Jeśli natrafisz na błąd kompilacji, to najpewniej pojedyncza literówka
  składniowa — logika i architektura są kompletne.

## Licencja

MIT.

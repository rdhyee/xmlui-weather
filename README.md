# xmlui-weather

A small weather dashboard built in [XMLUI](https://xmlui.org): multi-city sidebar,
current conditions, a 3-day forecast strip, and drag-and-drop reordering of the city
list via the [`xmlui-dnd-list`](https://github.com/rdhyee/xmlui-dnd-list) extension.

Weather data comes from [wttr.in](https://wttr.in) (`?format=j1`). **No API key, no
account, no server code** — the whole app is static files plus two `<DataSource>`
declarations.

## Run it

Everything needed is committed. There is no `npm install` and no build step. But the
app *must* be served over HTTP — `index.html` loads its scripts by absolute path
(`/xmlui/...`), so opening the file directly with `file://` will not work.

```bash
git clone https://github.com/rdhyee/xmlui-weather.git
cd xmlui-weather
python3 -m http.server 8000
# → http://localhost:8000
```

Any static server works (`npx serve`, `caddy file-server`, nginx). I run it behind
Caddy at `https://weather.myinfonet.localhost/` with `root * <this dir>` +
`file_server`.

## Layout

| Path | What it is |
|------|-----------|
| `Main.xmlui` | The whole app: state, layout, the detail-pane `<DataSource>` |
| `components/CitySidebarRow.xmlui` | One city row — its own `<DataSource>`, emits `select` / `remove` / `moveUp` / `moveDown` |
| `components/WeatherStat.xmlui` | Label-over-value stat cell |
| `config.json` | App globals (`xsVerbose` tracing on) |
| `index.html` | Script tags + a jsx-runtime shim (see below) |
| `xmlui/` | Vendored runtime + extension UMD + trace tooling |

## How it works

- **State lives in `<App>` vars**, persisted to `localStorage` under `weather.cities`
  and `weather.selectedCity`. Every write is wrapped in `try/catch`, so a browser with
  storage disabled degrades to session-only rather than throwing.
- **Each sidebar row fetches its own weather.** `CitySidebarRow` declares its own
  `<DataSource>` rather than receiving data from the parent, so rows load
  independently and the parent holds no fetch bookkeeping. N cities = N requests to
  wttr.in — fine at handful scale, not what you'd do at fifty.
- **Child → parent communication is `emitEvent`**, not shared state. A row emits
  `select` / `remove` / `moveUp` / `moveDown` carrying the city string; `Main.xmlui`
  owns every mutation of the `cities` array.
- **Reordering has two paths on purpose**: drag-and-drop via `<Dnd:DndItems>`, and
  up/down chevrons on each row. The chevrons are the keyboard-reachable path, and they
  were also how I tested reorder logic independently of the drag layer.
- **The °F/°C toggle is pure presentation.** wttr.in returns both units in one payload,
  so switching never refetches.

## Two things that will look odd

**1. The XMLUI runtime is vendored, not loaded from the CDN.** `index.html` has the CDN
line commented out and loads `xmlui/xmlui-standalone.umd.js` instead. That was to pin a
build while a startup-trace fix was in flight (May 2026). It's a pin, not a preference —
the `<!-- TODO: swap back to CDN -->` comment marks the spot.

**2. There's a hand-written `react_jsx_runtime` shim in `index.html`.** Extension UMDs
built by `xmlui build-lib` expected a global the build didn't actually emit. Fixed
upstream in [xmlui#3426](https://github.com/xmlui-org/xmlui/pull/3426) (merged
2026-05-02), but the runtime vendored here predates that merge, so the shim stays until
this app moves to a release carrying the fix. Removing the shim on a current XMLUI
should be safe; removing it against *this* vendored build will break the dnd extension.

## The drag-and-drop extension

`xmlui/xmlui-dnd-list.js` is a built UMD. Its source — plus a build log narrating what a
first-time XMLUI extension author actually runs into — lives in
[rdhyee/xmlui-dnd-list](https://github.com/rdhyee/xmlui-dnd-list). This app is that
extension's host; the `onReorder` handler in `Main.xmlui` is its real-world usage.

## Tracing

`xmlui/xs-trace.js` and `xmlui/xs-diff.html` are app-level tracing helpers for the XMLUI
Inspector, which is wired up as the profile menu in `Main.xmlui`'s `<AppHeader>`. Click
the Inspector icon to see semantic traces of clicks, data binds, and HTTP calls;
`xs-diff.html` compares two exported traces. See
[xmlui-org/trace-tools](https://github.com/xmlui-org/trace-tools).

## Known rough edges

- City names are free text passed straight to wttr.in. A typo yields an error message,
  not a suggestion — there's no geocoding or autocomplete.
- Duplicate detection is exact string match, so `Albany, CA` and `albany, ca` can both
  sit in the list.
- The forecast card reads `hourly[4]` (midday) as the day's representative condition — a
  simplification, not a daily aggregate.
- wttr.in rate-limits. Adding several cities in quick succession can leave rows spinning.

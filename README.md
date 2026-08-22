# YAML Type Coercion Auditor

Paste YAML and see every scalar that resolves to a different type, or a different value, depending on which parser reads the file.

**Live demo:** https://0xelitesystem.github.io/yaml-type-coercion-auditor/

## The receipt

The famous YAML gotcha is `NO` becoming `false`. The expensive one is quieter.

js-yaml changed integer resolution between major version 3 and major version 4. Bumping that one dependency silently changed `mode: 0644` from **420** to **644**, changed `012` from **10** to **12**, and flipped `0o644` from the string `"0o644"` to the number **420**. Every other scalar in the file resolved identically, so nothing else in the diff hinted at it. Neither version raised an error.

The inverse direction hurts just as much. A file written for the newer rules, read by the older ones, turns `0o644` back into a plain string, and a `chmod` call gets a string where it expected an int.

"We moved to YAML 1.2" is not a safe summary either. Measured, no widely used parser sits cleanly on either side of the 1.1 / 1.2 line:

- **js-yaml 4 is not YAML 1.2 core.** It still resolves binary literals (`0b101` becomes 5) and still returns `Date` objects for bare dates.
- **PyYAML is not YAML 1.1.** It resolves `on` and `yes` as booleans but leaves `y` and `n` as strings, even though the YAML 1.1 bool type lists them.

That is why this tool shows five named columns and not two.

## Use

1. Open the live demo, or open `index.html` from a local clone.
2. Paste a YAML document, or click one of the three samples.
3. Read the count first (`N of your M scalars are parser-dependent`), then the per-line detail.

Each finding shows the resolved **value** under all five resolvers, not just the type, because the value is what breaks configs. `0644` becoming `420` in one parser and `644` in another is the whole problem, and both are perfectly valid integers.

Findings are graded:

- **value drift** means at least two parsers both returned a number and the numbers differ. Nothing downstream throws. This is the dangerous one.
- **type drift** means the resolved kind differs: a string in one parser, a number or boolean or date in another.

Scalars are grouped into named classes, and sexagesimals are their own class, separate from octal. `8:00` resolving to `480` has nothing to do with the leading-zero rule; it is base-60 integer resolution, and it fires on values that look like clock times.

## What is measured

Every cell in the reference table was produced on 2026-08-21 by running the real parsers:

| Column     | Package                    | Version |
|------------|----------------------------|---------|
| PyYAML     | PyYAML (Python)            | 6.0.3   |
| js-yaml 3  | js-yaml                    | 3.15.1  |
| js-yaml 4  | js-yaml                    | 4.3.1   |
| YAML 1.1   | eemeli/yaml, `version: '1.1'` | 2.9.0 |
| YAML 1.2   | eemeli/yaml, `version: '1.2'` | 2.9.0 |

Of the 35 scalars in that table, 21 resolve to a different type or value depending on which of the five reads them. The page recomputes that count live rather than printing a stored number.

The 21 parser-dependent rows:

| scalar | PyYAML | js-yaml 3 | js-yaml 4 | YAML 1.1 | YAML 1.2 |
|---|---|---|---|---|---|
| `0644` | `number:420` | `number:420` | `number:644` | `number:420` | `number:644` |
| `012` | `number:10` | `number:10` | `number:12` | `number:10` | `number:12` |
| `0o644` | `string:0o644` | `string:0o644` | `number:420` | `string:0o644` | `number:420` |
| `1_000` | `number:1000` | `number:1000` | `string:1_000` | `number:1000` | `string:1_000` |
| `8:00` | `number:480` | `number:480` | `string:8:00` | `number:480` | `string:8:00` |
| `22:30` | `number:1350` | `number:1350` | `string:22:30` | `number:1350` | `string:22:30` |
| `NO` | `boolean:false` | `string:NO` | `string:NO` | `boolean:false` | `string:NO` |
| `no` | `boolean:false` | `string:no` | `string:no` | `boolean:false` | `string:no` |
| `off` | `boolean:false` | `string:off` | `string:off` | `boolean:false` | `string:off` |
| `y` | `string:y` | `string:y` | `string:y` | `boolean:true` | `string:y` |
| `n` | `string:n` | `string:n` | `string:n` | `boolean:false` | `string:n` |
| `on` | `boolean:true` | `string:on` | `string:on` | `boolean:true` | `string:on` |
| `yes` | `boolean:true` | `string:yes` | `string:yes` | `boolean:true` | `string:yes` |
| `Y` | `string:Y` | `string:Y` | `string:Y` | `boolean:true` | `string:Y` |
| `0b101` | `number:5` | `number:5` | `number:5` | `number:5` | `string:0b101` |
| `2026-08-20` | `Date:2026-08-20` | `Date:2026-08-20T00:00:00.000Z` | `Date:2026-08-20T00:00:00.000Z` | `Date:2026-08-20T00:00:00.000Z` | `string:2026-08-20` |
| `2026-08-20T10:00:00Z` | `Date:2026-08-20T10:00:00+00:00` | `Date:2026-08-20T10:00:00.000Z` | `Date:2026-08-20T10:00:00.000Z` | `Date:2026-08-20T10:00:00.000Z` | `string:2026-08-20T10:00:00Z` |
| `1:2:3` | `number:3723` | `number:3723` | `string:1:2:3` | `number:3723` | `string:1:2:3` |
| `1e3` | `string:1e3` | `number:1000` | `number:1000` | `number:1000` | `number:1000` |
| `00:00` | `string:00:00` | `string:00:00` | `string:00:00` | `number:0` | `string:00:00` |
| `60:00` | `number:3600` | `number:3600` | `string:60:00` | `number:3600` | `string:60:00` |

Four results are worth reading twice:

- **`1e3` is the one row where Python stands alone.** It is the string `"1e3"` under PyYAML and the number `1000` under all four JavaScript resolvers. PyYAML's float pattern requires a decimal point, and requires any exponent to carry an explicit sign, so `1e3`, `1e+3`, `1.e3` and `1.0e3` all stay strings while `1.0e+3` resolves to `1000.0`. A `timeout: 1e3` is a number in Node and text in Python.
- **`00:00` resolves to the number `0` under YAML 1.1 only.** eemeli/yaml's base-60 pattern starts `[0-9]`, while PyYAML's and js-yaml 3's require a leading `1` through `9`, so a padded midnight is a number in exactly one of the five.
- **`60:00`, which is not a valid clock time at all, coerces to `3600`** under PyYAML, js-yaml 3 and YAML 1.1.
- **`1.0` is a Python `float` and a JavaScript `number` holding the integer 1.** It is not in the list above, because the numbers agree, but the type identity does not. Nothing changes numerically; a round trip through js-yaml re-emits it as `1`.

## How it works

There is no YAML library on the page. Two layers do the work.

**A scalar scanner.** It walks the document line by line and pulls plain scalars out of block mappings, block sequences and flow collections, tracking keys as well as values, because a key like `on:` is resolved by the same rules. It skips comments, single- and double-quoted scalars, block scalar bodies under `|` and `>`, aliases, and any value carrying an explicit tag. A quoted `"0644"` is unambiguous and is never flagged; false positives on quoted strings would make the tool useless. It is a scanner, not a full parser: it does not build a document tree and it will not catch every exotic construction.

**Five resolvers.** Each column is a port of that parser's own resolution code, not an approximation of the spec. The PyYAML column uses the regular expressions from `yaml/resolver.py` and the constructors from `yaml/constructor.py`. The js-yaml columns use the character-scanning integer resolvers and float patterns from `lib/type/int.js` and `float.js` in each major version, plus the shared timestamp type. The YAML 1.1 and 1.2 columns use eemeli/yaml's tag lists in schema order, including the detail that the base-60 tags sit after the plain integer tags.

The page runs those five ports against the measured table on load and prints the score. At time of writing it reproduces 175 of 175 measured cells.

## Why this exists

Type coercion bugs in YAML do not throw. They hand you a valid value of the wrong kind and let the failure surface three systems later, as a permission that is subtly wrong or a timeout that is a thousand times too short. The usual advice is to quote your strings, plus a link to the Norway story. What was missing is a thing you can paste your actual file into.

It is one HTML file with no dependencies, no build step, no telemetry, and no network access of any kind. MIT licensed. Fork it, vendor it, put it behind your firewall.

**In the interest of not overselling this, and starting with my own work:** [yaml-to-json](https://0xelitesystem.github.io/yaml-to-json/) is mine, it is live, and it is a sixth set of scalar rules that agrees with none of the five columns above on the headline case. Its resolver deliberately keeps leading-zero values as strings:

```js
if (/^[-+]?\d+$/.test(s)){
  if (/^[-+]?0\d+/.test(s)) return s; // keep leading-zero strings (zip codes etc.)
  return parseInt(s, 10);
}
```

So `mode: 0644` comes out of it as the string `"0644"`, where PyYAML and js-yaml 3 give you `420` and js-yaml 4 gives you `644`. It also recognises only `true` and `false` as booleans, and has no octal, sexagesimal, timestamp or underscore handling at all. And it says none of that on the page: the words `octal` and `coercion` do not appear anywhere on it, and nothing warns you that the same document read by a real parser downstream would produce different numbers. That silence is the gap this tool exists to cover, and my own converter is the first place it shows up.

## Privacy

Everything runs in your browser. Nothing is uploaded, logged, or sent anywhere. There is no analytics, no font fetch, no CDN reference, and no `fetch` call in the source. Load the page once and it works offline. The only thing written to storage is your light or dark theme preference.

## Run locally

```
git clone https://github.com/0xelitesystem/yaml-type-coercion-auditor.git
cd yaml-type-coercion-auditor
```

Open `index.html` directly in a browser, or serve it:

```
python -m http.server 8000
```

Then visit http://localhost:8000/.

## Build

There is no build. It is a single self-contained `index.html` with inline CSS and JavaScript. Editing the file is the whole workflow.

## Related

- [json-precision-auditor](https://github.com/0xelitesystem/json-precision-auditor) is the declared sibling. That one shows IEEE 754 silently changing your **numbers**. This one shows the resolver silently changing your **types and values**, a class the JSON tool cannot reach because it starts from already-parsed JSON.
- [yaml-to-json](https://github.com/0xelitesystem/yaml-to-json) converts YAML to JSON with a sixth set of scalar rules of its own. This tool is the warning label for it.
- [jwt-inspector](https://github.com/0xelitesystem/jwt-inspector) decodes and checks a JWT locally.

## License

MIT. Copyright (c) 2026 0xelitesystem.

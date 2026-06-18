# Data & Serialization Utilities

JSON and XML parsers, plus text/file IO, are **bundled in core** and included via
`<TrussC.h>` — no addon, no extra dependency. Everything is in namespace `tc`.

Relative paths resolve through `getDataPath()` (the executable's `data/` dir, or
`.app/Contents/Resources/data/` in a macOS bundle) — the same rule as image
loading. A bare filename means "in `data/`". Pass an absolute path to bypass.

> Don't hand-roll a parser. If you're scanning a string for `"key":`, stop — use
> `tc::Json`. This trips people up because these helpers aren't obvious from the
> graphics-heavy API surface.

## JSON — `tcJson.h` (nlohmann/json)

`tc::Json` is an alias for `nlohmann::json`.

```cpp
Json j = loadJson("config.json");          // file  -> Json (getDataPath-resolved)
Json j = parseJson(httpResponseBody);      // string -> Json

float t   = j["main"].value("temp", 0.0f); // typed access with default
bool  has = j.contains("weather") && j["weather"].is_array();
auto  arr = j["weather"];                  // iterate, arr[0]["id"].get<int>() ...

saveJson(j, "out.json", 2);                // indent 2; pass -1 for compact
string s = toJsonString(j, -1);
```

- `.value(key, default)` returns `default` only when the key is **missing**. If the
  key exists but has the wrong type it **throws** — wrap untrusted/remote JSON in
  `try/catch`.
- On parse failure `loadJson`/`parseJson` log an error and return a null `Json()`.
  Guard with `j.is_object()` / `j.is_null()` before use.
- `tcJsonReflect.h` bridges `TC_REFLECT` members <-> JSON (read/write reflectors).

## XML — `tcXml.h` (pugixml)

`tc::Xml` wraps `pugi::xml_document`; aliases `XmlNode`, `XmlAttribute`,
`XmlDocument`, `XmlParseResult`.

```cpp
Xml xml = loadXml("settings.xml");         // getDataPath-resolved
Xml xml = parseXml(str);

XmlNode root = xml.root();
XmlNode n    = root.child("panel");
float   x    = n.attribute("x").as_float();
string  name = n.child("name").text().as_string();

// build + save
Xml out;
XmlNode r = out.addRoot("config");
r.append_child("item").append_attribute("v") = 3;
out.addDeclaration();
out.save("out.xml");
```

Node/attribute traversal is raw pugixml: `child()`, `next_sibling()`,
`attribute()`, `text()`, `.as_int()/.as_float()/.as_string()`,
`append_child()/append_attribute()`.

## Text & files — `tcFile.h`

```cpp
string txt = loadTextFile("notes.txt");    // getDataPath-resolved
saveTextFile("notes.txt", txt);
bool   e   = fileExists("secrets.json");
vector<string> entries = listDirectory("images");   // entry names within the dir
```

Path-manipulation and filesystem helpers — `getFileName/getBaseName/`
`getFileExtension/getParentDirectory/joinPath/getAbsolutePath`,
`directoryExists/createDirectory/removeFile/getFileSize` — are listed in
[app-lifecycle.md](app-lifecycle.md) under **File Utilities**.

## Path resolution — `getDataPath()`

The single resolver every loader above uses:

- absolute path → returned unchanged
- relative path → executable dir + data root (or the bundle's `Resources/data/`)

So `loadJson("x.json")`, `loadXml("x.xml")`, `loadTextFile("x.txt")` and image
loading all agree: a bare filename lives in `data/`. (`loadXml`/`Xml::save`
historically skipped this resolution and worked off the CWD — fixed so the three
loaders are now consistent. Older TrussC checkouts may still have the asymmetry.)

---
name: xaf-azure-maps-proxy
description: >
  Serve map tiles in a DevExpress XAF Blazor (or any ASP.NET Core) app from Azure Maps through a
  server-side proxy, rendered with MapLibre GL. Use when replacing OpenStreetMap, MapTiler or CARTO
  tiles in a commercial app, when the map key must not leak into generated or downloadable HTML
  reports, or when Azure Maps transactions run out too fast. Covers the /tiles-az proxy controller,
  Azure style placeholders, TileJSON rewriting, sprite and glyph api-version, a size-limited JSON and
  tile cache, a subscription-key pool with cooldown, per-IP rate limiting, MapLibre layer styling
  pitfalls, and offline HTML export with an embedded basemap. Do not use for a single SPA map where a
  domain-restricted key or Entra ID token is acceptable. Triggers: Azure Maps, MapLibre, Leaflet tiles,
  "Access blocked" OSM tile usage policy, {{azMapsDomain}}, subscription-key, TileJSON, tile proxy.
---

# XAF Azure Maps Tile Proxy with MapLibre GL

The browser talks only to your server. The proxy adds the Azure Maps key, rewrites every
`atlas.microsoft.com` address to itself and caches responses. The front end uses one library
(MapLibre GL) and one shared JavaScript engine for every map in the application.

## Why a proxy

- **The key stays on the server.** Dashboards rendered as HTML are often downloaded or e-mailed.
  Any key embedded in that file is public.
- **One provider, one billing account.** Raster and vector tiles, styles, sprites and glyphs all go
  through the same endpoint and the same key pool.
- **Caching saves transactions.** Azure bills every TileJSON request as a full transaction, and one
  style references about six sources. Without a cache, a single map open costs six transactions
  before any tile loads.

## Before changing code

- **Check the provider licence first.** MapTiler Free and CARTO basemaps are non-commercial.
  OpenStreetMap tiles have a usage policy; when blocked they return HTTP 200 with
  "Access blocked — App is not following the tile usage policy" burned into the image, so DevTools
  shows no error. Azure Maps (Gen2) bills to the customer's Azure subscription without a
  non-commercial clause.
- Create an Azure Maps account (Gen2) and collect its keys. Keys from different accounts add up,
  because limits are per account, not per key.
- Identify every map in the app, the library each uses, and whether maps end up in exported HTML.
- Decide the public base URL of the app. It must be absolute, because an exported file has no
  application context.

## Azure Maps specifics

| Topic | Rule |
|---|---|
| Authentication | Header `subscription-key`, not a query parameter |
| Style URL | `https://atlas.microsoft.com/styling/styles/{style}?styleVersion=2023-01-01&api-version=2.0` (public, no key; not in the REST docs) |
| Style names | `road`, `grayscale_light`, `road_shaded_relief`, `night` |
| Style placeholders | `{{azMapsDomain}}`, `{{azMapsStylingPath}}`, `{{azMapsLanguage}}`, `{{azMapsView}}` — normally replaced by the Azure SDK; the proxy must replace them |
| Sprites and glyphs | Append `api-version=2.0`; the style omits it and Azure returns 404 without it |
| TileJSON | Contains absolute `https://atlas.microsoft.com/...` tile URLs; rewrite every JSON response, not only the style |
| Raster tiles | `map/tile?api-version=2024-04-01&tilesetId=microsoft.base.road&zoom={z}&x={x}&y={y}&language=..` |
| Billing | One TileJSON request = 1 transaction; 15 tiles = 1 transaction |

## Configuration

```jsonc
"Maps": { "PublicBaseUrl": "https://myapp.azurewebsites.net" },
"AzureMaps": {
  "StyleVersion": "2023-01-01",
  "JsonCacheMinutes": 1440,
  "TileCacheMinutes": 10080,
  "CacheSizeMb": 160,
  "MaxCachedItemKb": 2048,
  "KeyCooldownMinutes": 30,
  "DefaultLanguage": "en-US",
  "View": "Auto",
  "Styles": { "streets": "road", "muted": "grayscale_light", "terrain": "road_shaded_relief", "dark": "night" },
  "SubscriptionKeys": []
}
```

Store keys only in environment configuration (`AzureMaps__SubscriptionKeys__0`, `__1`, ...), never in
the repository. Point `PublicBaseUrl` at localhost in development, otherwise local maps call production.

## Service registration

```csharp
services.AddRateLimiter(o => o.AddPolicy("tiles", ctx =>
    RateLimitPartition.GetFixedWindowLimiter(
        ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown",
        _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 600,               // one map open is ~65 requests
            Window = TimeSpan.FromMinutes(1),
            QueueLimit = 0
        })));

// HTML generators are often static; set the public URL once at startup.
MapTilesEndpoint.BaseUrl = Configuration.GetValue("Maps:PublicBaseUrl", MapTilesEndpoint.BaseUrl);

services.AddSingleton<MapResourceCache>();
services.AddSingleton(_ => new MapKeyPool(
    Configuration.GetSection("AzureMaps:SubscriptionKeys").Get<string[]>() ?? [],
    TimeSpan.FromMinutes(Configuration.GetValue("AzureMaps:KeyCooldownMinutes", 30))));

services.AddHttpClient(AzureTileController.HttpClientName, c =>
{
    c.DefaultRequestHeaders.UserAgent.ParseAdd("MyApp/1.0 (+https://myapp.azurewebsites.net)");
    c.Timeout = TimeSpan.FromSeconds(10);
});
```

Call `app.UseRateLimiter()` in the middleware pipeline.

```csharp
public static class MapTilesEndpoint
{
    private static string _baseUrl = "https://myapp.azurewebsites.net";
    public static string BaseUrl
    {
        get => _baseUrl;
        set => _baseUrl = string.IsNullOrWhiteSpace(value) ? _baseUrl : value.TrimEnd('/');
    }
    public static string StyleBaseUrl => $"{BaseUrl}/tiles-az/style/";
    public static string RasterTemplate => $"{BaseUrl}/tiles-az/{{z}}/{{x}}/{{y}}.png";
}
```

## Proxy controller

The controller is anonymous on purpose: an exported report has no application session. It is
protected by the per-IP rate limit, a closed list of style IDs and a server-built upstream URL.

```csharp
[AllowAnonymous]
[Route("tiles-az")]
[EnableRateLimiting("tiles")]
public sealed class AzureTileController(
    IHttpClientFactory httpClientFactory,
    IConfiguration configuration,
    MapKeyPool keyPool,
    MapResourceCache cache) : ControllerBase
{
    public const string HttpClientName = "AzureMapTiles";
    private const string Upstream = "https://atlas.microsoft.com/";

    // Style: the ID must come from the closed list, otherwise clients can request any upstream resource.
    [HttpGet("style/{id}.json")]
    public async Task<IActionResult> GetStyle(string id, [FromQuery] string? lang, CancellationToken ct)
    {
        var style = configuration.GetValue<string>($"AzureMaps:Styles:{id}");
        if (string.IsNullOrWhiteSpace(style)) return NotFound();

        var version = configuration.GetValue("AzureMaps:StyleVersion", "2023-01-01");
        var path = $"styling/styles/{style}?styleVersion={version}&api-version=2.0";

        var (bytes, _, status) = await GetCachedAsync(path, requiresKey: false, ct);
        if (bytes is null) return StatusCode(status);

        Response.Headers.CacheControl = "public, max-age=86400";
        return Content(RewriteUrls(Encoding.UTF8.GetString(bytes), lang), "application/json");
    }

    // Everything the style references: TileJSON, vector tiles, sprites, glyphs.
    [HttpGet("up/{**path}")]
    public async Task<IActionResult> GetUpstream(string path, CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(path) || path.Contains("..", StringComparison.Ordinal))
            return BadRequest();

        var upstreamPath = path + Request.QueryString.Value;
        if (!upstreamPath.Contains("api-version=", StringComparison.OrdinalIgnoreCase))
            upstreamPath += (upstreamPath.Contains('?') ? "&" : "?") + "api-version=2.0";

        // Sprites and glyphs are public; tiles and TileJSON need a key.
        var requiresKey = !path.StartsWith("styling/", StringComparison.OrdinalIgnoreCase);

        var (bytes, contentType, status) = await GetCachedAsync(upstreamPath, requiresKey, ct);
        if (bytes is null) return StatusCode(status);

        Response.Headers.CacheControl = "public, max-age=604800";

        // TileJSON carries absolute Azure URLs — rewrite them or the browser gets 401 from Azure.
        if (contentType?.Contains("json", StringComparison.OrdinalIgnoreCase) == true)
            return Content(RewriteUrls(Encoding.UTF8.GetString(bytes), null), contentType);

        return File(bytes, contentType ?? "application/octet-stream");
    }

    private async Task<(byte[]? Bytes, string? ContentType, int Status)> GetCachedAsync(
        string upstreamPath, bool requiresKey, CancellationToken ct)
    {
        var key = "az:" + upstreamPath;
        if (cache.TryGet(key, out var hit))
            return (hit.Bytes, hit.ContentType, StatusCodes.Status200OK);

        var result = await FetchAsync(upstreamPath, requiresKey, ct);
        if (result.Bytes is null) return result;

        if (result.Bytes.Length <= configuration.GetValue("AzureMaps:MaxCachedItemKb", 2048) * 1024)
        {
            var isJson = result.ContentType?.Contains("json", StringComparison.OrdinalIgnoreCase) == true;
            var minutes = isJson
                ? configuration.GetValue("AzureMaps:JsonCacheMinutes", 1440)
                : configuration.GetValue("AzureMaps:TileCacheMinutes", 10080);
            cache.Set(key, result.Bytes, result.ContentType, TimeSpan.FromMinutes(minutes));
        }
        return result;
    }

    private async Task<(byte[]? Bytes, string? ContentType, int Status)> FetchAsync(
        string upstreamPath, bool requiresKey, CancellationToken ct)
    {
        if (requiresKey && keyPool.Count == 0)
            return (null, null, StatusCodes.Status503ServiceUnavailable);

        var client = httpClientFactory.CreateClient(HttpClientName);
        var url = Upstream + upstreamPath.TrimStart('/');
        var lastStatus = StatusCodes.Status503ServiceUnavailable;
        var attempts = requiresKey ? Math.Max(keyPool.Count, 1) : 1;

        for (var i = 0; i < attempts; i++)
        {
            using var request = new HttpRequestMessage(HttpMethod.Get, url);
            string? apiKey = null;
            if (requiresKey)
            {
                // One key per client, not per tile: a map open is dozens of requests on one account.
                apiKey = keyPool.NextFor(HttpContext.Connection.RemoteIpAddress?.ToString());
                if (string.IsNullOrWhiteSpace(apiKey))
                    return (null, null, StatusCodes.Status503ServiceUnavailable);
                request.Headers.Add("subscription-key", apiKey);
            }

            using var response = await client.SendAsync(request, HttpCompletionOption.ResponseHeadersRead, ct);
            if (response.IsSuccessStatusCode)
            {
                if (apiKey is not null) keyPool.ReportHealthy(apiKey);
                var bytes = await response.Content.ReadAsByteArrayAsync(ct);
                return (bytes, response.Content.Headers.ContentType?.MediaType, StatusCodes.Status200OK);
            }

            lastStatus = (int)response.StatusCode;
            if (apiKey is null || lastStatus is not (401 or 402 or 403 or 429))
                return (null, null, lastStatus);

            keyPool.ReportExhausted(apiKey);
        }
        return (null, null, lastStatus);
    }

    // After this call the document must not contain any reference to atlas.microsoft.com.
    private string RewriteUrls(string json, string? lang)
    {
        var host = (configuration.GetValue<string>("Maps:PublicBaseUrl") ?? MapTilesEndpoint.BaseUrl)
            .Replace("https://", "", StringComparison.OrdinalIgnoreCase)
            .Replace("http://", "", StringComparison.OrdinalIgnoreCase)
            .TrimEnd('/');
        var language = string.IsNullOrWhiteSpace(lang)
            ? configuration.GetValue("AzureMaps:DefaultLanguage", "en-US")!
            : lang;

        return json
            .Replace("{{azMapsDomain}}", host + "/tiles-az/up", StringComparison.Ordinal)
            .Replace("{{azMapsStylingPath}}", "styling", StringComparison.Ordinal)
            .Replace("{{azMapsLanguage}}", language, StringComparison.Ordinal)
            .Replace("{{azMapsView}}", configuration.GetValue("AzureMaps:View", "Auto")!, StringComparison.Ordinal)
            .Replace(Upstream, $"https://{host}/tiles-az/up/", StringComparison.OrdinalIgnoreCase);
    }
}
```

Add a `{z:int}/{x:int}/{y:int}.png` action with the raster URL from the table only if some maps stay
on Leaflet. Validate `0 <= z <= 22` and `0 <= x, y < 2^z`.

## Cache

Use a dedicated `MemoryCache` instance. Setting `SizeLimit` on the shared `IMemoryCache` forces every
other consumer in the application to set `Size` on its entries, and they start throwing.

```csharp
public sealed class MapResourceCache : IDisposable
{
    private readonly MemoryCache _cache;

    public MapResourceCache(IConfiguration configuration) =>
        _cache = new MemoryCache(new MemoryCacheOptions
        {
            SizeLimit = configuration.GetValue("AzureMaps:CacheSizeMb", 160) * 1024L * 1024L
        });

    public bool TryGet(string key, out (byte[] Bytes, string? ContentType) value) =>
        _cache.TryGetValue(key, out value!);

    public void Set(string key, byte[] bytes, string? contentType, TimeSpan ttl) =>
        _cache.Set(key, (bytes, contentType), new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = ttl,
            Size = bytes.Length
        });

    public void Dispose() => _cache.Dispose();
}
```

Keep the cache in process memory with a hard limit. Do not write tiles to the App Service local
disk. If memory becomes a constraint, move the cache to Blob Storage.

## Key pool

```csharp
public sealed class MapKeyPool
{
    private readonly string[] _keys;
    private readonly TimeSpan _cooldown;
    private readonly ConcurrentDictionary<string, DateTimeOffset> _coolingUntil = new();
    private int _cursor = -1;

    public MapKeyPool(IEnumerable<string?> keys, TimeSpan cooldown)
    {
        _keys = keys.Select(k => k?.Trim() ?? "").Where(k => k.Length > 0)
            .Distinct(StringComparer.Ordinal).ToArray();
        _cooldown = cooldown;
    }

    public int Count => _keys.Length;

    // The same client starts from the same key while it works.
    public string? NextFor(string? client)
    {
        if (_keys.Length == 0) return null;
        if (string.IsNullOrEmpty(client)) return FirstAvailable(Interlocked.Increment(ref _cursor));
        return FirstAvailable((int)((uint)client.GetHashCode(StringComparison.Ordinal) % (uint)_keys.Length));
    }

    private string? FirstAvailable(int start)
    {
        var now = DateTimeOffset.UtcNow;
        for (var i = 0; i < _keys.Length; i++)
        {
            var key = _keys[(int)((uint)(start + i) % (uint)_keys.Length)];
            if (!_coolingUntil.TryGetValue(key, out var until) || until <= now) return key;
        }
        // All cooling down: better to try and fail than show no map.
        return _keys.MinBy(k => _coolingUntil.TryGetValue(k, out var u) ? u : DateTimeOffset.MinValue);
    }

    public void ReportExhausted(string key) => _coolingUntil[key] = DateTimeOffset.UtcNow.Add(_cooldown);
    public void ReportHealthy(string key) => _coolingUntil.TryRemove(key, out _);
}
```

Unit-test: stable assignment per client, skipping a cooling key, recovery after `ReportHealthy`,
and `null` for an empty pool.

## MapLibre front end

```js
var STYLE_BASE = 'https://myapp.azurewebsites.net/tiles-az/style/'; // injected from MapTilesEndpoint.StyleBaseUrl
var STYLE_QUERY = '?lang=' + encodeURIComponent(UI_LANG);

var map = new maplibregl.Map({
  container: 'map',
  style: STYLE_BASE + basemap + '.json' + STYLE_QUERY,
  center: [15, 51.5],
  zoom: 4,
  attributionControl: { compact: true }
});
```

Build one shared engine (map init, labels, export, event handlers) and let each dashboard add only
its own `refresh()`. When the engine is assembled from C# string parts, end every part with a blank
line.

### Styling basemap layers

```js
var MAIN_ROADS = ['motorway', 'trunk'];

function applyRoadEmphasis() {
  map.getStyle().layers.forEach(function (layer) {
    // Match on data, not on layer names: Azure names layers "Highway", "Minor road", ...
    if (layer.type !== 'line' || layer['source-layer'] !== 'transportation') return;
    try {
      var color = map.getPaintProperty(layer.id, 'line-color');
      // Only a plain value may be the match fallback. Zoom expressions and legacy {stops}
      // are rejected inside `match`, and the rejection aborts the whole loop.
      if (typeof color === 'string') {
        map.setPaintProperty(layer.id, 'line-color',
          ['match', ['get', 'class'], MAIN_ROADS, '#b45309', color]);
      }
    } catch (e) { }
  });
}

function applyLabelLanguage() {
  var field = ['coalesce', ['get', 'name:' + UI_LANG], ['get', 'name']];
  map.getStyle().layers.forEach(function (layer) {
    if (layer.type !== 'symbol') return;
    try {
      var tf = map.getLayoutProperty(layer.id, 'text-field');
      // Road shields use `ref` (A4, E40) and house numbers use `housenumber`.
      // Replacing those with a name field erases them.
      if (!tf || JSON.stringify(tf).indexOf('"name') < 0) return;
      map.setLayoutProperty(layer.id, 'text-field', field);
    } catch (e) { }
  });
}
```

`map.setStyle()` discards these changes. Reapply them after the new style loads.

## Offline HTML export

To make a downloaded report work without the server, embed the basemap for the visible area:

1. Fetch the style and keep only the first vector source and the layers that use it.
2. Fetch its TileJSON and every tile covering the current viewport up to the current zoom.
3. Fetch `sprite@2x.json`, `sprite@2x.png` and glyph ranges `0-255` and `256-511` for every `text-font` in the layers.
4. Store everything base64-encoded in `window.BASEMAP`, and point the style at `embedded://`.
5. Register the protocol in a script placed **before** the page script:

```js
function __bin(b64) {
  var s = atob(b64), a = new Uint8Array(s.length);
  for (var i = 0; i < s.length; i++) a[i] = s.charCodeAt(i);
  return a.buffer;
}
maplibregl.addProtocol('embedded', function (req) {
  var u = req.url.replace('embedded://', '');
  if (u.indexOf('style') === 0) return Promise.resolve({ data: BASEMAP.style });
  if (u === 'sprite.json') return Promise.resolve({ data: JSON.parse(BASEMAP.sprite.json || '{}') });
  if (u === 'sprite.png') return Promise.resolve({ data: __bin(BASEMAP.sprite.png || '') });
  if (u.indexOf('glyphs/') === 0) {
    var p = u.substring(7).split('/');
    var g = BASEMAP.glyphs[decodeURIComponent(p[0]) + '|' + p[1].replace('.pbf', '')];
    return Promise.resolve({ data: g ? __bin(g) : new ArrayBuffer(0) });
  }
  var t = BASEMAP.tiles[u];
  return Promise.resolve({ data: t ? __bin(t) : new ArrayBuffer(0) });
});
```

The rewritten style uses `tiles: ['embedded://{z}/{x}/{y}']`, `sprite: 'embedded://sprite'` and
`glyphs: 'embedded://glyphs/{fontstack}/{range}'`. A typical file is about 5–6 MB.

## Gotchas

| Symptom | Cause | Fix |
|---|---|---|
| OSM tile shows "Access blocked", network shows 200 | OSM tile usage policy | Change provider; a custom User-Agent does not fix it |
| Blank map, 401 from `atlas.microsoft.com` in DevTools | TileJSON not rewritten | Rewrite every JSON response in `up/` |
| Map without icons and labels, 404 on sprite or glyphs | Missing `api-version` | Append `api-version=2.0` in `up/` |
| Thousands of transactions a day | Each TileJSON request is billed; a style has ~6 sources | Cache JSON for a day |
| Other features throw after adding the cache | `SizeLimit` set on the shared `IMemoryCache` | Use a dedicated `MemoryCache` |
| Road styling and labels disappear after changing basemap | `setStyle` resets layers | Reapply after the style loads |
| Road numbers (A4, S8) vanish | `text-field` of shields overwritten | Replace only fields that contain `"name` |
| Road emphasis matches nothing | Layers matched by name | Filter on `source-layer === 'transportation'` and `['get','class']` |
| Styling loop stops halfway | Complex paint value used as `match` fallback | Use the original only when it is a string or number; wrap each layer in `try` |
| Whole page script fails with "Unexpected token" | `const esc` in a dashboard collides with `function esc` in the engine | Unique names or one IIFE per dashboard |
| First line of an engine part never runs | Previous part ends with a `//` comment and no newline | End every part with a blank line |
| Station icon too small | 28 px image with `pixelRatio: 2` renders at 14 px | Size `icon-size` for the pixel ratio |

## Completion checklist

- [ ] DevTools → Network filtered by `atlas.microsoft.com` shows **zero** browser requests.
- [ ] An exported HTML file contains neither a key nor `atlas.microsoft.com`.
- [ ] Second map open serves style and TileJSON from cache (tens of ms instead of hundreds).
- [ ] Azure Portal → Maps account → Metrics → Usage shows the expected transaction rate after an hour of use.
- [ ] Offline export opens with the application stopped.
- [ ] With one invalid key in configuration, maps keep working on the next key.
- [ ] Keys that appeared in logs or chats are rotated (`az maps account keys renew`) and replaced in app settings.

# mcp-chrome Güvenlik İncelemesi (Veri Sızıntısı Odaklı)

Bu inceleme, extension (`app/chrome-extension`) ve native-server (`app/native-server`) kodu üzerinde veri dışa sızdırma risklerini doğrulamak için yapılmıştır.

## 1) Ağ istekleri hangi adreslere gidiyor?

### Extension → Native server (lokal)

Ajan/sohbet ve yönetim çağrıları `127.0.0.1` üzerindeki yerel sunucuya gider:

```ts
// app/chrome-extension/entrypoints/sidepanel/composables/useAgentProjects.ts
const url = `http://127.0.0.1:${serverPort}/agent/projects`;
const response = await fetch(url);
```

```ts
// app/chrome-extension/entrypoints/background/web-editor/index.ts
const sseUrl = `http://127.0.0.1:${port}/agent/chat/${encodeURIComponent(sessionId)}/stream`;
const response = await fetch(sseUrl, {
  /* ... */
});
```

### Native server lokal host'a sabitlenmiş

```ts
// app/native-server/src/constant/index.ts
export const SERVER_CONFIG = {
  HOST: '127.0.0.1',
  CORS_ORIGIN: [/^chrome-extension:\/\//, /^moz-extension:\/\//, 'http://127.0.0.1'] as const,
} as const;
```

```ts
// app/native-server/src/agent/tool-bridge.ts
const url =
  options.mcpUrl || `http://127.0.0.1:${process.env.MCP_HTTP_PORT || NATIVE_SERVER_PORT}/mcp`;
```

### Kullanıcı komutuyla dış URL çağrısı yapılabilen noktalar (gizli değil)

```ts
// app/chrome-extension/.../record-replay/actions/handlers/http.ts
const response = await fetch(url, fetchOptions);
```

```js
// app/chrome-extension/inject-scripts/network-helper.js
const response = await fetch(url, { ...options, signal });
```

```ts
// app/native-server/src/file-handler.ts
const response = await fetch(fileUrl);
```

Bu üçü **kullanıcı/ajan tarafından verilen URL** ile çalışır; kodda gizli sabit dış endpoint yoktur.

## 2) Hassas veri okunup gizlice dışarı gönderiliyor mu?

### Çerez API erişimi

Manifest izinlerinde `cookies` yok:

```ts
// app/chrome-extension/wxt.config.ts
permissions: [
  'nativeMessaging', 'tabs', 'activeTab', 'scripting', 'webRequest', 'debugger', /* ... */
],
```

> `cookies` izni olmadığından `chrome.cookies` ile doğrudan çerez okuma yok.

### Hassas veri okunabilen başlıca akışlar

- Network capture araçları request/response header/body yakalayabilir (`network-capture-debugger.ts`, `network-capture-web-request.ts`).
- Sayfa içeriği alma aracı DOM/metin okuyabilir (`web-fetcher.ts`).
- `network-helper.js` replay sırasında `credentials: 'include'` kullanır (ilgili siteye kullanıcı adına istek atar).

Ancak bu veriler kod içinde sabit bir 3. parti adrese otomatik post edilmez; sonuçlar MCP tool çıktısı olarak yerel akışta kullanılır.

## 3) Hard-coded endpoint/domain/IP/webhook/analytics var mı?

Kod tarafında sabit ağ hedefleri pratikte `127.0.0.1`/`localhost` (lokal) ile sınırlıdır:

- `app/native-server/src/constant/index.ts`
- `app/native-server/src/mcp/stdio-config.json`
- `app/chrome-extension/entrypoints/**` içindeki agent URL üretimleri

Harici telemetri/beacon/analytics endpoint'i için gömülü bir gönderim noktası bulunmamıştır.

## 4) Backdoor / script injection / şüpheli bağımlılıklar

### Dinamik kod çalıştırma noktaları var (ama amaçlı)

```ts
// app/chrome-extension/.../tools/browser/inject-script.ts
func: (code) => new Function(code)(),
```

```ts
// app/chrome-extension/.../tools/browser/userscript.ts
new Function(userCode)();
```

Bunlar kullanıcı tarafından verilen scriptleri çalıştırmak için tasarımsal olarak mevcut. Gizli uzaktan kod çekme (`import('https://...')`) şeklinde bir backdoor izi görülmedi.

### Native server tarafı

```ts
// app/native-server/src/agent/engines/claude.ts
const sdk = await (Function(
  'moduleName',
  'return import(moduleName)',
)(sdkModuleName) as Promise<any>);
```

Burada dinamik import edilen modül adı sabit paket adıdır (`@anthropic-ai/claude-agent-sdk`), uzak URL import'u değildir.

### Bağımlılık gözlemi

`package.json` bağımlılıklarında telemetry/beacon odaklı zorunlu bir SDK görünmüyor; ağ davranışı araçların fonksiyonel amaçlarıyla sınırlı.

## 5) “Yerel AI istemcisi + tarayıcı otomasyonu dışında bağlantı olmasın” kontrolü

Kod varsayılanında extension ↔ native-server ↔ local MCP URL (`127.0.0.1`) akışı var.
Dış internete çağrı yapabilen yollar kullanıcı komutu veya açık tool parametresi ile tetiklenen otomasyon fonksiyonlarıdır; gizli/arka plan telemetrisi şeklinde sabit bir veri gönderimi tespit edilmedi.

---

## Net Sonuç

**Bu projede senin iznin ve haberin olmadan hiçbir verin dışarıya gönderilmez.**

Not: Araçlar (HTTP request, web fetch, userscript, network capture) gereği güçlüdür; kullanıcı/ajan komutuyla dış adrese istek yaptırmak mümkündür. Bu, gizli veri sızıntısı değil, açıkça tetiklenen otomasyon davranışıdır.

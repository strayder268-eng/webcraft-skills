# Terjemahan di Next.js (App Router)

Pola ini mengikuti panduan resmi Next.js dan tidak butuh library tambahan. Kalau proyek butuh fitur lebih
(mis. format ICU lengkap), pakai library seperti `next-intl` atau `intl-messageformat`, dicatat sebagai ADR.

## Daftar bahasa: satu tempat

```ts
// src/i18n/config.ts
export const locales = ['id', 'en'] as const;   // menambah bahasa: tambah di sini + folder messages/<kode>
export const defaultLocale = 'id';
export type Locale = (typeof locales)[number];
export const hasLocale = (x: string): x is Locale => (locales as readonly string[]).includes(x);
```

## Routing

```
src/app/[lang]/layout.tsx       <html lang={lang}>
src/app/[lang]/page.tsx
src/proxy.ts                    tanpa awalan bahasa → arahkan ke bahasa pilihan/Accept-Language/bawaan
```

Semua file route ada di bawah `app/[lang]/`. Daftar bahasa untuk halaman statis lewat `generateStaticParams`
di layout. [Next.js: Internationalization]

Bentuk lain yang sah: route tanpa `[lang]`, dan `proxy.ts` yang me-*rewrite* `/en/...` ke route yang sama
sambil menaruh bahasa di header/cookie. Syaratnya tetap: bahasa terlihat di URL (aturan 6), `<html lang>`
benar, dan hreflang terpasang. Catat bentuk yang dipakai di *Konvensi* `docs/PROJECT.md`.

## Memuat kamus (server)

```ts
// src/i18n/dictionary.ts
import 'server-only';
import type { Locale } from './config';

const load = (l: Locale) => ({
  common: () => import(`../../messages/${l}/common.json`).then((m) => m.default),
  billing: () => import(`../../messages/${l}/billing.json`).then((m) => m.default),
});

export async function getMessages(l: Locale, ns: 'common' | 'billing') {
  return load(l)[ns]();
}
```

Server Component memanggil `getMessages` dan meneruskan teks yang dibutuhkan ke Client Component lewat props
(atau provider kecil berisi namespace yang dipakai halaman itu saja), supaya kamus utuh tidak terkirim ke
browser.

## Fungsi t() kecil

```ts
// src/i18n/format.ts
export function t(msg: string, vars: Record<string, string | number> = {}, locale = 'id') {
  // jamak sederhana: "{count, plural, one {# key} other {# keys}}"
  msg = msg.replace(/\{(\w+), plural, ((?:\w+ \{[^}]*\}\s*)+)\}/g, (_, name, cases) => {
    const n = Number(vars[name]);
    const cat = new Intl.PluralRules(locale).select(n);
    const pick = (k: string) => cases.match(new RegExp(`${k} \\{([^}]*)\\}`))?.[1];
    return (pick(cat) ?? pick('other') ?? '').replace('#', new Intl.NumberFormat(locale).format(n));
  });
  return msg.replace(/\{(\w+)\}/g, (_, k) => String(vars[k] ?? `{${k}}`));
}
```

`Intl.PluralRules` bawaan browser/Node sudah tahu aturan jamak tiap bahasa (CLDR).

## Pengecekan kunci (CI)

```js
// scripts/check-messages.mjs
import { readdirSync, readFileSync } from 'node:fs';

const base = 'id';
const flat = (o, p = '') =>
  Object.entries(o).flatMap(([k, v]) => (typeof v === 'object' ? flat(v, `${p}${k}.`) : [[`${p}${k}`, v]]));
const vars = (s) => [...String(s).matchAll(/\{(\w+)/g)].map((m) => m[1]).sort().join(',');
const read = (l, f) => new Map(flat(JSON.parse(readFileSync(`messages/${l}/${f}`, 'utf8'))));

let fail = false;
for (const l of readdirSync('messages').filter((x) => x !== base)) {
  for (const f of readdirSync(`messages/${base}`)) {
    const a = read(base, f);
    const b = read(l, f);
    for (const [k, v] of a) {
      if (!b.has(k)) { console.error(`hilang: ${l}/${f} ${k}`); fail = true; }
      else if (vars(v) !== vars(b.get(k))) { console.error(`placeholder beda: ${l}/${f} ${k}`); fail = true; }
    }
    for (const k of b.keys()) if (!a.has(k)) console.warn(`kunci lebih: ${l}/${f} ${k}`);
  }
}
process.exit(fail ? 1 : 0);
```

## hreflang

```ts
// metadata halaman publik
export async function generateMetadata({ params }) {
  const { lang } = await params;
  return {
    alternates: {
      canonical: `/${lang}/pricing`,
      languages: { id: '/id/pricing', en: '/en/pricing', 'x-default': '/id/pricing' },
    },
  };
}
```

## Sumber

- [Next.js: Internationalization](https://nextjs.org/docs/app/guides/internationalization)
- [MDN: Intl.PluralRules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/PluralRules)
- [Next.js: generateMetadata alternates](https://nextjs.org/docs/app/api-reference/functions/generate-metadata#alternates)

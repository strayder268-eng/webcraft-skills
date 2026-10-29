# Keamanan di Next.js

## CSP dengan nonce

Next.js menerapkan nonce otomatis ke skripnya kalau proxy memasang header CSP berisi `'nonce-…'`.
Halaman harus dirender dinamis (nonce dibuat per request). [Next.js: Content Security Policy]

```ts
// src/proxy.ts
import { NextResponse, type NextRequest } from 'next/server';

export function proxy(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64');
  const isDev = process.env.NODE_ENV === 'development';
  const csp = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic'${isDev ? " 'unsafe-eval'" : ''};
    style-src 'self' 'nonce-${nonce}';
    img-src 'self' blob: data:;
    font-src 'self';
    connect-src 'self' ${process.env.NEXT_PUBLIC_API_URL ?? ''};
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
  `.replace(/\s{2,}/g, ' ').trim();

  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-nonce', nonce);
  requestHeaders.set('Content-Security-Policy', csp);

  const response = NextResponse.next({ request: { headers: requestHeaders } });
  response.headers.set('Content-Security-Policy', csp); // mulai dengan Content-Security-Policy-Report-Only
  return response;
}
```

Layanan luar (analitik, captcha, login Telegram) ditambahkan ke direktif yang sesuai secara eksplisit.

## Header lain

```js
// next.config.mjs
const securityHeaders = [
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
];

export default {
  poweredByHeader: false, // hapus X-Powered-By
  async headers() {
    return [{ source: '/:path*', headers: securityHeaders }];
  },
};
```

`Strict-Transport-Security` dipasang di reverse proxy HTTPS (`deploy-app`).

## HTML dari data

React meng-escape isi secara otomatis. `dangerouslySetInnerHTML` hanya untuk HTML yang sudah dibersihkan
dengan sanitizer, dan dicatat alasannya.

## Library

```sh
npm ci                    # instal persis sesuai package-lock.json
npm audit --omit=dev      # kerentanan pada dependensi produksi
```

## Sumber

- [Next.js: Content Security Policy](https://nextjs.org/docs/app/guides/content-security-policy)
- [Next.js: headers](https://nextjs.org/docs/app/api-reference/config/next-config-js/headers)
- [Next.js: poweredByHeader](https://nextjs.org/docs/app/api-reference/config/next-config-js/poweredByHeader)
- [npm audit](https://docs.npmjs.com/cli/v10/commands/npm-audit)

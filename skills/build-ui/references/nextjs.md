# UI di Next.js (App Router)

## Komponen dasar dengan varian

```tsx
// src/components/ui/Button.tsx
import type { ButtonHTMLAttributes } from 'react';

const variants = {
  primary: 'bg-primary text-primary-foreground hover:opacity-90',
  secondary: 'bg-surface text-text border border-border hover:bg-background',
  danger: 'bg-danger text-danger-foreground hover:opacity-90',
} as const;

const sizes = {
  sm: 'h-8 px-3 text-sm',
  md: 'h-10 px-4',
} as const;

type Props = ButtonHTMLAttributes<HTMLButtonElement> & {
  variant?: keyof typeof variants;
  size?: keyof typeof sizes;
  loading?: boolean;
};

export function Button({ variant = 'primary', size = 'md', loading, disabled, className = '', ...rest }: Props) {
  return (
    <button
      {...rest}
      disabled={disabled || loading}
      aria-busy={loading || undefined}
      className={`inline-flex items-center justify-center gap-2 rounded-md font-medium
        focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-focus
        disabled:opacity-50 ${variants[variant]} ${sizes[size]} ${className}`}
    />
  );
}
```

- Pilihan varian dan ukuran tetap terbatas (objek di atas). Varian baru ditambah di sini, bukan di halaman.
- Kalau proyek sudah memakai `class-variance-authority`/`tailwind-merge`, pakai itu dengan pola yang sama.

## Keadaan halaman

App Router punya file khusus untuk keadaan per route [Next.js: Error handling, Loading UI]:

```
src/app/(app)/api-keys/
  page.tsx          isi
  loading.tsx       skeleton saat data dimuat
  error.tsx         error + tombol coba lagi (Client Component, menerima reset())
```

Keadaan **kosong** ditangani di dalam halaman/komponen dengan `EmptyState`.

Bagian halaman yang memuat sendiri dibungkus `<Suspense fallback={<Skeleton …/>}>` supaya bagian lain tampil
lebih dulu.

## Client Component seperlunya

Halaman dan layout tetap Server Component. Hanya bagian interaktif (form, tombol dengan state, dialog) yang
`'use client'` (lihat `structure-project/references/nextjs.md`).

## Gambar

```tsx
import Image from 'next/image';

<Image src="/hero.png" alt="Dasbor pemakaian API" width={1200} height={630} priority />   // gambar utama
<Image src={item.thumb} alt="" width={160} height={90} />                                   // hiasan, lazy bawaan
```

`width`/`height` wajib (mencegah layout melompat). `priority` hanya untuk gambar utama di atas layar.
[Next.js: Image]

## Font

```tsx
// src/app/layout.tsx
import { Inter, JetBrains_Mono } from 'next/font/google';

const body = Inter({ subsets: ['latin'], variable: '--font-body', display: 'swap' });
const code = JetBrains_Mono({ subsets: ['latin'], variable: '--font-code', display: 'swap' });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="id" className={`${body.variable} ${code.variable}`}>
      <body className="bg-background text-text font-sans">{children}</body>
    </html>
  );
}
```

`next/font` meng-host font sendiri saat build; tidak ada request ke Google dari browser pengunjung.
Variabel `--font-body`/`--font-code` dipetakan ke token di `globals.css` (`setup-design-system`).
[Next.js: Fonts]

## Metadata dan SEO

```tsx
// halaman publik
export const metadata = {
  title: 'Harga',
  description: 'Harga per model, dibayar sesuai pemakaian.',
  alternates: { canonical: '/pricing' },
};

// layout area privat
export const metadata = { robots: { index: false, follow: false } };
```

`src/app/sitemap.ts` dan `src/app/robots.ts` menghasilkan `sitemap.xml` dan `robots.txt`.
[Next.js: Metadata]

## Pengecekan elemen mentah

Tambahkan ke skrip cek desain (`setup-design-system`):

```sh
# elemen interaktif mentah di luar komponen dasar
grep -rnE '<(button|input|select|textarea)[ >]' src --include='*.tsx' \
  | grep -v '^src/components/ui/' && exit 1
```

## Sumber

- [Next.js: Error handling](https://nextjs.org/docs/app/getting-started/error-handling)
- [Next.js: loading.js](https://nextjs.org/docs/app/api-reference/file-conventions/loading)
- [Next.js: Image optimization](https://nextjs.org/docs/app/getting-started/images)
- [Next.js: Font optimization](https://nextjs.org/docs/app/getting-started/fonts)
- [Next.js: Metadata and OG images](https://nextjs.org/docs/app/getting-started/metadata-and-og-images)

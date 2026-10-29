# Struktur proyek: Next.js + TypeScript (App Router)

Berlaku untuk Next.js 15+ dengan App Router. Untuk versi yang terpasang, dokumentasi resmi ikut terpasang di
`node_modules/next/dist/docs/`; baca itu kalau ada yang berbeda dari file ini.

## Struktur

Ini **contoh bentuk** untuk proyek baru. `src/` opsional di Next.js: proyek tanpa `src/` (`web/app/`,
`web/components/`, `web/lib/`) sah, asal tiga tingkat penempatan terpenuhi. Proyek lama yang belum punya
`features/` boleh memakai `components/<fitur>/` sebagai folder fitur (satu fitur satu folder). Bentuk yang
dipilih dicatat di *Konvensi* `docs/PROJECT.md`; kode baru mengikuti bentuk itu, bukan mencampur dua bentuk.

```
web/
  src/
    app/                          routing saja
      layout.tsx
      page.tsx
      (public)/                   route group: mengelompokkan tanpa mengubah URL
        pricing/page.tsx
      (app)/
        dashboard/
          page.tsx                tipis: merakit komponen dari features/
          _components/            komponen yang hanya dipakai halaman ini
    features/
      <fitur>/                    contoh: billing, api-keys, auth
        components/               komponen milik fitur ini
        api.ts                    pemanggil API fitur ini
        types.ts
        schema.ts                 validasi input form fitur ini
        use-<aksi>.ts             hook satu aksi user
    components/
      ui/                         komponen dasar lintas fitur: Button, Input, Dialog, Table, EmptyState
      layout/                     kerangka situs: header, sidebar, footer
    lib/                          kode bersama yang bukan UI: klien HTTP, format uang/tanggal (nama spesifik)
    config/
      env.ts                      baca + validasi environment variable, sekali
  public/                         file statis
  .env.example
```

Pemetaan ke tiga tingkat penempatan:

| Tingkat | Letak |
|---|---|
| satu halaman | `src/app/<route>/_components/` |
| satu fitur | `src/features/<fitur>/` |
| lintas fitur | `src/components/ui/`, `src/components/layout/`, `src/lib/` |

- Folder berawalan `_` adalah *private folder*: tidak ikut routing. Folder dalam kurung `(nama)` adalah *route
  group*: tidak mengubah URL. [Next.js: Project structure]
- Folder `src/` memisahkan kode aplikasi dari file konfigurasi di root. [Next.js: src folder]
- `components/ui/` tidak boleh mengambil data atau berisi teks bisnis. Teksnya datang lewat props.

## Server dan Client Component

- Bawaannya semua komponen adalah **Server Component**. Tambahkan `'use client'` hanya pada komponen yang
  butuh interaksi (state, event klik, efek browser).
- Taruh `'use client'` sedalam mungkin, di komponen interaktif kecil, bukan di halaman atau layout besar.
  Semua yang diimpor file `'use client'` ikut terkirim ke browser. [Next.js: Reducing JS bundle size]
- File yang memegang rahasia atau hanya boleh jalan di server diawali `import 'server-only'`, supaya build gagal
  kalau tidak sengaja diimpor dari client. [Next.js: Server and Client Components]

## Environment variable

- Hanya variabel berawalan `NEXT_PUBLIC_` yang sampai ke browser. Anggap nilainya **publik**; jangan pernah
  isi rahasia.
- Nilai `NEXT_PUBLIC_*` **dibekukan saat build**. Mengubahnya di server setelah build tidak berpengaruh;
  harus build ulang. [Next.js: Environment variables]
- Semua variabel dibaca di `src/config/env.ts` dan divalidasi di sana (mis. dengan zod kalau proyek sudah
  memakainya). Kode lain mengimpor dari `env.ts`, tidak membaca `process.env` langsung.

## Nama file

| Jenis | Pola | Contoh |
|---|---|---|
| Komponen React | `PascalCase.tsx` | `TopUpForm.tsx` |
| Hook | `use-<aksi>.ts` | `use-create-key.ts` |
| Lainnya | `kebab-case.ts` | `format-money.ts` |
| File khusus Next | nama baku | `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx` |

[keputusan webcraft; file khusus Next mengikuti dokumentasi]

## Impor

- Pakai alias `@/` ke `src/` (di `tsconfig.json`: `"paths": { "@/*": ["./src/*"] }`), bukan `../../../`.
- `features/a` tidak mengimpor dari `features/b`. Pindahkan bagian yang dipakai bersama ke `components/ui`
  atau `lib`.

## Perintah cek

```sh
npx tsc --noEmit        # typecheck
npx eslint .            # lint (Next 16 tidak lagi punya `next lint`)
```

Ikuti skrip di `package.json` proyek kalau sudah ada.

## Sumber

- [Next.js: Project structure and organization](https://nextjs.org/docs/app/getting-started/project-structure)
- [Next.js: src folder](https://nextjs.org/docs/app/api-reference/file-conventions/src-folder)
- [Next.js: Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)
- [Next.js: Environment variables](https://nextjs.org/docs/app/guides/environment-variables)
- [Next.js: Migrate from next lint to ESLint CLI](https://nextjs.org/docs/app/guides/upgrading/codemods)

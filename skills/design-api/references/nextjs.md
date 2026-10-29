# Memanggil API dari Next.js (frontend)

## Satu klien HTTP

Semua pemanggilan API lewat satu modul, `src/lib/http.ts`. Komponen dan fitur tidak memanggil `fetch` mentah.

```ts
// src/lib/http.ts
import { env } from '@/config/env';

export class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    public fieldErrors: { field: string; code: string }[] = [],
  ) {
    super(code);
  }
}

export async function api<T>(path: string, init: RequestInit = {}): Promise<T> {
  const res = await fetch(`${env.NEXT_PUBLIC_API_URL}${path}`, {
    ...init,
    credentials: 'include', // kirim cookie sesi
    headers: { 'Content-Type': 'application/json', ...init.headers },
    signal: init.signal ?? AbortSignal.timeout(15_000),
  });
  if (res.status === 204) return undefined as T;
  const body = await res.json().catch(() => null);
  if (!res.ok) {
    throw new ApiError(res.status, body?.code ?? 'unknown_error', body?.errors ?? []);
  }
  return body as T;
}
```

- Base URL dari `src/config/env.ts`, tidak ditulis langsung.
- Ada batas waktu (timeout). Request yang tidak pernah selesai membuat UI macet.
- Error selalu menjadi `ApiError` dengan `code`.

## Pemanggil per fitur

Setiap fitur punya `src/features/<fitur>/api.ts` yang memakai `api()` dan mengembalikan tipe fitur:

```ts
// src/features/api-keys/api.ts
import { api } from '@/lib/http';
import type { ApiKey } from './types';

export const listApiKeys = (cursor?: string) =>
  api<{ data: ApiKey[]; next_cursor: string | null }>(
    `/api/v1/api-keys${cursor ? `?cursor=${encodeURIComponent(cursor)}` : ''}`,
  );
```

## Menampilkan error

- `code` diterjemahkan lewat kamus (`manage-translations`), mis. `t('errors.' + err.code)`, dengan teks
  cadangan umum kalau `code` tidak dikenal.
- `fieldErrors` ditampilkan di dekat field form yang bersangkutan (`build-ui`).
- 401 → arahkan ke halaman login. 429 → pesan "coba lagi sebentar".

## Mencegah kirim ganda

- Tombol kirim dinonaktifkan selama request berjalan (`build-ui`).
- Aksi uang/pesanan: buat `Idempotency-Key` sekali saat form dibuka (`crypto.randomUUID()`), kirim ulang dengan
  kunci yang sama kalau user menekan lagi atau jaringan putus.

## Sumber

- [MDN: AbortSignal.timeout()](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static)
- [MDN: Crypto.randomUUID()](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID)

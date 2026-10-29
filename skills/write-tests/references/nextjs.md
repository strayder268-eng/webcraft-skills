# Test di Next.js

Rekomendasi: **Vitest + Testing Library** untuk unit/komponen, **Playwright** untuk end-to-end. Kalau proyek
sudah memakai alat lain, ikuti yang ada. Menambahkan alat test dicatat di ADR.

## Fungsi dan hook

```ts
// src/lib/format-money.test.ts
import { describe, it, expect } from 'vitest';
import { formatMoney } from './format-money';

describe('formatMoney', () => {
  it('memformat rupiah tanpa desimal', () => {
    expect(formatMoney(150000, 'IDR', 'id-ID')).toBe('Rp 150.000');
  });
});
```

Catatan: `Intl` bisa memakai spasi khusus (non-breaking) di hasil format; bandingkan setelah dinormalkan atau
pakai `toMatch`.

## Komponen

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

it('menampilkan error di field saat nominal kosong', async () => {
  render(<TopUpForm />);
  await userEvent.click(screen.getByRole('button', { name: 'Top up' }));
  expect(screen.getByRole('textbox', { name: 'Nominal' })).toHaveAccessibleDescription(/wajib diisi/i);
});
```

`toHaveAccessibleDescription` berasal dari `@testing-library/jest-dom` (didaftarkan di file setup Vitest).
Cari elemen lewat peran dan label, seperti user dan pembaca layar. Test seperti ini sekaligus menjaga
aksesibilitas (`build-ui`).

## End-to-end (Playwright)

```ts
import { test, expect } from '@playwright/test';

test('user bisa membuat kunci API', async ({ page }) => {
  await page.goto('/api-keys');
  await page.getByRole('button', { name: 'Buat kunci' }).click();
  await page.getByLabel('Nama').fill('ci');
  await page.getByRole('button', { name: 'Simpan' }).click();
  await expect(page.getByText('ci')).toBeVisible(); // menunggu otomatis, tanpa sleep
});
```

- Login disiapkan sekali dan disimpan sebagai *storage state*, bukan lewat form di setiap test.
- Setiap test mandiri; data dibuat lewat API test atau seed khusus, lalu dibersihkan.
- Jalankan terhadap backend test dengan database test, bukan layanan produksi.

## Perintah

```sh
npx vitest run
npx playwright test
```

## Sumber

- [Next.js: Testing (Vitest, Playwright)](https://nextjs.org/docs/app/guides/testing)
- [Vitest](https://vitest.dev/guide/)
- [Testing Library: Queries priority](https://testing-library.com/docs/queries/about#priority)
- [Playwright: Best practices](https://playwright.dev/docs/best-practices)
- [Playwright: Authentication](https://playwright.dev/docs/auth)

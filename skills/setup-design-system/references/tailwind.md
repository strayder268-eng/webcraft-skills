# Token desain di Tailwind CSS v4

Tailwind v4 mengatur tema di CSS (`@theme`), bukan di `tailwind.config.js`. Catatan v3 ada di bagian akhir.

## Satu file token

```css
/* src/app/globals.css */
@import "tailwindcss";

/* Mode gelap ikut pengaturan perangkat; ganti ke data-theme bila ada tombol tema:
   @custom-variant dark (&:where([data-theme=dark], [data-theme=dark] *)); */

/* Lapis 2, semantik: nilai mode terang */
:root {
  --background: #ffffff;
  --surface: #f6f7f9;
  --text: #111418;
  --text-muted: #5b6470;
  --border: #d9dde3;          /* garis hiasan/pemisah */
  --border-strong: #7d8590;   /* batas input/kontrol: ≥3:1 di kedua mode */
  --primary: #0b6bcb;
  --primary-foreground: #ffffff;
  --danger: #c62828;
  --danger-foreground: #ffffff;
  --success: #2e7d32;
  --warning: #9a6700;
  --info: #0b6bcb;
  --focus: #0b6bcb;
}

/* Mode gelap: hanya nilai yang berganti */
@media (prefers-color-scheme: dark) {
  :root {
    --background: #0d1117;
    --surface: #161b22;
    --text: #e6edf3;
    --text-muted: #9aa5b1;
    --border: #30363d;
    --primary: #4d9fff;
    --primary-foreground: #0d1117;
    /* … */
  }
}

/* Petakan ke utility Tailwind. inline: utility menunjuk variabel di atas, sehingga ikut berganti tema. */
@theme inline {
  --color-*: initial;                 /* matikan palet bawaan: warna di luar token tidak tersedia */
  --color-background: var(--background);
  --color-surface: var(--surface);
  --color-text: var(--text);
  --color-text-muted: var(--text-muted);
  --color-border: var(--border);
  --color-border-strong: var(--border-strong);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-danger: var(--danger);
  --color-danger-foreground: var(--danger-foreground);
  --color-success: var(--success);
  --color-warning: var(--warning);
  --color-info: var(--info);
  --color-focus: var(--focus);
  --color-transparent: transparent;
  --color-current: currentColor;

  --font-sans: var(--font-body);      /* diisi next/font, lihat build-ui */
  --font-mono: var(--font-code);
}

@theme {
  --spacing: 4px;                     /* p-1 = 4px, p-4 = 16px */
  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-full: 9999px;
}

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

Nilai warna di atas hanya contoh. Ganti sesuai identitas dan cek kontrasnya.

Dipakai di komponen: `bg-surface text-text border-border`, `bg-primary text-primary-foreground`.

## Pengecekan otomatis

Skrip yang dijalankan di CI dan sebelum commit. Gagal kalau ada nilai langsung di komponen:

```sh
#!/usr/bin/env sh
# scripts/check-design.sh
set -u
fail=0
# warna langsung (hex, rgb, hsl, oklch) di komponen
if grep -rnE '#[0-9a-fA-F]{3,8}\b|rgba?\(|hsla?\(|oklch\(' src --include='*.tsx' --include='*.ts'; then fail=1; fi
# nilai sembarang Tailwind: text-[15px], bg-[#fff], p-[13px]
if grep -rnE '\b[a-z-]+-\[[^]]+\]' src --include='*.tsx'; then fail=1; fi
exit $fail
```

Pengecualian yang sah (mis. ukuran gambar dari data) diberi komentar alasan dan dikecualikan secara eksplisit.

## Tailwind v3

Konsepnya sama, letaknya di `tailwind.config.js`:

```js
module.exports = {
  theme: {
    colors: {                         // `colors`, bukan `extend.colors`: palet bawaan ikut hilang
      background: 'var(--background)',
      surface: 'var(--surface)',
      text: 'var(--text)',
      primary: 'var(--primary)',
      // …
      transparent: 'transparent',
      current: 'currentColor',
    },
  },
};
```

Variabel `:root` dan mode gelap tetap di `globals.css`.

## Sumber

- [Tailwind CSS: Theme variables](https://tailwindcss.com/docs/theme): `@theme`, `@theme inline`, `--color-*: initial`.
- [Tailwind CSS: Dark mode](https://tailwindcss.com/docs/dark-mode): `@custom-variant`.

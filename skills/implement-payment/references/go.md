# Pembayaran di Go

Format tanda tangan **berbeda per gateway**. Contoh di bawah memakai HMAC-SHA256 sebagai pola; ganti dengan
cara persis dari dokumentasi gateway yang dipakai (header apa, string apa yang ditandatangani, encoding apa).

## Handler webhook

```go
func (h *Handler) webhook(w http.ResponseWriter, r *http.Request) {
    body, err := io.ReadAll(http.MaxBytesReader(w, r.Body, 64<<10)) // body MENTAH, sebelum di-parse
    if err != nil {
        writeProblem(w, Problem{Status: 400, Title: "Body tidak terbaca", Code: "bad_body"})
        return
    }
    if !validSignature(h.secret, r.Header.Get("X-Signature"), body) {
        slog.WarnContext(r.Context(), "webhook tanda tangan salah")
        writeProblem(w, Problem{Status: 401, Title: "Tanda tangan tidak valid", Code: "bad_signature"})
        return
    }
    var ev Event
    if err := json.Unmarshal(body, &ev); err != nil { /* 400 */ }

    err = h.svc.ApplyPaymentEvent(r.Context(), ev) // cocokkan nominal/invoice + kredit atomik
    switch {
    case err == nil, errors.Is(err, billing.ErrAlreadyProcessed):
        w.WriteHeader(http.StatusOK) // sudah diproses = tetap sukses, supaya gateway berhenti mengirim ulang
    case errors.Is(err, billing.ErrMismatch):
        slog.ErrorContext(r.Context(), "webhook tidak cocok", "invoice", ev.Invoice)
        w.WriteHeader(http.StatusOK) // jangan diulang gateway; tandai untuk diperiksa admin
    default:
        writeProblem(w, Problem{Status: 500, Title: "Gagal memproses", Code: "internal_error"}) // gateway akan mengulang
    }
}

func validSignature(secret []byte, got string, body []byte) bool {
    mac := hmac.New(sha256.New, secret)
    mac.Write(body)
    want := hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(want), []byte(got)) // waktu-konstan
}
```

## Kredit atomik

```go
func (s *Service) ApplyPaymentEvent(ctx context.Context, ev Event) error {
    return pgx.BeginFunc(ctx, s.pool, func(tx pgx.Tx) error {
        // 1. catat event; UNIQUE(event_id) → sudah pernah diproses
        if _, err := tx.Exec(ctx, `INSERT INTO payment_events (event_id, payment_id, raw) VALUES ($1, $2, $3)`,
            ev.ID, ev.Invoice, ev.Raw); err != nil {
            if isUniqueViolation(err) {
                return ErrAlreadyProcessed
            }
            return err
        }
        // 2. cocokkan dengan data kita
        var p Payment
        if err := tx.QueryRow(ctx, `SELECT id, user_id, amount, currency, status FROM payments WHERE id = $1 FOR UPDATE`,
            ev.Invoice).Scan(&p.ID, &p.UserID, &p.Amount, &p.Currency, &p.Status); err != nil {
            return err
        }
        if p.Amount != ev.Amount || p.Currency != ev.Currency {
            return ErrMismatch
        }
        if ev.Status != "paid" {
            return s.markFailedOrExpired(ctx, tx, p, ev)
        }
        // 3. status satu arah
        tag, err := tx.Exec(ctx, `UPDATE payments SET status = 'paid', paid_at = now() WHERE id = $1 AND status = 'pending'`, p.ID)
        if err != nil {
            return err
        }
        if tag.RowsAffected() == 0 {
            return ErrAlreadyProcessed
        }
        // 4. ledger + saldo (kunci idempotency = ID pembayaran)
        return s.ledger.Credit(ctx, tx, p.UserID, p.Amount, "topup", "payment:"+p.ID)
    })
}
```

Catatan: kalau `ErrMismatch` dikembalikan dari dalam `BeginFunc`, transaksi di-rollback sehingga baris
`payment_events` ikut batal. Simpan kejadian tidak cocok di tabel/log terpisah di luar transaksi supaya admin
bisa memeriksanya.

## Sumber

- [crypto/hmac: Equal](https://pkg.go.dev/crypto/hmac#Equal)
- [Stripe: Webhooks](https://docs.stripe.com/webhooks)

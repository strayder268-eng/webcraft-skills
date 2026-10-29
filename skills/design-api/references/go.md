# API di Go (net/http)

Butuh Go 1.22+ (pola route dengan method dan parameter path ada di `net/http` bawaan; router pihak ketiga
tidak diperlukan).

## Route

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /api/v1/api-keys", h.list)
mux.HandleFunc("POST /api/v1/api-keys", h.create)
mux.HandleFunc("DELETE /api/v1/api-keys/{id}", h.delete)

func (h *Handler) delete(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // …
}
```

[Go blog: Routing enhancements]

## Membaca body dengan aman

Satu helper dipakai semua handler:

```go
func decodeJSON(w http.ResponseWriter, r *http.Request, dst any) error {
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // batas 1 MB
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()                     // tolak field tak dikenal (mass assignment)
    if err := dec.Decode(dst); err != nil {
        return err                                  // → 400
    }
    if dec.More() {
        return errors.New("body berisi lebih dari satu objek JSON") // → 400
    }
    return nil
}
```

Setelah decode, validasi isinya (panjang, rentang, format) → 422 dengan `errors` per field.
Struct request **terpisah** dari struct database; field seperti `UserID` diisi dari sesi, bukan dari JSON.

## Menulis error

```go
type Problem struct {
    Type   string       `json:"type,omitempty"`
    Title  string       `json:"title"`
    Status int          `json:"status"`
    Detail string       `json:"detail,omitempty"`
    Code   string       `json:"code"`
    Errors []FieldError `json:"errors,omitempty"`
}

type FieldError struct {
    Field string `json:"field"`
    Code  string `json:"code"`
}

func writeProblem(w http.ResponseWriter, p Problem) {
    w.Header().Set("Content-Type", "application/problem+json")
    w.WriteHeader(p.Status)
    _ = json.NewEncoder(w).Encode(p)
}
```

Error tak terduga: catat lengkap di log (dengan ID request), kirim `writeProblem` 500 generik.

## Middleware

Urutan yang disarankan, dibungkus sekali di `main`:

```
recover (panic → 500) → request ID → log request → CORS → rate limit → auth → handler
```

Detail log, rate limit, dan auth: `build-backend`, `implement-auth`.

## Sumber

- [Go blog: Routing enhancements for Go 1.22](https://go.dev/blog/routing-enhancements)
- [net/http: MaxBytesReader](https://pkg.go.dev/net/http#MaxBytesReader)
- [encoding/json: Decoder.DisallowUnknownFields](https://pkg.go.dev/encoding/json#Decoder.DisallowUnknownFields)

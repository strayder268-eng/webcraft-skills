# Caddy (reverse proxy HTTPS)

Caddy meminta dan memperbarui sertifikat Let's Encrypt otomatis untuk setiap nama domain di Caddyfile, dan
mengalihkan HTTP ke HTTPS. Domain harus sudah mengarah ke server dan port 443 (atau 80) terbuka.
[Caddy: Automatic HTTPS]

```
# Caddyfile
(security) {
	header {
		Strict-Transport-Security "max-age=63072000; includeSubDomains"
		X-Content-Type-Options "nosniff"
		Referrer-Policy "strict-origin-when-cross-origin"
		-Server
	}
}

example.com {
	import security
	request_body {
		max_size 10MB
	}
	handle /api/* {
		reverse_proxy api:8080
	}
	handle {
		reverse_proxy web:3000
	}
}

www.example.com {
	redir https://example.com{uri} permanent
}
```

- Caddy menambahkan `X-Forwarded-For` dengan IP pengunjung. Aplikasi hanya mempercayai header itu kalau
  request datang dari container proxy.
- Kalau ada proxy/CDN lain di depan Caddy, daftarkan alamatnya di opsi global `trusted_proxies` supaya IP
  pengunjung terbaca benar.
- Endpoint streaming (mis. Server-Sent Events) memakai `flush_interval -1` di `reverse_proxy`.

## Sumber

- [Caddy: Automatic HTTPS](https://caddyserver.com/docs/automatic-https)
- [Caddy: reverse_proxy](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [Caddy: header](https://caddyserver.com/docs/caddyfile/directives/header)

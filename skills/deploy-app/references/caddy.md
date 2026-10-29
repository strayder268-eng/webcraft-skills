# Caddy (reverse proxy HTTPS)

Caddy meminta dan memperbarui sertifikat Let's Encrypt otomatis untuk setiap nama domain di Caddyfile, dan
mengalihkan HTTP ke HTTPS. Domain harus sudah mengarah ke server dan port 443 (atau 80) terbuka.
[Caddy: Automatic HTTPS]

```
# Caddyfile
{
	# Matikan API admin (bawaan aktif di localhost:2019). Akibatnya: config baru berlaku
	# setelah Caddy di-restart (`docker compose restart proxy`), `caddy reload` tidak bisa dipakai.
	admin off
}

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

- **`admin off`**: API admin Caddy bisa mengubah seluruh config tanpa autentikasi bagi siapa pun yang bisa
  menjangkaunya. Di produksi dimatikan; mengubah config cukup dengan edit Caddyfile lalu restart. Sertifikat
  tersimpan di volume `/data`, jadi restart tidak meminta sertifikat ulang. [Caddy: Global options]
- Caddy menambahkan `X-Forwarded-For` dengan IP pengunjung. Aplikasi hanya mempercayai header itu kalau
  request datang dari container proxy.
- Kalau ada proxy/CDN lain di depan Caddy, daftarkan rentang IP-nya di blok global
  `servers { trusted_proxies static <CIDR…> }` supaya IP pengunjung terbaca benar.
- Endpoint streaming (mis. Server-Sent Events) memakai `flush_interval -1` di `reverse_proxy`.

## Sumber

- [Caddy: Global options (admin, trusted_proxies)](https://caddyserver.com/docs/caddyfile/options)
- [Caddy: Automatic HTTPS](https://caddyserver.com/docs/automatic-https)
- [Caddy: reverse_proxy](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [Caddy: header](https://caddyserver.com/docs/caddyfile/directives/header)

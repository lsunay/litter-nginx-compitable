# Nginx + WireGuard Security Baseline (Codex Server)

Bu dokuman, su topoloji icin onerilen minimum guvenlik ayarlaridir:

- Android app -> `wss://codex.example.com:443` (public)
- VPS Nginx -> `http://<wireguard_ip>:8390` (private WG tunnel)

## Karar: Auth Nerede Olacak?

- Auth katmani internete acik ilk noktada, yani **Nginx** uzerinde olmali.
- WireGuard ikinci katman olur (network izolasyonu).
- Upstream Codex portu internete acilmamali.

## Onerilen Auth Yontemi

- Tercih: **Bearer token** (`Authorization: Bearer <token>`)
- Gecici fallback: query token (`?access_token=...`) sadece client header gonderemiyorsa
- Basic Auth sadece gecici/test amacli kullanilmali.

## Nginx UI Uzerinden Hedef Ayarlar

1. Public host/domain ekle (`codex.example.com`).
2. SSL/TLS:
- Let's Encrypt sertifika aktif et.
- `Force SSL` aktif et.
- HTTP/2 aktif olabilir.
3. Upstream:
- `http://<wireguard_ip>:8390`
- WebSocket support aktif et.
4. Access control:
- Bearer token kontrolu ekle (advanced/custom Nginx config alanindan).
5. Rate limit:
- En az `ip` bazli istek siniri koy.
6. Logging:
- Query string loglamayi kapat veya maskelenmis log format kullan.

## Advanced Nginx Snippet (Bearer Token + WebSocket)

`<YOUR_LONG_RANDOM_TOKEN>` kismina uzun, rastgele bir token koy (min 32+ byte).

```nginx
# http{} context'inde:
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

map $http_authorization $codex_auth_ok {
    default 0;
    "~^Bearer <YOUR_LONG_RANDOM_TOKEN>$" 1;
}

limit_req_zone $binary_remote_addr zone=codex_rl:10m rate=20r/m;

server {
    listen 443 ssl;
    server_name codex.example.com;

    # SSL sertifika satirlari panel tarafindan yonetilebilir.

    location / {
        if ($codex_auth_ok = 0) { return 401; }

        limit_req zone=codex_rl burst=20 nodelay;

        proxy_pass http://<wireguard_ip>:8390;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

## WireGuard ve Firewall Kurallari

VPS tarafinda:

- `8390` portunu public interface'de kapat.
- `8390` sadece WG interface/IP'den kabul et.
- `22` (SSH) icin IP allowlist + key-only login kullan.

Ornek mantik:

- `ACCEPT` only: source=`<vps_wireguard_peer_ip>` dest_port=`8390`
- `DROP` all: dest_port=`8390`

## Codex Server Tarafi

- Mumkunse server'i tum arayuzlerde degil, WG IP veya localhost'a bind et.
- Internetten direkt Codex erisimi olmamali; tek giris Nginx olmalı.

## Android App Notu (Bu repo icin kritik)

Mevcut Android Codex transport kodu `ws://` kullaniyor; `wss://` ve TLS socket destegi yoksa public erisimde trafik sifresiz kalir. Bu nedenle:

1. Public agda calisacaksa Android istemcinin `wss://` desteklemesi gerekir.
2. `ws://...:443` guvenli degildir; port 443 tek basina sifreleme saglamaz.

## Dogrulama Checklist

1. `curl -I http://codex.example.com` -> `301/308` ile `https`'e gitmeli.
2. Token olmadan baglanti -> `401` donmeli.
3. Token ile baglanti -> `101 Switching Protocols` (WebSocket) donmeli.
4. Public internetten `:8390` erisimi kapali olmali.
5. Sadece WG icinden upstream erisimi olmali.


# NPM Usage (Nginx Proxy Manager)

Bu rehber Nginx Proxy Manager (NPM) uzerinden Codex + WireGuard ters proxy kurulumunu anlatir.

Topoloji:

- Android -> `wss://codex.example.com`
- NPM (VPS) -> `http://<wireguard_ip>:8390`

## 1. Proxy Host Olustur

NPM panel:

1. `Hosts` -> `Proxy Hosts` -> `Add Proxy Host`
2. `Domain Names`: `codex.example.com`
3. `Scheme`: `http`
4. `Forward Hostname / IP`: `<wireguard_ip>`
5. `Forward Port`: `8390`
6. `Block Common Exploits`: aktif
7. `Websockets Support`: aktif
8. `Cache Assets`: kapali
9. `Save`

## 2. SSL Ayarlari

Proxy host'u edit et -> `SSL` tab:

1. `SSL Certificate`: `Request a new SSL Certificate`
2. `Force SSL`: aktif
3. `HTTP/2 Support`: aktif
4. `HSTS Enabled`: aktif
5. `HSTS Subdomains`: ihtiyaca gore (genelde kapali)
6. `Save`

## 3. Bearer Token Koruma (Advanced)

Proxy host'u edit et -> `Advanced` tab'a su snippet'i ekle:

```nginx
set $codex_auth_ok 0;
if ($http_authorization = "Bearer <YOUR_LONG_RANDOM_TOKEN>") {
    set $codex_auth_ok 1;
}
if ($codex_auth_ok = 0) {
    return 401;
}

proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_set_header X-Forwarded-Proto https;

proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```

Notlar:

- `<YOUR_LONG_RANDOM_TOKEN>` yerine uzun rastgele token koy.
- Token'i duz metin paylasma; gizli ortamda sakla.

## 4. Rate Limit (Opsiyonel ama onerilir)

NPM container icindeki global nginx config ile uygulanir (panel disi olabilir):

```nginx
limit_req_zone $binary_remote_addr zone=codex_rl:10m rate=20r/m;
```

Host advanced alanda:

```nginx
limit_req zone=codex_rl burst=20 nodelay;
```

## 5. Firewall / Network

VPS guvenlik grubu veya iptables/ufw:

- Public'ten `8390` kapali.
- `8390` sadece WireGuard peer IP'den erisilebilir.
- Public'te yalniz `443` acik.

## 6. Test

1. SSL zorlamasi:
- `http://codex.example.com` -> HTTPS'e yonlenmeli.
2. Auth:
- Token yok -> `401`
- Token var -> WebSocket `101 Switching Protocols`
3. Port izolasyonu:
- Internetten `codex.example.com:8390` erisimi olmamali.

## 7. Onemli Uyari (Bu repo)

Android uygulama kodunda Codex RPC icin halen `ws://` akisi var. Public kullanimda `wss://` desteklenmeden ag trafigi sifrelenmis sayilmaz.

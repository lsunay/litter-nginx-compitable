# Temporary Access Playbook (Without Tailscale)

Bu rehber, Android uygulamayi degistirmeden ve Tailscale kurmadan Codex server'a gecici olarak erismek icin en dusuk riskli operasyon modelini tanimlar.

Temel ilke:

- Codex server surekli acik kalmaz.
- Public erisim surekli acik kalmaz.
- Gerektiginde acilir, kontrol edilir, sonra kapatilir.

## Hedef Topoloji

- Android app -> `ws://<temporary-endpoint>:443` veya mevcut kullandigin host
- Nginx -> `http://<wireguard_ip>:8390`
- Codex app-server sadece ihtiyac aninda calisir

Not:

- Bu model mevcut Android istemci sinirlari nedeniyle ideal degildir.
- Public `ws://` sifresizdir. Bu yuzden surekli kullanim icin uygun degil, sadece kisa sureli ve kontrollu kullanim icin dusunulmelidir.

## Gecici Kullanim Modeli

1. Normal durumda Codex server kapali olsun.
2. Nginx public route normalde kapali olsun veya en azindan deny ile korunsun.
3. Gerektiginde Codex server'i sunucuda manuel baslat.
4. Nginx route'u gecici olarak ac.
5. Android ile kontrolu yap.
6. Is bitince Nginx route'u tekrar kapat.
7. Codex server process'ini kapat.

## Sunucuda Codex Baslatma

Ornek:

```bash
codex app-server --listen ws://0.0.0.0:8390
```

Daha iyi:

- Mümkünse sadece WireGuard IP'sine bind et.
- `0.0.0.0` sadece zorunluysa kullan.

Ornek:

```bash
codex app-server --listen ws://<wireguard_ip>:8390
```

## Nginx Gecici Ac/Kapat Mantigi

Varsayilan durum:

- Public proxy route kapali
- veya `return 403`
- veya source IP allowlist disindakilere kapali

Gecici acarken:

- Yalniz ihtiyac duydugun sure boyunca aktif et
- Mumkunse kendi mevcut public IP'ni allowlist'e ekle

## En Guvenli Gecici Nginx Yontemi

Eger IP'n sabitse veya o anki cikis IP'ni biliyorsan:

```nginx
location / {
    allow <YOUR_PUBLIC_IP>;
    deny all;

    proxy_pass http://<wireguard_ip>:8390;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
}
```

Bu sayede:

- Endpoint public olarak acik gorunse bile sadece senin IP'nden erisim olur.
- Token destegi olmayan mevcut Android build icin en pratik koruma katmani budur.

## Firewall Kurali

Sunucuda:

- `8390` public internetten direkt erisilemez olmali
- `8390` sadece VPS icinden veya WireGuard tarafindan erisilebilir olmali

Nginx public dinler, Codex dogrudan dinlemez mantigi korunmali.

## Kullanim Sirasi

1. Sunucuda Codex server'i baslat.
2. Nginx'te route'u gecici olarak aktif et.
3. Mumkunse IP allowlist ekle.
4. Android uygulamadan baglan.
5. Kontrolunu yap.
6. Android baglantisini kes.
7. Nginx route'u kapat veya `deny all` yap.
8. Codex process'ini sonlandir.

## Kapatma

Process ID biliyorsan:

```bash
kill <PID>
```

Veya process adiyla:

```bash
pkill -f "codex app-server"
```

## Minimum Operasyon Kurali

- Surekli acik servis yok
- Surekli acik public route yok
- Mümkünse IP allowlist var
- Is bitince proxy kapali
- Is bitince process kapali

## Risk Degerlendirmesi

Bu modelin riski halen vardir cunku:

- Android istemci `ws://` kullaniyor
- Trafik TLS ile korunmuyor
- Public erisim gecici de olsa exposure yaratir

Ama asagidaki durumlarda kabul edilebilir gecici model olabilir:

- Kisa sureli kullanim
- Nadir kullanim
- Tek kullanici
- IP allowlist ile sinirlanmis erisim
- Is bitince aninda kapatma disiplini

## Ne Zaman Bu Modeli Birakmali?

Su durumlarda bu gecici modeli birakip kalici cozum gerekir:

- Siklikla disaridan baglaniyorsan
- Farkli aglardan baglaniyorsan
- Baska kullanicilar da erisecekse
- Servis daha uzun sure acik kalacaksa

Kalici cozum yonu:

- Android app icin `wss://` destegi
- `Authorization: Bearer` destegi
- Nginx tarafinda token dogrulamasi

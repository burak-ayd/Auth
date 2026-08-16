# Auth — Yol Haritası

Self-hosted Authelia alternatifi. Next.js tabanlı. Coolify + Traefik ile deploy.

## Mimari Kararlar

| Karar | Seçim | Not |
|---|---|---|
| Repo yapısı | Tek Next.js app + `docker-compose.yml` | Monorepo YOK |
| Framework | Next.js 15.x (App Router) | Next.js 16 henüz stabil değil |
| Output | `standalone` | Docker için |
| DB | Postgres 16 (Drizzle ORM) | TS-first, lightweight |
| Cache/Session | Redis 7 | Sliding session TTL |
| Auth | Argon2id + opaque session token | Cookie'de token, Redis'te hash |
| 2FA v1 | TOTP (RFC 6238) | speakeasy/otplib |
| 2FA v2 | Push notification | mobile app, plug-in provider |
| Proxy | Coolify Traefik `forwardAuth` | Authelia-uyumlu header |
| Deploy | Coolify (GitHub webhook) | Multi-stage Dockerfile |

---

## Faz Durumu

- [ ] Faz 0 — Altyapı
- [ ] Faz 1 — Core Auth
- [ ] Faz 2 — Forward-Auth
- [ ] Faz 3 — Access Control Engine
- [ ] Faz 4 — 2FA (TOTP)
- [ ] Faz 4b — 2FA Push Notification (ileride)
- [ ] Faz 5 — Admin Panel
- [ ] Faz 6 — Hardening
- [ ] Faz 7 — Coolify Deploy

---

## Faz 0 — Altyapı `[ ]`

**Hedef:** Çalışan dev ortamı, Postgres + Redis + Next.js tek komutla ayakta.

**Deliverable:**
- [ ] `npx create-next-app@latest` (App Router, TS, ESLint, Tailwind yok)
- [ ] `package.json` scriptleri: `dev`, `build`, `start`, `db:generate`, `db:migrate`, `db:studio`
- [ ] Env validasyon: `lib/env.ts` — `zod` ile parse
  - `DATABASE_URL`, `REDIS_URL`, `SESSION_SECRET`, `JWT_SECRET`, `TRUSTED_PROXIES`, `NODE_ENV`, `APP_URL`
- [ ] `.env.example` + `.env.local` (git ignore)
- [ ] `docker-compose.yml`:
  - `postgres:16-alpine` (volume, healthcheck)
  - `redis:7-alpine` (volume, healthcheck)
  - `app` service (dev, hot reload)
- [ ] Drizzle kurulum + `drizzle.config.ts`
- [ ] `lib/db/client.ts`, `lib/db/schema/users.ts` (boş tablo)
- [ ] `lib/redis/client.ts` (ioredis)
- [ ] Healthcheck stub: `GET /api/health` → `{ ok: true }`

**Çıkış kriteri:**
```bash
docker compose up
curl http://localhost:3000/api/health  # {"ok":true}
```

---

## Faz 1 — Core Auth `[ ]`

**Hedef:** Login/logout, Redis'te session.

**Deliverable:**
- [ ] Schema: `users(id, email, username, password_hash, created_at, updated_at)`
- [ ] Migration: `0000_init.sql`
- [ ] Argon2id helper: `lib/auth/hash.ts` (`m=64MB, t=3, p=4`)
- [ ] Session helper: `lib/auth/session.ts`
  - Token: 32 byte random → base64url
  - Redis key: `sess:{sha256(token)}`
  - TTL: 1 saat sliding, max 30 gün
- [ ] `POST /api/auth/register` (ilk admin oluşturma seed)
- [ ] `POST /api/auth/login` — email/password kontrol, session oluştur, cookie set
- [ ] `POST /api/auth/logout` — Redis'te session sil, cookie temizle
- [ ] `GET /api/auth/me` — mevcut user
- [ ] Cookie: `__Host-session`, `Secure`, `HttpOnly`, `SameSite=Lax`, `Path=/`
- [ ] Login UI: `/login` sayfası (server component + server action)

**Çıkış kriteri:** Login → cookie var → `/api/auth/me` user döner → logout → 401.

---

## Faz 2 — Forward-Auth `[ ]`

**Hedef:** Traefik `forwardAuth` ile korunan servislere kimlik enjekte et.

**Deliverable:**
- [ ] `GET /api/verify` — Traefik uyumlu
  - Oku: `X-Forwarded-Method`, `X-Forwarded-Uri`, `X-Forwarded-Host`, `X-Forwarded-For`
  - Session validate (Redis)
  - `200` + headers:
    - `Remote-User`, `Remote-Groups`, `Remote-Name`, `Remote-Email`
    - `Remote-Email-Verified`, `Remote-2fa-Verified`
  - `401` → Traefik login'e yönlendirir
- [ ] Authelia header format uyumu test
- [ ] `docker-compose.test.yml` — whoami servisi + Traefik labels ile uçtan uca doğrula

**Çıkış kriteri:** `curl -H "X-Forwarded-..."` → 200/401 doğru. Whoami login sonrası erişilebilir.

---

## Faz 3 — Access Control Engine `[ ]`

**Hedef:** Domain/path bazlı kurallar.

**Deliverable:**
- [ ] Schema: `groups`, `group_members`, `access_rules`
- [ ] Rule shape:
  ```ts
  {
    domain: string,
    path_regex: string,  // anchor + regex
    subject: { kind: 'user'|'group', id: string },
    policy: 'bypass'|'one_factor'|'two_factor'|'deny'
  }
  ```
- [ ] Evaluation engine: domain match → path regex → rule priority → ilk match kazanır
- [ ] Default policy: `deny`
- [ ] `auth_level` session'a ekle: `none | one_factor | two_factor`
- [ ] `/api/verify` kuralları uygula, gerekirse 2FA için redirect URL döndür

**Çıkış kriteri:** `two_factor` policy'li sayfa → 1FA session ile 403/redirect.

---

## Faz 4 — 2FA (TOTP) `[ ]`

**Hedef:** TOTP ile ikinci faktör.

**Deliverable:**
- [ ] `lib/2fa/provider.ts` interface:
  ```ts
  interface TwoFactorProvider {
    id: string;
    enroll(userId): Promise<{ secret, qrUrl }>;
    challenge(userId): Promise<ChallengeRef>;
    verify(userId, response): Promise<{ ok, level }>;
  }
  ```
- [ ] `TotpProvider` implementasyonu (speakeasy/otplib)
- [ ] Schema: `user_2fa(user_id, provider, secret_encrypted, enrolled_at, backup_codes_hash[])`
- [ ] TOTP secret şifreleme: AES-256-GCM (`SESSION_SECRET` derived key)
- [ ] Enroll UI: `/settings/2fa` — QR + doğrulama input
- [ ] Login flow update: `one_factor` session → `/2fa` sayfasına redirect → verify → `two_factor`
- [ ] Backup codes: 10 adet, Argon2id hash, tek kullanımlık
- [ ] "Bu cihazı hatırla": signed trust token (30 gün), Redis whitelist

**Çıkış kriteri:** Enroll → login sonrası 2FA sor → doğru kod → `two_factor` level.

---

## Faz 4b — 2FA Push Notification (ileride) `[ ]`

**Not:** Bu faz mobil app ayrı repoda geliştirilecek. Backend interface ve altyapı şimdiden hazır olsun.

**Backend deliverable:**
- [ ] `PushChallengeProvider` skeleton implementasyonu
- [ ] WebSocket altyapısı: `POST /api/auth/2fa/push/challenge` + WebSocket channel
- [ ] Challenge model: DB row `{ id, user_id, code, created_at, expires_at, status: pending|approved|denied }`
- [ ] Timeout: 60s, WS mesajı: `challenge.created`, `challenge.approved`, `challenge.denied`
- [ ] Mobile app API: device register, device list, WS auth (long-lived token)

**Mobil app (ayrı repo):**
- [ ] Push notification backend (FCM/APNs)
- [ ] Mobile UI: pending challenge list, approve/deny, biometric

**Çıkış kriteri (backend):** `PushChallengeProvider` registered, WS bağlantısı + challenge flow çalışır. Mobil app stub'ı.

---

## Faz 5 — Admin Panel `[ ]`

**Hedef:** Kural ve kullanıcı yönetimi.

**Deliverable:**
- [ ] Route segment: `/admin/*` — `two_factor` + role guard
- [ ] Pages: `/admin/users`, `/admin/groups`, `/admin/rules`, `/admin/audit`
- [ ] Server Actions ile CRUD
- [ ] Audit log tablosu: `audit_logs(ts, actor, action, target, ip, ua, meta)`
- [ ] Zod validation her form'da
- [ ] Server Actions: `createUser`, `deleteUser`, `assignGroup`, `addRule`, `revokeSession`

**Çıkış kriteri:** Admin login olmadan `/admin/*` → 403. Admin ile user/group/rule yönetimi çalışır.

---

## Faz 6 — Hardening `[ ]`

**Hedef:** Saldırı yüzeyini kapat.

**Deliverable:**
- [ ] Rate limit (Redis token bucket):
  - Login: 5/dk/IP, 20/dk/email
  - 2FA verify: 10/dk
  - Register: 3/saat/IP
- [ ] Brute-force lockout: 5 başarısız → 15 dk (Redis sorted set)
- [ ] CSRF: double-subit cookie + Origin/Referer check
- [ ] Cookie ayarları finalize (Faz 1'den)
- [ ] Security headers: HSTS, CSP, X-Frame-Options=DENY, X-Content-Type-Options=nosniff
- [ ] Structured logging: `pino` → stdout
- [ ] Audit log her kritik aksiyona
- [ ] `/api/health` deep: DB ping, Redis ping, version

**Çıkış kriteri:** `npm run omnirule:security` temiz.

---

## Faz 7 — Coolify Deploy `[ ]`

**Hedef:** Coolify üzerinden GitHub → auto deploy.

**Deliverable:**
- [ ] Multi-stage `Dockerfile`:
  - `deps` (pnpm install)
  - `builder` (next build, standalone output)
  - `runner` (next start, non-root user)
- [ ] `.dockerignore`
- [ ] Coolify setup:
  - Service: GitHub repo + branch
  - `Resource: Postgres` (managed volume)
  - `Resource: Redis` (managed volume)
  - Internal DNS: `postgres.railway.internal` benzeri
- [ ] Env inject (Coolify UI): tüm Faz 0 değişkenleri
- [ ] Traefik labels (Coolify otomatik)
- [ ] Healthcheck: `GET /api/health` (DB + Redis)
- [ ] Auto-deploy: GitHub webhook aktif
- [ ] `docker-compose.yml` sadece dev için (postgres + redis local), prod Coolify managed

**Çıkış kriteri:** GitHub push → Coolify otomatik rebuild → whoami servisi auth sonrası erişilebilir.

---

## Açık Sorular

- DB katmanı: **Drizzle** (seçildi) vs Prisma
- Mobile app ayrı repo, ne zaman başlanacak? (Faz 4 sonrası)
- Rate limit policy: katı mı, tunable mı?
- Multi-tenant (birden fazla domain grubu) v1'de var mı?

---

*Son güncelleme: Faz durumu checklist olarak — her faz tamamlanınca `[ ]` → `[x]`*

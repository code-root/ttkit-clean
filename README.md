# ttkit_clean — TikTok Account Toolkit

**Language / اللغة:** English (default) · [العربية ↓](#العربية)

> ⚠️ **This repository contains documentation only.**  
> The full source code is a **paid / licensed product** and is **not** published here.  
> **To purchase, get updates, or request a demo:** contact me via [GitHub — code-root](https://github.com/code-root).

---

## What is ttkit_clean?

`ttkit_clean` is a **standalone Python package** for TikTok account automation:

- Device registration (`device_register`) with realistic fingerprints
- Cryptographic signing: **Argus / Gorgon / Ladon / TTEncrypt**
- Email OTP registration (mail.tm, 1secmail, custom IMAP)
- Post-register activation (FYP, likes, profile, avatar)
- Phone bind / change via **5SIM** + **Lite Go** passport path
- Lite Go follow (`/lite/v2/relation/follow/`)
- Optional **Frida MSSDK** real in-app signing
- Full session persistence (cookies, token, fingerprint, guard headers)

Everything runs from one folder — vendored deps, SignServer JAR, and runtime data.

---

## Package layout — file-by-file reference

Below is a **complete map** of every file in the paid package and what it does.

```
ttkit_clean/
├── __init__.py
├── __main__.py
├── _bootstrap.py
├── cli.py
├── create.py
├── device.py
├── register.py
├── fingerprints.py
├── signing.py
├── signatures.py
├── musical.py
├── cookies.py
├── otp.py
├── auth.py
├── profile.py
├── aweme.py
├── onboard.py
├── phone.py
├── send_phone.py
├── send_follow.py
├── proxy.py
├── session_store.py
├── mssdk.py
├── identity.py
├── config.py
├── logutil.py
├── probe_paths.py
├── bin/
├── _vendor/
├── _metasec/
├── utils/
└── data/
```

---

### Entry points

| File | Role |
|------|------|
| **`__main__.py`** | Allows `python3 -m ttkit_clean` — delegates to `cli.main()`. |
| **`cli.py`** | Full argparse CLI: create, onboard, bind phone, update phone, follow, drain-pending. Parses all flags (`--domain`, `--frida`, `--no-activate`, etc.) and routes to the correct module. |
| **`__init__.py`** | Public API exports: `Fingerprint`, `create_account`, `onboard_existing_account`, `update_phone_after_login`, lazy imports to keep startup fast. |
| **`_bootstrap.py`** | Prepends `_vendor/` to `sys.path` so the package works with **zero pip install**. |

---

### Account creation

| File | Role |
|------|------|
| **`create.py`** | **Main orchestrator** — `create_account()`. Full loop: proxy pick → fingerprint → device mint → mail inbox → passport send_code → OTP poll → verify → save session → optional activation + phone bind. Handles error 7, sticky burn, anti-detect delays, Frida bootstrap, abort callbacks for dashboard integration. |
| **`register.py`** | `DeviceRegisterClient` — wraps HTTP device_register with retries, host rotation (`log16-*`), and response parsing into `device_id` / `install_id`. |
| **`device.py`** | Low-level **device_register mint** (`mint_musical_device`), `passport_params()`, `passport_form()`, `musical_ua()`, `warm_device_guard()`. Talks to SignServer for TTEncrypt. Selects IDC host from mint response. |
| **`fingerprints.py`** | Generates **device fingerprints**: `make_real_phone()` (Samsung, OPPO, Xiaomi, Huawei, realme, OnePlus) and `make_emulator()` (AVD / Frida only). `Fingerprint` dataclass with openudid, cdid, resolution, DPI, build_id, timezone, MCC, etc. Lite app constants (`aid=1340`, version 44.5.52). |
| **`identity.py`** | `mix()` — XOR encoding for passport lean fields (email, mobile, code, type). `email_local()` random local part. `infer_region_from_proxy()` geo from proxy string. |
| **`musical.py`** | Musical-specific passport helpers: lean form builder, random birthday, passport lean mode flags. |

---

### Signing & crypto

| File | Role |
|------|------|
| **`signing.py`** | **`passport_post()`** — builds signed POST to passport hosts. Assembles headers: X-Argus, X-Ladon, X-Gorgon, X-SS-STUB, sdk-version. Chooses offline SignerPy vs Frida MSSDK. Handles gzip response decode. |
| **`signatures.py`** | `SignFace` dataclass + `FACE_MUSICAL` (aid 1233) and `FACE_LITE` (aid 1340). `sign_passport()` / `sign_headers()` with correct Argus key per app. Documents golden rules: api16 gets Argus, api2 gets Gorgon only, device_register uses full TTEncrypt hex payload. |
| **`mssdk.py`** | Frida bridge client (`:8790`). `ensure_ready()`, `set_device()`, `sign_request()`. Calls `signer_server.py` HTTP API when `--frida` is enabled. |
| **`cookies.py`** | Cookie jar management: `cookie_header()`, `merge_response_cookies()`, `ensure_odin_tt()` synth, `warm_ms_token()`, passport host routing by IDC, guard header store merge/apply, blocks auth cookies during mint, seeds only safe cookies. |
| **`auth.py`** | Email **login** for existing accounts: `login_and_confirm()` — used by `--do-login` and phone update flows. |

---

### Email & OTP

| File | Role |
|------|------|
| **`otp.py`** | **`create_mailtm_inbox()`** — registers mail.tm account (prefers `web-library.net` domain). `get_otp_mailtm()` polling. `get_otp_any()` dispatches by domain. 1secmail support. IMAP reader for custom domain (`storage-te.com`). `_extract_otp()` 6-digit parser from HTML/text. |

---

### Activation, profile & aweme API

| File | Role |
|------|------|
| **`profile.py`** | **`activate_account_after_register()`** — post-register pipeline: warm session → FYP browse → likes → soft follow → nickname/bio commit → avatar upload → phone bind. `commit_user_with_backoff()` handles rate limit `3002284` with 10–40 min backoff. Enqueues deferred jobs to `onboard_pending.json`. |
| **`aweme.py`** | Generic **`aweme_request()`** — signed GET/POST to aweme endpoints with guard headers. `browse_for_you_feed()`, `digg_aweme()`, `soft_follow_user()`, `fetch_user_posts()`, `check_account_health()`, `warm_logged_in_session()`, `report_aweme_play()`, `resolve_warm_users()`. |
| **`onboard.py`** | **`onboard_existing_account()`** — reload session from `sessions/*.json`, fresh proxy, run activation without new register. **`drain_onboard_pending()`** — process deferred profile/avatar queue after cooldown. |

---

### Phone binding & change

| File | Role |
|------|------|
| **`phone.py`** | **5SIM integration**: `fivesim_buy_phone()`, `fivesim_wait_code()`, `fivesim_finish()`. **`bind_phone_to_account()`** and **`change_phone_on_account()`** — full send_code → SMS → bind loop with type fallback (`34`, `24`, `3731`, `2`). Handles `3052`/`3053`, operator retry, manual OTP stage. **`update_phone_after_login()`** — CLI entry for `--update-phone-only`. `format_mobile_for_lite()` → `+1 647…` before XOR. |
| **`send_phone.py`** | **Lite Go passport mobile** — `lite_go_query()` (aid=1340 full query string), `lite_mobile_lean_form()` (XOR body), `lite_passport_post()` (headers matching real Lite 44.5.52 captures: `x-tt-ultra-lite`, `x-tt-bypass-dp`, `ttzip`, Argus). Used by `phone.py` instead of musical aid=1233 path. |

---

### Follow (Lite Go)

| File | Role |
|------|------|
| **`send_follow.py`** | **`lite_relation_follow()`** — `POST /lite/v2/relation/follow/` with `body=null`, stub `46C03B52742B3F2615A3ABDF1636B754`. `resolve_username()` scrapes `user_id` + `sec_uid` from TikTok web. **`follow_with_session()`** loads saved session and sends follow. Standalone: `python3 -m ttkit_clean.send_follow`. |

---

### Proxy & sessions

| File | Role |
|------|------|
| **`proxy.py`** | Proxy pool from `data/proxies.txt`. `pick_fresh_proxies()`, `probe_proxy()`, sticky key tracking, cooldown (`proxy_used_ttkit.json`), `mark_sticky_burned()` on error 7, GeoNode session rotation, `ipr()` / `ipr_fresh()` IPRoyal helpers. |
| **`session_store.py`** | **`save_account_session()`** / **`load_account_session()`** — JSON bundles in `data/sessions/`. `fingerprint_from_session()`, `ensure_auth_cookies()`, `save_mailtm_inbox()`, onboard pending queue read/write, `accounts_ttkit.txt` append. |

---

### Config & utilities

| File | Role |
|------|------|
| **`config.py`** | All paths (`DATA_DIR`, `SESSIONS_DIR`, `SIGNER_JAR`), IDC → host map (`PASSPORT_HOST_BY_IDC`), musical app version constants, guard header names, cookie allow/block lists, profile backoff timings, proxy cooldown seconds. |
| **`logutil.py`** | `step()`, `ok()`, `fail()`, `warn()`, `info()` — structured console output; `-v` verbose gate. |
| **`probe_paths.py`** | Diagnostic script — probes SignServer, Frida, proxy, mail.tm reachability. |

---

### `bin/` — SignServer

| File | Role |
|------|------|
| **`bin/tt-signer.jar`** | Java SignServer — TTEncrypt, heavy crypto endpoints on `http://127.0.0.1:8000`. |
| **`bin/start_signer.sh`** | Bash launcher — skips start if port 8000 already listening. |
| **`bin/start_signer.py`** | Python alternative launcher with port check. |

---

### `data/` — runtime (not in public repos)

| Path | Role |
|------|------|
| **`data/proxies.txt`** | Proxy pool — `host:port:user:pass` per line. |
| **`data/.env`** | Secrets: `FIVESIM_API_TOKEN`, country/operator overrides. |
| **`data/sessions/*.json`** | Per-account full session bundles. |
| **`data/accounts_ttkit.txt`** | Compact log: `email:pw:session_key:did:iid`. |
| **`data/mailtm_inboxes.json`** | mail.tm inbox passwords for OTP re-read. |
| **`data/onboard_pending.json`** | Deferred profile/avatar jobs after rate limit. |
| **`data/proxy_used_ttkit.json`** | Proxy burn timestamps / cooldowns. |
| **`data/warm_aweme_ids.json`** | Cached video IDs for warm-up likes. |
| **`data/pending_manual_phone.json`** | Manual phone OTP bind state. |

---

### `_vendor/` — bundled Python deps

Pre-installed, no `pip install` needed:

| Package | Use |
|---------|-----|
| **requests** + urllib3 | HTTP client |
| **SignerPy** | Gorgon, Argus, Ladon, TTEncrypt |
| **pycryptodome** | AES, RSA, hash primitives |
| **gmssl** | SM2/SM3/SM4 (Chinese crypto suite) |
| **pytz** | Timezone data |
| **certifi** | CA certificates |
| **user_agent** | Random User-Agent strings |

---

### `_metasec/` — local signing engine

Offline Argus/Ladon implementation extracted from Metasec research. Used by `signatures.py` when SignServer or SignerPy path is selected. Separate keys per `aid` (1233 vs 1340).

---

### `utils/` — legacy helpers

| File | Role |
|------|------|
| **`utils/api.py`** | `Applog`, `Device`, `Utils`, `Xlog` classes — TTEncrypt payload build, device_register low-level, xlog reporting. Used by `device.py` and mint path. |

---

## How account creation works (overview)

```
Proxy → device_register (mint did/iid)
      → create mail.tm inbox
      → passport send_code (Argus signed)
      → poll OTP
      → passport verify (register)
      → save session JSON
      → [optional] activate: FYP + likes + profile + 5SIM phone bind
```

### CLI commands (full package)

```bash
# Parent folder must contain ttkit_clean/
python3 -m ttkit_clean --domain mailtm --no-activate --no-bind-phone -v   # minimal
python3 -m ttkit_clean --domain mailtm -v                                  # full
python3 -m ttkit_clean --frida --domain mailtm -v                          # Frida MSSDK
python3 -m ttkit_clean --onboard-only --email=user@mail.tm -v
python3 -m ttkit_clean --bind-phone-only --email=user@mail.tm -v
python3 -m ttkit_clean --update-phone-only --email=user@mail.tm -v
python3 -m ttkit_clean --follow --email=user@mail.tm --target=coderoot.ksa -v
```

### System requirements

- Python ≥ 3.9
- Java (SignServer JAR)
- Residential/mobile proxies
- Optional: Android emulator + Frida + TikTok Lite
- Optional: 5SIM API token

---

## Common error codes

| Code | Stage | Meaning |
|------|-------|---------|
| `error 7` | passport | Session/proxy risk block |
| `3053` | phone send_code | Number rejected (VOIP / geo / risk) |
| `3052` | phone bind | Account/session/IP too risky |
| `3002284` | profile | Edit rate limit — retry later |

---

## Disclaimer

Educational / research documentation only. Use responsibly and in compliance with TikTok ToS and local laws.

---

## License & purchase

**Source code is not open source.** This repo is documentation only.

For the full package, updates, support, or commercial license:

**GitHub:** [code-root](https://github.com/code-root)

---

---

# العربية

> ⚠️ **هذا المستودع يحتوي على التوثيق فقط.**  
> الكود المصدري **منتج مدفوع / مرخّص** وغير منشور هنا.  
> **للشراء أو التحديثات أو العرض التجريبي:** تواصل معي عبر [GitHub — code-root](https://github.com/code-root).

---

## ما هو ttkit_clean؟

حزمة Python مستقلة لأتمتة حسابات TikTok:

- تسجيل جهاز (`device_register`) ببصمات واقعية
- توقيع: **Argus / Gorgon / Ladon / TTEncrypt**
- تسجيل بالإيميل + OTP (mail.tm، 1secmail، IMAP)
- تفعيل بعد التسجيل (FYP، لايك، بروفايل، صورة)
- ربط/تغيير رقم عبر **5SIM** + مسار **Lite Go**
- فولو Lite Go
- Frida MSSDK اختياري
- حفظ جلسة كاملة

---

## شرح الملفات — مرجع كامل

### نقاط الدخول

| الملف | الوظيفة |
|-------|---------|
| **`__main__.py`** | تشغيل `python3 -m ttkit_clean` |
| **`cli.py`** | واجهة CLI كاملة — كل الأوامر والخيارات |
| **`__init__.py`** | API عام: `create_account`, `Fingerprint`, … |
| **`_bootstrap.py`** | تفعيل `_vendor/` تلقائياً بدون pip |

### إنشاء الحساب

| الملف | الوظيفة |
|-------|---------|
| **`create.py`** | المنسّق الرئيسي — حلقة كاملة من البروكسي حتى حفظ الجلسة والتفعيل |
| **`register.py`** | عميل device_register مع إعادة محاولة وتدوير hosts |
| **`device.py`** | mint الجهاز، باراميترات passport، warm device guard |
| **`fingerprints.py`** | بصمات هواتف حقيقية ومحاكي — `Fingerprint` dataclass |
| **`identity.py`** | XOR `mix()` للحقول، توليد إيميل، استنتاج المنطقة من البروكسي |
| **`musical.py`** | مساعدات passport لـ musical (aid 1233) |

### التوقيع والتشفير

| الملف | الوظيفة |
|-------|---------|
| **`signing.py`** | `passport_post()` — تجميع الترويسات الموقّعة |
| **`signatures.py`** | تعريف FACE_MUSICAL / FACE_LITE ومفاتيح Argus |
| **`mssdk.py`** | عميل جسر Frida على `:8790` |
| **`cookies.py`** | إدارة الكوكيز، odin_tt، guard headers، توجيه hosts |
| **`auth.py`** | تسجيل دخول بالإيميل لحساب موجود |

### البريد و OTP

| الملف | الوظيفة |
|-------|---------|
| **`otp.py`** | إنشاء صندوق mail.tm، polling OTP، 1secmail، IMAP |

### التفعيل والبروفايل

| الملف | الوظيفة |
|-------|---------|
| **`profile.py`** | تفعيل كامل بعد التسجيل + backoff عند 3002284 |
| **`aweme.py`** | طلبات aweme موقّعة: FYP، لايك، فولو، صحة الحساب |
| **`onboard.py`** | تفعيل حساب محفوظ + استكمال المهام المؤجلة |

### الهاتف

| الملف | الوظيفة |
|-------|---------|
| **`phone.py`** | 5SIM + bind/change + معالجة 3052/3053 + OTP يدوي |
| **`send_phone.py`** | مسار Lite Go لـ passport mobile (aid 1340، XOR lean body) |

### الفولو

| الملف | الوظيفة |
|-------|---------|
| **`send_follow.py`** | `POST /lite/v2/relation/follow/` مع `body=null` |

### البروكسي والجلسات

| الملف | الوظيفة |
|-------|---------|
| **`proxy.py`** | ملف البروكسيات، cooldown، حرق sticky |
| **`session_store.py`** | حفظ/تحميل الجلسات JSON |

### إعدادات وأدوات

| الملف | الوظيفة |
|-------|---------|
| **`config.py`** | مسارات، خرائط IDC، ثوابت التطبيق |
| **`logutil.py`** | سجلات ملونة step/ok/fail |
| **`probe_paths.py`** | فحص SignServer وFrida والبروكسي |

### `bin/`

| الملف | الوظيفة |
|-------|---------|
| **`tt-signer.jar`** | SignServer Java على :8000 |
| **`start_signer.sh`** | تشغيل السيرفر |

### `data/` (بيانات تشغيل — لا تُرفع للعامة)

| المسار | الوظيفة |
|--------|---------|
| `proxies.txt` | قائمة البروكسيات |
| `.env` | توكن 5SIM |
| `sessions/*.json` | جلسات الحسابات |
| `accounts_ttkit.txt` | سجل مختصر |
| `mailtm_inboxes.json` | بيانات صناديق mail.tm |
| `onboard_pending.json` | مهام مؤجلة |
| `proxy_used_ttkit.json` | cooldown البروكسي |

### `_vendor/` و `_metasec/` و `utils/`

- **`_vendor/`** — requests, SignerPy, pycryptodome, gmssl, pytz (بدون pip)
- **`_metasec/`** — Argus/Ladon أوفلاين
- **`utils/api.py`** — TTEncrypt, Applog, device_register منخفض المستوى

---

## أوامر CLI (الحزمة الكاملة)

```bash
python3 -m ttkit_clean --domain mailtm --no-activate --no-bind-phone -v
python3 -m ttkit_clean --domain mailtm -v
python3 -m ttkit_clean --frida --domain mailtm -v
python3 -m ttkit_clean --follow --email=USER@mail.tm --target=coderoot.ksa -v
```

---

## الترخيص والشراء

**الكود غير مفتوح المصدر.** هذا المستودع للتوثيق فقط.

**GitHub:** [code-root](https://github.com/code-root)

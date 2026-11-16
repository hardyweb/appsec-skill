# appsec

AppSec secure coding skill untuk agent AI. Cherry-picked dari [VibeSec-Skill](https://github.com/BehiSecc/VibeSec-Skill) — bukan copy penuh.

Satu fail Markdown. Tiada dependency, tiada tooling. Jalan pada mana-mana agent yang baca skill file (PiG, Claude Code, Cursor, Codex, Copilot, Antigravity).

```
appsec/SKILL.md    # 172 baris
```

## Masalah yang ia selesaikan

LLM optimize untuk "kod jalan". Minta halaman detail invois, anda dapat halaman detail invois yang berfungsi — tanpa ownership check, sebab anda tak minta. Tiada syntax error, tiada test gagal. Ia cuma senyap-senyap pulangkan data orang lain.

Vibe coding bawa anda ke 40 endpoint dalam satu hujung minggu. Empat puluh endpoint bermakna empat puluh peluang lupa authorization check.

Skill ini ajar model baca kod dari perspektif bug hunter, bukan code generator.

## Kenapa bukan copy penuh VibeSec

VibeSec 757 baris. Banyak overlap dengan skill infra dan framework yang saya dah ada:

| Kandungan VibeSec | Status |
|---|---|
| Secrets policy, least privilege, audit logging, CIS hardening, WAF, fail2ban, FIM | Buang — `security` dah cover |
| FormRequest, Blade, queue, Horizon, versi L11-L13 | Buang — `laravel` dah cover |
| IDOR, XSS sinks, CSRF pre-auth, open redirect bypass, SSRF bypass, upload magic bytes, SQLi edge cases, XXE flags, path traversal, JWT, GraphQL | **Ambil** |

Hasil: 172 baris. Setiap baris adalah sesuatu yang belum ada di tempat lain.

Prinsip: **skill patut miliki satu layer, bukan satu topik.** Overlap bakar context dan latih anda untuk abaikan peraturan sendiri.

## Kandungan

| Seksyen | Isi |
|---|---|
| `[ACCESS CONTROL]` | Ownership di data layer, tenant isolation, 404 bukan 403, UUID, mass assignment, token revocation |
| `[CLIENT-SIDE]` | XSS ikut konteks, sumber XSS tak langsung, CSP, header, CSRF termasuk endpoint pre-auth |
| `[OPEN REDIRECT]` | Allowlist/indirect ref, jadual bypass (`@`, `%252f`, null byte, IDN homograph) |
| `[PASSWORDS]` | Panjang minimum, Argon2id/bcrypt/scrypt sahaja, tiada MD5/SHA1 |
| `[SSRF]` | Allowlist, DNS pin, format IP bypass, cloud metadata `169.254.169.254` |
| `[FILE UPLOAD]` | Allowlist extension + magic bytes + parse sebenar, rename UUID, ZIP slip, polyglot |
| `[SQL INJECTION]` | Parameterized utama; ORDER BY/LIMIT/IN/LIKE kena allowlist |
| `[XXE]` | Flag khusus PHP/Python/Java/.NET |
| `[PATH TRAVERSAL]` | Canonicalize + `commonpath` |
| `[JWT]` | Algorithm allowlist, `none` reject, `exp`, `jti`, HttpOnly cookie |
| `[GRAPHQL]` | Introspection off, depth/cost limit |
| `[AUTHZ CHECKLIST]` | Enam semakan pantas sebelum merge |
| `[AUDIT MODE — READ-ONLY]` | Audit app sedia ada tanpa ubah kod |
| `[SECRET SCAN]` | Kesan key/token bocor guna grep + git sahaja |
| `[STATE]` | Approval gate untuk tindakan destruktif |

## Bahagian yang paling berguna

### SSRF — jadual bypass

Semua orang tahu "validate URL". Hampir tiada siapa tahu senarainya:

```
2130706433              # perpuluhan 127.0.0.1
0177.0.0.1              # oktal
0x7f.0.0.1              # heks
[::1]                   # IPv6 loopback
[::ffff:127.0.0.1]      # IPv4-mapped
127.1                   # dipendekkan
[fe80::1%25eth0]        # IPv6 scope ID
```

Tambah DNS rebinding: resolve, validate, **pin IP**, jangan re-resolve. Dan block `169.254.169.254` — endpoint metadata cloud. Perbezaan antara feature link preview dan serahkan credential instance.

### SQLi — bahagian yang tak boleh parameterize

Parameterized query dah jadi muscle memory. Yang menarik ialah apa yang tak boleh di-bind:

- Nama kolum `ORDER BY`
- `LIMIT` / `OFFSET`
- Nama table dan kolum
- `IN (...)` dengan senarai dinamik
- Pattern `LIKE` (dan `%` / `_` perlu di-escape)

Semua itu perlu allowlist. Di sinilah "saya guna ORM" senyap-senyap jadi tak betul.

### File upload = XXE menyamar

DOCX, XLSX, PPTX dan SVG semuanya ZIP+XML. Jadi "kami hanya benarkan upload dokumen" juga bermakna "kami terima XML kawalan user masuk parser kami".

Magic bytes:

```
JPEG       FF D8 FF
PNG        89 50 4E 47 0D 0A 1A 0A
GIF        47 49 46 38
PDF        25 50 44 46
ZIP/DOCX   50 4B 03 04
```

## Audit mode (read-only)

Untuk app yang dah siap. Default: **baca sahaja, tak ubah kod**.

Tanpa peraturan eksplisit, agent yang bersemangat akan mula refactor production sementara anda masih baca diff.

**Dibenarkan:** `read`, `rg`/`grep`, `git log/diff/show`, semak manifest dependency, semak config, semak versi framework.

**Tak dibenarkan tanpa kelulusan:** tulis/refactor, migration, `composer install`, restart service, tulis ke DB, jalankan app, exploit aktif (`sqlmap`, ZAP, `curl` payload) ke prod.

Format laporan — kumpul ikut severity (Critical/High/Medium/Low):

```
file:line — kelas vuln — kenapa boleh dieksploit — cadangan fix (satu baris, tiada patch)
```

Diakhiri dengan: apa yang disemak, apa yang **tak** disemak (perlu kelulusan), senarai fix ikut keutamaan.

Kod hanya diubah bila diminta secara eksplisit selepas baca laporan.

### Grep Laravel

```bash
rg '\$request->all\(\)|->update\(\$request'   # mass assignment
rg 'DB::raw|whereRaw|orderByRaw'               # SQL mentah
rg '\{!!.*!!\}'                                # Blade tak escape
rg '->withoutMiddleware'                       # auth dilangkau
rg 'APP_DEBUG=true'                            # debug di prod
rg 'unserialize|eval\('                        # surface RCE
rg 'Http::get\(\$'                             # calon SSRF
rg 'Storage::.*\$'                             # calon path traversal
```

Sepuluh grep, lima minit, anda dah ada shortlist sebelum baca satu controller pun.

## Secret scan — grep + git sahaja

Tak perlu gitleaks atau trufflehog. Tak semua mesin ada.

**1. Working tree** — regex untuk AWS (`AKIA`/`ASIA`), GitHub (`ghp_`/`github_pat_`), Stripe (`sk_live_`/`sk_test_`/`rk_live_`), Slack (`xox[baprs]-`), Google (`AIza`/`ya29.`), SendGrid (`SG.`), private key PEM, JWT, generic `api_key=`/`password=`/`token=`, credential dalam URL.

**2. Built assets** — `public/ dist/ build/ static/ .next/`. Paling bahaya, sebab memang awam. Semak juga `NEXT_PUBLIC_*`/`REACT_APP_*`/`VITE_*` dalam `.env`.

**3. Git history** — secret yang dah dipadam masih hidup di sini:

```bash
git log --all -p -S 'AKIA' --oneline | head -50
git log --all --diff-filter=D --name-only --pretty=format: | sort -u | grep -E '\.env|\.pem|\.key|id_rsa'
```

**4. Noise reduction** — skip `.env.example`, `*.md`, `*.lock`, vendor; skip placeholder (`changeme`, `your-key-here`, `${...}`, `env(`); heuristik high-entropy untuk string 32+ aksara bercampur huruf besar/kecil/digit.

**5. Report** — mask nilai (`AKIAIOSF...MPLE`), label exposed (git history / built asset / local only), action ladder.

**Prinsip utama: rotate, bukan padam.** Secret dalam git history atau bundle awam = dah compromised. Padam baris tak betul-betul selamatkan apa-apa.

## Pemasangan

PiG:

```bash
mkdir -p ~/.pig/agent/skills/appsec
curl -o ~/.pig/agent/skills/appsec/SKILL.md \
  https://raw.githubusercontent.com/hardyweb/appsec-skill/main/appsec/SKILL.md
```

Claude Code: `~/.claude/skills/appsec/`
Cursor: `~/.cursor/skills/appsec/`
Codex: `~/.agents/skills/appsec/`
Copilot: `~/.copilot/skills/appsec/`
Antigravity: `~/.gemini/antigravity/skills/appsec/`

Atau letak dalam direktori project untuk skop project sahaja. Restart session selepas tambah.

## Guna

```
/skill appsec
```

Atau cuma tanya:

> Audit app Laravel dalam `~/sandbox/invois`. Read-only, jangan ubah apa-apa.

> Saya nak bina endpoint webhook yang terima URL dari user. Semak SSRF dulu.

> Semak repo ni untuk secret yang bocor.

## Skop

| Skill | Skop |
|---|---|
| `security` | Infra: SSH, firewall, WAF, auditd, hardening, secrets management |
| `laravel` | Framework: L11-L13, FormRequest, Blade, queue, Horizon |
| `appsec` | Kod app: authz, XSS, CSRF, SSRF, SQLi, XXE, upload, JWT, redirect, secret scan |

Skill ini tak cover: SAST penuh, dependency CVE scanning, pen-testing aktif, atau compliance framework. Untuk pen-testing guna `security-testing`; untuk dependency, `composer audit` / `npm audit` / `govulncheck`.

## Kredit

Kandungan asal: [BehiSecc/VibeSec-Skill](https://github.com/BehiSecc/VibeSec-Skill) oleh [@Behi_Sec](https://x.com/Behi_Sec). Liputan berbayar lebih mendalam di [vibesec.sh](https://vibesec.sh).

Perubahan dalam repo ini: dipotong kepada layer AppSec sahaja, tambah audit mode read-only, tambah secret scan tanpa dependency, tambah grep Laravel.

## Lesen

Ikut lesen asal VibeSec-Skill. Lihat `LICENSE` jika ada.

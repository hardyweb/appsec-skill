# AppSec

Secure coding skill untuk AI coding agent.

Skill ini diadaptasi daripada [VibeSec-Skill](https://github.com/BehiSecc/VibeSec-Skill). Saya tidak mengambil keseluruhan kandungannya kerana sebahagian panduan keselamatan sudah tersedia dalam skill lain yang saya gunakan.

Fokus `appsec` ialah keselamatan pada peringkat kod aplikasi seperti access control, XSS, CSRF, SSRF, SQL injection dan file upload.

Semua panduan disimpan dalam satu fail Markdown tanpa dependency atau tooling tambahan.

```text
appsec/SKILL.md    # 172 baris
```

## Kenapa buat skill ini?

AI coding agent boleh menghasilkan kod yang berfungsi, tetapi kod yang berjaya dijalankan tidak semestinya selamat.

Contohnya, apabila diminta membina halaman detail invois, agent mungkin menghasilkan query dan paparan yang betul. Tetapi adakah ia memastikan pengguna hanya boleh melihat invois milik sendiri?

Perkara seperti ini mudah terlepas jika agent hanya menumpukan kepada requirement yang diberikan.

Saya mahu agent turut melihat kod daripada sudut keselamatan, bukan sekadar memastikan fungsi yang diminta berjalan.

## Kenapa tidak ambil keseluruhan VibeSec?

VibeSec-Skill mempunyai sekitar 757 baris panduan. Banyak kandungannya bertindih dengan skill `security` dan `laravel` yang sudah saya gunakan.

Saya hanya mengambil bahagian yang berkaitan secara langsung dengan keselamatan kod aplikasi.

| Kandungan VibeSec | Tindakan |
|---|---|
| Secrets policy, least privilege, audit logging, CIS hardening, WAF, fail2ban, FIM | Tidak diambil — sudah diliputi oleh `security` |
| FormRequest, Blade, queue, Horizon dan panduan Laravel L11-L13 | Tidak diambil — sudah diliputi oleh `laravel` |
| IDOR, XSS, CSRF, SSRF, SQLi, XXE, path traversal, JWT, GraphQL dan beberapa teknik bypass | Dikekalkan |

Hasilnya ialah satu skill yang lebih kecil, dengan skop yang lebih jelas.

Prinsip yang saya gunakan: setiap skill perlu mempunyai tanggungjawab tersendiri. Kandungan yang sama tidak perlu diulang kerana ia menggunakan context agent dan boleh menyebabkan arahan menjadi sukar diselenggara.

## Kandungan

| Seksyen | Skop |
|---|---|
| `[ACCESS CONTROL]` | Ownership check, tenant isolation, 404/403, UUID, mass assignment dan token revocation |
| `[CLIENT-SIDE]` | XSS, CSP, security headers dan CSRF |
| `[OPEN REDIRECT]` | Allowlist dan teknik bypass URL validation |
| `[PASSWORDS]` | Password hashing menggunakan Argon2id, bcrypt dan scrypt |
| `[SSRF]` | URL validation, DNS rebinding, IP bypass dan cloud metadata |
| `[FILE UPLOAD]` | Extension, magic bytes, file parsing, ZIP slip dan polyglot files |
| `[SQL INJECTION]` | Parameterized queries dan allowlist untuk dynamic SQL |
| `[XXE]` | Konfigurasi XML parser untuk PHP, Python, Java dan .NET |
| `[PATH TRAVERSAL]` | Path canonicalization dan validation |
| `[JWT]` | Algorithm validation, expiration dan token revocation |
| `[GRAPHQL]` | Introspection, query depth dan complexity limits |
| `[AUTHZ CHECKLIST]` | Semakan authorization sebelum merge |
| `[AUDIT MODE — READ-ONLY]` | Audit kod sedia ada tanpa membuat perubahan |
| `[SECRET SCAN]` | Mencari secret dalam working tree, build assets dan Git history |
| `[STATE]` | Approval sebelum tindakan yang boleh mengubah atau memusnahkan data |

## Beberapa perkara yang diberi perhatian

### SSRF

URL validation bukan sekadar memeriksa sama ada URL menggunakan protokol HTTP atau HTTPS.

Alamat IP boleh ditulis dalam beberapa format yang berbeza. Contohnya:

```text
2130706433          # Decimal representation of 127.0.0.1
0177.0.0.1          # Octal
0x7f.0.0.1          # Hexadecimal
[::1]               # IPv6 loopback
[::ffff:127.0.0.1]  # IPv4-mapped IPv6
127.1               # Shortened IPv4
```

Skill ini turut mengingatkan agent tentang DNS rebinding, IP pinning dan akses kepada cloud metadata endpoint seperti `169.254.169.254`.

### SQL Injection

Parameterized query merupakan amalan asas, tetapi tidak semua bahagian SQL boleh menggunakan bind parameter.

Antara contoh yang memerlukan validation dan allowlist:

- Dynamic column dalam `ORDER BY`.
- Table dan column names.
- Dynamic `LIMIT` dan `OFFSET`.
- Senarai nilai dalam `IN`.
- Pattern dalam `LIKE`, termasuk escaping wildcard.

Penggunaan ORM juga tidak bermaksud semua query secara automatik selamat. Penggunaan raw expression dan dynamic query masih perlu diperiksa.

### File Upload

File upload memerlukan lebih daripada sekadar pemeriksaan extension.

Fail seperti DOCX, XLSX dan PPTX menggunakan format ZIP yang mengandungi XML. SVG pula merupakan dokumen XML.

Oleh itu, proses upload perlu mengambil kira jenis fail sebenar, kandungan fail dan cara parser memprosesnya.

Antara magic bytes yang digunakan untuk pemeriksaan awal:

```text
JPEG       FF D8 FF
PNG        89 50 4E 47 0D 0A 1A 0A
GIF        47 49 46 38
PDF        25 50 44 46
ZIP        50 4B 03 04
```

Magic bytes sahaja tidak mencukupi untuk mengesahkan keselamatan fail. Parsing dan validation masih diperlukan.

## Audit mode (read-only)

Satu bahagian yang saya tambah sendiri ialah audit mode.

Ia digunakan apabila agent diminta menyemak aplikasi yang sudah dibangunkan. Secara default, agent hanya dibenarkan membaca dan menganalisis kod.

**Dibenarkan:**

- Membaca fail dan direktori.
- Menggunakan `rg` atau `grep`.
- Menyemak `git log`, `git diff` dan `git show`.
- Memeriksa dependency manifest dan konfigurasi.
- Menyemak versi framework dan dependency.

**Tidak dibenarkan tanpa kelulusan:**

- Mengubah atau refactor kod.
- Menjalankan migration.
- Memasang dependency.
- Restart service.
- Menulis ke database.
- Menjalankan aplikasi atau ujian yang memberi kesan kepada production.
- Menjalankan exploit aktif terhadap production.

Laporan audit menggunakan format berikut:

```text
file:line — vulnerability class — exploitation risk — recommended fix
```

Finding dikumpulkan mengikut severity: Critical, High, Medium dan Low.

Laporan juga perlu menyatakan bahagian yang telah diperiksa, perkara yang belum diperiksa dan cadangan pembaikan mengikut keutamaan.

Sebarang perubahan kod dibuat hanya selepas mendapat arahan yang jelas.

## Laravel quick checks

Saya turut masukkan beberapa arahan `rg` untuk membantu mencari bahagian kod yang perlu diperiksa semasa audit Laravel.

```bash
rg '\$request->all\(\)|->update\(\$request'   # Mass assignment
rg 'DB::raw|whereRaw|orderByRaw'               # Raw SQL
rg '\{!!.*!!\}'                                # Unescaped Blade output
rg '->withoutMiddleware'                       # Middleware bypass
rg 'APP_DEBUG=true'                             # Debug configuration
rg 'unserialize|eval\('                         # Potential code execution
rg 'Http::get\(\$'                              # Potential SSRF
rg 'Storage::.*\$'                              # Potential path traversal
```

Arahan ini hanya digunakan untuk mencari calon isu keselamatan. Setiap hasil masih perlu diperiksa secara manual kerana tidak semua padanan semestinya vulnerability.

## Secret scan

Saya juga masukkan pemeriksaan secret menggunakan `grep` dan Git tanpa memerlukan gitleaks atau trufflehog.

Pemeriksaan meliputi:

1. Working tree — API keys, tokens, private keys dan credentials.
2. Built assets — fail dalam `public`, `dist`, `build`, `static` dan `.next`.
3. Git history — secret yang pernah dimasukkan dan kemudian dipadam.
4. Noise reduction — mengabaikan placeholder dan fail yang biasanya tidak mengandungi secret sebenar.
5. Reporting — mask nilai secret dan nyatakan lokasi pendedahan.

Satu perkara penting: jika secret pernah masuk ke Git history atau public build, jangan anggap ia selamat hanya kerana baris tersebut sudah dipadam.

Secret berkenaan perlu dianggap compromised dan diganti.

## Pemasangan

### PiG

```bash
mkdir -p ~/.pig/agent/skills/appsec

curl -o ~/.pig/agent/skills/appsec/SKILL.md \
  https://raw.githubusercontent.com/hardyweb/appsec-skill/main/appsec/SKILL.md
```

### Lokasi skill lain

| Agent | Direktori |
|---|---|
| Claude Code | `~/.claude/skills/appsec/` |
| Cursor | `~/.cursor/skills/appsec/` |
| Codex | `~/.agents/skills/appsec/` |
| Copilot | `~/.copilot/skills/appsec/` |
| Antigravity | `~/.gemini/antigravity/skills/appsec/` |

Lokasi ini bergantung pada format dan konfigurasi skill yang disokong oleh setiap agent. Untuk penggunaan dalam satu projek sahaja, letakkan fail dalam direktori skill projek.

## Penggunaan

Aktifkan skill melalui mekanisme yang disediakan oleh agent masing-masing.

Contoh arahan:

```text
Audit aplikasi Laravel dalam ~/sandbox/invois.
Gunakan appsec dan read-only mode. Jangan ubah kod.
```

```text
Saya mahu membina webhook endpoint yang menerima URL
daripada pengguna. Semak risiko SSRF sebelum mencadangkan
implementation.
```

```text
Semak repository ini untuk secret yang mungkin terdedah,
termasuk Git history dan built assets.
```

## Skop dan pembahagian skill

Dalam setup saya, panduan keselamatan dibahagikan kepada tiga bahagian:

| Skill | Skop |
|---|---|
| `security` | Infrastruktur: SSH, firewall, WAF, auditd, hardening dan secrets management |
| `laravel` | Laravel: FormRequest, Blade, queue, Horizon dan framework-specific practices |
| `appsec` | Application security: authorization, XSS, CSRF, SSRF, SQLi, XXE, file upload dan JWT |

`appsec` bukan pengganti kepada SAST, dependency vulnerability scanning atau penetration testing.

Untuk pemeriksaan tambahan, gunakan tooling yang bersesuaian seperti:

```bash
composer audit
npm audit
govulncheck
```

Untuk penetration testing, gunakan skill atau tooling khusus seperti `security-testing`.

## Kredit

Skill ini diadaptasi daripada [BehiSecc/VibeSec-Skill](https://github.com/BehiSecc/VibeSec-Skill) oleh [@Behi_Sec](https://x.com/Behi_Sec).

Perubahan yang dibuat:

- Memilih kandungan yang berkaitan dengan application security.
- Mengelakkan pertindihan dengan skill `security` dan `laravel`.
- Menambah audit mode read-only.
- Menambah secret scanning menggunakan grep dan Git.
- Menambah beberapa quick checks khusus Laravel.

## Lesen

Sila rujuk lesen asal VibeSec-Skill dan fail `LICENSE` dalam repository ini untuk maklumat penggunaan dan pengagihan semula.

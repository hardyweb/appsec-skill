
  # appsec

   AppSec secure coding skill untuk PiG. Cherry-picked dari [VibeSec-Skill](https://github.com/BehiSecc/VibeSec-Skill) (BehiSecc),
 bukan copy penuh.

   ## Kenapa bukan copy penuh

   VibeSec 757 baris, banyak overlap dengan skill sedia ada:

   | Kandungan VibeSec | Status |
   |---|---|
   | Secrets policy, least privilege, audit logging, hardening, WAF, fail2ban, FIM | Buang — dah ada dalam `security` |
   | Idiom framework, FormRequest, Blade, queue, versi L11-L13 | Buang — dah ada dalam `laravel` |
   | IDOR, XSS sinks, CSRF pre-auth, open redirect bypass, SSRF bypass, upload magic bytes, SQLi non-parameterizable, XXE flags, path
 traversal, JWT, GraphQL | Ambil |

   Hasil: 118 baris. Fokus kod aplikasi web sahaja.

   ## Bila guna

   - Tulis kod web app baru — secure-by-default dari mula
   - Review PR / diff sebelum merge
   - Audit app yang dah dibangunkan (read-only, lihat bawah)

   ## Skop

   | Skill | Skop |
   |---|---|
   | `security` | Infra: SSH, firewall, WAF, auditd, hardening, secrets management |
   | `laravel` | Framework: L11-L13, FormRequest, Blade, queue, Horizon |
   | `appsec` | Kod app: authz, XSS, CSRF, SSRF, SQLi, XXE, upload, JWT, redirect |

   ## Kandungan

   - **Access control** — ownership di data layer, tenant isolation, 404 bukan 403, UUID, mass assignment, token revocation
   - **Client-side** — XSS ikut konteks, sumber XSS tak langsung, CSP, header, CSRF termasuk endpoint pre-auth
   - **Open redirect** — allowlist/indirect ref, jadual bypass (`@`, `%252f`, null byte, IDN homograph)
   - **Password** — panjang minimum, Argon2id/bcrypt/scrypt sahaja
   - **SSRF** — allowlist, DNS pin, format IP bypass (decimal/octal/hex/IPv6), cloud metadata `169.254.169.254`
   - **File upload** — allowlist extension + magic bytes + parse sebenar, rename UUID, ZIP slip, polyglot
   - **SQL injection** — parameterized utama; ORDER BY/LIMIT/IN/LIKE kena allowlist
   - **XXE** — flag khusus PHP/Python/Java/.NET
   - **Path traversal** — canonicalize + `commonpath`
   - **JWT** — algorithm allowlist, `none` reject, `exp`, `jti`, HttpOnly cookie
   - **GraphQL** — introspection off, depth/cost limit

   ## Audit mode (read-only)

   Untuk app yang dah siap. Default: **baca sahaja, tak ubah kod**.

   Dibenarkan: `read`, `rg`/`grep`, `git log/diff/show`, semak manifest dependency, semak config, semak versi framework.

   Tak dibenarkan tanpa kelulusan: tulis/refactor, migration, `composer install`, restart service, tulis ke DB, jalankan app, exploit
 aktif (`sqlmap`, ZAP, `curl` payload) ke prod.

   Format laporan — kumpul ikut severity (Critical/High/Medium/Low):



 file:line — kelas vuln — kenapa boleh dieksploit — cadangan fix (satu baris, tiada patch)



   Akhir laporan: apa yang disemak, apa yang **tak** disemak (perlu kelulusan), senarai fix ikut keutamaan.

   Kod hanya diubah bila diminta secara eksplisit selepas baca laporan.

   Grep Laravel khusus: `$request->all()`, `DB::raw`, `whereRaw`, `orderByRaw`, `{!! !!}` dalam Blade, `->withoutMiddleware`,
 `APP_DEBUG=true`, `APP_KEY` dalam git, `authorize()` hilang, `Storage::` dengan path user, `Http::get($userUrl)`, `unserialize`,
 `eval`.

   Auto-discovered. Load dengan `/skill appsec`. Restart session selepas tambah/ubah skill.

   ## Kredit

   Kandungan asal: [BehiSecc/VibeSec-Skill](https://github.com/BehiSecc/VibeSec-Skill). Versi penuh berbayar di
 [vibesec.sh](https://vibesec.sh).




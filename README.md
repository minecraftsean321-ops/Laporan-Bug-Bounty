# Laporan-Bug-Bounty

| Nama                 | NRP        |
|----------------------|------------|
| Hasheemi Rafsanjani  | 5027251015 |
| Sean Arthur Tamajaya | 5027251050 |
| Mahrinza Redouane Z. | 5027251074 |

Link github: https://github.com/minecraftsean321-ops/Laporan-Bug-Bounty

## 1. Exposed .git Directory (Access Restricted) - Found by Sean (5027251050)

Ditemukannya file .git/config yang saat ini sudah di blok aksesnya tetapi bisa terbuka jika ada misconfig kecil (atau ada endpoint lain dalam .git/ yang nggak ke-cover rule block-nya).

code yang digunakan: 

```
curl https://servicedesk.its.ac.id/.git/config
```

<img width="788" height="195" alt="image" src="https://github.com/user-attachments/assets/e0879938-b747-4794-aed1-e86e4a417626" />

Percobaan scope .git/ yang lain:  

<img width="695" height="549" alt="image" src="https://github.com/user-attachments/assets/e0d19e25-a204-438c-9601-8fb4ffeacd80" />

Dari ketiga percobaan tersebut dapat disimpulkan bahwa block-nya itu bersifat folder wide.

Dampak: Saat ini rendah karena akses diblok. Tapi risiko residual tetap ada — perubahan konfigurasi server (misal migrasi, update vhost, atau salah deploy ulang) bisa membuka akses tanpa sengaja, yang berpotensi membocorkan seluruh source code & history commit.

Rekomendasi: Hapus folder .git dari webroot production sepenuhnya (bukan cuma block akses), dan pastikan proses deployment ke depan nggak nge-copy folder VCS ke server production.


## 2. Clickjacking — Missing X-Frame-Options / CSP frame-ancestors - Found by Sean (5027251050)

<img width="977" height="235" alt="image" src="https://github.com/user-attachments/assets/56dd9c7f-5dee-4c33-b3de-f6e53abee878" />  

Halaman service.its.ac.id tidak mengirimkan header X-FRAME-Options maupun Content-Security-Policy dengan direktif frame-ancestors. Akibatnya, halaman dapat di-embed secara penuh di dalam <iframe> milik domain pihak ketida manapun. 

Disini saya menggunakan kode HTML di bawah ini untuk mencoba meng-embed halaman service.its.ac.id:

```html
<!DOCTYPE html>
<html>
<head><title>Clickjacking PoC</title></head>
<body>
  <h3>Clickjacking Test - servicedesk.its.ac.id</h3>
  <iframe src="https://servicedesk.its.ac.id/" width="800" height="600"></iframe>
</body>
</html>
```

Dan saat file dibuka hasilnya seperti ini:  

<img width="1920" height="901" alt="image" src="https://github.com/user-attachments/assets/54382eca-5daf-4db0-a766-b05d74e6a238" />  

Dampak: Halaman ini rawan disalahgunakan untuk UI redress attack — attacker bisa overlay elemen transparan/tersamar di atas iframe buat mengelabui user supaya klik tombol tertentu (misal “Masuk”) tanpa sadar, berpotensi memicu aksi tidak diinginkan kalau dikombinasikan dengan teknik social engineering lain.

Rekomendasi: Tambahkan header X-Frame-Options: SAMEORIGIN (atau DENY kalau nggak butuh embed sama sekali), dan/atau Content-Security-Policy: frame-ancestors 'self'.


## 3. Deprecated TLS Protocol Versions Supported (TLS 1.0 & TLS 1.1) - Found by Sean (5027251050)

Deskripsi: Berdasarkan scan yang saya lakukan menggunakan testssl.sh, terdapat protokol TLS 1.0 & TLS 1.1 yang masih aktif padahal protokol tersebut telah di-deprecate resmi oleh IETF (RFC 8996, 2021) karena kelemahan kriptografi.

Langkah": 

Download tools testssl.sh.git terlebih dahulu: 

```
git clone --depth 1 https://github.com/drwetter/testssl.sh.git
```

Masuk ke directory github tersebut, lalu ubah akses ke file testssl.sh menjadi executable:

```
chmod +x testssl.sh
```

Lalu scan testssl.sh:

```
./testssl.sh https://servicedesk.its.ac.id
```

<img width="999" height="617" alt="image" src="https://github.com/user-attachments/assets/e1eace12-3c09-4d9c-afe7-af5724dc3429" />

<img width="1000" height="628" alt="image" src="https://github.com/user-attachments/assets/9165f3cb-bc28-4258-b420-16adefec588f" />


Severity: Medium

CWE: CWE-326 (Inadequate Encryption Strength) / CWE-327 (Use of a Broken or Risky Cryptographic Algorithm)

Dampak: 

1. Kerentanan protokol turunan — TLS 1.0/1.1 rentan terhadap serangan klasik seperti BEAST (eksploitasi cipher CBC) dan POODLE-style downgrade, meski mitigasi di sisi browser modern udah mengurangi risiko praktisnya.
2. Downgrade attack surface — selama protokol lama masih di-offer, ada celah teoretis buat attacker memaksa client “turun” ke protokol yang lebih lemah lewat manipulasi handshake (man-in-the-middle), terutama buat client lawas yang masih support TLS 1.0/1.1.
3. Non-compliance — banyak standar keamanan (PCI DSS sejak 2018, NIST SP 800-52 Rev. 2) udah melarang penggunaan TLS 1.0/1.1 di sistem yang menangani data sensitif. Kalau servicedesk ini nyimpan data pribadi mahasiswa/staff, ini jadi concern compliance juga.
4. Reputasi & grading eksternal — tools audit eksternal (SSL Labs, dll) bakal cap grade maks B (capped) selama protokol lama masih aktif, walau konfigurasi lain udah bagus.

Rekomendasi: 

1. Disable TLS 1.0 dan TLS 1.1 di konfigurasi server (nginx/Apache), sisakan TLS 1.2 minimum.
2. Aktifkan TLS 1.3 — saat ini hasil scan nunjukin TLS 1.3 tidak didukung sama sekali, padahal ini standar modern yang lebih cepat dan lebih aman (menghilangkan banyak kelas serangan lama by design).
3. Setelah disable, verifikasi ulang pakai command yang sama (openssl s_client -tls1) — harusnya hasilnya berubah jadi connection refused/handshake failure.
4. Review juga cipher suite yang di-offer, prioritaskan AEAD cipher (GCM) dibanding CBC untuk protokol yang masih dipertahankan.

Bukti bahwa TLS 1.0 masih aktif:

```
openssl s_client -connect servicedesk.its.ac.id:443 -tls1
```

<img width="956" height="595" alt="image" src="https://github.com/user-attachments/assets/f89417f1-831e-474e-a422-c43e3b1c7c61" />

Bukti bahwa TLS 1.1 masih aktif:

```
openssl s_client -connect servicedesk.its.ac.id:443 -tls1_1
```

<img width="959" height="594" alt="image" src="https://github.com/user-attachments/assets/209f8a45-9fa6-4f0d-b01e-a452aa0d81f2" />


## 4. Sensitive Information Disclosure via Directory Listing - Found by Sean (5027251050)

Deskripsi: Directory /upload/ pada aplikasi SIMPEG (Sistem Kepegawaian) ITS tidak memiliki proteksi index/listing, sehingga seluruh isi direktori dapat diakses dan dienumerasi oleh pihak yang tidak terautentikasi. Direktori ini berisi file dengan pola penamaan yang mengindikasikan nomor identitas pegawai (contoh: 58296-1.pdf, 58297-1.pdf) serta folder skp_xls/ yang kemungkinan berisi dokumen SKP (Sasaran Kerja Pegawai).

Sevirity: Medium - High

Steps to Reproduce: 

1. Coba akses `https://kepegawaian.its.ac.id/upload/` menggunakan curl

   ```
    curl -s https://kepegawaian.its.ac.id/upload/ | head -50
   ```

   Disitu dapat terlihat bahwa ada file dari pegawai dengan kode yang memiliki pola penamaan seperi yang disebutkan di deskripsi.

2. File individual dapat diakses langsung, contoh:

   ```
    curl -sI https://kepegawaian.its.ac.id/upload/58296-1.pdf
   ```
   Disitu terlihat respons nya HTTP/1.1 200 OK, yang menandakan file dapat diunduh tanpa autentikasi.

Dampak:
Pihak yang tidak berwenang dapat mengenumerasi dan mengunduh dokumen yang berpotensi berisi data pribadi karyawan (nama, NIP, penilaian kinerja), yang jika benar merupakan pelanggaran terhadap kerahasiaan data pribadi (berpotensi melanggar UU PDP).

Rekomendasi: 
1. Nonaktifkan directory listing pada konfigurasi Apache (Options -Indexes) untuk direktori /upload/.
2. Tambahkan file index.php/index.html kosong di setiap subdirektori sebagai mitigasi tambahan.
3. Pindahkan file-file sensitif ke luar web root, dan sajikan hanya melalui endpoint yang mewajibkan autentikasi & otorisasi.


## 5. Pengujian Tambahan - Found by Mahrinza Redouane Z. (5027251074)

**Waktu:** 24–25 Oktober 2026  
**Target:** `portal.its.ac.id`, `its.ac.id`, `its.id`  
**Metode:** Recon ringan, pemeriksaan HTTP/TLS, Firefox Developer Tools, dan analisis HTML/JavaScript lokal.

**Hasil utama:** Pada `portal.its.ac.id`, API `user-info` mengembalikan **401 sebelum login, 200 setelah login, dan 401 setelah logout**. Pemeriksaan akses anonim, pengumuman, browser storage, serta JavaScript tidak menunjukkan kebocoran data atau bypass yang terbukti. `its.ac.id` melakukan redirect ke `www.its.ac.id`, sedangkan `its.id` ke `shortener.its.ac.id`; tujuan redirect tidak diuji lebih lanjut karena cakupannya belum dikonfirmasi.

### Observasi Risiko Keamanan

| Observasi | Potensi risiko | Rekomendasi |
|---|---|---|
| Cookie `myits_portal_api_session` dan `PHPSESSID` terlihat tanpa `Secure`; `PHPSESSID` juga tanpa `HttpOnly`. | Paparan cookie pada kondisi tertentu; dampak bergantung pada fungsi cookie dan mitigasi HSTS yang sudah aktif. | Tinjau atribut `Secure` dan `HttpOnly` sesuai fungsi cookie. |
| Header API menampilkan `PHP/8.0.13`. | Jika merupakan versi runtime sebenarnya tanpa patch tambahan, perangkat lunak dapat berada di luar dukungan keamanan. | Verifikasi versi aktual dan status patch PHP. |
| CSP dan `X-Content-Type-Options` tidak terlihat pada respons root portal yang diperiksa. | Lapisan pertahanan browser berpotensi kurang lengkap; tidak membuktikan XSS. | Evaluasi CSP dan `X-Content-Type-Options`. |
| JavaScript merujuk `localStorage.getItem("token")`; salah satu request mengirim `Bearer null`. | Perlu peninjauan konsistensi mekanisme autentikasi; token tersimpan atau bypass tidak terbukti. | Tinjau pengelolaan token dan header otorisasi. |


**Bukti screenshot dari pengujian portal:**

<img width="1919" height="170" alt="bukti_1" src="https://github.com/user-attachments/assets/f6c15b19-7b5c-420c-8f97-b60d5caf94b5" />

*Bukti 1. Pengaturan `Secure` dan `HttpOnly` yang ditampilkan Firefox. Nilai cookie telah disensor; keberadaan atribut yang belum diaktifkan bukan bukti eksploitasi.*

<img width="1375" height="408" alt="bukti_2" src="https://github.com/user-attachments/assets/5cf22a14-d3c9-47ef-9719-8743ce22350c" />

*Bukti 2. Header `X-Powered-By: PHP/8.0.13` terlihat pada respons API. Ini membuktikan pengungkapan versi di header, bukan verifikasi versi runtime aktual.*


Tentu, berikut adalah tambahan ringkasan untuk laporan Anda (Hasheemi). Saya buatkan dalam format markdown yang rapi, singkat, dan menyediakan 2 slot gambar untuk dokumentasi (satu untuk Laravel Debug/SQL, satu untuk Weak File Validation).

Silakan *copy-paste* bagian ini dan letakkan **paling bawah** setelah bagian nomor 5 (Mahrinza) di file `README.md` Anda.

---

## 6. Pengujian Tambahan - Found by Hasheemi (5027251015)

**Waktu:** 21–22 Oktober 2026  
**Target:** `thesis.its.ac.id`  
**Metode:** Analisis error (Error-based Analysis), manipulasi HTTP Request via Burp Suite, dan *source code review* (view-source).

**Hasil Utama:** Ditemukan dua kelas kerentanan utama pada aplikasi myITS Thesis. Pertama, *Information Disclosure* akibat konfigurasi `APP_DEBUG=true` yang membocorkan *stack trace*, *path* direktori server (`/var/www/html/...`), dan *raw query* SQL Server. Kedua, *Weak File Type Validation* pada *endpoint* unggah berkas yang memungkinkan *bypass* validasi menggunakan manipulasi *Content-Type* dan *Polyglot File* (PHP yang disamarkan sebagai GIF).

### Observasi Risiko Keamanan

| Observasi | Potensi Risiko | Rekomendasi |
|---|---|---|
| **Laravel Debug Mode Aktif (`APP_DEBUG=true`)** pada environment produksi. Error seperti `TypeError` dan konversi tipe SQL Server memunculkan halaman *Whoops* yang membocorkan struktur database (`sdm`, `lab_sdm`, `unit_organisasi`), *path* direktori, dan nama *namespace*. | **Medium:** Penyerang dapat memetakan skema database, memahami alur logika bisnis, dan mempersiapkan serangan lanjutan (seperti SQLi jika ada *endpoint* lain yang rentan). | Set `APP_DEBUG=false` dan `APP_ENV=production` di file `.env`. Terapkan *custom error handler* (HTTP 500) tanpa detail teknis. Validasi tipe data (UUID, Integer) di Controller/FormRequest. |
| **Weak File Validation pada `/tesis/usulan/upload`**. Backend menerima file PHP asli (dengan manipulasi `Content-Type`) dan file *Polyglot* (`GIF89a` + kode PHP) karena hanya bergantung pada *Magic Bytes* dan *input* klien. | **Medium:** Melanggar prinsip *defense in depth*. Jika konfigurasi S3 berubah atau file diproses oleh *endpoint* lain, dapat berujung pada *Stored XSS* atau *RCE*. | Terapkan validasi ketat di *backend* menggunakan aturan Laravel `mimes:pdf,png,jpg,jpeg|max:5120`. Gunakan *library* `finfo` untuk inspeksi MIME *type* aktual. Selalu ganti nama file dengan UUID. |

### Dokumentasi (Bukti Screenshot)

**1. Bukti Kebocoran Informasi (Laravel Debug & SQL Server)**
*(Menunjukkan error `TypeError`, *path* `/var/www/html/...`, dan *query* `SELECT ... FROM sdm`)*

<img width="959" height="539" alt="Screenshot 2026-10-21 183734" src="https://github.com/user-attachments/assets/d9ec0c65-e52a-4cc9-bab4-d989fb450e60" />

<img width="959" height="539" alt="Screenshot 2026-10-21 083302" src="https://github.com/user-attachments/assets/b82acbf6-683c-45f7-be7d-ba1f4a1b801f" />



**2. Bukti Bypass Validasi File (Polyglot/Native PHP)**
*(Menunjukkan respons JSON Burp Suite dengan `file_ext: "gif"` atau `application/x-php` dan `status: "OK"`)*

<img width="717" height="313" alt="Screenshot 2026-10-22 081026" src="https://github.com/user-attachments/assets/d956055b-327d-4a7e-86ad-37eb1e5965b8" />

<img width="866" height="332" alt="Screenshot 2026-10-21 171853" src="https://github.com/user-attachments/assets/7d037a71-0841-402a-b982-fa1ac297245d" />


--- 

## 7. Reflected DOM-based XSS pada Verifikasi Email — Found by Mahrinza Redouane Z. (5027251074)

**Tanggal pengujian:** 25 Oktober 2026  
**Target:** `https://portal.its.ac.id/verify/[code-id]`  
**Parameter yang terpengaruh:** `e` (nilai Base64 pada query string)  
**Jenis:** DOM-based Cross-Site Scripting (XSS), sumber data URL  
**CWE:** CWE-79 — Improper Neutralization of Input During Web Page Generation  
**Severity usulan:** Medium, perlu penilaian akhir DPTSI berdasarkan prasyarat dan konteks autentikasi.

### 7.1 Deskripsi

Halaman verifikasi email memproses parameter URL `e` yang dikodekan dalam Base64 dan memasukkan hasil decoding ke elemen `#statusVerifikasi` menggunakan `innerHTML`. Karena data dari URL tidak di-escape sebagai teks, markup dan atribut event handler dapat diinterpretasikan oleh browser.

Pemeriksaan JavaScript frontend menemukan jalur pemanggilan berikut:

1. Route `/verify/[code-id]` membaca `code-id` dan `e`, lalu memanggil fungsi ekspor `UG` (`confirmEmail`).
2. Fungsi `confirmEmail` membaca `query.e` dan melakukan decoding Base64.
3. Hasil decoding digunakan untuk membentuk HTML pada cabang respons sukses (`then`) maupun gagal (`catch`).
4. PoC browser dengan parameter dummy membuktikan bahwa HTML dapat dimasukkan ke DOM dan kode JavaScript sederhana dapat berjalan ketika callback error dipicu secara lokal akibat pemblokiran request verifikasi oleh Firefox DevTools.

**Cuplikan alur rentan (disederhanakan dari bundle frontend):**

```javascript
const decodedEmail = Buffer.from(router.query.e, 'base64').toString('ascii');

// Baik pada hasil sukses maupun gagal, kode aslinya membangun HTML serupa:
document.getElementById('statusVerifikasi').innerHTML =
  message + ' <span class="tx-medium">' + decodedEmail + '</span>.';
```

Tidak terlihat encoding HTML untuk `decodedEmail` sebelum assignment ke `innerHTML`. *Base64 adalah encoding, bukan mekanisme sanitasi.*

### 7.2 Bukti dan langkah reproduksi yang telah dilakukan

Seluruh uji dilakukan dengan **kode verifikasi dummy** (`BB-DUMMY`), tanpa email/OTP asli. Tidak ada upaya membaca cookie, menyalin token, atau mengakses data pengguna lain.

1. **HTML injection via URL.** Parameter `e` berisi encoding Base64 dari markup aman `<b>BB-URL-TEST-2026</b>`. Halaman menampilkan marker sebagai huruf tebal, dan Console menunjukkan `hasBoldElement: true` serta `renderedHTML` yang mengandung elemen `<b>`.
2. **Eksekusi JavaScript tanpa eksfiltrasi.** Untuk pengujian berikutnya, parameter `e` diisi dengan encoding Base64 dari elemen HTML uji yang memicu event lokal dan hanya menetapkan `document.documentElement.dataset.bbImgPoc = 'executed'`. Tidak ada request keluar yang dipicu oleh kode uji.
3. **Kontrol keselamatan:** Sebelum membuka URL kedua, Firefox DevTools dikonfigurasi untuk memblokir `POST /api/api/sso/verify/confirm/email`. Panel Network menampilkan **Blocked by DevTools** dan transfer `0 B` untuk request tersebut.
4. **Hasil eksekusi:** Console menunjukkan `javascriptExecuted: true`, `imageInserted: true`, dan `handlerType: function`. Ini mengonfirmasi bahwa event handler dari markup yang dikendalikan oleh URL telah berjalan dalam browser uji.

**Batasan penting:** Eksekusi JavaScript dibuktikan pada browser uji **dengan request verifikasi diblokir melalui DevTools**. Pada percobaan HTML marker sebelumnya, request dummy sempat mencapai server dan memperoleh HTTP 500; HTML tetap dirender, tetapi eksekusi JavaScript pada kondisi tanpa pemblokiran tidak diuji. Oleh sebab itu, laporan **tidak** mengklaim bypass verifikasi email, pengambilalihan akun, pencurian cookie, atau eskalasi hak akses.

### 7.3 Dampak dan prasyarat

Pengguna yang membuka URL verifikasi yang dibentuk pihak lain berpotensi mengeksekusi JavaScript di origin `portal.its.ac.id`, jika jalur DOM yang sama tercapai. Dampak yang mungkin termasuk manipulasi isi halaman atau interaksi dalam konteks sesi korban, tergantung kontrol keamanan tambahan. Dampak-dampak tersebut merupakan **risiko potensial**, bukan tindakan yang dilakukan pada pengujian.

### 7.4 Rekomendasi

- Ganti penyisipan konten URL dengan `innerHTML` menjadi `textContent`, atau bangun elemen melalui DOM API `createElement`/`append` dengan nilai teks dari `textContent`.
- Validasi dan batasi parameter `e` untuk format email yang memang diharapkan; lakukan validasi juga di sisi server. Validasi format merupakan lapisan tambahan, bukan pengganti output encoding.
- Pertimbangkan Content Security Policy (CSP) sebagai pertahanan tambahan, terutama pembatasan inline event handler; CSP tidak menggantikan perbaikan sink DOM.
- Tinjau pemanggilan otomatis endpoint verifikasi agar kode dummy/tidak valid tidak memicu respons error internal `500` yang tidak perlu.

**Contoh perbaikan penulisan ke DOM:**

```javascript
const target = document.getElementById('statusVerifikasi');
const span = document.createElement('span');
span.className = 'tx-medium';
span.textContent = decodedEmail;

target.replaceChildren(
  document.createTextNode(`${message} `),
  span,
  document.createTextNode('.')
);
```

### 7.5 Dokumentasi Screenshot

**Bukti 7.1 — Halaman verifikasi menampilkan marker HTML dari parameter URL:**

<img width="1918" height="866" alt="xss_1" src="https://github.com/user-attachments/assets/ce06b41e-f0e0-441a-93d6-1ecbac84c3df" />

**Bukti 7.2 — Console menunjukkan elemen `<b>` benar-benar terbentuk (`hasBoldElement: true`):**

<img width="849" height="301" alt="xss_2" src="https://github.com/user-attachments/assets/38c74561-84e3-4b1e-9909-9a72c7a00098" />

**Bukti 7.3 — Sumber kode `confirmEmail`: parameter `query.e` didekode dengan Base64 lalu dimasukkan melalui `innerHTML`:**

<img width="1220" height="488" alt="xss_3" src="https://github.com/user-attachments/assets/c17c6327-e457-45b3-9a4c-d7079ad45a61" />

**Bukti 7.4 — Console mengonfirmasi PoC eksekusi JavaScript (`javascriptExecuted: true`):**

<img width="1452" height="413" alt="xss_4" src="https://github.com/user-attachments/assets/326c3b46-1496-4e25-bca1-f81e403b3331" />

**Bukti 7.5 — Network menunjukkan request POST verifikasi diblokir oleh Firefox DevTools saat PoC eksekusi:**

<img width="1324" height="506" alt="xss_5" src="https://github.com/user-attachments/assets/663a20e4-4407-4cd0-8b1b-1db74fe59ce1" />

---

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
   

  

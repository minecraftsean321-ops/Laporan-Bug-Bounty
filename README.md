# Laporan-Bug-Bounty

## 1. Exposed .git Directory (Access Restricted)

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

## 2. Clickjacking — Missing X-Frame-Options / CSP frame-ancestors

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

## 3. Deprecated TLS Protocol Versions Supported (TLS 1.0 & TLS 1.1)

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



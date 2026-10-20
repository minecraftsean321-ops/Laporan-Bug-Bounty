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



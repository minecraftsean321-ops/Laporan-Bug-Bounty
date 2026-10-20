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

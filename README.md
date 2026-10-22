# Laporan-Bug-Bounty

## Tahap Reconnaissance

```
theHarvester -d its.ac.id -b otx,crtsh,hackertarget,rapiddns,subdomaincenter -l 200 
```

Hasil:

[*] Hosts found: 662
---------------------
aavi.its.ac.id
abmas.its.ac.id:202.46.129.184
absensi3.if.its.ac.id
ad01.its.ac.id:202.46.129.24
ad02.its.ac.id
ad02.its.ac.id:202.46.129.24
ad02.its.ac.id:202.46.129.25
addi.is.its.ac.id
adfs.its.ac.id
adfs.its.ac.id:202.46.129.34
adfs02.its.ac.id
adfs02.its.ac.id:202.46.129.43
adfs029.its.ac.id
adm.icts.its.ac.id
adm.icts.its.ac.id:202.46.129.105
admission.its.ac.id:103.94.189.59
admissione.its.ac.id
agusza.if.its.ac.id
aip-pk2.its.ac.id
ais.its.ac.id
akademik.its.ac.id:103.94.189.58
akademik.its.ac.id:103.94.189.59
akademik.its.ac.id:202.46.129.164
akademik3.its.ac.id
akademik3.its.ac.id:202.46.129.10
akademik3e.its.ac.id
alumni.despro.its.ac.id
alumni.its.ac.id:103.94.189.35
analitik.chem.its.ac.id
antasena.its.ac.id
apdsupport.its.ac.id
api.its.ac.id
api.its.ac.id:103.94.188.15
apps-lb-189-58.its.ac.id:103.94.189.58
aquarius.its.ac.id:202.46.129.22
arch.its.ac.id
arch.its.ac.id:103.94.189.4
arch.its.ac.id:202.46.129.115
arek.its.ac.id:202.46.129.184
ares.its.ac.id
ares.its.ac.id:202.46.129.13
ars.its.ac.id
asrama.its.ac.id:myits-apps-lb.its.ac.id
asramai.its.ac.id
audit.its.ac.id

```
whois its.ac.id
```

<img width="668" height="356" alt="image" src="https://github.com/user-attachments/assets/568e0ff9-33f1-4980-a724-a1f8190d6b6d" />

```
dig its.ac.id ANY
```

<img width="981" height="477" alt="image" src="https://github.com/user-attachments/assets/3c3665af-49d1-4cf5-a23c-676f8deadd82" />

Kita melihat status dari subdomain yang berhasil kita temukan:

```
~/go/bin/httpx -l tier1-3.txt -silent -status-code -title -tech-detect -o alive.txt
```
<img width="971" height="393" alt="image" src="https://github.com/user-attachments/assets/e095ff35-2b3b-47b4-b1ab-3c0591238819" />

Scan nuclei ke host yang alive: 

```
cat alive.txt | awk '{print $1}' | nuclei -tags cve,exposure,misconfig,tech -o nuclei-results.txt
```

<img width="1005" height="624" alt="image" src="https://github.com/user-attachments/assets/eac2113d-53e5-4bca-8067-41674c5f70d5" />

<img width="983" height="616" alt="image" src="https://github.com/user-attachments/assets/272c3ca7-afc8-4e37-b2ec-cdc9bddb5014" />



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

## 4. Sensitive Information Disclosure via Directory Listing

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
   

  

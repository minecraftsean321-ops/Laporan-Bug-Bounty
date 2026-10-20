# Laporan-Bug-Bounty

1. Ditemukannya file .git/config yang saat ini sudah di clok aksesnya tetapi bisa terbuka jika ada misconfig kecil (atau ada endpoint lain dalam .git/ yang nggak ke-cover rule block-nya).

code yang digunakan: 

```
curl https://servicedesk.its.ac.id/.git/config
```

<img width="788" height="195" alt="image" src="https://github.com/user-attachments/assets/e0879938-b747-4794-aed1-e86e4a417626" />

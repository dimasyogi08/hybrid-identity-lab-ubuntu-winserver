# Hybrid Identity Lab: Ubuntu 24.04 & Windows Server 2022 Integration

Proyek ini mendemonstrasikan cara menghubungkan sistem operasi Linux (Ubuntu) ke dalam domain Windows Server (Active Directory) untuk memungkinkan manajemen user terpusat.

## Tech Stack
- **OS Server:** Windows Server 2022 (Domain: dimsum.local)
- **OS Client:** Ubuntu 24.04 LTS
- **Tools:** SSSD, Realmd, Adcli, Kerberos v5
- **Platform:** VirtualBox (Bridged Network)

## Key Implementation Steps
1. **DNS & Network:** Mengonfigurasi Ubuntu agar menggunakan Windows Server sebagai DNS utama.
2. **Kerberos Setup:** Melakukan instalasi `krb5-user` dan konfigurasi `krb5.conf` untuk autentikasi tiket.
3. **Domain Join:** Menggunakan `adcli` untuk melakukan join domain secara manual guna menghindari konflik mDNS pada domain `.local`.
4. **Home Directory:** Mengaktifkan `pam_mkhomedir` agar folder user otomatis terbuat saat login pertama kali.

## Troubleshooting (Problem Solving)
- **GSSAPI Error:** Menyelesaikan masalah autentikasi Kerberos dengan sinkronisasi waktu (NTP) antara Server dan Client.
- **mDNS Conflict:** Memperbaiki resolusi nama domain `.local` yang sering bermasalah di sistem Linux.

## Result
User dari Active Directory (contoh: `budi01@dimsum.local`) berhasil login ke desktop Ubuntu dengan identitas terpusat.

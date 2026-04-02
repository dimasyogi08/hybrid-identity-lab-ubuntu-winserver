# Enterprise Hybrid Identity Lab: Ubuntu 24.04 & Windows Server 2022 Integration

## Project Summary
Merancang dan mengimplementasikan arsitektur Manajemen Identitas Terpusat (Centralized Identity Management) dengan mengintegrasikan sistem operasi klien Linux (Ubuntu 24.04 LTS) ke dalam lingkungan Windows Server 2022 Active Directory. Proyek ini bertujuan untuk menstandarisasi manajemen akses, meningkatkan postur keamanan, dan mencapai kapabilitas Single Sign-On (SSO) lintas platform, yang merupakan fondasi krusial bagi infrastruktur Hybrid Cloud (Microsoft Entra ID/Azure AD).

## Architecture & Lab Environment

### 1. Homelab Infrastructure
* **Hypervisor:** Oracle VirtualBox (Bridged Networking)
* **Host Hardware:** Ryzen 7, 32GB RAM, NVMe Storage (Acer Nitro V16)

### 2. Network Topology
| Role | OS | Hostname / FQDN | IP Address (Static) |
| :--- | :--- | :--- | :--- |
| **Domain Controller & DNS** | Windows Server 2022 | `win2022-lab.dimsum.local` | `192.168.18.100` |
| **Linux Client Node** | Ubuntu 24.04 LTS | `client-budi.dimsum.local` | `DHCP / Static` |

### 3. Core Technologies & Protocols
* **Active Directory Domain Services (AD DS):** Sebagai *Identity Provider (IdP)*.
* **Kerberos (v5):** Protokol autentikasi jaringan untuk manajemen tiket (TGT).
* **SSSD (System Security Services Daemon):** Penghubung (*bridge*) antara sistem Linux lokal dengan direktori eksternal.
* **PAM (Pluggable Authentication Modules):** Modul untuk manajemen *session* dan *home directory*.

## Step-by-Step Implementation

### Phase 1: Windows Server Preparation (The Identity Provider)
1. Melakukan *provisioning* Active Directory dengan nama domain `dimsum.local`.
2. Mengonfigurasi **DNS Server** untuk menangani *Forward Lookup* dan **Reverse Lookup Zone** (krusial untuk validasi Kerberos).
3. Membuat *Organizational Unit (OU)* khusus untuk komputer dan *User Object* (`budi01@dimsum.local`).
4. Menyesuaikan *Inbound Rules* pada Windows Firewall untuk mengizinkan lalu lintas DNS (Port 53), Kerberos (Port 88), dan LDAP (Port 389).

### Phase 2: Linux Client Provisioning
Memodifikasi *resolver* DNS di Ubuntu agar merujuk sepenuhnya ke IP Windows Server (`192.168.18.100`), dilanjutkan dengan instalasi komponen integrasi identitas:

`sudo apt update`

`sudo apt install realmd adcli sssd sssd-tools samba-common krb5-user packagekit`

### Phase 3: Kerberos Configuration & Time Synchronization
Kerberos sangat sensitif terhadap asinkronisasi waktu dan ketidakcocokan identitas (FQDN).

Melakukan sinkronisasi NTP (Network Time Protocol) antara Klien dan Server (Toleransi deviasi maksimal 5 menit).

Mendefinisikan pemetaan Realm secara eksplisit pada /etc/krb5.conf:
[libdefaults]
    default_realm = DIMSUM.LOCAL

[realms]
    DIMSUM.LOCAL = {
        kdc = 192.168.18.100
        admin_server = 192.168.18.100
    }

[domain_realm]
    .dimsum.local = DIMSUM.LOCAL
    dimsum.local = DIMSUM.LOCAL

### Phase 4: Active Directory Domain Join
Mengeksekusi proses penggabungan domain menggunakan backdoor command adcli untuk bypass masalah resolusi GUI/realm standar, serta memastikan Computer Object terdaftar di database AD:
sudo adcli join --verbose --domain dimsum.local --domain-controller 192.168.18.100 --login-user Administrator

### Phase 5: Post-Join Configuration (SSSD & PAM)
Membangun ulang konfigurasi `/etc/sssd/sssd.conf` dengan cara `sudo cat /etc/sssd/sssd.conf` agar daemon membaca database Active Directory:

`[sssd]`

`domains = dimsum.local`

`config_file_version = 2`

`services = nss, pam`


`[domain/dimsum.local]`

`ad_domain = dimsum.local`

`krb5_realm = DIMSUM.LOCAL`

`realmd_tags = manages-system joined-with-adcli`

`cache_credentials = True`

`id_provider = ad`

`krb5_store_password_if_offline = True`

`default_shell = /bin/bash`

`ldap_id_mapping = True`

`use_fully_qualified_names = True`

`fallback_homedir = /home/%u@%d`

`access_provider = ad`


Menyetel izin file secara ketat (chmod 600) dan me-restart layanan SSSD. Terakhir, mengaktifkan pam_mkhomedir melalui sudo pam-auth-update agar direktori /home/budi01@dimsum.local otomatis terbuat saat otentikasi pertama.

## Deep Dive: Troubleshooting & Incident Resolution
Bagian ini membedah problem-solving dari masalah kritis yang terjadi selama fase integrasi:

### 1. Insiden: Resolusi Nama Domain .local (mDNS Conflict)
- Gejala: Ubuntu gagal meresolusi dimsum.local, menghasilkan error realm: No such realm found.
- Root Cause: Secara default, sistem Linux menggunakan protokol systemd-resolved dan Avahi/Bonjour yang mengasumsikan bahwa domain berakhiran .local adalah jaringan Multicast lokal, bukan DNS eksternal.
- Resolusi: Melakukan intervensi pada file Name Service Switch (/etc/nsswitch.conf). Saya memprioritaskan resolusi dns standar di atas mdns4_minimal agar Ubuntu langsung melempar permintaan query ke Windows Server.

### 2. Insiden: Kegagalan Autentikasi Tiket (GSSAPI Error)
- Gejala: Error Message: SASL(-1): generic failure: GSSAPI Error: Server not found in Kerberos database.
- Root Cause: Identitas Service Principal Name (SPN) mengalami mismatch. Windows Server mengidentifikasi dirinya sebagai win2022-lab.dimsum.local, sementara query dari Ubuntu tidak memiliki kepastian jalur identitas tersebut.
- Root Cause: Identitas Service Principal Name (SPN) mengalami mismatch. Windows Server mengidentifikasi dirinya sebagai win2022-lab.dimsum.local, sementara query dari Ubuntu tidak memiliki kepastian jalur identitas tersebut.
- Resolusi Berlapis:
  a. Membuat Reverse DNS (PTR Record) di Windows Server untuk memvalidasi identitas IP 192.168.18.100.
  b. Melakukan modifikasi Host Binding manual pada /etc/hosts di sisi Ubuntu (192.168.18.100 win2022-lab.dimsum.local win2022-lab).
  c.Menyempurnakan pemetaan realm di /etc/krb5.conf menggunakan blok [domain_realm].

## Validation & Security Posture
- Domain Status: Eksekusi realm list menampilkan status configured: kerberos-member.
- Identity Resolution: Perintah id budi01@dimsum.local di Ubuntu berhasil memetakan UID dan GID tinggi yang di-generate langsung oleh sistem Active Directory.
- SSO Login: Autentikasi berhasil dilakukan pada Desktop Login Screen Ubuntu menggunakan kredensial Windows tanpa perlu membuat Local User terlebih dahulu.

## Alignment with Cloud Architecture
Arsitektur lokal ini merupakan representasi fundamental dari apa yang terjadi di lingkungan Cloud computing. Pengetahuan mengenai DNS terpusat, Kerberos, dan sinkronisasi identitas ini adalah fondasi krusial untuk mengelola Microsoft Entra ID (Azure AD), Microsoft Entra Domain Services, serta implementasi Hybrid Cloud Connect di masa mendatang.




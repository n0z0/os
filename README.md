# Technical Brief: Portable Redox OS Deployment via GitHub CI/CD

## 1. Latar Belakang & Objektif
**Tujuan:** Menerapkan OS Redox secara portabel di PC Lab Kampus (UEFI) melalui *Frugal Install* yang dipicu oleh PowerShell *One-Liner*.
**Infrastruktur:** 100% *Serverless* menggunakan ekosistem GitHub (Repo, Actions, Pages, dan Releases).
**Kendala Utama:** PC Lab memiliki fitur Secure Boot aktif. Limit ukuran file GitHub (100MB).
**Solusi:** 
- *Chain of Trust* (`shimx64.efi` + MOK Manager) untuk bypass Secure Boot.
- File ringan (*script*, EFI) di-host di **GitHub Pages**.
- File berat (`redox.iso`) di-host di **GitHub Releases** via **GitHub Actions**.

---

## 2. Arsitektur Sistem (Infrastruktur GitHub)

### A. Repositori GitHub (`github.com/Organisasi/Redox-Lab`)
Berisi kode sumber (*source of truth*):
- `scripts/masuk-redox.ps1` (Script utama)
- `efi/shimx64.efi`, `efi/grubx64.efi`, `efi/mmx64.efi` (File bootloader)
- `efi/grub.cfg` (Konfigurasi GRUB)
- `.github/workflows/deploy.yml` (Script otomasi CI/CD)

### B. CI/CD Pipeline (GitHub Actions)
Setiap kali ada pembaruan di *branch* `main` (atau dijadwalkan setiap bulan), GitHub Actions akan:
1. Menyalin isi folder `scripts` dan `efi` ke *branch* `gh-pages`.
2. Mengunduh rilis `redox.iso` terbaru dari GitLab resmi Redox OS.
3. Membuat **GitHub Release** baru di repositori Anda dan melampirkan `redox.iso` ke rilis tersebut.

### C. Alur Eksekusi di PC Lab (End-User)
1. Mahasiswa mengetik *One-Liner* yang mengarah ke URL GitHub Pages Anda.
2. PowerShell mengunduh *script* dari Pages.
3. *Script* mengunduh file EFI dari Pages, dan `redox.iso` dari URL *Latest Release* GitHub repositori Anda.
4. *Script* memodifikasi BCD Windows dan melakukan *Reboot*.

---

## 3. Pembagian Tugas (Workstreams)

### A. Tim DevOps / Infrastruktur (GitHub)
**Tugas:** Menyiapkan repositori dan *pipeline* otomatis.
**Langkah Teknis:**
1. Kumpulkan file EFI dari ISO Ubuntu (`shimx64.efi`, `grubx64.efi`, `mmx64.efi`).
2. Buat file `.github/workflows/deploy.yml`. 
   *Gambaran Workflow:*
   ```yaml
   name: Build and Deploy
   on:
     push:
       branches: [ "main" ]
   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3
         - name: Download latest Redox ISO
           run: wget -O redox.iso https://gitlab.redox-os.org/redox-os/redox/-/jobs/artifacts/master/raw/build/x86_64/livedisk.iso?job=livedisk
         - name: Create GitHub Release for ISO
           uses: softprops/action-gh-release@v1
           with:
             files: redox.iso
             tag_name: "latest-build"
         - name: Deploy Script & EFI to GitHub Pages
           uses: peaceiris/actions-gh-pages@v3
           with:
             github_token: ${{ secrets.GITHUB_TOKEN }}
             publish_dir: ./files_to_publish
   ```

### B. Tim Developer Script (PowerShell)
**Tugas:** Menulis *script* `masuk-redox.ps1` dengan Endpoint GitHub.
**Langkah Logika Script:**
1. Cek hak akses Administrator.
2. Buat direktori tersembunyi `C:\RedoxBoot`.
3. Unduh file statis menggunakan URL statis (Contoh: `https://[org].github.io/Redox-Lab/grubx64.efi`).
4. Unduh `redox.iso` dari URL *Release* (Contoh: `https://github.com/[org]/Redox-Lab/releases/download/latest-build/redox.iso`).
5. Set *One-Time Boot* menggunakan `bcdedit` mengarah ke `C:\RedoxBoot\shimx64.efi`.
6. `Restart-Computer -Force`.

### C. Tim Dokumentasi / UX Mahasiswa
**Tugas:** Menyiapkan instruksi praktikum untuk mahasiswa.
**Instruksi Mahasiswa:**
1. Buka PowerShell (Run as Administrator).
2. Salin dan jalankan *One-Liner* berikut:
   `iex ((New-Object System.Net.WebClient).DownloadString('https://[org].github.io/Redox-Lab/masuk-redox.ps1'))`
3. Ikuti instruksi pendaftaran *MOK Manager* (layar biru) saat PC pertama kali *restart*.
4. Masuk ke Redox OS.

---

## 4. Konfigurasi GRUB (`grub.cfg`)
Tim infrastruktur harus menyertakan `grub.cfg` ini ke dalam repositori:
```text
set timeout=0
menuentry "Redox OS Lab Session" {
    insmod part_gpt
    insmod loopback
    # Pastikan (hd0,gptX) disesuaikan dengan skema partisi lab (umumnya C: ada di gpt3)
    loopback loop (hd0,gpt3)/RedoxBoot/redox.iso
    kernel (loop)/kernel.bin
    module (loop)/initfs.img
    boot
}
```

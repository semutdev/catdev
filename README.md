# catdev

**Docker, PHP, Node, MySQL, PostgreSQL, dan Claude Code di MacBook Intel lama yang mentok di macOS Catalina.**

catdev adalah satu script bash yang menyalakan Linux VM ringan lewat QEMU. Tanpa ganti OS, tanpa patch macOS, dan tanpa Docker Desktop.

> *English: a tiny bash script that runs a lightweight Debian VM via QEMU + Hypervisor.framework, so old Intel Macs stuck on macOS Catalina can still run Docker and modern dev tools.*

🌐 Landing page: [catdev.semut.dev](https://catdev.semut.dev)

---

## Kenapa catdev?

Banyak MacBook Intel lama berhenti di macOS Catalina. Hardware-nya masih layak untuk ngoding, apalagi kalau RAM sudah di-upgrade, tapi tool developer satu per satu berhenti mendukungnya:

- **Docker Desktop**: versi yang masih jalan di Catalina sudah lama ditinggal.
- **Homebrew**: Catalina tidak lagi didukung, banyak paket harus dikompilasi sendiri.
- **Editor dan aplikasi Electron baru**: sebagian besar sekarang butuh minimal macOS 11.
- **Multipass**: jalan, tapi berat. Di MacBook Pro 2012 bisa memakan sekitar 7 GB RAM dan bikin laptop lag.

catdev mengambil jalan paling sederhana: satu VM Debian minimal, dengan pengaturan yang sepenuhnya bisa kamu kontrol.

## Hasilnya

| | RAM yang dipakai di Mac |
|---|---|
| Multipass | ± 7 GB |
| **catdev** | **± 1,1 GB** |

Diukur di MacBook Pro 2012 dengan RAM 16 GB dan macOS Catalina, setelah VM selesai boot. Saat dipakai kerja, pemakaian catdev bisa naik sampai batas `VM_MEM` (default 3 GB), tapi tidak lebih.

## Cara kerjanya

Docker butuh kernel Linux, jadi catdev menyalakan Debian 13 minimal di dalam QEMU. Prosesor Intel menjalankan VM-nya langsung lewat Hypervisor.framework bawaan macOS (akselerasi HVF), sehingga kecepatannya mendekati native.

```
MacBook (macOS Catalina)
│  Terminal, VS Code Remote-SSH, browser
│  localhost:2222 → SSH ke VM
│  localhost:8000, :5173, :3000, :8080 → aplikasimu
│
└── QEMU + HVF  (RAM & CPU dibatasi lewat VM_MEM / VM_CPUS)
    │
    └── Debian 13 VM
        Docker · MySQL · PostgreSQL · PHP & Node (via container)
        Claude Code · tmux · zram
```

Setup awal VM (user, SSH key, Docker, zram) dikerjakan otomatis oleh cloud-init saat boot pertama.

## Persyaratan

Cukup disiapkan sekali di awal:

- **Mac Intel yang mendukung virtualisasi.** Cek dengan `sysctl kern.hv_support`, hasilnya harus `1`.
- **Xcode Command Line Tools**, dibutuhkan MacPorts untuk membangun paket.
- **MacPorts**, lebih bersahabat dengan macOS lama dibanding Homebrew.
- **QEMU** dari MacPorts.
- **SSH key** di `~/.ssh/id_ed25519.pub`.
- Ruang disk kosong sekitar 10 GB untuk mulai.

## Instalasi

### 1. Xcode Command Line Tools

```bash
xcode-select --install
```

Kalau muncul pesan tidak tersedia, unduh **Command Line Tools for Xcode 12.4** dari [developer.apple.com/download/all](https://developer.apple.com/download/all/) (login dengan Apple ID gratis), lalu pasang manual.

### 2. MacPorts

Unduh installer untuk macOS Catalina dari [macports.org/install.php](https://www.macports.org/install.php), lalu jalankan seperti aplikasi biasa.

### 3. QEMU

```bash
sudo port selfupdate
sudo port -b install qemu || sudo port install qemu
qemu-system-x86_64 -accel help
```

Pastikan `hvf` muncul di daftar akselerator. Kalau QEMU harus dikompilasi dari source, prosesnya bisa satu jam atau lebih. Colokkan charger dan biarkan jalan.

### 4. catdev

```bash
mkdir -p ~/bin
curl -fsSL https://raw.githubusercontent.com/semutdev/catdev/main/catdev -o ~/bin/catdev
chmod +x ~/bin/catdev
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

### 5. Buat dan nyalakan VM

```bash
[ -f ~/.ssh/id_ed25519.pub ] || ssh-keygen -t ed25519
catdev init
catdev up
```

`init` mengunduh image Debian (sekitar 300–400 MB) dan menyiapkan disk. Boot pertama dengan `up` makan waktu beberapa menit karena Docker sedang dipasang di dalam VM. Tunggu sampai muncul `✓ VM siap`.

Tes:

```bash
catdev ssh docker run --rm hello-world
catdev status
```

## Pemakaian

| Perintah | Fungsi |
|---|---|
| `catdev init` | Unduh image dan siapkan VM (sekali saja) |
| `catdev up` | Nyalakan VM |
| `catdev ssh` | Masuk ke VM, atau jalankan satu perintah: `catdev ssh docker ps` |
| `catdev status` | Lihat RAM yang dipakai VM dan container yang sedang jalan |
| `catdev down` | Matikan VM dengan rapi |

### Pengaturan

Semua pengaturan lewat environment variable. Taruh di `~/.zshrc` supaya permanen. Perubahan berlaku setelah `catdev down` lalu `catdev up`.

| Variabel | Default | Keterangan |
|---|---|---|
| `VM_MEM` | `3072` | RAM VM dalam MB |
| `VM_CPUS` | `2` | Jumlah CPU VM |
| `FWD_PORTS` | `3000 5173 8000 8080` | Port yang diteruskan dari `localhost` Mac ke VM |
| `SSH_PORT` | `2222` | Port SSH di Mac |
| `VM_DISK_SIZE` | `40G` | Ukuran maksimal disk VM (hanya saat `init`) |
| `SSH_PUBKEY` | `~/.ssh/id_ed25519.pub` | Public key yang dipasang di VM (hanya saat `init`) |
| `CATDEV_HOME` | `~/.catdev` | Lokasi disk VM, image, dan log |

Contoh untuk Claude Code + Docker + database sekaligus:

```bash
export VM_MEM=4096
export FWD_PORTS="3000 5173 8000 8080 3306 5432"
```

## Alur kerja yang disarankan

### Simpan kode di dalam VM

Jalankan `git clone` di dalam VM, **jangan mount folder dari Mac**. Mount folder lewat jaringan adalah penyebab utama lag: Docker, npm, composer, dan Claude Code membaca ribuan file kecil, dan semuanya jadi lambat kalau harus lewat jaringan.

```bash
catdev ssh
mkdir -p ~/projects && cd ~/projects
git clone git@github.com:kamu/project.git
```

Agent forwarding sudah aktif, jadi `git push` dari VM memakai SSH key yang ada di Mac. Pastikan key-nya sudah dimuat: `ssh-add -K ~/.ssh/id_ed25519`.

### Edit dengan VS Code Remote-SSH

Tambahkan ke `~/.ssh/config` di Mac:

```
Host catdev
  HostName 127.0.0.1
  Port 2222
  User dev
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  ForwardAgent yes
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```

Lalu di VS Code: **Remote-SSH: Connect to Host → catdev**. Editornya di Mac, semua prosesnya jalan di VM. Pakai versi VS Code yang masih mendukung Catalina, dan matikan update otomatis.

### Pakai tmux untuk sesi panjang

```bash
catdev ssh sudo apt install -y tmux
catdev ssh tmux new -A -s main
```

Sesi tmux tetap hidup walaupun terminal ditutup atau koneksi putus. Cocok untuk Claude Code dan dev server.

### Akses aplikasi dari browser Mac

Pastikan dev server listen di `0.0.0.0`, lalu buka `http://localhost:<port>` di Mac:

```bash
php artisan serve --host=0.0.0.0
npm run dev -- --host
```

### Matikan VM sebelum mematikan Mac

Mematikan Mac saat VM menyala sama seperti mencabut listrik VM. Biasakan `catdev down` dulu. Supaya database otomatis menyala lagi setelah `catdev up`, pakai `restart: unless-stopped` di `docker-compose.yml`.

## Troubleshooting

**`Error: Neither Xcode nor the Command Line Tools were found` saat `port install`.**
Command Line Tools rusak atau tidak lengkap. Hapus dan pasang ulang:

```bash
sudo rm -rf /Library/Developer/CommandLineTools
xcode-select --install
```

**`qemu-system-x86_64` lama sekali saat pertama dijalankan.**
macOS memeriksa binary baru saat pertama kali dijalankan. Tunggu sebentar. Selanjutnya akan normal.

**`catdev up` berhenti di "Menunggu SSH".**
Cek log console VM di `~/.catdev/serial.log`.

**`Too many authentication failures` saat `catdev ssh`.**
SSH agent di Mac menyodorkan terlalu banyak key. Tambahkan `IdentityFile ~/.ssh/id_ed25519` dan `IdentitiesOnly yes` di blok `Host catdev` pada `~/.ssh/config`.

**Port baru tidak bisa diakses.**
Port diteruskan saat `catdev up`. Tambahkan ke `FWD_PORTS`, lalu `catdev down` dan `catdev up`.

**Mulai ulang dari nol.**
Hapus folder `~/.catdev`, lalu jalankan `catdev init` lagi. Semua isi VM, termasuk kode yang belum di-push, ikut terhapus.

## Batasan

- Hanya untuk Mac Intel dengan dukungan VT-x.
- RAM yang sudah dipakai VM baru kembali ke Mac setelah `catdev down`. Atur batasnya dengan `VM_MEM`.
- Kode tinggal di file disk VM (`~/.catdev/disk.qcow2`). Biasakan `git push`, dan sesekali cadangkan file disk itu saat VM mati.
- Baru diuji di MacBook Pro 2012 dengan macOS Catalina dan QEMU 11.1.2 dari MacPorts.

## Coba di Mac kamu? Kabari hasilnya

catdev baru diuji di satu jenis mesin. Kalau kamu mencobanya di Mac lain, buka [Issue](https://github.com/semutdev/catdev/issues) dan sertakan:

- Model Mac dan jumlah RAM
- Versi macOS (`sw_vers`)
- Versi QEMU (`qemu-system-x86_64 --version`)
- Hasil `catdev status` setelah VM menyala

Pull request untuk perbaikan dan fitur baru juga sangat diterima.

## Kredit

catdev hanya menyatukan beberapa proyek open source yang luar biasa:

- [QEMU](https://www.qemu.org/), emulator dan virtualizer
- [Debian cloud images](https://cloud.debian.org/images/cloud/)
- [cloud-init](https://cloud-init.io/)
- [Docker](https://www.docker.com/)

## Lisensi

[MIT](LICENSE). Bebas dipakai, diubah, dan dibagikan.

---

Dibuat oleh [@semutdev](https://www.threads.com/@semutdev) untuk teman-teman yang masih setia dengan MacBook lama.
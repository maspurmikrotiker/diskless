# Panduan Lengkap Setup Diskless Server (iPXE + iSCSI + Nginx + Ubuntu 24.04) untuk WARNET hingga LAB Komputer 

## Oleh: Purwanto (Network Engineer PT InfraSolusi)
## Contact: 

Panduan dari A-Z ini dirancang khusus untuk spesifikasi hardware Anda:
*   **Server:** Dell PowerEdge R730, RAM 128GB, RAID 10 (4x SSD Samsung EVO 1TB)
*   **Network:** Intel X710-DA2 10GbE SFP+
*   **Switch:** MikroTik CRS326-24S+
*   **Client:** Windows 10/11 (Diskless via iSCSI)
*   **OS Server:** Ubuntu Server 24.04 LTS

---

## Tahap 1: Persiapan Jaringan (Network & Switch Preparation)

Karena kita menggunakan koneksi 10GbE berkecepatan tinggi, kita wajib mengaktifkan **Jumbo Frames (MTU 9000)** untuk mengurangi beban CPU dan mendongkrak *throughput* saat *boot storm*.

1. **Konfigurasi MikroTik CRS326:**
   * Pastikan port yang mengarah ke server dan klien diset memiliki `mtu=9000` (atau `9216` tergantung tipe switch).
   * Buat VLAN khusus diskless (opsional, tapi disarankan agar trafik bersih dari broadcast storm).

2. **Konfigurasi Netplan di Ubuntu Server 24.04 (`/etc/netplan/01-netcfg.yaml`):**
   Identifikasi nama interface Intel X710 Anda (misal: `enp3s0f0`), lalu sesuaikan konfigurasinya:
   ```yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       enp3s0f0:
         dhcp4: no
         addresses:
           - 192.168.10.1/24
         mtu: 9000
         optional: true
   ```
   Terapkan konfigurasi:
   ```bash
   sudo netplan apply
   ```

---

## Tahap 2: Optimasi Kernel Linux & Memori (Memanfaatkan RAM 128GB)

Agar server tidak kewalahan saat puluhan klien melakukan *booting* secara bersamaan (mengisi *Page Cache* RAM 128GB), kita perlu menyetel parameter kernel.

1. Buat file konfigurasi sysctl baru:
   ```bash
   sudo nano /etc/sysctl.d/99-diskless.conf
   ```
2. Masukkan parameter *best practices* berikut:
   ```text
   # Mengoptimalkan buffer jaringan 10G
   net.core.rmem_max = 16777216
   net.core.wmem_max = 16777216
   net.core.rmem_default = 1048576
   net.core.wmem_default = 1048576
   net.core.netdev_max_backlog = 10000
   net.ipv4.tcp_rmem = 4096 87380 16777216
   net.ipv4.tcp_wmem = 4096 65536 16777216

   # Optimasi penggunaan RAM (Page Cache agresif untuk SSD RAID 10)
   vm.swappiness = 10
   vm.dirty_background_ratio = 5
   vm.dirty_ratio = 10
   ```
3. Simpan dan terapkan:
   ```bash
   sudo sysctl --system
   ```

---

## Tahap 3: Instalasi Paket Core Stack

Install seluruh dependensi yang dibutuhkan langsung dari repositori Ubuntu 24.04:

```bash
sudo apt update
sudo apt install -y dnsmasq nginx targetcli-fb syslinux ipxe genisoimage
```

---

## Tahap 4: Konfigurasi Nginx (HTTP Boot Server)

Penggunaan HTTP jauh lebih ngebut dibanding TFTP murni untuk mentransfer *boot image* berukuran besar.

1. Buat direktori untuk menampung file boot iPXE:
   ```bash
   sudo mkdir -p /var/www/html/boot
   ```
2. Salin file *image* iPXE universal (`undionly.kpxe` untuk BIOS dan `ipxe.efi` untuk UEFI) ke direktori tersebut:
   ```bash
   sudo cp /usr/lib/ipxe/undionly.kpxe /var/www/html/boot/
   sudo cp /usr/lib/ipxe/ipxe.efi /var/www/html/boot/
   ```
3. Buat file *script* iPXE agar klien tahu langkah selanjutnya setelah mengunduh bootloader:
   ```bash
   sudo nano /var/www/html/boot/boot.ipxe
   ```
   Isi dengan script berikut:
   ```text
   #!ipxe
   echo === MENJALANKAN DISKLESS CLIENT (10G NETWORK) ===
   dhcp
   chain http://192.168.10.1/boot/menu.ipxe
   ```
4. Buat file menu interaktif iPXE (`/var/www/html/boot/menu.ipxe`):
   ```text
   #!ipxe
   set server-ip 192.168.10.1

   item --default windows Boot Windows 10/11 iSCSI
   item local Boot from Local Drive
   choose target && goto ${target}

   :windows
   echo Menghubungkan ke Storage iSCSI Windows...
   sanboot iscsi:${server-ip}::::iqn.2024-04.com.diskless:windows.client01
   goto end

   :local
   exit

   :end
   ```

---

## Tahap 5: Konfigurasi Dnsmasq (ProxyDHCP & DNS)

Kita menggunakan `dnsmasq` sebagai **ProxyDHCP** agar tidak bertabrakan dengan router utama (MikroTik utama), sekaligus melayani permintaan *network boot*.

1. Backup konfigurasi asli:
   ```bash
   sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
   sudo nano /etc/dnsmasq.conf
   ```
2. Masukkan konfigurasi berikut:
   ```text
   # Interface jaringan server
   interface=enp3s0f0
   bind-interfaces

   # Nonaktifkan fungsi DHCP utama, gunakan ProxyDHCP agar aman bagi jaringan existing
   dhcp-range=192.168.10.1,proxy

   # Deteksi arsitektur klien (BIOS vs UEFI) dan arahkan ke iPXE HTTP
   dhcp-match=set:bios,option:client-arch,0
   dhcp-match=set:efi32,option:client-arch,6
   dhcp-match=set:efi64,option:client-arch,7
   dhcp-match=set:efibc,option:client-arch,9

   # Arahkan bootfile berdasarkan arsitektur ke Nginx HTTP Server
   dhcp-boot=tag:bios,http://192.168.10.1/boot/undionly.kpxe
   dhcp-boot=tag:efi64,http://192.168.10.1/boot/ipxe.efi
   dhcp-boot=tag:efibc,http://192.168.10.1/boot/ipxe.efi
   ```
3. Restart layanan dnsmasq dan nginx:
   ```bash
   sudo systemctl restart dnsmasq
   sudo systemctl enable dnsmasq
   sudo systemctl restart nginx
   sudo systemctl enable nginx
   ```

---

## Tahap 6: Konfigurasi iSCSI Target (`targetcli`)

Kita akan membuat virtual disk di atas RAID 10 SSD Samsung EVO untuk dijadikan *root drive* klien Windows.

1. Masuk ke interactive shell `targetcli`:
   ```bash
   sudo targetcli
   ```
2. Jalankan perintah konfigurasi berikut secara berurutan di dalam *prompt* `targetcli`:
   * **Membuat Backstore File** (Gunakan kapasitas sesuai kebutuhan, misal 60GB untuk Windows 10/11):
     ```text
     /> backstores/fileio create name=win_client_disk file_size=60G path=/var/lib/libvirt/images/win_client.img
     ```
   * **Membuat iSCSI Target IQN**:
     ```text
     /> iscsi create iqn.2024-04.com.diskless:windows.client01
     ```
   * **Membuat LUN (Logical Unit Number) dan mapping ke target**:
     ```text
     /> cd iscsi/iqn.2024-04.com.diskless:windows.client01/tpg1/luns
     /iscsi/iqn.20...ient01/tpg1/luns> create /backstores/fileio/win_client_disk
     ```
   * **Mengatur Authentication (Bebaskan tanpa password/ACL untuk kemudahan awal atau set open access)**:
     ```text
     /iscsi/iqn.20...ient01/tpg1> set attribute authentication=0 demo_mode_write_protect=0 generate_node_acls=1
     ```
   * **Simpan konfigurasi agar permanen setelah reboot**:
     ```text
     /> saveconfig
     /> exit
     ```

---

## Tahap 7: Menyiapkan Master Image Windows 10/11

Bagian ini adalah cara memasukkan sistem operasi Windows ke dalam file `win_client.img` yang ada di server.

1. **Cara Termudah (Metode Burn via WinPE / Image Builder):**
   * Buat satu PC fisik uji coba. Install Windows 10/11 secara normal pada PC tersebut.
   * Install **Intel X710 NIC Driver** dan pastikan *driver iSCSI initiator* berfungsi dengan baik.
   * Gunakan tool *Image Preparation* seperti **Microsoft Sysprep** agar *image* bersih dari *hardware ID* unik:
     ```cmd
     C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
     ```
   * Ambil *disk* tersebut (atau cloning disk menggunakan Clonezilla/Acronis) dan masukkan isinya langsung ke dalam file image target di server: `/var/lib/libvirt/images/win_client.img`.

---

## Tahap 8: Pengujian Klien Pertama (Client Booting)

1. Nyalakan komputer klien (pastikan menggunakan LAN card yang mendukung PXE/UEFI Network Boot, disarankan menggunakan Intel NIC fisik agar stabil).
2. Atur urutan *boot* utama di BIOS klien ke **Network Boot (PXE)**.
3. Klien akan meminta IP ke server, mengunduh iPXE via HTTP dari Nginx, memuat menu, dan langsung melakukan *streaming* pembacaan OS Windows melalui iSCSI target dengan kecepatan tinggi memanfaatkan jaringan 10GbE.

---

## Tahap 9: Best Practices & Tips Perawatan Harian

* **Write Filter (Penting untuk Windows Diskless):** 
  Karena Windows menulis *log* dan *temp file* secara masif, aktifkan fitur **Unified Write Filter (UWF)** di Windows 10/11 Enterprise/Education Anda. Hal ini membuat semua perubahan file sistem dibuang ke RAM klien, sehingga *image* utama di server tidak cepat rusak (*corrupt*) dan performa SSD RAID 10 tetap terjaga abadi.
* **Monitoring I/O:** Gunakan perintah `htop` atau `iftop` di Ubuntu Server untuk memantau trafik jaringan 10G dan konsumsi RAM saat jam sibuk.

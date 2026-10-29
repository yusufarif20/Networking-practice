# Dokumentasi Jaringan PT Maju Jaya: Campus Expansion

Disusun oleh: \[yusufarif20\] | Tanggal: 29 Oktober 2026 | Alat: Cisco Packet Tracer (Physical + Logical workspace)

## 1. Ringkasan

PT Maju Jaya memiliki tiga gedung: Building A (HQ), Building B (Warehouse, 60 m dari HQ), dan Building C (Sales Annex, 180 m dari HQ). Jaringan flat 192.168.1.0/24 diganti dengan jaringan tersegmentasi VLAN yang aman. Poin desain utama:

- Empat VLAN (Management, Staff, Sales, Servers) dengan inter-VLAN routing memakai router-on-a-stick pada R-A.
- Building B dihubungkan dengan tembaga (60 m, di bawah batas 100 m).
- Building C dihubungkan dengan fiber multimode karena 180 m melebihi batas tembaga.
- DHCP dari router untuk VLAN 20 dan 30, VLAN 10 dan 40 statis.
- Keamanan: ACL, port security, native VLAN 99, port tidak terpakai di-shutdown ke VLAN 999, SSH only, password terenkripsi.

## 2. Topologi Fisik (Physical View)

Semua perangkat jaringan berada di rack wiring closet. Closet dibuat dari City view dan diletakkan dekat gedung masing-masing. Server Room, Office Floor, Warehouse Floor, dan Annex Floor adalah label (Place Note) pada denah gedung.

```
Intercity
└── Home City
    ├── Building A (HQ)
    │   ├── Server Room - Building A
    │   │   └── Server-A      VLAN 40  10.10.40.2 /28
    │   └── Office Floor - Building A
    │       ├── PC-A1, PC-A2, PC-A3   VLAN 20  DHCP
    │       ├── PC-A4                 VLAN 10  10.10.10.10 /24 (host admin)
    │       └── Printer-A             VLAN 20  DHCP
    ├── Wiring Closet WC-A
    │   ├── R-A (Router 2911)
    │   └── SW-CORE (Switch-PT-Empty + modul)
    ├── Building B (Warehouse)   60 m dari A (copper)
    │   └── Warehouse Floor - Building B
    │       └── PC-B1, PC-B2, PC-B3   VLAN 20  DHCP
    ├── Wiring Closet WC-B
    │   └── SW-B (Switch 2960-24TT)
    ├── Building C (Sales Annex)   180 m dari A (fiber)
    │   └── Annex Floor - Building C
    │       ├── PC-C1, PC-C2, PC-C3   VLAN 30  DHCP
    │       ├── AP-C                  VLAN 30  SSID SALES (WPA2-PSK)
    │       └── Laptop-C1, Laptop-C2  VLAN 30  DHCP (wireless)
    └── Wiring Closet WC-C
        └── SW-C (Switch-PT-Empty + modul)
```

| Level navigasi | Isi tampilan |
| --- | --- |
| Intercity | Antar kota (hanya Home City yang dipakai) |
| City | Posisi gedung dan closet, jarak antar gedung |
| Building | Denah gedung tempat end device ditempatkan |
| Wiring Closet | Rack tempat router dan switch dipasang |

## 3. Pemilihan Medium dan Hardware

| Link | Jarak | Medium | Alasan |
| --- | --- | --- | --- |
| SW-CORE ke SW-B | 60 m | Copper straight-through (Gi2/1 ke Gi0/1) | Di bawah batas 100 m |
| SW-CORE ke SW-C | 180 m | Fiber multimode (modul PT-SWITCH-NM-1FGE di kedua sisi) | Melebihi 100 m; multimode Gigabit mampu hingga sekitar 550 m, single-mode tidak diperlukan |

Switch 2960-24TT dan 3560 standar tidak punya slot modul. Karena itu SW-CORE dan SW-C memakai **Switch-PT-Empty** yang modular, dan portnya ditambahkan lewat modul (saat power off).

| Perangkat | Modul | Jumlah | Fungsi |
| --- | --- | --- | --- |
| SW-CORE | PT-SWITCH-NM-1FGE | 1 | Fiber ke SW-C |
| SW-CORE | PT-SWITCH-NM-1CGE | 2 | Trunk ke R-A dan SW-B |
| SW-CORE | PT-SWITCH-NM-1CFE | 6 | Server, 4 PC, Printer |
| SW-C | PT-SWITCH-NM-1FGE | 1 | Fiber ke SW-CORE |
| SW-C | PT-SWITCH-NM-1CFE | 4 | 3 PC dan 1 AP |
| Laptop-C1, C2 | WPC300N (menggantikan modul Ethernet) | 1 per laptop | Koneksi wireless 2,4 GHz |

## 4. Rencana VLAN dan IP

| VLAN | Nama | Subnet | Gateway | Penghuni | Pengalamatan |
| --- | --- | --- | --- | --- | --- |
| 10 | Management | 10.10.10.0/24 | 10.10.10.1 | Switch, router, PC-A4 | Statis |
| 20 | Staff | 10.10.20.0/24 | 10.10.20.1 | PC HQ, Printer, PC Warehouse | DHCP (.11 dst) |
| 30 | Sales | 10.10.30.0/24 | 10.10.30.1 | PC Annex, laptop wireless | DHCP (.11 dst) |
| 40 | Servers | 10.10.40.0/28 | 10.10.40.1 | Server-A | Statis |
| 99 | Native-Unused | - | - | Native VLAN trunk | - |
| 999 | Blackhole | - | - | Port tidak terpakai | - |

| Perangkat | IP |
| --- | --- |
| R-A Gi0/0.10 / .20 / .30 / .40 | 10.10.10.1 / 10.10.20.1 / 10.10.30.1 / 10.10.40.1 (/28) |
| SW-CORE (VLAN 10) | 10.10.10.2 |
| SW-B (VLAN 10) | 10.10.10.3 |
| SW-C (VLAN 10) | 10.10.10.4 |
| PC-A4 (admin) | 10.10.10.10 |
| Server-A | 10.10.40.2 (DNS: intranet.maju.local) |

DHCP: alamat .1 sampai .10 dikecualikan di pool STAFF dan SALES; DNS server 10.10.40.2.

## 5. Wiring

| No | Dari | Port | Ke | Port | Kabel | Jarak |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | R-A | Gi0/0 | SW-CORE | Gi1/1 | Copper straight-through | di closet |
| 2 | SW-CORE | Fa3/1 | Server-A | FastEthernet0 | Copper straight-through | Gedung A |
| 3 | SW-CORE | Fa4/1 | PC-A1 | FastEthernet0 | Copper straight-through | Gedung A |
| 4 | SW-CORE | Fa5/1 | PC-A2 | FastEthernet0 | Copper straight-through | Gedung A |
| 5 | SW-CORE | Fa6/1 | PC-A3 | FastEthernet0 | Copper straight-through | Gedung A |
| 6 | SW-CORE | Fa7/1 | PC-A4 | FastEthernet0 | Copper straight-through | Gedung A |
| 7 | SW-CORE | Fa8/1 | Printer-A | FastEthernet0 | Copper straight-through | Gedung A |
| 8 | SW-CORE | Gi2/1 | SW-B | Gi0/1 | Copper straight-through | 60 m |
| 9 | SW-CORE | Gi0/1 (fiber) | SW-C | Gi0/1 (fiber) | Fiber | 180 m |
| 10-12 | SW-B | Fa0/1-3 | PC-B1..B3 | FastEthernet0 | Copper straight-through | Gedung B |
| 13-15 | SW-C | Fa1/1-3 | PC-C1..C3 | FastEthernet0 | Copper straight-through | Gedung C |
| 16 | SW-C | Fa4/1 | AP-C | Port 0 | Copper straight-through | Gedung C |
| 17 | AP-C | wireless | Laptop-C1, C2 | Wireless0 | WPA2-PSK | Gedung C |

Verifikasi CDP pada SW-CORE menunjukkan tiga tetangga sesuai rencana: R-A (Gig1/1 ke Gig0/0), SW-B (Gig2/1 ke Gig0/1), SW-C (Gig0/1 ke Gig0/1).

## 6. Konfigurasi Utama

Konfigurasi dasar semua perangkat: hostname, `no ip domain-lookup`, `ip domain-name maju.local`, `enable secret`, `service password-encryption`, user lokal `admin` dengan `secret`.

### 6.1 R-A: router-on-a-stick dan DHCP

```
interface gi0/0
 no ip address
 no shutdown
interface gi0/0.10
 encapsulation dot1Q 10
 ip address 10.10.10.1 255.255.255.0
interface gi0/0.20
 encapsulation dot1Q 20
 ip address 10.10.20.1 255.255.255.0
interface gi0/0.30
 encapsulation dot1Q 30
 ip address 10.10.30.1 255.255.255.0
interface gi0/0.40
 encapsulation dot1Q 40
 ip address 10.10.40.1 255.255.255.240
interface gi0/0.99
 encapsulation dot1Q 99 native

ip dhcp excluded-address 10.10.20.1 10.10.20.10
ip dhcp excluded-address 10.10.30.1 10.10.30.10
ip dhcp pool STAFF
 network 10.10.20.0 255.255.255.0
 default-router 10.10.20.1
 dns-server 10.10.40.2
ip dhcp pool SALES
 network 10.10.30.0 255.255.255.0
 default-router 10.10.30.1
 dns-server 10.10.40.2
```

### 6.2 Trunk (SW-CORE)

| Port | Tujuan | Native | Allowed VLAN |
| --- | --- | --- | --- |
| Gi1/1 | R-A | 99 | 10,20,30,40,99 |
| Gi2/1 | SW-B | 99 | 10,20 |
| Gi0/1 | SW-C (fiber) | 99 | 10,30 |

SW-B Gi0/1 memakai allowed 10,20. SW-C Gi0/1 memakai allowed 10,30. Native VLAN 99 di semua trunk.

### 6.3 Port akses dan port security

```
interface range fa0/1 - 3          ! contoh SW-B
 switchport mode access
 switchport access vlan 20
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
```

Port AP-C (SW-C Fa4/1) memakai `maximum 4` karena membawa MAC AP dan dua laptop di belakangnya. Port tidak terpakai (SW-B Fa0/4-24 dan Gi0/2): `shutdown` dan `switchport access vlan 999`. Pada SW-CORE semua port terpakai, sehingga tidak ada port sisa. `interface vlan 1` di-shutdown pada semua switch.

### 6.4 ACL (di R-A, arah masuk pada subinterface sumber)

```
ip access-list extended SALES-IN
 permit udp any eq bootpc any eq bootps
 permit tcp 10.10.30.0 0.0.0.255 host 10.10.40.2 eq 80
 permit icmp 10.10.30.0 0.0.0.255 host 10.10.40.2
 permit udp 10.10.30.0 0.0.0.255 host 10.10.40.2 eq 53
 permit icmp 10.10.30.0 0.0.0.255 any echo-reply
 permit tcp 10.10.30.0 0.0.0.255 any established
 deny ip any any

ip access-list extended STAFF-IN
 permit udp any eq bootpc any eq bootps
 deny ip 10.10.20.0 0.0.0.255 10.10.10.0 0.0.0.255
 permit ip any any

interface gi0/0.30
 ip access-group SALES-IN in
interface gi0/0.20
 ip access-group STAFF-IN in
```

Catatan desain: aturan DHCP diperlukan agar klien dapat IP; UDP 53 ke server diizinkan agar Sales dapat me-resolve intranet.maju.local (tambahan di luar spesifikasi, didokumentasikan di sini); `echo-reply` dan `established` diperlukan karena ACL bersifat stateless.

### 6.5 SSH

```
ip domain-name maju.local
crypto key generate rsa            ! jawab 1024 saat ditanya modulus
ip ssh version 2
access-list 10 permit 10.10.10.0 0.0.0.255
line vty 0 4                       ! 0 15 pada switch
 login local
 transport input ssh
 access-class 10 in
```

### 6.6 Server dan wireless

- Server-A: IP 10.10.40.2/28, gateway 10.10.40.1, DNS 10.10.40.2; layanan HTTP ON; DNS ON dengan record A `intranet.maju.local` ke 10.10.40.2.
- AP-C: SSID SALES, WPA2-PSK, passphrase \[isi\], port tersambung ke SW-C Fa4/1 (VLAN 30).
- Laptop-C1 dan C2: modul WPC300N, Wireless0 dengan SSID SALES, WPA2-PSK, IP DHCP.

## 7. Kendala yang Ditemui dan Solusinya

| Kendala | Penyebab | Solusi |
| --- | --- | --- |
| `crypto key generate rsa modulus 1024` ditolak (tanda ^ menunjuk ke modulus) | Versi IOS Packet Tracer tidak menerima parameter `modulus` | Ketik `crypto key generate rsa`, lalu isi 1024 saat diminta |
| Paket keamanan router `security: disable` | Lisensi securityk9 belum diaktifkan | `license boot module c2900 technology-package securityk9` di mode config, terima EULA, `write memory`, `reload`; hasil: securityk9 Evaluation |
| `switchport trunk encapsulation dot1q` ditolak | Switch hanya mendukung dot1q | Perintah dilewati, trunk tetap berfungsi (802.1q) |
| Log `%CDP-4-NATIVE_VLAN_MISMATCH` muncul berulang | SW-CORE native 99, SW-B dan SW-C masih native 1 | Samakan native VLAN 99 di semua trunk; `logging synchronous` di console agar log tidak memotong ketikan |
| Gi0/0 R-A masih memiliki IP 192.168.1.1 | Sisa konfigurasi tugas sebelumnya | `no ip address` pada interface induk |
| Laptop tidak dapat terhubung wifi | Modul bawaan Ethernet | Ganti dengan WPC300N saat power off |
| Nama port berbeda dari rencana awal (Fa0/1 dst) | Switch modular menamai port per slot modul | Nama port disesuaikan dari `show ip interface brief` |

## 8. Bukti Verifikasi (hasil nyata)

- SW-CORE `show interfaces status`: 9 port berstatus connected (Gig0/1, Gig1/1, Gig2/1, Fa3/1 sampai Fa8/1).
- `show interfaces trunk` pada SW-CORE, SW-B, SW-C: native VLAN 99 dan allowed VLAN sesuai rencana.
- `show vlan brief` SW-CORE: VLAN 10, 20, 30, 40, 99, 999 aktif; SW-C: Fa1/1 sampai Fa4/1 di VLAN 30.
- `show ip dhcp binding` R-A: klien memperoleh alamat mulai .11, membuktikan excluded-address bekerja; VLAN 20 melayani HQ dan Warehouse, VLAN 30 melayani Annex. Setelah pemeriksaan lanjutan, seluruh PC dan laptop memperoleh IP 10.10.x.x.
- `show port-security` SW-C: Fa1/1-3 maksimum 2, Fa4/1 maksimum 4, mode Restrict, violation count 0.
- Link fiber SW-CORE ke SW-C: \[lampirkan screenshot Physical view 180 m dan status connected\]. Catatan: kolom Type pada `show interfaces status` menampilkan 10/100/1000BaseTX; pastikan kabel yang dipakai bertipe Fiber (garis putus-putus oranye di Logical view).

## 9. Test Plan

Isi kolom Actual setelah pengujian dilakukan dengan ACL aktif.

| No | Test | Expected | Actual | Status |
| --- | --- | --- | --- | --- |
| 1 | Staff PC ping Warehouse Staff PC | Success |  |  |
| 2 | Staff PC buka intranet.maju.local | Success |  |  |
| 3 | Sales PC ping Staff PC | Fail |  |  |
| 4 | Sales PC buka web server (http://10.10.40.2) | Success |  |  |
| 5 | Sales PC ping router VLAN 10 (10.10.10.1) | Fail |  |  |
| 6 | Staff PC ping Management (10.10.10.1) | Fail |  |  |
| 7 | Rogue PC (hub dengan 3 PC) di port Warehouse | Port restricts (violation count naik) |  |  |
| 8 | SSH dari Staff PC ke switch | Fail |  |  |
| 9 | SSH dari PC-A4 ke switch (`ssh -l admin 10.10.10.3`) | Success |  |  |

Perintah bukti: `show access-lists`, `show ip interface gi0/0.30`, `show port-security interface fa0/1`.

## 10. Troubleshooting Twist

Format tiap kasus: gejala, perintah show, temuan, root cause, perbaikan.

| Kasus | Gejala | Perintah diagnosis | Root cause | Perbaikan |
| --- | --- | --- | --- | --- |
| 1. Native VLAN mismatch (sudah terjadi secara nyata) | Log CDP-4-NATIVE\_VLAN\_MISMATCH berulang tiap sekitar 60 detik | `show interfaces trunk`, `show cdp neighbors detail` | Native VLAN core 99, switch lawan 1 | `switchport trunk native vlan 99` di sisi lawan |
| 2. VLAN 30 dihapus dari allowed list trunk ke SW-C | Klien Sales tidak dapat IP | `show interfaces trunk` | VLAN 30 tidak ada di allowed dan active | Tambahkan `switchport trunk allowed vlan add 30` |
| 3. ACL STAFF-IN dipasang arah `out` | Staff masih dapat ping VLAN 10 | `show ip interface gi0/0.20`, `show access-lists` | ACL di arah salah | Ubah ke `ip access-group STAFF-IN in` |
| 4 (opsional). `no shutdown` Gi0/0 R-A dihapus | Semua VLAN kehilangan gateway | `show ip interface brief` | Interface induk down | `no shutdown` |

Isi bagian temuan dan screenshot untuk tiap kasus: \[lengkapi\].

## 11. Daftar Deliverable

- [ ] Screenshot City view (tiga gedung dengan jarak 60 m dan 180 m)
- [ ] Screenshot rack tiap wiring closet
- [ ] Screenshot Physical tab perangkat dengan modul terpasang (SW-CORE, SW-C)
- [ ] Link fiber hijau dan `show interfaces status`
- [ ] Output `show vlan brief`, `show interfaces trunk`, `show ip dhcp binding`, `show access-lists`, `show port-security`
- [ ] Test plan lengkap (Expected vs Actual)
- [ ] Dokumentasi troubleshooting

## 12. Kesimpulan

Jaringan tiga gedung berhasil dirancang dengan segmentasi VLAN, inter-VLAN routing, DHCP terpusat, dan kontrol akses berbasis ACL. Jarak 60 m dilayani tembaga, sedangkan 180 m dilayani fiber multimode sehingga tidak melanggar batas 100 m tembaga. Penempatan perangkat di wiring closet memberi keamanan fisik, kerapian kabel, dan kemudahan perawatan. \[Tambahkan kesimpulan hasil test plan setelah kolom Actual terisi.\]

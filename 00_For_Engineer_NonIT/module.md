# Maintenance Enhancement Workflow
### Panduan Eksekusi Perubahan Logic SAP — Untuk Maintenance Engineer

---

## 1. Pendahuluan

### 1.1 Target Pembaca
Dokumen ini untuk maintenance engineer yang sudah menguasai SAP PM dari sisi bisnis (equipment, functional location, notification, work order) namun belum terbiasa dengan proses perubahan sistem (enhancement). Fokus dokumen: eksekusi dan verifikasi, bukan pengenalan modul PM.

### 1.2 Enhancement vs Modification
Enhancement adalah penyisipan logic baru melalui titik resmi yang disediakan SAP (User Exit, BAdI, Enhancement Point) tanpa mengubah kode standar. Modification adalah perubahan langsung pada kode standar SAP — dihindari karena hilang saat upgrade dan membutuhkan access key khusus dari SAP.

### 1.3 Pembagian Peran
Requester (business/maintenance) merumuskan kebutuhan menjadi Functional Specification. ABAP Developer mengimplementasikan. Requester memverifikasi hasil melalui debugger, bukan membaca kode. Dokumen ini menutup kompetensi di peran pertama dan ketiga.

---

## 2. Siklus Enhancement di Proyek Nyata

Berikut tahapan standar untuk satu Change Request (CR) enhancement skala kecil-menengah, dengan estimasi durasi realistis. Durasi bervariasi tergantung kompleksitas dan antrean di tim IT/ABAP.

| Fase | Aktivitas | Output | Pihak Terlibat | Estimasi Durasi |
|---|---|---|---|---|
| 1. Requirement Gathering | Kebutuhan mentah dari lapangan digali sampai jadi kebutuhan spesifik (bukan solusi) | Notes/minutes of meeting | Requester, end user | 1-2 hari |
| 2. Data Feasibility Check | Verifikasi apakah data pendukung sudah tersedia di SAP (native atau interface) | Catatan hasil cek | Requester, functional consultant | 1-2 hari |
| 3. Functional Specification | Business rule dituangkan ke FS: trigger, kondisi, data, output, test scenario | Dokumen FS | Requester | 1-3 hari |
| 4. Technical Analysis | Developer menelusuri program terkait dan titik enhancement yang tersedia | Technical Spec / catatan analisis | ABAP developer | 2-4 hari |
| 5. Build (Development) | Implementasi logic di environment DEV, pada titik enhancement yang dipilih | Objek ABAP di DEV | ABAP developer | 3-7 hari |
| 6. Verifikasi Bersama | Requester dan developer memverifikasi logic lewat debugger (breakpoint/watchpoint) sebelum masuk testing formal | Catatan verifikasi | Requester, developer | 0.5-1 hari |
| 7. Testing di QAS | Positive, negative, boundary, regression case dijalankan di Quality System | Test result / sign-off | Requester, developer, QA | 2-5 hari |
| 8. Approval & Transport Scheduling | Sign-off requester, review CAB (jika ada), penjadwalan transport ke Production | TR approved, jadwal transport | Requester, approver, Basis | 1-3 hari |
| 9. Go-Live | Transport dieksekusi ke Production, biasanya di luar jam operasional kritikal | Perubahan live di PRD | Basis, developer | 1 hari |
| 10. Hypercare | Pemantauan ketat pasca go-live untuk menangkap false positive/negative yang tidak muncul saat testing | Log monitoring, defect list (jika ada) | Requester, developer | 5-10 hari |

Total durasi tipikal untuk CR skala kecil-menengah: **3-5 minggu**. Ini bukan indikasi ABAP-nya rumit — mayoritas waktu terpakai di analisis, testing berlapis, dan approval. Penting dikomunikasikan ke atasan agar ekspektasi timeline realistis.

Catatan praktis:
- Fase 2 (Data Feasibility Check) sering dilewati, padahal ini penyebab paling umum CR "macet": kebutuhan sebenarnya adalah masalah integrasi data (data belum masuk SAP), bukan masalah logic.
- Fase 6 (Verifikasi Bersama) sering dianggap opsional oleh developer karena requester dianggap tidak perlu tahu detail teknis. Ini yang membuat requester tidak punya kontrol atas hasil akhir sebelum testing formal — sebaiknya diminta secara eksplisit.
- Fase 10 (Hypercare) sering diabaikan setelah go-live dianggap "selesai". Requester adalah pihak yang paling tepat memantau fase ini karena berada di lapangan.

---

## 3. Requirement dan Business Rule

### 3.1 Requirement vs Solution
Requirement adalah kebutuhan; solusi ditentukan setelah analisis. Requester sering mengajukan solusi langsung (contoh: "buatkan notification otomatis"). Tugas maintenance engineer adalah menggali kebutuhan yang mendasarinya sebelum solusi ditentukan.

### 3.2 Business Rule
Business rule dirumuskan dalam format kondisi-aksi yang presisi dan bisa diuji:

"JIKA status breaker berubah menjadi TRIP DAN breaker berada di Functional Location kategori kritikal, MAKA sistem membuat Notification tipe M2 prioritas tinggi ke role Shift Supervisor."

### 3.3 Functional Specification (FS)
FS ditulis oleh requester (template pada bagian 15), memuat trigger, business rule, data yang terlibat, output yang diharapkan, dan skenario testing. FS menjadi acuan resmi bagi developer dan dokumentasi approval.

---

## 4. Analisis Sebelum Eksekusi

### 4.1 Ketersediaan Data
Verifikasi apakah data pendukung business rule sudah tersedia di SAP, baik dari entry manual maupun interface dari sistem lain (SCADA/PLC). Jika belum tersedia, masalahnya adalah integrasi data, bukan enhancement logic.

### 4.2 Kebutuhan Struktur Data Baru
Jika business rule membutuhkan atribut yang belum memiliki tempat di sistem (contoh: kategori kritikal pada Functional Location), diperlukan field atau tabel tambahan.

### 4.3 Dampak Lintas Proses
Perubahan pada modul PM berpotensi berdampak ke modul lain (misalnya MM untuk kebutuhan material pada Work Order) atau proses notifikasi/workflow lain yang sudah berjalan.

### 4.4 Penentuan Enhancement vs Modification
Pertanyaan wajib ke developer setiap CR: apakah requirement dapat dipenuhi melalui titik enhancement resmi, atau memerlukan modifikasi kode standar. Jawaban ini menentukan risiko jangka panjang terhadap upgrade sistem.

---

## 5. Desain Logic

### 5.1 Decision Table
Format standar untuk business rule dengan banyak kondisi:

| Status | Kategori FL | Aksi |
|---|---|---|
| TRIP | Kritikal | Notification prioritas tinggi ke Shift Supervisor |
| TRIP | Non-kritikal | Notification prioritas normal |
| NORMAL | — | Tidak ada aksi |

### 5.2 Pseudo Code
```
JIKA status_breaker = "TRIP" DAN kategori_FL = "KRITIKAL"
    MAKA buat_notification(tipe="M2", prioritas="TINGGI")
```
Level ini yang dapat ditulis dan diverifikasi requester sebelum masuk ke implementasi ABAP.

---

## 6. Menemukan Program dan Titik Enhancement

### 6.1 Identifikasi Program dari Transaksi
Menu System > Status pada transaksi yang sedang dijalankan menampilkan nama program dan screen number aktif. Ini titik awal standar untuk memastikan program yang dianalisis benar.

### 6.2 Identifikasi Tabel/Field dari Layar
F1 pada field tertentu, dilanjutkan klik Technical Information, menampilkan nama tabel dan field teknis yang mendasari tampilan tersebut, serta program dan screen number terkait.

### 6.3 Runtime Analysis (SAT)
Transaksi SAT (pengganti ST12/SE30) merekam seluruh program, FORM, dan Function Module yang dieksekusi saat satu aksi dilakukan di layar. Digunakan dengan cara: aktifkan recording, jalankan transaksi yang dianalisis di sesi terpisah, lakukan aksi yang diteliti, hentikan recording. Hasil rekaman menunjukkan urutan eksekusi program secara aktual, sehingga analisis tidak bergantung pada dugaan.

### 6.4 Identifikasi Titik Enhancement yang Sudah Ada
Sebelum mengusulkan enhancement baru, periksa apakah implementasi sudah ada di titik tersebut:
- SE38/SE80, menu Edit > Enhancement Operations > Show Implicit Enhancement Options, untuk melihat titik enhancement yang tersedia maupun yang sudah terisi.
- SE18 (definisi BAdI) / SE19 (implementasi BAdI).
- SMOD (definisi User Exit) / CMOD (project implementasi).

### 6.5 Penelusuran dari Pesan Sistem
Message class dan nomor message (didapat via F1 pada pesan yang muncul di layar) memungkinkan penelusuran where-used untuk menemukan FORM/program yang memicu pesan tersebut secara langsung.

### 6.6 Where-Used List
Klik kanan pada Function Module/tabel di SE38/SE80, pilih Where-Used List (Ctrl+Shift+F3), untuk melihat seluruh program lain yang memakai objek yang sama. Digunakan untuk memastikan enhancement baru tidak berbenturan dengan proses lain.

Urutan analisis yang disarankan: 6.1 untuk identifikasi program awal, 6.3 untuk memastikan alur eksekusi aktual (satu transaksi dapat memanggil banyak program), 6.4 untuk mengecek ketersediaan titik enhancement, sebelum masuk ke desain logic pada bagian 5.


## 7. Satu Sesi Enhancement di ABAP Editor

Bab sebelumnya menjelaskan bagaimana program dan titik enhancement ditemukan. Bab ini menjelaskan langkah setelahnya: apa yang terjadi ketika program tersebut sudah terbuka di layar, dan bagaimana mengikuti penjelasan developer tanpa perlu menguasai sintaks ABAP secara penuh.

### 7.1 Membuka Program (SE38)
Nama program yang didapat dari bagian 6.1/6.3 dimasukkan ke transaksi SE38, lalu dijalankan dalam mode Display. Program akan terbuka sebagai teks, disusun dari atas ke bawah, namun struktur ini **tidak dibaca berurutan** seperti dokumen biasa.

### 7.2 Bagian yang Dilewati vs Bagian yang Diikuti
Struktur umum sebuah program ABAP:

```
REPORT ...              -> deklarasi nama program, dilewati
TABLES ...               -> deklarasi tabel yang dipakai, dilewati
DATA ...                 -> deklarasi variabel, dilewati saat pertama kali dibaca
START-OF-SELECTION.      -> titik ini yang menjadi acuan mulai membaca
  PERFORM ...
```

`START-OF-SELECTION` adalah titik masuk eksekusi program yang sesungguhnya. Bagian di atasnya (REPORT, TABLES, DATA) adalah deklarasi yang baru relevan dicek belakangan, saat memang dibutuhkan (misalnya untuk mengecek nama variabel tertentu).

### 7.3 Mengikuti PERFORM ke FORM
Baris berbentuk `PERFORM <nama>.` adalah pemanggilan blok logic bernama `<nama>`. Meletakkan kursor pada baris tersebut lalu double-click akan membawa tampilan ke definisi blok logic tersebut, ditandai dengan `FORM <nama>. ... ENDFORM.`. Ini cara program ABAP disusun sebagai rangkaian blok yang saling memanggil, bukan satu blok besar berurutan.

### 7.4 Mencari Blok Logic Tertentu
Ketika developer menyebutkan nama blok logic secara spesifik (contoh: "logicnya ada di FORM CREATE_NOTIFICATION"), gunakan Ctrl+F pada editor, cari teks `CREATE_NOTIFICATION`, dan navigasi ke baris `FORM CREATE_NOTIFICATION`. Tidak diperlukan membaca ulang seluruh program dari awal.

### 7.5 Membaca Perubahan Logic, Bukan Sintaks
Ketika developer menunjukkan perubahan, fokus pada **apa yang berubah dari sisi keputusan bisnis**, bukan pada sintaksnya.

Logic sebelum perubahan:
```abap
IF status = 'TRIP'.
  PERFORM create_notification.
ENDIF.
```

Logic setelah perubahan:
```abap
IF status = 'TRIP'.
  IF running_hour > 6000.
    PERFORM create_notification.
  ENDIF.
ENDIF.
```

Cara membaca perubahan ini: sebelumnya notification dibuat setiap kali status TRIP. Setelah perubahan, notification hanya dibuat jika status TRIP **dan** running hour motor sudah di atas 6000 jam. Developer menambahkan satu syarat keputusan baru, bukan mengganti keseluruhan program. Perubahan pada level ini yang relevan diverifikasi terhadap business rule pada bagian 3.2 — bukan detail penulisan `IF`/`ENDIF` itu sendiri.

### 7.6 Setelah Logic Ditambahkan
Setelah baris ditambahkan, developer menjalankan Check (validasi sintaks), Activate (mengaktifkan objek), lalu Execute untuk menjalankan program dan melihat hasilnya. Tahap ini singkat secara durasi, namun baru bisa dilakukan setelah analisis pada bagian 6 selesai.

### 7.7 Alokasi Waktu Developer Selama Build
Fase Build pada tabel di bagian 2 (estimasi 3-7 hari) tidak terpakai seluruhnya untuk menulis kode. Rincian tipikal:

| Aktivitas | Porsi Waktu |
|---|---|
| Mencari program dan menelusuri alur (bagian 6.1, 6.3) | 20-30% |
| Mencari/menentukan titik enhancement yang tepat (bagian 6.4) | 20-30% |
| Menulis logic (IF, PERFORM, dsb.) | 15-20% |
| Debugging dan penyesuaian | 15-20% |
| Persiapan testing awal sebelum diserahkan ke QAS | 10-15% |

Rincian ini relevan dikomunikasikan ke atasan atau pihak yang menganggap proses ini seharusnya selesai dalam hitungan jam.


---

## 8. Verifikasi via Debugger

### 7.1 Aktivasi Debugger
Ketik `/h` pada command bar, jalankan transaksi seperti biasa. Aksi berikutnya (misalnya Save) akan menghentikan eksekusi dan masuk ke mode debug.

### 7.2 Breakpoint
Titik jeda pada program, ditandai developer di titik business rule yang relevan. Diminta secara eksplisit pada sesi verifikasi bersama (Fase 6, bagian 2).

### 7.3 Watchpoint
Berhenti otomatis ketika variabel tertentu berubah ke nilai tertentu (misalnya status berubah menjadi TRIP), tanpa perlu mengetahui posisi baris kode.

### 7.4 Pembacaan Nilai Variabel
Saat program berhenti, nilai seluruh variabel tampil dalam tabel pada layar debugger — memungkinkan requester memverifikasi hasil logic tanpa membaca kode.

---

## 9. Testing

| Jenis | Contoh |
|---|---|
| Positive Case | Trip di FL kritikal menghasilkan notification prioritas tinggi |
| Negative Case | Status normal tidak menghasilkan notification |
| Boundary Case | FL yang belum diklasifikasi kritikal/non-kritikal |
| Regression Test | Proses notification lain yang tidak dimaksud berubah tetap berjalan normal |

Testing dijalankan di Quality System (QAS) dengan keterlibatan langsung requester, bukan berdasarkan laporan sepihak dari developer.

---

## 10. Transport dan Approval

### 9.1 Transport Request (TR)
Paket resmi yang mencatat perubahan, dipindahkan dari Development ke Quality ke Production. Tidak ada perubahan yang masuk ke Production tanpa melalui TR.

### 9.2 Penjadwalan
Transport ke Production umumnya dijadwalkan di luar jam operasional kritikal untuk meminimalkan risiko gangguan.

### 9.3 Approval
Requester memberikan sign-off tertulis atas hasil testing di QAS sebelum TR dieksekusi ke Production.

---

## 11. Governance

- Approval: requester, reviewer teknis, dan atasan terkait.
- Dokumentasi: FS, hasil testing, dan nomor TR disimpan sebagai jejak audit.
- Rollback plan: disiapkan sebelum transport ke Production.
- Hypercare: pemantauan 1-2 minggu pasca go-live (Fase 10, bagian 2).

---

## 12. Kesalahan yang Sering Terjadi

1. Requester mengajukan solusi jadi, bukan kebutuhan yang mendasarinya.
2. Data Feasibility Check (bagian 4.1) dilewati — masalah integrasi data disalahartikan sebagai kebutuhan enhancement.
3. Modifikasi kode standar dilakukan tanpa mempertimbangkan titik enhancement resmi.
4. Requester tidak dilibatkan dalam testing, hanya menerima laporan dari developer.
5. Transport ke Production dieksekusi tanpa approval atau jadwal yang jelas.
6. Hypercare tidak dijalankan — defect baru diketahui berminggu-minggu setelah go-live.

---

## 13. Glosarium dan T-Code

| Istilah/T-Code | Kegunaan |
|---|---|
| System > Status | Melihat nama program dan transaksi yang sedang berjalan |
| F1 → Technical Information | Melihat tabel/field teknis di balik field pada layar |
| SAT (dulu ST12/SE30) | Merekam program/FORM/FM yang dieksekusi saat satu aksi dilakukan |
| SE38 | Membuka program ABAP tunggal |
| SE80 | Object Navigator |
| Edit > Enhancement Operations | Melihat titik enhancement yang tersedia/terisi pada suatu program |
| SE18 / SE19 | Definisi / implementasi BAdI |
| SMOD / CMOD | User Exit klasik: definisi / project implementasi |
| Where-Used List (Ctrl+Shift+F3) | Melihat seluruh pemakaian objek yang sama di program lain |
| /h | Mengaktifkan mode debug |
| Transport Request (TR) | Paket resmi perpindahan perubahan antar environment |

---

## 14. Template Functional Specification (FS)

```
FUNCTIONAL SPECIFICATION

1. Judul Requirement          :
2. Requester & Tanggal        :
3. Latar Belakang & Tujuan    :
4. Trigger                    :
5. Business Rule (JIKA...MAKA...) :
6. Data yang Terlibat (entity/field/sumber) :
7. Output yang Diharapkan     :
8. Dampak ke Proses/Modul Lain:
9. Skenario Pengujian
   - Positive Case :
   - Negative Case  :
   - Boundary Case  :
10. Approval (Requester / Reviewer / Approver) :
```

---

Bagian sintaks ABAP tersedia terpisah di repository. Dokumen ini membahas proses eksekusi dan verifikasi enhancement.

# Modul 7 - Profiling 

**Nama:** Izzudin Abdul Rasyid  
**NPM:** 2406495786  

---

## 🔍 Before Optimization 

Pada tahap ini, aplikasi dijalankan menggunakan data *dummy* sebanyak 5.000 data `Student` dan 10.000 data relasi `StudentCourse`. Pengujian beban (*load testing*) dilakukan menggunakan **Apache JMeter** dengan skenario 10 *Thread* (Users) yang dieksekusi secara bersamaan.

Berikut adalah hasil analisis performa dan identifikasi masalah (*bottleneck*) pada ketiga *endpoint* sebelum dilakukan optimasi:

### 1. Endpoint `/all-student` 

**Penjelasan Masalah:**
Endpoint ini memiliki performa yang sangat lambat karena mengalami masalah **N+1 Query** pada Hibernate. Di dalam `StudentService.java`, aplikasi menarik seluruh daftar mahasiswa dari database (1 *query*), kemudian melakukan *looping* untuk setiap mahasiswa guna menarik data *course* mereka menggunakan `findByStudentId` (5.000 *query* tambahan). Total terdapat **5.001 query** yang dieksekusi ke PostgreSQL dalam satu kali *request*.

**Hasil Uji Coba JMeter:**
Berdasarkan pengujian, masalah N+1 Query ini menyebabkan antrean proses I/O ke database yang sangat berat, membuat waktu respons anjlok.
* **Average Load Time (GUI):** 6.798 ms (Lebih dari 6 detik).
* **Average Load Time (CLI):** 6.250 ms.

**Bukti Eksekusi JMeter GUI:**
![JMeter GUI - All Student Before](screenshots/all-studentBefore.png)

**Bukti Eksekusi JMeter Terminal (CLI):**
![JMeter CLI - All Student Before](screenshots/all-studentBeforeTerminal.png)

**Hasil JMeter Test Logs:**
![JMeter JTL - All Student Before](screenshots/hasil_test_1.png)

---

### 2. Endpoint `/all-student-name`

**Penjelasan Masalah:**
Walaupun waktu respons tergolong cepat, terdapat pemborosan memori dan CPU yang signifikan:
1. **Overfetching:** Aplikasi menggunakan `studentRepository.findAll()` yang setara dengan `SELECT * FROM students`. Seluruh kolom (termasuk `id`, `faculty`, `gpa`) ditarik ke RAM, padahal yang dibutuhkan hanyalah kolom `name`.
2. **Inefficient Concatenation:** Penggunaan operator `+=` di dalam *looping* sangat membebani *Garbage Collector* Java karena sifat *String* yang *immutable* (menciptakan ribuan objek *String* baru secara repetitif).

**Hasil Uji Coba JMeter:**
* **Average Load Time (GUI):** 52 ms.
* **Average Load Time (CLI):** 52 ms.

**Bukti Eksekusi JMeter GUI:**
![JMeter GUI - All Student Name Before](screenshots/all-student-nameBefore.png)

**Bukti Eksekusi JMeter Terminal (CLI):**
![JMeter CLI - All Student Name Before](screenshots/all-student-nameBeforeTerminal.png)

**Hasil JMeter Test Logs:**
![JMeter JTL - All Student Name Before](screenshots/hasil_test_2.png)

---

### 3. Endpoint `/highest-gpa`

**Penjelasan Masalah:**
Metode `findStudentWithHighestGpa()` melakukan *anti-pattern* dengan menarik seluruh 5.000 baris data mahasiswa ke memori aplikasi Java hanya untuk mencari satu nilai GPA tertinggi melalui *looping*. Proses komputasi/pencarian ini seharusnya diserahkan kepada Database Management System (DBMS). Jika data diskalakan hingga jutaan, *endpoint* ini berpotensi besar menyebabkan **Out of Memory (OOM) Error**.

**Hasil Uji Coba JMeter:**
* **Average Load Time (GUI):** 30 ms.
* **Average Load Time (CLI):** 35 ms.

**Bukti Eksekusi JMeter GUI:**
![JMeter GUI - Highest GPA Before](screenshots/highest-gpaBefore.png)

**Bukti Eksekusi JMeter Terminal (CLI):**
![JMeter CLI - Highest GPA Before](screenshots/highest-gpaBeforeTerminal.png)

**Hasil JMeter Test Logs:**
![JMeter JTL - Highest GPA Before](screenshots/hasil_test_3.png)

---

## 🚀 After Optimization

Setelah mengidentifikasi masalah pada tahap baseline, dilakukan optimasi pada layer `Repository` dan `Service`. Berikut adalah hasil perbandingan performa setelah dilakukan *refactoring* kode:

### 1. Endpoint `/all-student` (Optimasi N+1 Query)

**Langkah Optimasi:**
Menggunakan teknik **JOIN FETCH** pada `StudentCourseRepository` untuk menarik data `Student` dan `Course` dalam satu *query* tunggal, sehingga menghilangkan 5.000 *query* tambahan yang sebelumnya terjadi.

**Hasil Performa:**
* **Average Load Time:** Turun drastis dari **6.250 ms** menjadi sekitar **71 - 96 ms**.
* **Peningkatan:** >98% lebih cepat.

**Bukti Eksekusi JMeter GUI:**
![JMeter GUI - All Student After](screenshots/allstudentafter1.png)

**Hasil JMeter Test Logs:**
![JMeter JTL - All Student After](screenshots/allstudentafter2.png)

---

### 2. Endpoint `/all-student-name` (Optimasi Memori & String)

**Langkah Optimasi:**
1. Menerapkan **Database Projection** untuk hanya mengambil kolom `name` dari database.
2. Mengganti penggabungan string manual (`+=`) dengan **`String.join()`** yang jauh lebih efisien dalam penggunaan memori dan CPU.

**Hasil Performa:**
* **Average Load Time:** Turun dari **52 ms** menjadi sekitar **11 - 12 ms**.
* **Peningkatan:** >75% lebih cepat.

**Bukti Eksekusi JMeter GUI:**
![JMeter GUI - All Student Name After](screenshots/allstudentnameafter1.png)

**Hasil JMeter Test Logs:**
![JMeter JTL - All Student Name After](screenshots/allstudentnameafter2.png)

---

### 3. Endpoint `/highest-gpa` (Optimasi Database Sorting)

**Langkah Optimasi:**
Delegasi proses pencarian GPA tertinggi ke database menggunakan fungsi `findFirstByOrderByGpaDesc()`. Aplikasi tidak lagi menarik seluruh data ke RAM, melainkan hanya menerima **1 baris data** hasil sortir dari database.

**Hasil Performa:**
* **Average Load Time:** Turun dari **35 ms** menjadi sekitar **7 - 10 ms**.
* **Peningkatan:** >70% lebih cepat.

**Bukti Eksekusi JMeter GUI:**
![JMeter GUI - Highest GPA After](screenshots/highestgpaafater1.png)

**Hasil JMeter Test Logs:**
![JMeter JTL - Highest GPA After](screenshots/highestgpaafter2.png)

---

## 💡 Kesimpulan
Melalui teknik profiling dan optimasi yang tepat, aplikasi berhasil mencapai target peningkatan performa minimum 20%. Peningkatan paling signifikan terlihat pada endpoint `/all-student` yang berhasil mengatasi masalah I/O bottleneck (N+1 Query), menjadikannya ribuan kali lebih efisien dan siap untuk menangani beban data yang lebih besar.


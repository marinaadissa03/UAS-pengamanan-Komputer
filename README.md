# UAS-pengamanan-Komputer

## Identitas
- Nama: Marina Adissa Damayanti
- NIM: 221080200082
- Kelas: Pengamanan Sistem Komputer 6/B1
- Repo GitHub: [https://github.com/marinaadissa03/UAS-pengamanan-Komputer/blob/main/jawaban%20disa082]

---

## Bagian A – Bug Fixing JWT REST API

### Bug 1: Token tetap aktif setelah logout
Penjelasan:
Token yang sudah dibuat saat login tetap bisa digunakan setelah logout karena tidak ada sistem blacklist atau revocation list untuk token. Artinya, logout tidak benar-benar menghapus token yang sedang aktif, sehingga bisa dimanfaatkan oleh pihak yang tidak sah.

Solusi:
Tambahkan sistem blacklist token di cache atau storage sementara (misalnya array/Redis). Saat logout, token ditambahkan ke daftar blacklist. Filter harus memverifikasi apakah token sudah diblacklist sebelum memproses permintaan lebih lanjut.

Commit:
feat: implement token blacklist system for logout

###Bug 2: Tidak ada pembatasan akses endpoint berdasarkan role
Penjelasan:
Semua user bisa mengakses semua endpoint meskipun hanya admin yang seharusnya boleh. Tidak ada validasi role user yang sedang login.

Solusi:
Tambahkan filter role berbasis JWT yang memeriksa nilai role di dalam token. Hanya user dengan role='admin' yang boleh akses endpoint tertentu.

Commit:
fix: add role-based access control filter

### Bug 3: User bisa mengakses atau mengubah data user lain
Penjelasan:
API tidak membatasi ID user yang boleh diubah. Siapapun bisa mengirimkan id berbeda di URL atau body untuk mengubah data user lain (termasuk admin).

Solusi:
Tambahkan logika validasi: jika role bukan admin, hanya boleh mengakses atau mengubah data dengan ID miliknya sendiri (dari payload token).

Commit:
fix: enforce user ownership check for update endpoint

---

## Bagian B – Simulasi Serangan dan Solusi

### Jenis Serangan: Broken Access Control  
**Simulasi Postman:**  
User biasa bisa mengubah data user lain dengan mengganti id di body request (misalnya menjadi id: 1 milik admin). Karena tidak ada validasi pemilik data, maka terjadilah pelanggaran otorisasi akses.

Simulasi Postman:

Method: PUT /api/users/update/1

Header: Authorization: Bearer <token-user-biasa>

Body:

{
  "username": "hacked_admin",
  "email": "admin_hacked@example.com"
}

Jika tidak dibatasi, maka data admin akan terganti.

**Solusi Implementasi:**  
Tambahkan pengecekan role & kepemilikan data pada filter atau di controller:
// Di dalam UserController.php
public function update($id = null)
{
    $authHeader = $this->request->getHeaderLine('Authorization');
    $token = str_replace('Bearer ', '', $authHeader);
    $decoded = JWT::decode($token, getenv('JWT_SECRET'), ['HS256']);

    // Hanya admin atau pemilik ID yang bisa update
    if ($decoded->role !== 'admin' && $decoded->id != $id) {
        return $this->fail('Forbidden', 403);
    }

    $data = $this->request->getJSON();
    $this->model->update($id, [
        'username' => $data->username,
        'email' => $data->email,
    ]);
    return $this->respond(['message' => 'Updated']);
}

Commit:
fix: protect update endpoint from unauthorized ID change



---

## Bagian C – Refleksi Teori & Etika

### 1. CIA Triad dalam Keamanan Informasi  
- Confidentiality: Menjamin bahwa hanya pihak yang berwenang dapat mengakses informasi.
- Integrity: Menjamin informasi tidak diubah oleh pihak yang tidak sah.
- Availability: Menjamin bahwa data dan sistem selalu tersedia saat dibutuhkan.

### 2. UU ITE yang relevan  
- Pasal 30 Ayat (1): Larangan mengakses sistem elektronik milik orang lain secara tidak sah.
- Pasal 32 Ayat (1): Larangan mengubah, menambah, mengurangi, mentransmisikan, atau merusak data milik orang lain secara ilegal.

### 3. Pandangan Al-Qur'an  
- Surah Al-Baqarah: 205  
"Dan apabila ia berpaling (dari kamu), ia berjalan di muka bumi untuk membuat kerusakan padanya dan merusak tanaman-tanaman dan binatang ternak. Dan Allah tidak menyukai kerusakan."

Tafsir: Allah melarang segala bentuk perusakan, termasuk perusakan data, informasi, dan sistem dalam dunia maya. Etika Islam menekankan tanggung jawab digital.

### 4. Etika Cyber dan Kejujuran  
Etika cyber mengharuskan profesional untuk menjaga amanah, tidak menyalahgunakan akses, serta bertanggung jawab atas kerahasiaan data. Kejujuran menjadi dasar dari semua praktik keamanan siber agar sistem dapat dipercaya dan berfungsi adil untuk semua pihak.

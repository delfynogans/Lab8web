## Praktikum 7-10 

| Atribut         | Keterangan            |
| --------------- | --------------------- |
| **Nama**        | DELFYNO DWI PRASTYO   |
| **NIM**         | 312310480             |
| **Kelas**       | TI.23.A.5             |
| **Mata Kuliah** | Pemrograman Website 2 |

## Membuat Model: `KategoriModel`

membuat model untuk mengelola data kategori melalui CodeIgniter 4.

![G_1](https://github.com/user-attachments/assets/f973a5f0-823f-4d44-9928-2c98ed791edf)

##  Memodifikasi Model: `ArticleModel`

Setelah membuat `KategoriModel`, langkah selanjutnya adalah memodifikasi model `ArticleModel` agar mendukung relasi dengan tabel `kategori`.
![G_2](https://github.com/user-attachments/assets/b65e9f52-c6ae-41ce-9c2d-00634ae65abb)
![G_3](https://github.com/user-attachments/assets/25bb83b5-ba7d-4d1e-9297-89716bf93e82)

##  Memodifikasi Controller: `Article.php`
Modifikasi `Artikel.php` untuk menggunakan model baru dan menampilkan data relasi:
```php
<?php

namespace App\Controllers;

use App\Models\ArtikelModel;
use App\Models\KategoriModel;

class Artikel extends BaseController
{
    
    public function index()
    {
        $title = 'Daftar Artikel';
        $model = new ArtikelModel();

        $artikel = $model
            ->select('artikel.*, kategori.nama_kategori')
            ->join('kategori', 'kategori.id_kategori = artikel.id_kategori', 'left')
            ->where('artikel.status', 1) // Hanya tampilkan artikel yang aktif
            ->orderBy('artikel.id', 'DESC') // Urutkan dari yang terbaru
            ->findAll();

        return view('artikel/index', compact('artikel', 'title'));
    }

    public function admin_index()
    {
        $title = 'Daftar Artikel';
        $model = new ArtikelModel();
        $kategoriModel = new KategoriModel();

        // Ambil keyword pencarian & filter kategori
        $q = $this->request->getGet('q') ?? '';
        $kategori_id = $this->request->getGet('kategori_id') ?? '';

        // Bangun query
        $builder = $model
            ->select('artikel.*, kategori.nama_kategori')
            ->join('kategori', 'kategori.id_kategori = artikel.id_kategori', 'left');

        if (!empty($q)) {
            $builder->like('artikel.judul', $q);
        }

        if (!empty($kategori_id)) {
            $builder->where('artikel.id_kategori', $kategori_id);
        }

        $data = [
            'title' => $title,
            'artikel' => $builder->paginate(10),
            'pager' => $model->pager,
            'q' => $q,
            'kategori_id' => $kategori_id,
            'kategori' => $kategoriModel->findAll(),
        ];

        return view('artikel/admin_index', $data);
    }

    public function add()
    {
        $validation = \Config\Services::validation();

        $validation->setRules([
            'judul' => 'required',
            'isi' => 'required',
            'id_kategori' => 'required|integer',
            'gambar' => [
                'label' => 'Gambar',
                'rules' => 'uploaded[gambar]|is_image[gambar]|max_size[gambar,2048]',
                'errors' => [
                    'uploaded' => 'Gambar wajib diunggah.',
                    'is_image' => 'File harus berupa gambar.',
                    'max_size' => 'Ukuran gambar maksimal 2MB.',
                ],
            ],
        ]);

        if ($validation->withRequest($this->request)->run()) {
            $file = $this->request->getFile('gambar');
            $fileName = $file->getRandomName();
            $file->move(ROOTPATH . 'public/gambar', $fileName);

            $model = new ArtikelModel();
            $model->insert([
                'judul' => $this->request->getPost('judul'),
                'isi' => $this->request->getPost('isi'),
                'slug' => url_title($this->request->getPost('judul'), '-', true),
                'gambar' => $fileName,
                'status' => 1, // Ubah dari 0 (draft) ke 1 (public)
                'id_kategori' => $this->request->getPost('id_kategori'),
            ]);

            return redirect()->to(base_url('admin/artikel'));
        }

        $kategoriModel = new KategoriModel();

        // Cek apakah tabel kategori ada dan memiliki data
        try {
            $kategori = $kategoriModel->findAll();
            if (empty($kategori)) {
                // Jika tidak ada kategori, buat kategori default
                $kategoriModel->insertBatch([
                    ['nama_kategori' => 'Teknologi', 'slug_kategori' => 'teknologi'],
                    ['nama_kategori' => 'Olahraga', 'slug_kategori' => 'olahraga'],
                    ['nama_kategori' => 'Politik', 'slug_kategori' => 'politik'],
                    ['nama_kategori' => 'Ekonomi', 'slug_kategori' => 'ekonomi'],
                    ['nama_kategori' => 'Hiburan', 'slug_kategori' => 'hiburan'],
                ]);
                $kategori = $kategoriModel->findAll();
            }
        } catch (\Exception $e) {
            // Jika tabel kategori tidak ada, buat kategori default
            $kategori = [
                ['id_kategori' => 1, 'nama_kategori' => 'Umum', 'slug_kategori' => 'umum']
            ];
        }

        return view('artikel/form_add', [
            'title' => 'Tambah Artikel',
            'validation' => $validation,
            'kategori' => $kategori
        ]);
    }

    public function edit($id)
    {
        $model = new ArtikelModel();
        $validation = \Config\Services::validation();

        $validation->setRules([
            'judul' => 'required',
            'isi' => 'required',
            'id_kategori' => 'required|integer',
        ]);

        if ($validation->withRequest($this->request)->run()) {
            $model->update($id, [
                'judul' => $this->request->getPost('judul'),
                'isi' => $this->request->getPost('isi'),
                'id_kategori' => $this->request->getPost('id_kategori'),
            ]);

            return redirect()->to('admin/artikel');
        }

        $data = $model->find($id);
        if (!$data) {
            throw new \CodeIgniter\Exceptions\PageNotFoundException("Artikel dengan ID $id tidak ditemukan.");
        }

        $kategoriModel = new KategoriModel();

        // Cek apakah tabel kategori ada dan memiliki data
        try {
            $kategori = $kategoriModel->findAll();
            if (empty($kategori)) {
                $kategori = [
                    ['id_kategori' => 1, 'nama_kategori' => 'Umum', 'slug_kategori' => 'umum']
                ];
            }
        } catch (\Exception $e) {
            $kategori = [
                ['id_kategori' => 1, 'nama_kategori' => 'Umum', 'slug_kategori' => 'umum']
            ];
        }

        return view('artikel/form_edit', [
            'title' => 'Edit Artikel',
            'data' => $data,
            'kategori' => $kategori,
            'validation' => $validation
        ]);
    }

    public function delete($id)
    {
        $model = new ArtikelModel();
        $model->delete($id);

        return redirect()->to('admin/artikel');
    }

    public function view($slug)
    {
        $model = new ArtikelModel();
        $artikel = $model
            ->select('artikel.*, kategori.nama_kategori')
            ->join('kategori', 'kategori.id_kategori = artikel.id_kategori', 'left')
            ->where('slug', $slug)
            ->where('artikel.status', 1) // Hanya tampilkan artikel yang aktif
            ->first();

        if (!$artikel) {
            throw new \CodeIgniter\Exceptions\PageNotFoundException("Artikel dengan slug '$slug' tidak ditemukan atau belum dipublikasikan.");
        }

        return view('artikel/view', [
            'title' => $artikel['judul'],
            'artikel' => $artikel
        ]);
    }
}
```

## Memodifikasi View
Buka folder view/artikel sesuaikan masing-masing view.
- index.php
![{417A4590-0293-42DD-9C99-33CB3BF73483}](https://github.com/user-attachments/assets/9eb6e44d-4de3-4c2e-902c-d0a0da4ffe4c)

admin_index.php
```php
<?= $this->include('template/admin_header'); ?>

<h2 class="mt-4 mb-4"><?= esc($title); ?></h2>

<!-- Form Pencarian dan Filter -->
<form method="get" class="row g-2 mb-4" role="search">
    <div class="col-md-4">
        <input type="text" name="q" value="<?= esc($q); ?>" class="form-control" placeholder="Cari judul artikel">
    </div>
    <div class="col-md-4">
        <select name="kategori_id" class="form-select">
            <option value="">Semua Kategori</option>
            <?php foreach ($kategori as $k): ?>
                <option value="<?= esc($k['id_kategori']); ?>" <?= ($kategori_id == $k['id_kategori']) ? 'selected' : ''; ?>>
                    <?= esc($k['nama_kategori']); ?>
                </option>
            <?php endforeach; ?>
        </select>
    </div>
    <div class="col-md-4">
        <button type="submit" class="btn btn-primary w-100">Cari</button>
    </div>
</form>

<!-- Tabel Artikel -->
<div class="table-responsive">
    <table class="table table-bordered table-hover align-middle">
        <thead class="table-light text-center">
            <tr>
                <th>ID</th>
                <th>Gambar</th>
                <th>Judul</th>
                <th>Kategori</th>
                <th>Status</th>
                <th>Aksi</th>
            </tr>
        </thead>
        <tbody>
            <?php if (!empty($artikel)): ?>
                <?php foreach ($artikel as $item): ?>
                    <tr>
                        <td class="text-center"><?= esc($item['id']); ?></td>
                        <td class="text-center">
                            <?php if (!empty($item['gambar'])): ?>
                                <img src="<?= base_url('gambar/' . $item['gambar']); ?>" 
                                     alt="<?= esc($item['judul']); ?>" 
                                     class="img-thumbnail" 
                                     style="width: 80px; height: 60px; object-fit: cover;">
                            <?php else: ?>
                                <span class="text-muted">No Image</span>
                            <?php endif; ?>
                        </td>
                        <td>
                            <strong><?= esc($item['judul']); ?></strong>
                            <p><small><?= esc(substr($item['isi'], 0, 50)); ?>...</small></p>
                        </td>
                        <td class="text-center"><?= esc($item['nama_kategori']); ?></td>
                        <td class="text-center">
                            <span class="badge bg-<?= $item['status'] ? 'success' : 'secondary'; ?>">
                                <?= $item['status'] ? 'Aktif' : 'Draft'; ?>
                            </span>
                        </td>
                        <td class="text-center">
                            <a href="<?= base_url('admin/artikel/edit/' . $item['id']); ?>" class="btn btn-sm btn-warning">Ubah</a>
                            <a href="<?= base_url('admin/artikel/delete/' . $item['id']); ?>" 
                               class="btn btn-sm btn-danger" 
                               onclick="return confirm('Yakin ingin menghapus artikel ini?');">
                               Hapus
                            </a>
                        </td>
                    </tr>
                <?php endforeach; ?>
            <?php else: ?>
                <tr>
                    <td colspan="6" class="text-center">Data tidak ditemukan.</td>
                </tr>
            <?php endif; ?>
        </tbody>
    </table>
</div>

<!-- Pagination -->
<div class="d-flex justify-content-start">
    <?= $pager->only(['q', 'kategori_id'])->links(); ?>
</div>

<?= $this->include('template/admin_footer'); ?>
```

## form_add.php
![G_4](https://github.com/user-attachments/assets/0c82353c-923b-4489-8511-1cb31b2a8a4e)

## form_edit.php
![G_5](https://github.com/user-attachments/assets/5c638fe7-49b9-4c23-925e-56ca6027ce25)

##  Testing Fitur Relasi Artikel dan Kategori

Setelah seluruh konfigurasi selesai (tabel, model, controller, dan view), saatnya melakukan **pengujian** untuk memastikan bahwa relasi artikel ↔ kategori berjalan sebagaimana mestinya.
![G_6](https://github.com/user-attachments/assets/64d8b765-27b1-4117-9a7c-ed0288ab89fa)

##  Testing: Menambah Artikel Baru dengan Memilih Kategori

Fitur ini menguji apakah proses input artikel baru berhasil menyimpan data **termasuk kategori yang dipilih**, serta apakah relasi artikel ↔ kategori bekerja sesuai harapan.
![G_7](https://github.com/user-attachments/assets/ffc1fa51-bba9-40fe-b168-6a2834399548)

##  Testing: Mengedit Artikel dan Mengubah Kategorinya

Fitur ini menguji apakah admin dapat mengubah data artikel **termasuk mengganti kategori** yang telah dipilih sebelumnya.
![G_8](https://github.com/user-attachments/assets/77d0814b-b473-4756-a734-40a94b9d4c3a)

# Testing: Menghapus Artikel

Fitur ini menguji apakah admin dapat **menghapus artikel** yang sudah ada, dan memastikan data benar-benar terhapus dari database.
## Sebelum
![G_9](https://github.com/user-attachments/assets/4e7208a9-6d6c-4a01-bd48-1606c6efad1e)

## Sesudah
![G_10](https://github.com/user-attachments/assets/d9b7f730-f9d5-4133-be89-3088e8cdf332)


##  Membuat AJAX Controller

AJAX Controller digunakan untuk menangani permintaan data secara dinamis dari **frontend** (seperti Vue.js atau jQuery), **tanpa perlu me-refresh halaman**. Umumnya digunakan untuk proses **CRUD** melalui RESTful API.
yaitu Ajax.php :
![G_11](https://github.com/user-attachments/assets/68b99c1c-3189-4b1e-99c7-dffa4c5b5c91)

## Membuat View

Dalam arsitektur MVC CodeIgniter 4, **View** bertanggung jawab untuk menampilkan data yang dikirim oleh Controller ke pengguna. View dapat berupa HTML, dengan sedikit PHP untuk menampilkan variabel atau melakukan loop.
![G_12](https://github.com/user-attachments/assets/0cc263fa-3b92-41c8-9807-c0afb892b744)
![G_13](https://github.com/user-attachments/assets/c6a04446-9389-4c93-a396-30a26cacbeaa)

## Persiapan

- Pastikan MySQL Server sudah berjalan.  
- Buka database `lab_ci4`.  
- Pastikan tabel `artikel` dan `kategori` sudah ada dan terisi data.  
- Pastikan library jQuery sudah terpasang atau dapat diakses melalui Content Delivery Network.

##  Modifikasi Controller Artikel

Ubah method `admin_index()` di `Artikel.php` untuk mengembalikan data dalam format JSON jika request adalah AJAX. (Sama seperti modul sebelumnya)

setelah saya upgrade kodenya :

```php
public function admin_index()
    {
        $title = 'Daftar Artikel';
        $model = new ArtikelModel();
        $kategoriModel = new KategoriModel();

        // Ambil keyword pencarian & filter kategori
        $q = $this->request->getGet('q') ?? '';
        $kategori_id = $this->request->getGet('kategori_id') ?? '';

        // Bangun query
        $builder = $model
            ->select('artikel.*, kategori.nama_kategori')
            ->join('kategori', 'kategori.id_kategori = artikel.id_kategori', 'left');

        if (!empty($q)) {
            $builder->like('artikel.judul', $q);
        }

        if (!empty($kategori_id)) {
            $builder->where('artikel.id_kategori', $kategori_id);
        }

        $data = [
            'title' => $title,
            'artikel' => $builder->paginate(10),
            'pager' => $model->pager,
            'q' => $q,
            'kategori_id' => $kategori_id,
            'kategori' => $kategoriModel->findAll(),
        ];

        return view('artikel/admin_index', $data);
    }
```
## 🛠️ Modifikasi Controller Artikel

Penjelasan:
• `$page = $this->request->getVar('page') ?? 1;`: Mendapatkan nomor halaman dari request. Jika tidak ada, default ke halaman 1.  
• `$builder->paginate(10, 'default', $page);`: Menerapkan pagination dengan nomor halaman yang diberikan.  
• `$this->request->isAJAX()`: Memeriksa apakah request yang datang adalah AJAX.  
• Jika AJAX, kembalikan data artikel dan pager dalam format JSON.  
• Jika bukan AJAX, tampilkan view seperti biasa.


# Pertanyaan dan Tugas

Selesaikan semua langkah praktikum di atas.  
Modifikasi tampilan data artikel dan pagination sesuai kebutuhan desain.
![G_14](https://github.com/user-attachments/assets/9ee9602e-bf3f-4ff1-9825-d00361e8536a)

Implementasikan fitur sorting (mengurutkan artikel berdasarkan judul, dll.) dengan AJAX.  
- Berdasarkan Judul dan kategori :
![G_15](https://github.com/user-attachments/assets/3d616edc-f2c0-4ecf-993a-8cb1c78a44f1)

- Berdasarkan Judul:
![G_16](https://github.com/user-attachments/assets/eb935baf-c27c-4be9-ab0e-990a2ba4ac3c)

- Berdasarkan Kategori
![G_17](https://github.com/user-attachments/assets/7566dd25-8ea0-4e44-bf32-e38cda48f0f6)

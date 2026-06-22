# Dokumentasi Workflow Models DRIP Coffee

File ini menjelaskan bagian **Models**: fungsi setiap model, kenapa syntax-nya ditulis seperti itu, siapa yang memanggil model tersebut, dan hasilnya terhubung ke controller/view mana.

Dalam MVC CodeIgniter:

```text
View
    ↓ input / klik user
Controller
    ↓ panggil model
Model
    ↓ query database
Database
    ↓ hasil query
Model
    ↓ return data
Controller
    ↓ kirim data
View
```

Model tidak mengatur tampilan. Model hanya fokus ke **data dan query database**.

---

## 1. Daftar Model Dan Tanggung Jawabnya

| Model | File | Fungsi Utama | Dipakai Oleh |
|---|---|---|---|
| `MenuModel` | `application/models/MenuModel.php` | Ambil data menu untuk pelanggan | `Drip.php` |
| `OrderModel` | `application/models/OrderModel.php` | Buat order pelanggan, riwayat, order aktif | `Drip.php`, `OrderController.php` |
| `QueueModel` | `application/models/QueueModel.php` | Membuat nomor antrian | `OrderController.php` |
| `User_model` | `application/models/User_model.php` | Login, register, daftar user | `Auth.php`, `Admin.php` |
| `Product_model` | `application/models/Product_model.php` | CRUD produk/menu sisi admin | `Admin.php`, `Kasir.php` |
| `Order_model` | `application/models/Order_model.php` | Dashboard kasir, status order, rekap order | `Kasir.php`, `Admin.php` |
| `Transaction_model` | `application/models/Transaction_model.php` | Simpan transaksi pembayaran dan rekap transaksi | `Kasir.php`, `Admin.php` |

Catatan: ada dua model order, yaitu `OrderModel` dan `Order_model`.

`OrderModel` dipakai untuk alur pelanggan/AJAX order.

`Order_model` dipakai untuk alur kasir/admin.

---

## 2. Syntax Dasar Model CodeIgniter

### Kode Umum

```php
class Order_model extends CI_Model {
```

### Cara Kerja

Class model dibuat dengan `extends CI_Model`. Artinya model mewarisi class bawaan CodeIgniter supaya bisa memakai fitur framework, terutama database Query Builder melalui `$this->db`.

### Kenapa Ditulis Begitu

Kalau tidak `extends CI_Model`, model tidak otomatis punya akses ke `$this->db`, `$this->session`, dan fitur lain dari CodeIgniter.

---

## 3. Syntax Query Builder Yang Sering Dipakai

### `where()`

```php
$this->db->where('status', 'completed');
```

Artinya membuat kondisi SQL:

```sql
WHERE status = 'completed'
```

Kenapa dipakai: lebih aman dan lebih mudah dibaca daripada menulis SQL manual. CodeIgniter juga membantu escaping value untuk mengurangi risiko SQL Injection.

### `order_by()`

```php
$this->db->order_by('created_at', 'DESC');
```

Artinya mengurutkan data berdasarkan kolom `created_at` dari terbaru ke terlama.

Kenapa dipakai: data order, riwayat, dan rekap biasanya ingin ditampilkan dari yang terbaru.

### `get()`

```php
$this->db->get('orders');
```

Artinya menjalankan query SELECT ke tabel `orders`.

### `result_array()`

```php
return $this->db->get('orders')->result_array();
```

Dipakai jika hasil query bisa banyak baris.

Contoh: daftar order, daftar menu, daftar user.

### `row_array()`

```php
return $this->db->get('users')->row_array();
```

Dipakai jika hasil query hanya satu baris.

Contoh: cari user berdasarkan email, cari order berdasarkan ID.

### `insert()`

```php
$this->db->insert('users', $data);
```

Artinya menyimpan data baru ke tabel.

Kenapa `$data` berbentuk array: key array menjadi nama kolom, value array menjadi isi kolom.

### `update()`

```php
$this->db->where('id', $id);
$this->db->update('orders', ['status' => $status]);
```

Artinya mengubah data order tertentu.

Kenapa `where()` harus ada sebelum `update()`: agar hanya satu order yang berubah, bukan semua order.

### `delete()`

```php
$this->db->where('id', $id);
$this->db->delete('menu_items');
```

Artinya menghapus data produk tertentu.

Kenapa `where()` penting: tanpa `where`, semua data pada tabel bisa terhapus.

---

## 4. Workflow Menu Pelanggan

### Model

`MenuModel.php`

### Method Yang Terlibat

```php
public function getAllMenu()
```

### Dipanggil Oleh

`application/controllers/Drip.php`

```php
public function menu() {
    $data['menus'] = $this->MenuModel->getAllMenu();
    $this->load->view('drip/menu', $data);
}
```

### Alur

```text
User buka /menu
    ↓
Drip::menu()
    ↓
MenuModel::getAllMenu()
    ↓
SELECT * FROM menu_items ORDER BY category ASC, name ASC
    ↓
Data menu dikembalikan sebagai array
    ↓
Drip::menu() mengirim $data['menus'] ke drip/menu.php
```

### Kenapa Syntax-nya Begitu

```php
$this->db->order_by('category', 'ASC');
$this->db->order_by('name', 'ASC');
```

Menu diurutkan berdasarkan kategori dulu supaya tampil lebih terstruktur. Setelah itu diurutkan berdasarkan nama supaya dalam kategori yang sama tetap rapi.

```php
return $query->result_array();
```

Karena menu jumlahnya banyak, maka dipakai `result_array()`, bukan `row_array()`.

---

## 5. Workflow Checkout Pelanggan Sampai Order Masuk Database

### Model

`QueueModel.php` dan `OrderModel.php`

### Method Yang Terlibat

```php
QueueModel::getCurrentQueue()
OrderModel::createOrder($orderData)
```

### Dipanggil Oleh

`application/controllers/OrderController.php`

```php
$queue = $this->queueModel->getCurrentQueue();
$orderId = $this->orderModel->createOrder($orderData);
```

### Alur Dari Frontend Sampai Model

```text
layouts/footer.php
    ↓ tombol bayar onclick="confirmPayment()"
assets/js/script.js
    ↓ fetch POST JSON
OrderController::create()
    ↓ QueueModel::getCurrentQueue()
    ↓ OrderModel::createOrder($orderData)
Database: orders + order_items
    ↓ response JSON
script.js tampilkan nomor antrian
```

### Detail QueueModel

```php
$this->db->select('MAX(CAST(SUBSTRING(queue_number, 3) AS UNSIGNED)) as last_queue');
$this->db->where('DATE(created_at) = CURDATE()', null, false);
```

### Cara Kerja

`SUBSTRING(queue_number, 3)` mengambil angka setelah prefix `A-`. Contoh `A-10` menjadi `10`.

`CAST(... AS UNSIGNED)` mengubah string menjadi angka.

`MAX(...)` mencari nomor terbesar hari ini.

`CURDATE()` membatasi antrian hanya untuk hari ini.

### Kenapa Ditulis Begitu

Nomor antrian berbentuk teks `A-10`, bukan angka murni. Supaya bisa mencari angka terbesar, bagian angka harus dipotong dan diubah dulu menjadi integer.

Parameter `false` pada `where()` dipakai karena kondisi `DATE(created_at) = CURDATE()` adalah SQL mentah. Kalau diescape otomatis, fungsi SQL bisa tidak bekerja benar.

### Detail OrderModel::createOrder()

```php
$this->db->trans_start();
```

Transaction dipakai karena proses membuat order menyimpan data ke dua tabel:

```text
orders
order_items
```

Kalau `orders` berhasil insert tetapi `order_items` gagal, data akan rusak. Dengan transaction, query dianggap satu paket.

```php
$this->db->insert('orders', $data);
$orderId = $this->db->insert_id();
```

Order utama disimpan dulu. Setelah itu `insert_id()` mengambil ID order baru untuk dipakai sebagai foreign key di `order_items`.

```php
foreach ($orderData['items'] as $item) {
    $this->db->insert('order_items', $itemData);
}
```

`foreach` dipakai karena satu order bisa berisi banyak menu.

---

## 6. Workflow Riwayat Pesanan Pelanggan

### Model

`OrderModel.php`

### Method

```php
getOrderHistory()
```

### Dipanggil Oleh

`Drip::riwayat()` dan `OrderController::getHistory()`

### Alur

```text
User buka /riwayat
    ↓
Drip::riwayat()
    ↓ jika user login
OrderModel::getOrderHistory()
    ↓
Join orders + order_items + menu_items
    ↓
Data dikirim ke drip/riwayat.php
```

### Syntax Penting

```php
$this->db->select("o.*, GROUP_CONCAT(CONCAT(m.name, ' x', oi.quantity) SEPARATOR ' · ') as items");
```

### Cara Kerja

`GROUP_CONCAT()` menggabungkan banyak item order menjadi satu teks.

Contoh:

```text
Americano x1 · Croissant x2
```

### Kenapa Ditulis Begitu

Riwayat pesanan butuh ringkasan item. Kalau tidak memakai `GROUP_CONCAT()`, controller/view harus melakukan query tambahan untuk setiap order. Itu lebih rumit dan lebih berat.

```php
$this->db->where('1 = 0', null, false);
```

Jika user tidak login, model sengaja mengembalikan data kosong. Riwayat guest diambil dari `localStorage` oleh JavaScript, bukan dari database user.

---

## 7. Workflow Antrian Pelanggan

### Model

`OrderModel.php`

### Method

```php
getActiveOrder()
```

### Dipanggil Oleh

`Drip::antrian()` dan `OrderController::getActive()`

### Alur

```text
User buka /antrian
    ↓
Drip::antrian()
    ↓ kalau user login
OrderModel::getActiveOrder()
    ↓
SELECT order terbaru dengan status pending/processing/completed
    ↓
Data dikirim ke drip/antrian.php
```

### Syntax Penting

```php
$this->db->where_in('status', ['pending', 'processing', 'completed']);
$this->db->order_by('created_at', 'DESC');
$this->db->limit(1);
```

### Kenapa Ditulis Begitu

`where_in()` dipakai karena status aktif bukan cuma satu. Pesanan yang baru masuk, sedang diproses, atau selesai tetap relevan untuk halaman antrian.

`order_by DESC` dan `limit(1)` dipakai agar hanya order terbaru yang ditampilkan.

---

## 8. Workflow Login Dan Register User

### Model

`User_model.php`

### Method

```php
get_by_email($email)
create($data)
get_all()
```

### Dipanggil Oleh

`Auth.php` dan `Admin.php`

### Login

```text
auth/login.php
    ↓ form POST
Auth::do_login()
    ↓ User_model::get_by_email($email)
Database users
    ↓ data user
Auth::do_login()
    ↓ password_verify()
Session login dibuat
```

### Register

```text
auth/register.php
    ↓ form POST
Auth::do_register()
    ↓ validasi form
User_model::create($data)
    ↓
INSERT INTO users
```

### Syntax Penting

```php
$this->db->where('email', $email);
return $this->db->get('users')->row_array();
```

Email dipakai karena login menggunakan email. `row_array()` dipakai karena email harus unik, jadi hasilnya satu user.

---

## 9. Workflow Produk/Menu Sisi Admin

### Model

`Product_model.php`

### Method

```php
get_all()
get($id)
create($data)
update($id, $data)
delete($id)
```

### Dipanggil Oleh

`Admin.php`

### Alur Daftar Produk

```text
Admin buka /admin/products
    ↓
Admin::products()
    ↓ Product_model::get_all()
Database menu_items
    ↓
Data produk dikirim ke admin/products.php
```

### Alur Tambah Produk

```text
Form tambah produk
    ↓ POST
Admin::add_product()
    ↓ upload gambar
Product_model::create($data)
    ↓ INSERT menu_items
Redirect ke admin/products
```

### Alur Hapus Produk

```text
Admin klik hapus
    ↓
Admin::delete_product($id)
    ↓ Product_model::delete($id)
Database menu_items
    ↓ DELETE berdasarkan id
```

### Kenapa Ada `where()` Sebelum Update/Delete

```php
$this->db->where('id', $id);
$this->db->delete('menu_items');
```

`where()` menentukan baris mana yang akan dihapus. Tanpa `where()`, query delete bisa menghapus semua produk.

---

## 10. Workflow Dashboard Kasir

### Model

`Order_model.php`

### Dipanggil Oleh

`Kasir::index()`

### Method Yang Dipakai

```php
get_all_pending()
get_all_processing()
get_all_completed()
get_daily_completed_revenue($start_date, $end_date)
```

### Alur

```text
Kasir buka /kasir
    ↓
Kasir::index()
    ↓
Order_model::get_all_pending()
Order_model::get_all_processing()
Order_model::get_all_completed()
Order_model::get_daily_completed_revenue()
    ↓
Data dikirim ke kasir/dashboard.php
```

### Relasi Ke View

| Data Controller | Asal Model | Dipakai Di View |
|---|---|---|
| `$data['orders']` | `get_all_pending()` | Tab Pesanan Baru |
| `$data['processing_orders']` | `get_all_processing()` | Tab Pesanan Diproses |
| `$data['completed_orders']` | `get_all_completed()` | Tab Pesanan Selesai |
| `$data['daily_revenue']` | `get_daily_completed_revenue()` | Tab Rekap Pendapatan |

---

## 11. Workflow Proses Order Oleh Kasir

### Model

`Order_model.php`

### Dipanggil Oleh

`Kasir::proses($order_id)`

### Alur

```text
Kasir klik Proses
    ↓
Kasir::proses($order_id)
    ↓ Order_model::get($order_id)
    ↓ cek status pending
Order_model::update_status($order_id, 'processing')
    ↓
Redirect ke dashboard kasir
```

### Kenapa Dicek Dulu Statusnya

```php
if ($order && $order['status'] == 'pending') {
```

Kasir hanya boleh memproses order yang statusnya masih `pending`. Jika order sudah processing/completed, status tidak boleh diubah sembarangan.

---

## 12. Workflow Pembayaran Oleh Kasir

### Model

`Order_model.php` dan `Transaction_model.php`

### Dipanggil Oleh

`Kasir::bayar($order_id)`

### Alur

```text
Kasir klik Bayar
    ↓
Kasir::bayar($order_id)
    ↓ Order_model::get($order_id)
    ↓ cek status processing
Transaction_model::create($data)
    ↓ INSERT transactions
Order_model::update_status($order_id, 'completed')
    ↓ UPDATE orders
Redirect ke dashboard kasir
```

### Kenapa Transaksi Dipisah Dari Order

`orders` menyimpan data pesanan: nomor antrian, meja, total, status order.

`transactions` menyimpan data pembayaran: order mana yang dibayar, kasir siapa, metode pembayaran, status paid.

Pemisahan ini membuat data lebih jelas: order adalah pesanan, transaction adalah pembayaran.

---

## 13. Workflow Detail Order Dan Struk

### Model

`Order_model.php`

### Method

```php
get_with_items($id)
```

### Dipanggil Oleh

```php
Kasir::order_detail($order_id)
Kasir::print_struk($order_id)
```

### Alur Detail Order

```text
Kasir klik Detail
    ↓
Kasir::order_detail($order_id)
    ↓ Order_model::get_with_items($order_id)
    ↓ orders + order_items + menu_items
Data dikirim ke kasir/order_detail.php
```

### Alur Struk

```text
Kasir klik Print Struk
    ↓
Kasir::print_struk($order_id)
    ↓ Order_model::get_with_items($order_id)
Data dikirim ke kasir/struk.php
    ↓ window.print()
```

### Kenapa Pakai Join `menu_items`

`order_items` hanya menyimpan `menu_id`, `quantity`, dan `price`. Agar view bisa menampilkan nama menu, model perlu join ke `menu_items`.

---

## 14. Workflow Rekap Pendapatan Kasir

### Model

`Order_model.php`

### Method

```php
get_daily_completed_revenue($start_date, $end_date)
get_completed_by_date($date)
```

### Dipanggil Oleh

```php
Kasir::index()
Kasir::revenue_detail($date)
```

### Alur Rekap Harian

```text
Kasir buka tab Rekap Pendapatan
    ↓
Kasir::index()
    ↓ Order_model::get_daily_completed_revenue()
    ↓ group by DATE(updated_at)
Data dikirim ke kasir/dashboard.php
```

### Alur Detail Tanggal

```text
Kasir klik tanggal rekap
    ↓
Kasir::revenue_detail($date)
    ↓ Order_model::get_completed_by_date($date)
Data dikirim ke kasir/revenue_detail.php
```

### Kenapa Pakai `updated_at`

Pendapatan dihitung saat order selesai/dibayar, bukan saat order dibuat. Karena itu tanggal yang dipakai adalah `updated_at`, yaitu waktu status order berubah menjadi completed.

---

## 15. Workflow Dashboard Admin

### Model

`Order_model.php` dan `Transaction_model.php`

### Dipanggil Oleh

`Admin::index()`

### Method Yang Dipakai

```php
Transaction_model::get_summary($month_start, $month_end)
Order_model::get_completed_revenue_by_date($selected_date)
Order_model::get_completed_by_date($selected_date)
Transaction_model::get_daily_summary($selected_date)
```

### Alur

```text
Admin buka /admin
    ↓
Admin::index()
    ↓
Transaction_model dan Order_model mengambil data pendapatan/order
    ↓
Data dikirim ke admin/dashboard.php
```

### Relasi Ke View

| Data Controller | Asal Model | Dipakai Di View |
|---|---|---|
| `$data['total_income']` | `Transaction_model::get_summary()` | Total income bulanan |
| `$data['selected_income']` | `Order_model::get_completed_revenue_by_date()` | Pendapatan tanggal terpilih |
| `$data['selected_orders']` | `Order_model::get_completed_by_date()` | Daftar order tanggal terpilih |
| `$data['daily_data']` | `Transaction_model::get_daily_summary()` | Data chart per jam |

---

## 16. Workflow Orders Recap Admin

### Model

`Order_model.php`

### Dipanggil Oleh

`Admin::orders_recap()`

### Method

```php
get_all_completed()
sum_revenue()
```

### Alur

```text
Admin buka /admin/orders_recap
    ↓
Admin::orders_recap()
    ↓
Order_model::get_all_completed()
Order_model::sum_revenue()
    ↓
Data dikirim ke admin/orders_recap.php
```

### Kenapa Hanya Completed

Rekap pendapatan tidak boleh menghitung order yang masih pending atau processing, karena belum benar-benar selesai/dibayar.

---

## 17. Workflow Transaction_model Untuk Rekap

### Method

```php
get_summary($start_date, $end_date)
get_daily_summary($date)
```

### Syntax Penting

```php
$this->db->select_sum('total');
$this->db->where('status', 'paid');
```

### Cara Kerja

`select_sum('total')` menjumlahkan total transaksi. `where('status', 'paid')` memastikan hanya transaksi yang sudah dibayar yang dihitung.

### Kenapa Ditulis Begitu

Admin dashboard membutuhkan angka total pendapatan, bukan daftar transaksi satu per satu. Jadi query agregasi lebih tepat daripada mengambil semua data lalu menjumlahkan di PHP.

---

## 18. Peta Pemanggilan Model

### Auth.php

```text
Auth::__construct()
    ↓ load User_model

Auth::do_login()
    ↓ User_model::get_by_email()

Auth::do_register()
    ↓ User_model::create()
```

### Drip.php

```text
Drip::__construct()
    ↓ load MenuModel dan OrderModel

Drip::menu()
    ↓ MenuModel::getAllMenu()

Drip::antrian()
    ↓ OrderModel::getActiveOrder()

Drip::riwayat()
    ↓ OrderModel::getOrderHistory()
```

### OrderController.php

```text
OrderController::__construct()
    ↓ load OrderModel sebagai orderModel
    ↓ load QueueModel sebagai queueModel

OrderController::create()
    ↓ QueueModel::getCurrentQueue()
    ↓ OrderModel::createOrder()

OrderController::getHistory()
    ↓ OrderModel::getOrderHistory()

OrderController::getActive()
    ↓ OrderModel::getActiveOrder()
```

### Kasir.php

```text
Kasir::__construct()
    ↓ load Product_model, Order_model, Transaction_model

Kasir::index()
    ↓ Order_model::get_all_pending()
    ↓ Order_model::get_all_processing()
    ↓ Order_model::get_all_completed()
    ↓ Order_model::get_daily_completed_revenue()

Kasir::revenue_detail()
    ↓ Order_model::get_completed_by_date()

Kasir::order_detail()
    ↓ Order_model::get_with_items()

Kasir::proses()
    ↓ Order_model::get()
    ↓ Order_model::update_status()

Kasir::print_struk()
    ↓ Order_model::get_with_items()

Kasir::bayar()
    ↓ Order_model::get()
    ↓ Transaction_model::create()
    ↓ Order_model::update_status()
```

### Admin.php

```text
Admin::__construct()
    ↓ load Product_model, Transaction_model, User_model, Order_model

Admin::index()
    ↓ Transaction_model::get_summary()
    ↓ Order_model::get_completed_revenue_by_date()
    ↓ Order_model::get_completed_by_date()
    ↓ Transaction_model::get_daily_summary()

Admin::orders_recap()
    ↓ Order_model::get_all_completed()
    ↓ Order_model::sum_revenue()

Admin::products()
    ↓ Product_model::get_all()

Admin::add_product()
    ↓ Product_model::create()

Admin::delete_product()
    ↓ Product_model::delete()

Admin::users()
    ↓ User_model::get_all()
```

---

## 19. Jawaban Singkat Kalau Ditanya Saat Presentasi

Kalau ditanya:

"Model itu fungsinya apa dan terhubung ke mana?"

Jawab:

"Model adalah layer yang berhubungan langsung dengan database. Controller memanggil model untuk mengambil, menyimpan, mengubah, atau menghapus data. Misalnya saat user checkout, JavaScript mengirim data ke `OrderController::create()`, lalu controller memanggil `QueueModel::getCurrentQueue()` untuk membuat nomor antrian dan `OrderModel::createOrder()` untuk menyimpan order ke tabel `orders` dan `order_items`. Setelah itu data dikembalikan ke controller lalu ke JavaScript sebagai JSON. Untuk kasir, `Kasir.php` memakai `Order_model` untuk mengambil pesanan pending, processing, completed, dan mengubah status order. Untuk admin, `Admin.php` memakai `Order_model` dan `Transaction_model` untuk rekap pendapatan. Jadi model tidak menampilkan halaman, tapi menyediakan data yang nanti dikirim controller ke view."

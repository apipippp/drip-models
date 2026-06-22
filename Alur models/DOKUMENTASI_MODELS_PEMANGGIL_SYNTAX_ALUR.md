# Dokumentasi Models: Pemanggil, Syntax, Alur, Dan Alasan

File ini menjelaskan **Models dipakai oleh siapa saja**, syntax lengkap dari controller yang memanggil model, alur datanya, dan alasan kenapa syntax di model ditulis seperti itu.

Tujuan file ini adalah membantu menjawab pertanyaan presentasi seperti:

"Model ini dipanggil dari mana?"

"Kenapa pakai `result_array()` bukan `row_array()`?"

"Alurnya dari controller ke model lalu database bagaimana?"

"Kenapa query-nya pakai `where()`, `join()`, `group_by()`, atau `transaction`?"

---

## 1. Gambaran Umum Cara Controller Memanggil Model

### Syntax Load Model Di Controller

```php
$this->load->model('Order_model');
```

### Cara Kerja

Syntax ini menyuruh CodeIgniter memuat file:

```text
application/models/Order_model.php
```

Setelah dimuat, model bisa dipakai dengan:

```php
$this->Order_model->nama_method();
```

### Kenapa Ditulis Begitu

CodeIgniter 3 menggunakan loader `$this->load->model()` agar controller tidak perlu melakukan `include` atau `require` manual. Framework akan mencari file model berdasarkan nama yang diberikan.

---

## 2. Cara Model Mengakses Database

### Syntax Dasar Model

```php
class Order_model extends CI_Model {
```

### Cara Kerja

`extends CI_Model` artinya class model mewarisi fitur bawaan CodeIgniter, termasuk database Query Builder melalui:

```php
$this->db
```

### Kenapa Ditulis Begitu

Kalau model tidak `extends CI_Model`, maka `$this->db` tidak otomatis tersedia. Akibatnya model tidak bisa memakai syntax seperti:

```php
$this->db->where();
$this->db->get();
$this->db->insert();
```

---

## 3. Peta Semua Model Dan Pemanggilnya

| Model | Dipanggil Oleh | Method Controller | Method Model Yang Dipakai |
|---|---|---|---|
| `MenuModel` | `Drip.php` | `menu()` | `getAllMenu()` |
| `OrderModel` | `Drip.php` | `antrian()`, `riwayat()` | `getActiveOrder()`, `getOrderHistory()` |
| `OrderModel` | `OrderController.php` | `create()`, `getHistory()`, `getActive()` | `createOrder()`, `getOrderHistory()`, `getActiveOrder()` |
| `QueueModel` | `OrderController.php` | `create()` | `getCurrentQueue()` |
| `User_model` | `Auth.php` | `do_login()`, `do_register()` | `get_by_email()`, `create()` |
| `User_model` | `Admin.php` | `users()` | `get_all()` |
| `Product_model` | `Admin.php` | `products()`, `add_product()`, `delete_product()` | `get_all()`, `create()`, `delete()` |
| `Product_model` | `Kasir.php` | `__construct()` | hanya diload, belum aktif dipakai di method utama |
| `Order_model` | `Kasir.php` | `index()`, `proses()`, `bayar()`, `order_detail()`, `print_struk()`, `revenue_detail()` | banyak method order kasir |
| `Order_model` | `Admin.php` | `index()`, `orders_recap()` | method rekap dan completed order |
| `Transaction_model` | `Kasir.php` | `bayar()` | `create()` |
| `Transaction_model` | `Admin.php` | `index()` | `get_summary()`, `get_daily_summary()` |

---

## 4. MenuModel Dipanggil Oleh Drip::menu()

### File Controller

`application/controllers/Drip.php`

### Syntax Load Model

```php
public function __construct() {
    parent::__construct();
    $this->load->model('MenuModel');
    $this->load->model('OrderModel');
}
```

### Syntax Pemanggilan Model

```php
public function menu() {
    $data['menus'] = $this->MenuModel->getAllMenu();
    $data['title'] = 'Menu - DRIP* Coffee';

    $this->load->view('layouts/header', $data);
    $this->load->view('drip/menu', $data);
    $this->load->view('layouts/footer');
}
```

### Syntax Di Model

`application/models/MenuModel.php`

```php
public function getAllMenu()
{
    $this->db->order_by('category', 'ASC');
    $this->db->order_by('name', 'ASC');
    $query = $this->db->get('menu_items');
    return $query->result_array();
}
```

### Alur Lengkap

```text
User buka halaman /menu
    ↓
Route mengarah ke Drip::menu()
    ↓
Drip::menu() memanggil MenuModel::getAllMenu()
    ↓
MenuModel query tabel menu_items
    ↓
Database mengembalikan semua menu
    ↓
Model return data dalam bentuk array
    ↓
Controller memasukkan ke $data['menus']
    ↓
View drip/menu.php menerima $menus
```

### Kenapa Syntax Model Seperti Itu

```php
$this->db->order_by('category', 'ASC');
```

Dipakai agar menu terurut berdasarkan kategori. Ini membuat data lebih rapi saat diterima controller atau view.

```php
$this->db->order_by('name', 'ASC');
```

Dipakai sebagai pengurutan kedua. Kalau ada banyak menu dalam kategori yang sama, namanya tetap urut A-Z.

```php
$query = $this->db->get('menu_items');
```

`get('menu_items')` menjalankan query SELECT ke tabel `menu_items`. Query Builder dipakai supaya tidak perlu menulis SQL manual.

```php
return $query->result_array();
```

`result_array()` dipakai karena hasil menu bisa lebih dari satu baris. Kalau memakai `row_array()`, hanya satu menu pertama yang akan diambil.

---

## 5. OrderModel::getActiveOrder() Dipanggil Oleh Drip::antrian()

### File Controller

`application/controllers/Drip.php`

### Syntax Pemanggilan Model

```php
public function antrian() {
    $user_id = $this->session->userdata('user_id');
    $data['active_order'] = null;

    if ($user_id) {
        $data['active_order'] = $this->OrderModel->getActiveOrder();
    } else {
        $session_order = $this->session->userdata('current_order');
        if ($session_order) {
            $this->db->where('id', $session_order['id']);
            $data['active_order'] = $this->db->get('orders')->row_array();
        }
    }

    $data['title'] = 'Antrian - DRIP* Coffee';
    $this->load->view('layouts/header', $data);
    $this->load->view('drip/antrian', $data);
    $this->load->view('layouts/footer');
}
```

### Syntax Di Model

`application/models/OrderModel.php`

```php
public function getActiveOrder()
{
    $userId = $this->session->userdata('user_id');

    if ($userId) {
        $this->db->where('user_id', $userId);
    } else {
        return null;
    }

    $this->db->where_in('status', ['pending', 'processing', 'completed']);
    $this->db->order_by('created_at', 'DESC');
    $this->db->limit(1);
    $query = $this->db->get('orders');
    return $query->row_array();
}
```

### Alur Lengkap

```text
User login membuka /antrian
    ↓
Drip::antrian() mengecek session user_id
    ↓
Jika user_id ada, controller memanggil OrderModel::getActiveOrder()
    ↓
Model mengambil user_id dari session
    ↓
Model mencari order milik user dengan status pending/processing/completed
    ↓
Model mengurutkan dari order terbaru
    ↓
Model mengambil satu order saja
    ↓
Data dikirim ke drip/antrian.php sebagai $active_order
```

### Kenapa Syntax Model Seperti Itu

```php
$userId = $this->session->userdata('user_id');
```

Dipakai agar model tahu user mana yang sedang login. Order aktif harus milik user tersebut, bukan order orang lain.

```php
$this->db->where('user_id', $userId);
```

Dipakai supaya query hanya mengambil order berdasarkan user login.

```php
$this->db->where_in('status', ['pending', 'processing', 'completed']);
```

`where_in()` dipakai karena status aktif ada beberapa kemungkinan. Kalau memakai `where('status', 'pending')`, order yang sedang diproses tidak akan tampil.

```php
$this->db->order_by('created_at', 'DESC');
$this->db->limit(1);
```

Dipakai agar hanya order terbaru yang ditampilkan. Halaman antrian tidak perlu menampilkan semua order.

```php
return $query->row_array();
```

Dipakai karena hasil yang dibutuhkan hanya satu order aktif terbaru.

---

## 6. OrderModel::getOrderHistory() Dipanggil Oleh Drip::riwayat()

### File Controller

`application/controllers/Drip.php`

### Syntax Pemanggilan Model

```php
public function riwayat() {
    $user_id = $this->session->userdata('user_id');
    $data['orders'] = [];

    if ($user_id) {
        $data['orders'] = $this->OrderModel->getOrderHistory();
    }

    $data['title'] = 'Riwayat - DRIP* Coffee';
    $this->load->view('layouts/header', $data);
    $this->load->view('drip/riwayat', $data);
    $this->load->view('layouts/footer');
}
```

### Syntax Di Model

```php
public function getOrderHistory()
{
    $userId = $this->session->userdata('user_id');

    $this->db->select("o.*, GROUP_CONCAT(CONCAT(m.name, ' x', oi.quantity) SEPARATOR ' · ') as items");
    $this->db->from('orders o');
    $this->db->join('order_items oi', 'o.id = oi.order_id', 'left');
    $this->db->join('menu_items m', 'oi.menu_id = m.id', 'left');

    if ($userId) {
        $this->db->where('o.user_id', $userId);
    } else {
        $this->db->where('1 = 0', null, false);
    }

    $this->db->group_by('o.id');
    $this->db->order_by('o.created_at', 'DESC');
    $this->db->limit(50);
    return $this->db->get()->result_array();
}
```

### Alur Lengkap

```text
User login membuka /riwayat
    ↓
Drip::riwayat() cek session user_id
    ↓
Controller memanggil OrderModel::getOrderHistory()
    ↓
Model join tabel orders, order_items, menu_items
    ↓
Model menggabungkan nama item menjadi ringkasan teks
    ↓
Model filter berdasarkan user_id login
    ↓
Model return maksimal 50 riwayat terbaru
    ↓
Controller mengirim $orders ke drip/riwayat.php
```

### Kenapa Syntax Model Seperti Itu

```php
$this->db->select("o.*, GROUP_CONCAT(CONCAT(m.name, ' x', oi.quantity) SEPARATOR ' · ') as items");
```

Syntax ini dipakai untuk mengambil data order sekaligus membuat ringkasan item. Contohnya menjadi:

```text
Americano x2 · Croissant x1
```

Tanpa `GROUP_CONCAT`, view harus melakukan looping dan query tambahan untuk setiap order.

```php
$this->db->from('orders o');
```

`orders` diberi alias `o` supaya query lebih pendek. Daripada menulis `orders.id`, cukup menulis `o.id`.

```php
$this->db->join('order_items oi', 'o.id = oi.order_id', 'left');
```

Join ini menghubungkan order utama dengan item-itemnya.

```php
$this->db->join('menu_items m', 'oi.menu_id = m.id', 'left');
```

Join ini dipakai agar model bisa mengambil nama menu dari `menu_items`.

```php
$this->db->where('1 = 0', null, false);
```

Ini sengaja membuat hasil query kosong untuk guest. Guest history ditangani JavaScript lewat `localStorage`, bukan database berdasarkan `user_id`.

---

## 7. QueueModel::getCurrentQueue() Dipanggil Oleh OrderController::create()

### File Controller

`application/controllers/OrderController.php`

### Syntax Pemanggilan Model

```php
public function __construct()
{
    parent::__construct();
    $this->load->model('OrderModel', 'orderModel');
    $this->load->model('QueueModel', 'queueModel');
}

public function create()
{
    $data = json_decode(file_get_contents('php://input'), true);

    $queue = $this->queueModel->getCurrentQueue();

    $orderData = [
        'user_id' => $this->session->userdata('user_id'),
        'queue' => $queue,
        'table' => $data['table'],
        'subtotal' => $data['subtotal'],
        'tax' => $data['tax'],
        'total' => $data['total'],
        'payment' => $data['payment'],
        'items' => $data['items']
    ];

    $orderId = $this->orderModel->createOrder($orderData);
}
```

### Syntax Di Model

```php
public function getCurrentQueue()
{
    $this->db->select('MAX(CAST(SUBSTRING(queue_number, 3) AS UNSIGNED)) as last_queue');
    $this->db->where('DATE(created_at) = CURDATE()', null, false);
    $query = $this->db->get('orders');
    $result = $query->row_array();
    $nextNum = ($result['last_queue'] ?? 9) + 1;
    return "A-" . $nextNum;
}
```

### Alur Lengkap

```text
User checkout dari frontend
    ↓
script.js mengirim data order lewat fetch
    ↓
OrderController::create() menerima JSON
    ↓
Controller memanggil QueueModel::getCurrentQueue()
    ↓
Model mencari nomor antrian terbesar hari ini
    ↓
Model menambah 1 nomor baru
    ↓
Controller menerima queue, misalnya A-10
    ↓
Queue dipakai untuk OrderModel::createOrder()
```

### Kenapa Syntax Model Seperti Itu

```php
SUBSTRING(queue_number, 3)
```

Nomor antrian berbentuk `A-10`. Bagian angka dimulai dari karakter ke-3, jadi `SUBSTRING` mengambil `10`.

```php
CAST(... AS UNSIGNED)
```

Hasil substring masih berupa teks. Supaya bisa dibandingkan secara angka, teks harus diubah menjadi integer.

```php
MAX(...)
```

Dipakai untuk mencari nomor antrian terbesar hari ini.

```php
$this->db->where('DATE(created_at) = CURDATE()', null, false);
```

Dipakai agar nomor antrian dihitung ulang per hari. `false` berarti CodeIgniter tidak melakukan escaping karena query ini memakai fungsi SQL mentah.

```php
$nextNum = ($result['last_queue'] ?? 9) + 1;
```

Jika belum ada order hari ini, `last_queue` null. Maka default `9 + 1` membuat nomor pertama menjadi `A-10`.

---

## 8. OrderModel::createOrder() Dipanggil Oleh OrderController::create()

### File Controller

`application/controllers/OrderController.php`

### Syntax Pemanggilan Model

```php
$orderId = $this->orderModel->createOrder($orderData);
```

### Syntax Di Model

```php
public function createOrder($orderData)
{
    $this->db->trans_start();

    $data = [
        'user_id' => $orderData['user_id'] ?? null,
        'queue_number' => $orderData['queue'],
        'table_number' => $orderData['table'],
        'subtotal' => $orderData['subtotal'],
        'tax' => $orderData['tax'],
        'total' => $orderData['total'],
        'payment_method' => $orderData['payment'],
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s')
    ];

    $this->db->insert('orders', $data);
    $orderId = $this->db->insert_id();

    foreach ($orderData['items'] as $item) {
        $itemData = [
            'order_id' => $orderId,
            'menu_id' => $item['id'],
            'quantity' => $item['qty'],
            'price' => $item['price']
        ];
        $this->db->insert('order_items', $itemData);
    }

    $this->db->trans_complete();

    if ($this->db->trans_status() === FALSE) {
        return false;
    }

    return $orderId;
}
```

### Alur Lengkap

```text
OrderController::create()
    ↓
Membuat array $orderData dari JSON frontend
    ↓
Memanggil OrderModel::createOrder($orderData)
    ↓
Model mulai database transaction
    ↓
Insert data utama ke tabel orders
    ↓
Ambil id order baru dengan insert_id()
    ↓
Loop semua item dari cart
    ↓
Insert setiap item ke tabel order_items
    ↓
Transaction selesai
    ↓
Jika sukses return orderId
    ↓
Controller kirim JSON success ke frontend
```

### Kenapa Syntax Model Seperti Itu

```php
$this->db->trans_start();
```

Dipakai karena proses checkout menyimpan data ke dua tabel. Jika salah satu gagal, semua harus dibatalkan agar data tidak setengah masuk.

```php
'user_id' => $orderData['user_id'] ?? null,
```

`?? null` dipakai karena guest user tidak punya `user_id`. Dengan syntax ini, order guest tetap bisa masuk database tanpa error undefined index.

```php
'status' => 'pending',
```

Order baru selalu dimulai dari status `pending` karena belum diproses kasir.

```php
'created_at' => date('Y-m-d H:i:s')
```

Format ini sesuai format `DATETIME` MySQL.

```php
$orderId = $this->db->insert_id();
```

Setelah insert ke `orders`, model perlu ID order baru untuk menghubungkan item di tabel `order_items`.

```php
foreach ($orderData['items'] as $item)
```

Satu order bisa berisi banyak item. Karena itu perlu loop.

---

## 9. User_model::get_by_email() Dipanggil Oleh Auth::do_login()

### File Controller

`application/controllers/Auth.php`

### Syntax Pemanggilan Model

```php
public function __construct() {
    parent::__construct();
    $this->load->model('User_model');
    $this->load->library('form_validation');
}

public function do_login() {
    $email = $this->input->post('email');
    $password = $this->input->post('password');
    $user = $this->User_model->get_by_email($email);

    if (!$user) {
        $this->session->set_flashdata('error', 'User tidak ditemukan dengan email tersebut');
        redirect('auth/login');
        return;
    }

    if (password_verify($password, $user['password'])) {
        $this->session->set_userdata([
            'user_id' => $user['id'],
            'name' => $user['name'],
            'role' => $user['role'],
            'email' => $user['email']
        ]);
        redirect('dashboard');
    }
}
```

### Syntax Di Model

```php
public function get_by_email($email) {
    $this->db->where('email', $email);
    return $this->db->get('users')->row_array();
}
```

### Alur Lengkap

```text
User submit form login
    ↓
Auth::do_login()
    ↓
Controller mengambil email dan password dari POST
    ↓
Controller memanggil User_model::get_by_email($email)
    ↓
Model mencari user berdasarkan email
    ↓
Model return satu data user
    ↓
Controller menjalankan password_verify()
    ↓
Jika benar, controller membuat session login
```

### Kenapa Syntax Model Seperti Itu

```php
$this->db->where('email', $email);
```

Email dipakai karena form login menggunakan email sebagai identitas. Query Builder juga membantu escaping value email.

```php
return $this->db->get('users')->row_array();
```

`row_array()` dipakai karena satu email seharusnya hanya punya satu user. Kalau pakai `result_array()`, controller harus mengambil index `[0]` lagi.

---

## 10. User_model::create() Dipanggil Oleh Auth::do_register()

### File Controller

`application/controllers/Auth.php`

### Syntax Pemanggilan Model

```php
public function do_register() {
    $this->form_validation->set_rules('name', 'Nama', 'required');
    $this->form_validation->set_rules('email', 'Email', 'required|valid_email|is_unique[users.email]');
    $this->form_validation->set_rules('password', 'Password', 'required|min_length[6]');

    if ($this->form_validation->run() == FALSE) {
        $this->load->view('auth/register');
    } else {
        $data = [
            'name' => $this->input->post('name'),
            'email' => $this->input->post('email'),
            'password' => password_hash($this->input->post('password'), PASSWORD_DEFAULT),
            'role' => $this->input->post('role') ?: 'user'
        ];
        $this->User_model->create($data);
        redirect('auth/login');
    }
}
```

### Syntax Di Model

```php
public function create($data) {
    return $this->db->insert('users', $data);
}
```

### Alur Lengkap

```text
User submit form register
    ↓
Auth::do_register()
    ↓
Controller validasi input
    ↓
Controller hash password
    ↓
Controller memanggil User_model::create($data)
    ↓
Model insert data ke tabel users
    ↓
Controller redirect ke halaman login
```

### Kenapa Syntax Model Seperti Itu

```php
return $this->db->insert('users', $data);
```

`insert()` dipakai karena register membuat data user baru. `$data` berbentuk array supaya key seperti `name`, `email`, `password`, dan `role` otomatis dipetakan ke kolom database.

---

## 11. Order_model Dipanggil Oleh Kasir::index()

### File Controller

`application/controllers/Kasir.php`

### Syntax Load Model

```php
public function __construct() {
    parent::__construct();
    if (!$this->session->userdata('user_id') || $this->session->userdata('role') != 'kasir') {
        redirect('auth/login');
    }
    $this->load->model('Product_model');
    $this->load->model('Order_model');
    $this->load->model('Transaction_model');
}
```

### Syntax Pemanggilan Model

```php
public function index() {
    $start_date = $this->input->get('start_date');
    $end_date = $this->input->get('end_date');
    $tab = $this->input->get('tab');

    $data['orders'] = $this->Order_model->get_all_pending();
    $data['processing_orders'] = $this->Order_model->get_all_processing();
    $data['completed_orders'] = $this->Order_model->get_all_completed();
    $data['daily_revenue'] = $this->Order_model->get_daily_completed_revenue($start_date, $end_date);

    $data['start_date'] = $start_date;
    $data['end_date'] = $end_date;
    $data['active_tab'] = $tab ?: ($start_date || $end_date ? 'revenue' : 'pending');

    $this->load->view('templates/header_kasir');
    $this->load->view('kasir/dashboard', $data);
    $this->load->view('templates/footer_kasir');
}
```

### Syntax Di Model

```php
public function get_all_pending() {
    $this->db->where('status', 'pending');
    $this->db->order_by('queue_number', 'ASC');
    return $this->db->get('orders')->result_array();
}

public function get_all_processing() {
    $this->db->where('status', 'processing');
    $this->db->order_by('queue_number', 'ASC');
    return $this->db->get('orders')->result_array();
}

public function get_all_completed() {
    $this->db->where('status', 'completed');
    $this->db->order_by('updated_at', 'DESC');
    return $this->db->get('orders')->result_array();
}
```

### Alur Lengkap

```text
Kasir login membuka /kasir
    ↓
Kasir::index()
    ↓
Controller mengambil filter tanggal dari URL GET
    ↓
Controller memanggil beberapa method Order_model
    ↓
Model mengambil order pending, processing, completed, dan rekap revenue
    ↓
Data dikirim ke kasir/dashboard.php
    ↓
View menampilkan tab Pesanan Baru, Diproses, Selesai, dan Rekap Pendapatan
```

### Kenapa Syntax Model Seperti Itu

```php
$this->db->where('status', 'pending');
```

Dipakai untuk memisahkan order berdasarkan status. Dashboard kasir memiliki tab berbeda untuk status berbeda.

```php
$this->db->order_by('queue_number', 'ASC');
```

Pending dan processing diurutkan berdasarkan nomor antrian supaya kasir memproses sesuai urutan.

```php
$this->db->order_by('updated_at', 'DESC');
```

Completed diurutkan berdasarkan waktu selesai terbaru, karena completed lebih cocok dilihat dari waktu selesai/dibayar.

---

## 12. Order_model::update_status() Dipanggil Oleh Kasir::proses()

### File Controller

`application/controllers/Kasir.php`

### Syntax Pemanggilan Model

```php
public function proses($order_id) {
    $order = $this->Order_model->get($order_id);
    if ($order && $order['status'] == 'pending') {
        $this->Order_model->update_status($order_id, 'processing');
    }
    redirect('kasir');
}
```

### Syntax Di Model

```php
public function get($id) {
    return $this->db->get_where('orders', ['id' => $id])->row_array();
}

public function update_status($id, $status) {
    $this->db->where('id', $id);
    return $this->db->update('orders', ['status' => $status]);
}
```

### Alur Lengkap

```text
Kasir klik tombol Proses
    ↓
URL membawa order_id ke Kasir::proses($order_id)
    ↓
Controller memanggil Order_model::get($order_id)
    ↓
Model mengambil order berdasarkan id
    ↓
Controller cek apakah status masih pending
    ↓
Jika pending, controller memanggil update_status()
    ↓
Model update status menjadi processing
    ↓
Controller redirect ke /kasir
```

### Kenapa Syntax Model Seperti Itu

```php
return $this->db->get_where('orders', ['id' => $id])->row_array();
```

`get_where()` dipakai karena query-nya sederhana: ambil satu order berdasarkan id. `row_array()` dipakai karena id order unik.

```php
$this->db->where('id', $id);
```

Wajib ada sebelum update agar hanya order tersebut yang berubah.

```php
return $this->db->update('orders', ['status' => $status]);
```

`update()` dipakai untuk mengubah status order, bukan membuat data baru.

---

## 13. Transaction_model::create() Dipanggil Oleh Kasir::bayar()

### File Controller

`application/controllers/Kasir.php`

### Syntax Pemanggilan Model

```php
public function bayar($order_id) {
    $order = $this->Order_model->get($order_id);
    if ($order && $order['status'] == 'processing') {
        $data = [
            'order_id' => $order_id,
            'kasir_id' => $this->session->userdata('user_id'),
            'total' => $order['total'],
            'payment_method' => $order['payment_method'] ?: 'cash',
            'status' => 'paid'
        ];
        $this->Transaction_model->create($data);
        $this->Order_model->update_status($order_id, 'completed');
    }
    redirect('kasir');
}
```

### Syntax Di Model

`application/models/Transaction_model.php`

```php
public function create($data) {
    return $this->db->insert('transactions', $data);
}
```

### Alur Lengkap

```text
Kasir klik Bayar
    ↓
Kasir::bayar($order_id)
    ↓
Controller ambil order dengan Order_model::get()
    ↓
Controller cek status harus processing
    ↓
Controller membuat array data transaksi
    ↓
Controller memanggil Transaction_model::create($data)
    ↓
Model insert ke tabel transactions
    ↓
Controller memanggil Order_model::update_status(..., 'completed')
    ↓
Order berpindah ke tab Pesanan Selesai
```

### Kenapa Syntax Model Seperti Itu

```php
return $this->db->insert('transactions', $data);
```

Transaksi adalah data baru setiap pembayaran terjadi, jadi digunakan `insert()`. Data transaksi dipisah dari order agar sistem tahu kapan order dibayar, kasir siapa yang memproses, dan metode pembayarannya.

---

## 14. Order_model::get_with_items() Dipanggil Oleh Kasir::order_detail() Dan Kasir::print_struk()

### File Controller

`application/controllers/Kasir.php`

### Syntax Pemanggilan Model

```php
public function order_detail($order_id) {
    $data['order'] = $this->Order_model->get_with_items($order_id);
    $this->load->view('kasir/order_detail', $data);
}

public function print_struk($order_id) {
    $data['order'] = $this->Order_model->get_with_items($order_id);
    $this->load->view('kasir/struk', $data);
}
```

### Syntax Di Model

```php
public function get_with_items($id) {
    $order = $this->get($id);
    if ($order) {
        $this->db->select('order_items.*, menu_items.name');
        $this->db->join('menu_items', 'menu_items.id = order_items.menu_id', 'left');
        $this->db->where('order_id', $id);
        $order['items'] = $this->db->get('order_items')->result_array();
    }
    return $order;
}
```

### Alur Lengkap

```text
Kasir klik Detail atau Print Struk
    ↓
Controller menerima order_id
    ↓
Controller memanggil Order_model::get_with_items($order_id)
    ↓
Model mengambil data order utama
    ↓
Model join order_items dengan menu_items
    ↓
Model menambahkan key items ke array order
    ↓
Controller mengirim data ke view order_detail.php atau struk.php
```

### Kenapa Syntax Model Seperti Itu

```php
$order = $this->get($id);
```

Model memakai method `get()` yang sudah ada agar query order utama tidak ditulis ulang.

```php
$this->db->select('order_items.*, menu_items.name');
```

`order_items.*` mengambil quantity dan price, sedangkan `menu_items.name` mengambil nama menu.

```php
$this->db->join('menu_items', 'menu_items.id = order_items.menu_id', 'left');
```

Join dibutuhkan karena `order_items` hanya menyimpan `menu_id`, bukan nama menu.

```php
$order['items'] = ...
```

Data item dimasukkan ke dalam array order supaya view cukup menerima satu variabel `$order` yang sudah lengkap.

---

## 15. Order_model::get_daily_completed_revenue() Dipanggil Oleh Kasir::index()

### Syntax Controller

```php
$start_date = $this->input->get('start_date');
$end_date = $this->input->get('end_date');
$data['daily_revenue'] = $this->Order_model->get_daily_completed_revenue($start_date, $end_date);
```

### Syntax Model

```php
public function get_daily_completed_revenue($start_date = null, $end_date = null) {
    $this->db->select('DATE(updated_at) as date, COUNT(id) as total_orders, SUM(total) as revenue');
    $this->db->where('status', 'completed');
    if ($start_date) $this->db->where('DATE(updated_at) >=', $start_date);
    if ($end_date) $this->db->where('DATE(updated_at) <=', $end_date);
    $this->db->group_by('DATE(updated_at)');
    $this->db->order_by('DATE(updated_at)', 'DESC');
    return $this->db->get('orders')->result_array();
}
```

### Alur Lengkap

```text
Kasir buka tab Rekap Pendapatan
    ↓
Kasir bisa mengisi filter start_date dan end_date
    ↓
Kasir::index() mengambil filter dari URL GET
    ↓
Controller memanggil get_daily_completed_revenue()
    ↓
Model menghitung total order dan revenue per tanggal
    ↓
Data dikirim ke kasir/dashboard.php
```

### Kenapa Syntax Model Seperti Itu

```php
DATE(updated_at) as date
```

Dipakai untuk mengambil tanggal tanpa jam. Rekap harian tidak butuh detail jam.

```php
COUNT(id) as total_orders
```

Menghitung jumlah order selesai per tanggal.

```php
SUM(total) as revenue
```

Menghitung total pendapatan per tanggal.

```php
if ($start_date) ...
if ($end_date) ...
```

Filter tanggal bersifat opsional. Jika user tidak memilih tanggal, model tetap bisa menampilkan semua rekap.

```php
group_by('DATE(updated_at)')
```

Wajib dipakai karena ada agregasi `COUNT` dan `SUM`. Tanpa `group_by`, database tidak tahu rekapnya harus dipisah per tanggal.

---

## 16. Order_model Dan Transaction_model Dipanggil Oleh Admin::index()

### File Controller

`application/controllers/Admin.php`

### Syntax Pemanggilan Model

```php
public function index() {
    $selected_date = $this->input->get('date') ?: date('Y-m-d');
    $month_start = date('Y-m-01 00:00:00');
    $month_end = date('Y-m-t 23:59:59');

    $data['selected_date'] = $selected_date;
    $data['total_income'] = $this->Transaction_model->get_summary($month_start, $month_end)['total'] ?? 0;
    $data['selected_income'] = $this->Order_model->get_completed_revenue_by_date($selected_date);
    $data['selected_orders'] = $this->Order_model->get_completed_by_date($selected_date);
    $data['daily_data'] = $this->Transaction_model->get_daily_summary($selected_date);

    $this->load->view('templates/header_admin');
    $this->load->view('admin/dashboard', $data);
    $this->load->view('templates/footer_admin');
}
```

### Syntax Transaction_model

```php
public function get_summary($start_date, $end_date) {
    $this->db->select_sum('total');
    $this->db->where('status', 'paid');
    $this->db->where('created_at >=', $start_date);
    $this->db->where('created_at <=', $end_date);
    return $this->db->get('transactions')->row_array();
}

public function get_daily_summary($date) {
    $this->db->select('HOUR(created_at) as hour, SUM(total) as total');
    $this->db->where('status', 'paid');
    $this->db->where('DATE(created_at)', $date);
    $this->db->group_by('HOUR(created_at)');
    return $this->db->get('transactions')->result_array();
}
```

### Syntax Order_model

```php
public function get_completed_revenue_by_date($date) {
    $this->db->select_sum('total');
    $this->db->where('status', 'completed');
    $this->db->where('DATE(updated_at)', $date);
    $result = $this->db->get('orders')->row_array();
    return (float)($result['total'] ?? 0);
}

public function get_completed_by_date($date) {
    $this->db->where('status', 'completed');
    $this->db->where('DATE(updated_at)', $date);
    $this->db->order_by('queue_number', 'ASC');
    $orders = $this->db->get('orders')->result_array();

    foreach ($orders as &$order) {
        $this->db->select('order_items.*, menu_items.name');
        $this->db->join('menu_items', 'menu_items.id = order_items.menu_id', 'left');
        $this->db->where('order_id', $order['id']);
        $order['items'] = $this->db->get('order_items')->result_array();
    }

    return $orders;
}
```

### Alur Lengkap

```text
Admin buka /admin
    ↓
Admin::index() menentukan tanggal yang dipilih dan rentang bulan
    ↓
Transaction_model::get_summary() menghitung income bulanan
    ↓
Order_model::get_completed_revenue_by_date() menghitung income tanggal terpilih
    ↓
Order_model::get_completed_by_date() mengambil order tanggal terpilih beserta item
    ↓
Transaction_model::get_daily_summary() mengambil data chart per jam
    ↓
Controller mengirim semua data ke admin/dashboard.php
```

### Kenapa Syntax Model Seperti Itu

```php
$this->db->select_sum('total');
```

Dipakai untuk menghitung total pendapatan langsung di database. Ini lebih efisien daripada mengambil semua transaksi lalu menjumlahkan di PHP.

```php
$this->db->where('status', 'paid');
```

Hanya transaksi yang benar-benar sudah dibayar yang dihitung sebagai income.

```php
HOUR(created_at) as hour
```

Dipakai untuk membuat data chart per jam pada dashboard admin.

```php
foreach ($orders as &$order)
```

Tanda `&` berarti data `$order` direferensikan langsung ke array aslinya. Jadi saat ditambahkan `$order['items']`, perubahan itu masuk ke `$orders` yang nanti direturn.

---

## 17. Order_model Dipanggil Oleh Admin::orders_recap()

### Syntax Controller

```php
public function orders_recap() {
    $data['orders'] = $this->Order_model->get_all_completed();
    $data['total_revenue'] = $this->Order_model->sum_revenue();
    $data['total_orders'] = count($data['orders']);

    $this->load->view('templates/header_admin');
    $this->load->view('admin/orders_recap', $data);
    $this->load->view('templates/footer_admin');
}
```

### Syntax Model

```php
public function get_all_completed() {
    $this->db->where('status', 'completed');
    $this->db->order_by('updated_at', 'DESC');
    return $this->db->get('orders')->result_array();
}

public function sum_revenue() {
    $this->db->select_sum('total');
    $this->db->where('status', 'completed');
    $result = $this->db->get('orders')->row();
    return (float)($result->total ?? 0);
}
```

### Alur Lengkap

```text
Admin buka halaman orders recap
    ↓
Admin::orders_recap()
    ↓
Order_model::get_all_completed() mengambil daftar order selesai
    ↓
Order_model::sum_revenue() menghitung total pendapatan
    ↓
Controller menghitung total_orders dengan count()
    ↓
Data dikirim ke admin/orders_recap.php
```

### Kenapa Syntax Model Seperti Itu

```php
$this->db->where('status', 'completed');
```

Rekap hanya boleh menghitung order yang sudah selesai. Order pending atau processing belum dianggap pendapatan final.

```php
return (float)($result->total ?? 0);
```

Dipakai supaya hasil selalu angka. Kalau belum ada transaksi, hasil `SUM` bisa null, maka diganti 0.

---

## 18. Product_model Dipanggil Oleh Admin Untuk Produk

### File Controller

`application/controllers/Admin.php`

### Syntax Pemanggilan Model

```php
public function products() {
    $data['products'] = $this->Product_model->get_all();
    $this->load->view('templates/header_admin');
    $this->load->view('admin/products', $data);
    $this->load->view('templates/footer_admin');
}

public function add_product() {
    $data = [
        'name' => $this->input->post('name'),
        'category' => $this->input->post('category'),
        'price' => $this->input->post('price'),
        'description' => $this->input->post('description'),
        'image_url' => $image_url
    ];
    $this->Product_model->create($data);
    redirect('admin/products');
}

public function delete_product($id) {
    $this->Product_model->delete($id);
    redirect('admin/products');
}
```

### Syntax Model

```php
public function get_all() {
    return $this->db->get('menu_items')->result_array();
}

public function create($data) {
    return $this->db->insert('menu_items', $data);
}

public function delete($id) {
    $this->db->where('id', $id);
    return $this->db->delete('menu_items');
}
```

### Alur Lengkap

```text
Admin membuka /admin/products
    ↓
Admin::products()
    ↓ Product_model::get_all()
    ↓ data menu_items dikirim ke view produk

Admin tambah produk
    ↓
Admin::add_product()
    ↓ Product_model::create($data)
    ↓ INSERT ke menu_items

Admin hapus produk
    ↓
Admin::delete_product($id)
    ↓ Product_model::delete($id)
    ↓ DELETE menu_items berdasarkan id
```

### Kenapa Syntax Model Seperti Itu

```php
return $this->db->get('menu_items')->result_array();
```

Produk/menu jumlahnya banyak, maka dipakai `result_array()`.

```php
return $this->db->insert('menu_items', $data);
```

Tambah produk berarti membuat baris baru, maka pakai `insert()`.

```php
$this->db->where('id', $id);
return $this->db->delete('menu_items');
```

`where()` wajib dipakai supaya produk yang dihapus hanya produk dengan id tertentu.

---

## 19. Ringkasan Jawaban Cepat Saat Ditanya

Kalau ditanya:

"Model ini digunakan oleh siapa saja?"

Jawab:

"Model digunakan oleh controller. Controller memuat model dengan `$this->load->model()`, lalu memanggil method model seperti `$this->Order_model->get_all_pending()`. Model menjalankan query database dengan `$this->db`, hasilnya dikembalikan ke controller, lalu controller mengirim data itu ke view. Contohnya `Kasir::index()` memanggil `Order_model::get_all_pending()` untuk mengambil pesanan baru, lalu datanya dikirim ke `kasir/dashboard.php`."

Kalau ditanya:

"Kenapa syntax model memakai Query Builder seperti `where()`, `get()`, `result_array()`?"

Jawab:

"Karena CodeIgniter menyediakan Query Builder agar query database lebih rapi, mudah dibaca, dan lebih aman daripada SQL manual. `where()` untuk filter data, `get()` untuk menjalankan SELECT, `result_array()` untuk mengambil banyak baris, `row_array()` untuk satu baris, `insert()` untuk tambah data, `update()` untuk ubah data, dan `delete()` untuk hapus data."

Kalau ditanya:

"Contoh alur lengkap model?"

Jawab:

"Contohnya checkout. User klik bayar di view, JavaScript `confirmPayment()` mengirim data lewat `fetch()` ke `OrderController::create()`. Controller memanggil `QueueModel::getCurrentQueue()` untuk nomor antrian, lalu memanggil `OrderModel::createOrder()` untuk menyimpan order ke `orders` dan item ke `order_items`. Setelah sukses, model mengembalikan `orderId`, controller membalas JSON, dan JavaScript menampilkan nomor antrian ke user."

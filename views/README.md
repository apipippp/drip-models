# Dokumentasi Workflow Frontend Views DRIP Coffee

File ini khusus untuk bagian **Views / Frontend**. Isinya menjelaskan alur dari tampilan HTML/PHP ke JavaScript, lalu bagaimana JavaScript menjalankan logic di halaman.

Bagian ini cocok dipakai untuk menjawab pertanyaan seperti:

"Kalau user klik tombol di tampilan, masuk ke JS mana? Logic-nya jalan bagaimana?"

---

## 1. Alur Navigasi Dari Home Ke Menu Dan Antrian

### File View

`application/views/drip/home.php`

### Kode Di Tampilan

```php
<button class="btn btn-dark px-4 py-3" onclick="location.href='<?= site_url('menu') ?>'">lihat menu →</button>

<button class="btn btn-outline-dark px-4 py-3" onclick="location.href='<?= site_url('antrian') ?>'">cek antrian</button>
```

### Alur Kerja

1. User membuka halaman home.
2. User klik tombol **lihat menu** atau **cek antrian**.
3. Atribut `onclick` menjalankan JavaScript langsung dari HTML.
4. `location.href` mengubah URL browser.
5. `site_url('menu')` dibuat oleh PHP CodeIgniter untuk menghasilkan URL yang benar ke route menu.
6. Browser berpindah halaman ke controller sesuai URL tersebut.

### Kenapa Ditulis Begitu

`location.href` dipakai karena navigasi ini sederhana, hanya pindah halaman. Tidak perlu fungsi JavaScript khusus karena tidak ada logic tambahan seperti validasi, modal, atau AJAX.

`site_url()` dipakai supaya URL aman mengikuti konfigurasi CodeIgniter. Kalau domain atau base path berubah saat hosting, link tetap benar.

---

## 2. Alur Menu Ditampilkan Dari View Ke JavaScript

### File View

`application/views/drip/menu.php`

### Kode Di Tampilan

```html
<div id="menuGrid" class="menu-grid"></div>
```

### File JavaScript

`assets/js/script.js`

### Kode Di JavaScript

```javascript
function renderMenu(items = menuItems) {
    const grid = document.getElementById('menuGrid');

    if (!grid) return;

    grid.innerHTML = items.map(item => `
        <div class="menu-card">
            <img src="${ASSET_URL}/images/${item.image}" alt="${item.name}">
            <div class="menu-info">
                <h3>${item.name}</h3>
                <p>${item.desc}</p>
                <div class="menu-footer">
                    <span>Rp ${item.price.toLocaleString('id-ID')}</span>
                    <button onclick="addToCart(${item.id})">+</button>
                </div>
            </div>
        </div>
    `).join('');
}
```

### Alur Kerja

1. `menu.php` hanya menyediakan wadah kosong dengan `id="menuGrid"`.
2. Saat halaman selesai dimuat, `script.js` mencari elemen tersebut dengan `document.getElementById('menuGrid')`.
3. Data menu berada di array JavaScript bernama `menuItems`.
4. Fungsi `renderMenu()` mengubah setiap data menu menjadi kartu HTML.
5. `.map()` dipakai untuk looping array menu dan menghasilkan string HTML.
6. `.join('')` menyatukan semua string HTML menjadi satu tampilan utuh.
7. `grid.innerHTML = ...` memasukkan kartu menu ke dalam `<div id="menuGrid">`.

### Kenapa Ditulis Begitu

Menu dibuat lewat JavaScript supaya bisa difilter secara cepat tanpa reload halaman. View cukup menyediakan container, lalu JS bertanggung jawab membuat isi menu secara dinamis.

Pendekatan ini cocok untuk frontend karena data menu bisa dimanipulasi langsung di browser, misalnya filter kategori atau update tampilan kartu.

---

## 3. Alur Filter Menu Berdasarkan Kategori

### File View

`application/views/drip/menu.php`

### Kode Di Tampilan

```html
<button class="filter-btn active" data-category="all">Semua</button>
<button class="filter-btn" data-category="kopi">Kopi</button>
<button class="filter-btn" data-category="non-kopi">Non Kopi</button>
<button class="filter-btn" data-category="makanan">Makanan</button>
<button class="filter-btn" data-category="dessert">Dessert</button>
```

### File JavaScript

`assets/js/script.js`

### Kode Di JavaScript

```javascript
document.querySelectorAll('.filter-btn').forEach(btn => {
    btn.addEventListener('click', function () {
        document.querySelectorAll('.filter-btn').forEach(b => b.classList.remove('active'));
        this.classList.add('active');

        const category = this.dataset.category;
        const filtered = category === 'all'
            ? menuItems
            : menuItems.filter(item => item.cat === category);

        renderMenu(filtered);
    });
});
```

### Alur Kerja

1. Tombol kategori di HTML diberi class `filter-btn`.
2. Setiap tombol punya `data-category`, misalnya `kopi`, `makanan`, atau `dessert`.
3. JavaScript mengambil semua tombol dengan `document.querySelectorAll('.filter-btn')`.
4. `.forEach()` memasang event click ke setiap tombol.
5. Saat tombol diklik, class `active` dihapus dari semua tombol.
6. Tombol yang sedang diklik diberi class `active`.
7. JS membaca kategori dari `this.dataset.category`.
8. Kalau kategori `all`, semua menu ditampilkan.
9. Kalau kategori tertentu, `.filter()` memilih item menu yang `item.cat`-nya sama dengan kategori.
10. Hasil filter dikirim ke `renderMenu(filtered)` untuk ditampilkan ulang.

### Kenapa Ditulis Begitu

`data-category` dipakai supaya kategori bisa disimpan langsung di HTML tanpa membuat banyak fungsi berbeda. Satu logic JavaScript bisa menangani semua tombol kategori.

`.filter()` dipakai karena tugasnya memang menyaring array berdasarkan kondisi tertentu.

---

## 4. Alur Tambah Menu Ke Keranjang

### File View / HTML Yang Dibuat JS

Tombol ini dibuat oleh `renderMenu()` di `assets/js/script.js`.

### Kode Tombol

```html
<button onclick="addToCart(${item.id})">+</button>
```

### File JavaScript

`assets/js/script.js`

### Kode Logic

```javascript
function addToCart(id) {
    const item = menuItems.find(menu => menu.id === id);
    const existing = cart.find(cartItem => cartItem.id === id);

    if (existing) {
        existing.qty += 1;
    } else {
        cart.push({ ...item, qty: 1 });
    }

    renderCart();
    updateCartCount();
    showToast(`${item.name} ditambahkan ke keranjang`);
}
```

### Alur Kerja

1. User klik tombol `+` pada salah satu menu.
2. HTML memanggil fungsi `addToCart(id)`.
3. Parameter `id` adalah ID menu yang diklik.
4. JS mencari data menu di `menuItems` menggunakan `.find()`.
5. JS mengecek apakah item itu sudah ada di array `cart`.
6. Kalau sudah ada, quantity (`qty`) ditambah 1.
7. Kalau belum ada, item baru dimasukkan ke `cart` dengan `qty: 1`.
8. `renderCart()` dipanggil untuk menggambar ulang isi keranjang.
9. `updateCartCount()` memperbarui angka bubble keranjang.
10. `showToast()` menampilkan notifikasi kecil bahwa item berhasil ditambahkan.

### Kenapa Ditulis Begitu

Array `cart` dipakai sebagai state keranjang di frontend. Jadi selama user belum checkout, semua item tersimpan di JavaScript dulu.

`.find()` cocok dipakai karena kita hanya butuh mencari satu item berdasarkan ID.

`renderCart()` dipanggil ulang setelah data berubah supaya tampilan selalu sinkron dengan isi array `cart`.

---

## 5. Alur Buka Dan Tutup Cart Drawer

### File View

`application/views/layouts/header.php`

```php
<div class="cart-bubble ms-3" onclick="toggleCart()">
```

`application/views/layouts/footer.php`

```php
<a href="javascript:void(0)" class="mobile-nav-link" onclick="toggleCart()">

<button class="btn-close" onclick="toggleCart()"></button>
```

### File JavaScript

`assets/js/script.js`

### Kode Logic

```javascript
function toggleCart() {
    const cartDrawer = document.getElementById('cartDrawer');
    cartDrawer.classList.toggle('open');
}
```

### Alur Kerja

1. User klik ikon keranjang di header desktop atau bottom nav mobile.
2. HTML memanggil `toggleCart()`.
3. JS mengambil elemen drawer dengan `id="cartDrawer"`.
4. `classList.toggle('open')` mengecek apakah class `open` sudah ada.
5. Kalau belum ada, class `open` ditambahkan sehingga drawer muncul.
6. Kalau sudah ada, class `open` dihapus sehingga drawer tertutup.

### Kenapa Ditulis Begitu

Cart drawer tidak perlu reload halaman. Cukup ubah class CSS. Ini lebih ringan dan responsif karena hanya memanipulasi tampilan di browser.

`toggle()` dipakai karena satu fungsi bisa menangani dua keadaan sekaligus: buka dan tutup.

---

## 6. Alur Render Isi Keranjang

### File View

`application/views/layouts/footer.php`

### Kode Container

```html
<div id="cartItems"></div>
<span id="cartTotal">Rp 0</span>
```

### File JavaScript

`assets/js/script.js`

### Kode Logic

```javascript
function renderCart() {
    const cartItems = document.getElementById('cartItems');
    const cartTotal = document.getElementById('cartTotal');

    if (cart.length === 0) {
        cartItems.innerHTML = '<p class="text-muted">Keranjang masih kosong</p>';
        cartTotal.textContent = 'Rp 0';
        return;
    }

    cartItems.innerHTML = cart.map(item => `
        <div class="cart-item">
            <div>
                <strong>${item.name}</strong>
                <p>Rp ${item.price.toLocaleString('id-ID')}</p>
            </div>
            <div class="qty-control">
                <button onclick="updateQty(${item.id}, -1)">-</button>
                <span>${item.qty}</span>
                <button onclick="updateQty(${item.id}, 1)">+</button>
            </div>
        </div>
    `).join('');

    const total = cart.reduce((sum, item) => sum + (item.price * item.qty), 0);
    cartTotal.textContent = `Rp ${total.toLocaleString('id-ID')}`;
}
```

### Alur Kerja

1. View menyediakan container kosong `cartItems` dan teks total `cartTotal`.
2. JavaScript mengisi container berdasarkan array `cart`.
3. Kalau cart kosong, JS menampilkan teks keranjang kosong.
4. Kalau ada item, JS membuat HTML item keranjang dengan `.map()`.
5. Tombol `-` dan `+` di cart memanggil `updateQty()`.
6. Total dihitung dengan `.reduce()`.
7. Hasil total ditampilkan ke `cartTotal`.

### Kenapa Ditulis Begitu

Cart dirender ulang setiap ada perubahan supaya tampilan tidak beda dengan data. Ini prinsip umum frontend: state berubah, UI ikut dirender ulang.

`.reduce()` dipakai karena kita ingin mengubah array item menjadi satu angka total harga.

---

## 7. Alur Lanjut Pembayaran

### File View

`application/views/layouts/footer.php`

### Kode Tombol

```html
<button class="btn btn-dark w-100 py-3" onclick="openPayment()">Lanjut Pembayaran →</button>
```

### File JavaScript

`assets/js/script.js`

### Kode Logic

```javascript
function openPayment() {
    if (cart.length === 0) {
        showToast('Keranjang masih kosong');
        return;
    }

    const tableNumber = document.getElementById('tableNumber').value;
    const subtotal = cart.reduce((sum, item) => sum + (item.price * item.qty), 0);
    const tax = Math.round(subtotal * 0.1);
    const total = subtotal + tax;

    currentOrder = {
        cart: [...cart],
        sub: subtotal,
        tax: tax,
        tot: total,
        table: tableNumber
    };

    document.getElementById('paymentModal').style.display = 'flex';
}
```

### Alur Kerja

1. User klik tombol **Lanjut Pembayaran**.
2. HTML memanggil `openPayment()`.
3. JS mengecek apakah cart kosong.
4. Kalau kosong, tampil toast dan proses berhenti.
5. Kalau ada isi, JS mengambil nomor meja dari input.
6. Subtotal dihitung dari `cart`.
7. Pajak dihitung 10% dari subtotal.
8. Total dihitung dari subtotal + pajak.
9. Data disimpan ke `currentOrder` sebagai snapshot pesanan.
10. Modal pembayaran ditampilkan dengan mengubah `display` menjadi `flex`.

### Kenapa Ditulis Begitu

Validasi cart kosong dilakukan di frontend agar user langsung mendapat feedback tanpa perlu kirim data ke server.

`currentOrder` dipakai untuk menyimpan data saat modal dibuka. Jadi data pembayaran tidak berubah-ubah saat user memilih metode pembayaran.

---

## 8. Alur Pilih Metode Pembayaran

### File View

`application/views/layouts/footer.php`

### Kode Tampilan

```html
<div class="pay-method" onclick="selectPayment('qris', this)">QRIS / E-Wallet</div>
<div class="pay-method" onclick="selectPayment('cash', this)">Tunai (Kasir)</div>

<button id="payBtn" class="btn btn-dark w-100 py-3 mt-2" disabled onclick="confirmPayment()">
    pilih metode pembayaran dulu
</button>
```

### File JavaScript

`assets/js/script.js`

### Kode Logic

```javascript
function selectPayment(method, element) {
    selectedPayment = method;

    document.querySelectorAll('.pay-method').forEach(el => {
        el.classList.remove('selected');
    });

    element.classList.add('selected');

    const payBtn = document.getElementById('payBtn');
    payBtn.disabled = false;
    payBtn.textContent = method === 'qris' ? 'Bayar dengan QRIS' : 'Bayar di Kasir';
}
```

### Alur Kerja

1. User memilih QRIS atau Tunai.
2. HTML memanggil `selectPayment(method, this)`.
3. `method` berisi string pilihan, misalnya `qris` atau `cash`.
4. `this` mengirim elemen HTML yang sedang diklik.
5. JS menyimpan pilihan ke variabel global `selectedPayment`.
6. Semua class `selected` dihapus dari metode pembayaran lain.
7. Elemen yang diklik diberi class `selected`.
8. Tombol bayar diaktifkan dengan `disabled = false`.
9. Teks tombol diubah sesuai metode pembayaran.

### Kenapa Ditulis Begitu

Tombol bayar awalnya `disabled` agar user tidak bisa checkout sebelum memilih metode pembayaran.

Parameter `this` dipakai supaya JS tahu elemen mana yang harus diberi class aktif tanpa perlu mencari ulang berdasarkan ID.

---

## 9. Alur Konfirmasi Pembayaran Sampai Kirim Ke Server

### File View

`application/views/layouts/footer.php`

### Kode Tombol

```html
<button id="payBtn" class="btn btn-dark w-100 py-3 mt-2" disabled onclick="confirmPayment()">
    pilih metode pembayaran dulu
</button>
```

### File JavaScript

`assets/js/script.js`

### Kode Logic

```javascript
function confirmPayment() {
    if (!selectedPayment || !currentOrder) {
        showToast('Pilih metode pembayaran dulu');
        return;
    }

    fetch(SITE_URL + '/submit_order_ajax', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            items: currentOrder.cart,
            table_number: currentOrder.table,
            subtotal: currentOrder.sub,
            tax: currentOrder.tax,
            total: currentOrder.tot,
            payment_method: selectedPayment
        })
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            cart = [];
            renderCart();
            updateCartCount();
            document.getElementById('paymentModal').style.display = 'none';
            document.getElementById('successQueue').textContent = data.queue_number;
            document.getElementById('successOverlay').style.display = 'flex';
        } else {
            showToast(data.message || 'Gagal membuat pesanan');
        }
    });
}
```

### Alur Kerja

1. User klik tombol bayar.
2. HTML memanggil `confirmPayment()`.
3. JS mengecek apakah metode pembayaran dan data order sudah ada.
4. Jika belum, tampil toast dan proses berhenti.
5. Jika sudah lengkap, JS mengirim data ke server memakai `fetch()`.
6. URL yang dipakai adalah `SITE_URL + '/submit_order_ajax'`.
7. `method: 'POST'` berarti data dikirim sebagai request POST.
8. Header `Content-Type: application/json` memberitahu server bahwa body request berupa JSON.
9. `JSON.stringify()` mengubah object JavaScript menjadi string JSON.
10. Server mengembalikan response JSON.
11. `.then(response => response.json())` mengubah response menjadi object JavaScript.
12. Jika `data.success` true, cart dikosongkan dan tampilan diperbarui.
13. Modal pembayaran ditutup.
14. Nomor antrian dari server ditampilkan ke success overlay.
15. Success overlay ditampilkan ke user.

### Kenapa Ditulis Begitu

`fetch()` dipakai karena checkout tidak perlu reload halaman. User tetap di halaman yang sama, lalu mendapat feedback sukses secara langsung.

JSON dipakai karena formatnya mudah dibaca JavaScript dan mudah diproses server.

Cart dikosongkan setelah server sukses, bukan sebelum request, supaya kalau server gagal data belanja user tidak hilang.

---

## 10. Alur Ke Controller Setelah Dari JavaScript

### File JavaScript

`assets/js/script.js`

```javascript
fetch(SITE_URL + '/submit_order_ajax', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({...})
})
```

### File Controller

`application/controllers/Drip.php`

### Alur Besar Backend

```text
View footer.php
    ↓ onclick="confirmPayment()"
script.js
    ↓ fetch POST JSON
Drip.php method submit_order_ajax()
    ↓ validasi data request
OrderModel.php
    ↓ simpan order dan order_items ke database
Database
    ↓ response JSON
script.js
    ↓ tampilkan success overlay dan nomor antrian
```

### Penjelasan

Bagian frontend berhenti di `fetch()`. Setelah itu request masuk ke controller CodeIgniter. Controller membaca data JSON, lalu memanggil model untuk menyimpan data ke database.

Setelah database berhasil menyimpan order, controller mengembalikan response JSON seperti:

```json
{
    "success": true,
    "queue_number": "A001"
}
```

Response itu dibaca lagi oleh JavaScript untuk menampilkan nomor antrian ke user.

---

## 11. Alur Success Overlay Setelah Checkout

### File View

`application/views/layouts/footer.php`

### Kode Tampilan

```html
<div id="successOverlay" class="success-overlay">
    <h1 id="successQueue">A001</h1>
    <button class="btn btn-dark px-5 py-3" onclick="closeSuccess()">Tutup</button>
</div>
```

### File JavaScript

`assets/js/script.js`

### Kode Logic

```javascript
document.getElementById('successQueue').textContent = data.queue_number;
document.getElementById('successOverlay').style.display = 'flex';

function closeSuccess() {
    document.getElementById('successOverlay').style.display = 'none';
    location.href = SITE_URL + '/antrian';
}
```

### Alur Kerja

1. Setelah checkout sukses, server mengembalikan nomor antrian.
2. JS memasukkan nomor itu ke elemen `successQueue`.
3. JS menampilkan overlay sukses.
4. User klik tombol **Tutup**.
5. Tombol memanggil `closeSuccess()`.
6. Overlay ditutup.
7. Browser diarahkan ke halaman antrian.

### Kenapa Ditulis Begitu

Success overlay dipakai supaya user langsung melihat bahwa transaksi berhasil tanpa pindah halaman mendadak.

Setelah user klik tutup, baru diarahkan ke halaman antrian agar bisa memantau status pesanan.

---

## 12. Alur Halaman Antrian Dari PHP Dan JavaScript

### File View

`application/views/drip/antrian.php`

### Kode View Server-Side

```php
<?php if (!empty($active_order)): ?>
    <h1><?= htmlspecialchars($active_order['queue_number']) ?></h1>
<?php else: ?>
    <div id="localQueueStatus"></div>
<?php endif; ?>
```

### File JavaScript

`assets/js/script.js`

### Kode Logic

```javascript
function loadQueueStatus() {
    const container = document.getElementById('localQueueStatus');

    if (!container) return;

    const history = JSON.parse(localStorage.getItem('orderHistory') || '[]');
    const lastOrder = history[history.length - 1];

    if (!lastOrder) {
        container.innerHTML = '<p>Belum ada pesanan aktif</p>';
        return;
    }

    container.innerHTML = `
        <h1>${lastOrder.queue}</h1>
        <p>Status: ${lastOrder.status}</p>
    `;
}
```

### Alur Kerja

1. Saat halaman antrian dibuka, PHP mengecek apakah `$active_order` ada.
2. Kalau user login dan punya pesanan aktif, data ditampilkan langsung dari server/database.
3. Kalau tidak ada data server, view menyediakan container kosong `localQueueStatus`.
4. JavaScript menjalankan `loadQueueStatus()`.
5. JS membaca `orderHistory` dari `localStorage`.
6. JS mengambil order terakhir.
7. Jika ada, JS menampilkan nomor antrian dan status ke container.
8. Jika tidak ada, JS menampilkan pesan belum ada pesanan.

### Kenapa Ditulis Begitu

Sistem ini hybrid: user login memakai database, sedangkan guest bisa memakai data lokal dari browser. Ini membuat guest tetap bisa melihat antrian walaupun tidak membuat akun.

---

## 13. Alur Riwayat Pesanan Dari LocalStorage

### File View

`application/views/drip/riwayat.php`

### Kode Container

```html
<div id="orderHistoryContainer"></div>
```

### File JavaScript

`assets/js/script.js`

### Kode Logic

```javascript
function loadOrderHistory() {
    const container = document.getElementById('orderHistoryContainer');

    if (!container) return;

    const history = JSON.parse(localStorage.getItem('orderHistory') || '[]');

    if (history.length === 0) {
        container.innerHTML = '<p>Belum ada riwayat pesanan</p>';
        return;
    }

    container.innerHTML = history.map(order => `
        <div class="history-card">
            <h3>${order.queue}</h3>
            <p>Total: Rp ${order.total.toLocaleString('id-ID')}</p>
        </div>
    `).join('');
}
```

### Alur Kerja

1. View menyediakan container kosong untuk riwayat.
2. JS mencari container dengan `getElementById()`.
3. JS membaca data `orderHistory` dari `localStorage`.
4. `JSON.parse()` mengubah string JSON menjadi array JavaScript.
5. Kalau kosong, JS menampilkan pesan belum ada riwayat.
6. Kalau ada data, JS mengubah setiap order menjadi card HTML.
7. HTML hasil `.map()` dimasukkan ke container.

### Kenapa Ditulis Begitu

`localStorage` dipakai supaya riwayat guest tetap ada walaupun halaman direload atau browser ditutup. Kalau memakai variabel biasa, data akan hilang saat halaman refresh.

---

## 14. Alur Global URL Dari PHP Ke JavaScript

### File View

`application/views/layouts/header.php`

### Kode

```php
<script>
    const SITE_URL = "<?= site_url() ?>";
    const ASSET_URL = "<?= base_url('assets') ?>";
</script>
```

### Dipakai Di JavaScript

```javascript
fetch(SITE_URL + '/submit_order_ajax', {...})

`<img src="${ASSET_URL}/images/${item.image}">`
```

### Alur Kerja

1. PHP CodeIgniter membuat URL aplikasi lewat `site_url()` dan `base_url()`.
2. Nilai URL itu dicetak ke variabel JavaScript global.
3. `script.js` memakai `SITE_URL` untuk request ke controller.
4. `script.js` memakai `ASSET_URL` untuk memanggil file gambar menu.

### Kenapa Ditulis Begitu

JavaScript tidak bisa langsung memanggil helper PHP seperti `site_url()`, karena JS berjalan di browser sedangkan PHP berjalan di server.

Solusinya, PHP mencetak nilai URL ke dalam variabel JS sebelum file `script.js` dipakai. Dengan begitu JS tetap tahu alamat aplikasi tanpa hardcode domain.

---

## 15. Alur Toast Notification

### File View

`application/views/layouts/footer.php`

### Kode Container

```html
<div class="toast-container position-fixed bottom-0 end-0 p-3">
    <div id="liveToast" class="toast" role="alert">
        <div class="toast-body" id="toastMessage"></div>
    </div>
</div>
```

### File JavaScript

`assets/js/script.js`

### Kode Logic

```javascript
function showToast(message) {
    const toastMessage = document.getElementById('toastMessage');
    const toastElement = document.getElementById('liveToast');

    toastMessage.textContent = message;

    const toast = new bootstrap.Toast(toastElement);
    toast.show();
}
```

### Alur Kerja

1. View menyediakan elemen toast Bootstrap.
2. JavaScript menerima pesan lewat parameter `message`.
3. JS memasukkan pesan ke `toastMessage`.
4. JS membuat instance `bootstrap.Toast`.
5. `toast.show()` menampilkan notifikasi.

### Kenapa Ditulis Begitu

Toast dipakai untuk feedback singkat seperti item berhasil ditambahkan, keranjang kosong, atau checkout gagal. Toast lebih ringan daripada alert browser karena tidak mengganggu alur user.

---

## 16. Ringkasan File Yang Terlibat

| Alur | View | JavaScript | Fungsi Utama |
|---|---|---|---|
| Navigasi home | `drip/home.php` | inline `location.href` | Pindah halaman |
| Render menu | `drip/menu.php` | `script.js` | `renderMenu()` |
| Filter menu | `drip/menu.php` | `script.js` | event listener `.filter-btn` |
| Tambah cart | HTML dari `renderMenu()` | `script.js` | `addToCart()` |
| Buka cart | `layouts/header.php`, `layouts/footer.php` | `script.js` | `toggleCart()` |
| Render cart | `layouts/footer.php` | `script.js` | `renderCart()` |
| Lanjut bayar | `layouts/footer.php` | `script.js` | `openPayment()` |
| Pilih metode | `layouts/footer.php` | `script.js` | `selectPayment()` |
| Submit order | `layouts/footer.php` | `script.js` + `Drip.php` | `confirmPayment()` |
| Sukses order | `layouts/footer.php` | `script.js` | `closeSuccess()` |
| Antrian | `drip/antrian.php` | `script.js` | `loadQueueStatus()` |
| Riwayat | `drip/riwayat.php` | `script.js` | `loadOrderHistory()` |
| URL global | `layouts/header.php` | `script.js` | `SITE_URL`, `ASSET_URL` |
| Notifikasi | `layouts/footer.php` | `script.js` | `showToast()` |

---

## 17. Jawaban Singkat Kalau Ditanya Saat Presentasi

Kalau ditanya:

"Alur dari tampilan ke JavaScript itu bagaimana?"

Jawab:

"Di bagian frontend, View PHP menyiapkan struktur HTML seperti tombol, container kosong, cart drawer, payment modal, dan toast. Beberapa tombol diberi event seperti `onclick="addToCart()"`, `onclick="openPayment()"`, atau `onclick="confirmPayment()"`. Saat user klik tombol itu, browser menjalankan fungsi di `assets/js/script.js`. JavaScript lalu memproses logic seperti menambah item ke array cart, menghitung total, membuka modal pembayaran, memilih metode bayar, sampai mengirim data order ke controller lewat `fetch()` dalam format JSON. Setelah server mengembalikan response, JavaScript memperbarui tampilan lagi, misalnya menampilkan success overlay dan nomor antrian. Jadi View bertugas menyediakan tampilan dan titik event, sedangkan JavaScript menjalankan interaksi dan logic frontend."

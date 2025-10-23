# 1) Penjelasan singkat: "Universal tables" (apa maksudnya)

Dalam konteks database aplikasi kafe/perpustakaan kecil seperti skenario *Caffe Kenangan*, **universal tables** = tabel-tabel generik/umum yang dipakai ulang di banyak modul untuk menghindari duplikasi. Contoh: `person`, `address`, `contact`, `status_lookup`, `payment_method`, `category_lookup`.
Manfaat: konsistensi data, lebih gampang maintain, mempermudah relasi (FK) ke satu sumber kebenaran.

**Contoh universal tables yang kubuat untuk skenario ini**

* `person` (menyimpan semua orang: pelanggan, pegawai)
* `contact` (telepon, email—bisa banyak per person)
* `role` (role pegawai: barista, kasir, manager)
* `payment_method` (tunai, kartu, e-wallet)
* `status_lookup` (status reservasi, status voucher, status buku)
* `category_lookup` (kategori menu, genre buku, tipe voucher)

---

# 2) Daftar entitas (improved), atribut kunci, hubungan — versi *lebih rapi*

Aku mempertahankan entitas yang kamu buat dan menambahkan beberapa untuk kelengkapan + universalization.

## Entitas utama (unnormalized list — langsung sesuai skenario)

1. **person** (universal)

   * person_id (PK)
   * name
   * person_type (enum: CUSTOMER, EMPLOYEE) — untuk generalisasi
   * notes

2. **customer** (spesialisasi person) — *generalization dari person*

   * customer_id (PK, FK → person.person_id)
   * prefer_contact_note (opsional)

3. **member** (subtype dari customer)

   * member_id (PK, FK → customer.customer_id)
   * email
   * join_date
   * member_number
   * promo_eligible (bool)

4. **employee** (spesialisasi person)

   * employee_id (PK, FK → person.person_id)
   * username
   * password_hash
   * role_id (FK → role.role_id)
   * address (atau FK → address jika memakai universal address)

5. **role** (universal lookup)

   * role_id (PK)
   * role_name (BARISTA, CASHIER, MANAGER, LIBRARIAN)

6. **menu_item**

   * menu_id (PK)
   * name
   * price
   * description
   * category_id (FK → category_lookup)
   * availability_status_id (FK → status_lookup)
   * image_url

7. **reservation**

   * reservation_id (PK)
   * customer_id (FK → customer.customer_id)
   * reservation_date (date)
   * reservation_time (time)
   * party_size
   * note
   * status_id (FK → status_lookup)
   * handled_by_employee_id (FK → employee.employee_id, nullable)

8. **payment**

   * payment_id (PK)
   * customer_id (FK → customer.customer_id)
   * employee_id (FK → employee.employee_id) — kasir yg input
   * payment_date_time (timestamp)
   * payment_method_id (FK → payment_method.payment_method_id)
   * total_amount
   * voucher_code (FK → voucher.code, nullable)

9. **payment_line** (agar pembayaran bisa memuat beberapa item) — *aggregates menu_item*:

   * payment_line_id (PK)
   * payment_id (FK → payment.payment_id)
   * menu_id (FK → menu_item.menu_id)
   * qty
   * price_each

10. **voucher**

    * code (PK)
    * description
    * voucher_type_id (FK → category_lookup maybe)
    * discount_value (nominal atau persen)
    * valid_from, valid_to
    * status_id (FK → status_lookup)

11. **event**

    * event_id (PK)
    * name
    * start_date, end_date
    * description

12. **event_participant** (M:N between customer and event)

    * event_id (FK)
    * customer_id (FK)
    * registration_date

13. **book**

    * isbn (PK)
    * title
    * author
    * publisher
    * year_published
    * genre_id (FK → category_lookup)
    * copies_total

14. **book_loan** (peminjaman buku)

    * loan_id (PK)
    * isbn (FK → book.isbn)
    * customer_id (FK → customer.customer_id)
    * loan_date
    * due_date
    * return_date (nullable)
    * status_id (FK → status_lookup)

15. **feedback** (`kritik_dan_saran`)

    * feedback_id (PK)
    * customer_id (FK)
    * content
    * created_at

16. **audit_log** (universal, optional)

    * log_id, actor_person_id, action, target_table, target_id, timestamp

17. **lookup tables**: `payment_method`, `status_lookup`, `category_lookup` — seperti dijelaskan di atas.

---

# 3) Mana yang **Generalization (IS-A)** dan mana yang **Aggregation (HAS-A)**

## Generalization / Inheritance (IS-A)

* `person` **generalizes** → `customer`, `employee`
  Penjelasan: `customer` dan `employee` adalah *jenis* dari `person`. Gunakan table inheritance atau separate tables dengan FK ke `person`.

* `customer` **generalizes** → `member`
  Penjelasan: semua `member` adalah `customer` tapi tidak semua customer member → pewarisan.

## Aggregation / Composition (HAS-A)

* `payment` **aggregates** `payment_line` (payment HAS MANY payment_line) — *agregasi transaksi*
* `payment_line` **references** `menu_item` (line item mengandung menu item)
* `reservation` **aggregates** `customer` dan dapat `handled_by` `employee` (reservation HAS-A customer, handled by employee)
* `event` **aggregates** banyak `customer` melalui `event_participant` (event HAS MANY participants)
* `book_loan` **aggregates** `book` dan `customer` (loan HAS-A book and customer)
* `employee` **can manage** many `menu_item` (not strict aggregation, lebih ke relationship M:N jika beberapa pegawai buat/ubah menu) — bisa pakai junction `employee_menu` jika perlu.

---

# 4) ER Diagram (textual/ASCII, unnormalized → cocok dicetak/gambar)

Keterangan cardinality singkat: 1, 1..*, 0..*.

```
[person] 1---< (is) ---1 [customer] 1---< (is member maybe) ---0..1 [member]
         \
          \---< (is) ---1 [employee]

[customer] 1---< (makes) ---0..* [reservation]  *---1 [status_lookup]
[reservation] *---0..1 handled_by ---1 [employee]

[payment] PK payment_id
[customer] 1---< (pays) ---0..* [payment]
[employee] 1---< (recorded_by) ---0..* [payment]
[payment] 1---< (contains) ---1..* [payment_line] >---1 [menu_item]

[menu_item] *---0..* managed_by ---* [employee]   (option: employee_menu junction)

[customer] *---0..* participates ---* [event]  (via event_participant)

[customer] *---0..* borrows ---* [book]  (via book_loan)

[customer] 1---< (leaves) ---0..* [feedback]

Lookup tables: [role], [payment_method], [status_lookup], [category_lookup], [payment_type] etc referenced by FK.

```

> Catatan: ini adalah *unnormalized ER* logical view yang menunjukkan hubungan dan entitas. Untuk diagram visual, tiap kotak entitas dan garis cardinality.

---

# 5) Normalisasi — langkah demi langkah (contoh skema dan alasan)

Aku mulai dari satu tabel besar *sebelum normalisasi* (mirip PDF) lalu progresif per NF.

## A. **Unnormalized (Sebelum)** — contoh tabel `Pembayaran_un` (satu tabel penuh field)

Contoh kolom:
`payment_un`( payment_id, customer_name, customer_phone, payment_date, total_amount, menu_items (text garbled repeating), voucher_code, employee_name, employee_role, ...)

Masalah: ada nilai berulang (menu_items), data non-atomic, duplikasi person info.

---

## B. **1NF (First Normal Form)** — *atomisasi & tidak ada repeating groups*

Prinsip: setiap kolom harus atomic; tidak ada kolom berisi list.

**Contoh tabel setelah 1NF (subset)**

* `payment`( payment_id PK, customer_id FK, employee_id FK, payment_datetime, total_amount, payment_method_id, voucher_code )
* `payment_line`( payment_line_id PK, payment_id FK, menu_id FK, qty, price_each )

Untuk pelanggan dan pegawai:

* `person`( person_id PK, name, person_type )
* `customer`( customer_id PK FK→person.person_id, contact_pref )
* `employee`( employee_id PK FK→person.person_id, username, password_hash, role_id )

Semua repeating groups (menu items list) dipindah ke `payment_line`. Semua atribut atomic.

---

## C. **2NF (Second Normal Form)** — *hapus partial dependencies (khusus PK composite)*

Prinsip: jika tabel punya composite PK, atribut non-PK harus tergantung penuh pada PK, bukan sebagian.

Contoh masalah awal: jika kita menyimpan `menu_name` di `payment_line` dan `payment_line` PK adalah (payment_id, menu_id) — `menu_name` tergantung ke `menu_id` saja (partial). Jadi **pindahkan** `menu_name` ke `menu_item`.

**Skema 2NF (perubahan penting):**

* `menu_item`( menu_id PK, name, price, category_id ) — atribut terkait menu.

* `payment_line`( payment_id FK, menu_id FK, qty, price_each ) — PK composite (payment_id, menu_id) atau buat payment_line_id PK tunggal.

  * Tidak menyimpan `menu_name` atau `category` di sini.

* `customer` — tidak menyimpan promo/piece yang tergantung pada membership; buat `member_promo` kalau perlu.

Intinya: semua attribut yang hanya bergantung pada sebagian PK dipindah ke tabel tempat ketergantungan penuh.

---

## D. **3NF (Third Normal Form)** — *hapus transitive dependencies*

Prinsip: Tidak boleh ada A → B → C; semua non-key harus bergantung langsung ke PK, bukan melalui atribut non-key lain.

Contoh masalah: kita menyimpan `customer_name` dan `customer_email` di `payment`. Itu menyebabkan transitive dependency (payment → customer_id → customer_name). Solusi: simpan hanya `customer_id` FK di `payment`. `customer_name` berada di `person`/`customer`.

**Skema 3NF final (ringkasan utama)**

* `person`( person_id PK, name, dob?, notes )
* `customer`( customer_id PK FK→person.person_id, default_contact_id?)
* `member`( member_id PK FK→customer.customer_id, email, join_date, member_number, promo_eligible )
* `employee`( employee_id PK FK→person.person_id, username, password_hash, role_id )
* `role`( role_id PK, role_name )
* `menu_item`( menu_id PK, name, price, description, category_id, availability_status_id )
* `category_lookup`( category_id PK, category_name )
* `status_lookup`( status_id PK, status_name, status_type )
* `payment_method`( method_id PK, method_name )
* `payment`( payment_id PK, customer_id FK, employee_id FK, datetime, payment_method_id FK, total_amount, voucher_code FK )
* `payment_line`( payment_line_id PK, payment_id FK, menu_id FK, qty, price_each )
* `voucher`( code PK, description, discount_value, valid_from, valid_to, status_id FK )
* `reservation`( reservation_id PK, customer_id FK, date, time, party_size, note, status_id FK, handled_by_employee_id FK )
* `event`( event_id PK, name, start_date, end_date, description )
* `event_participant`( event_id FK, customer_id FK, registration_date, PK composite )
* `book`( isbn PK, title, author, publisher, year, genre_id FK, copies_total )
* `book_loan`( loan_id PK, isbn FK, customer_id FK, loan_date, due_date, return_date, status_id FK )
* `feedback`( feedback_id PK, customer_id FK, content, created_at )
* `audit_log`( log_id PK, actor_person_id FK, action, target_table, target_id, timestamp )

Semua atribut non-key bergantung langsung pada PK tabel masing-masing; tidak ada duplikasi informasi person di tabel transaksi; voucher dan status disimpan terpisah.

---

# 6) Contoh transformasi singkat (dari UN → 1NF → 2NF → 3NF) untuk **Pembayaran + Menu Items**

**Unnormalized (satu row menyimpan list menu):**

```
payment_un(payment_id, customer_name, items)
items = "Caramel Macchiato x1; Hazelnut Latte x2"
```

**1NF:** pemecahan items → `payment` + `payment_line`

```
payment(payment_id, customer_id, payment_datetime, total_amount, voucher_code)
payment_line(payment_line_id, payment_id, menu_id, qty, price_each)
```

**2NF:** pastikan `menu_name` dipindah ke `menu_item` (karena tergantung pada menu_id)

```
menu_item(menu_id, name, price)
payment_line(payment_line_id, payment_id, menu_id, qty, price_each)
```

**3NF:** pastikan `customer` info hanya di `customer`/`person`, bukan di payment

```
payment(payment_id, customer_id, employee_id, datetime, total_amount)
person(person_id, name, ...)
customer(customer_id FK->person)
```

---

# 7) Petunjuk singkat untuk menggambar ERD (untuk tugas)

* Kotakkan entitas utama; tulis PK di baris pertama, atribut non-key di bawahnya.
* Tanda panah/garis: gunakan crow's foot untuk cardinality (1, 1..*, 0..*).
* Tandai generalisasi: garis bertiga (triangle) atau turunan dari `person` ke `customer` & `employee`.
* Tandai agregasi: connection dengan diamond atau cukup beri label hubungan (HAS_A / CONTAINS).
* Keterangan: beri label FK di atribut yang merupakan FK.

---

# 8) Ringkasan: mana yang bisa di-aggregate / digeneralisasi (singkat)

* **Generalize (IS-A):**

  * `person` → `customer`, `employee`
  * `customer` → `member`
* **Aggregate (HAS-A / composition):**

  * `payment` HAS MANY `payment_line` (aggregates menu_item via payment_line)
  * `reservation` HAS-A `customer` and optional handled_by `employee`
  * `event` HAS MANY `event_participant` (which references `customer`)
  * `book_loan` aggregates `book` and `customer`
  * `menu_item` may be managed/created by many `employee` (M:N) — treat sebagai relationship, tidak inheritance.

---

# 9) Contoh DDL ringkas (3 tabel utama contoh, 3NF) — buat cepat kalau perlu:

> Catatan: ini contoh minimal; sesuaikan tipe data dan constraints sesuai RDBMS.

```sql
CREATE TABLE person (
  person_id SERIAL PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  person_type VARCHAR(20)
);

CREATE TABLE customer (
  customer_id INT PRIMARY KEY REFERENCES person(person_id)
);

CREATE TABLE employee (
  employee_id INT PRIMARY KEY REFERENCES person(person_id),
  username VARCHAR(50) UNIQUE,
  password_hash VARCHAR(255),
  role_id INT REFERENCES role(role_id)
);

CREATE TABLE menu_item (
  menu_id SERIAL PRIMARY KEY,
  name VARCHAR(150),
  price DECIMAL(12,2),
  category_id INT REFERENCES category_lookup(category_id),
  availability_status_id INT REFERENCES status_lookup(status_id)
);

CREATE TABLE payment (
  payment_id SERIAL PRIMARY KEY,
  customer_id INT REFERENCES customer(customer_id),
  employee_id INT REFERENCES employee(employee_id),
  payment_datetime TIMESTAMP,
  payment_method_id INT REFERENCES payment_method(method_id),
  total_amount DECIMAL(12,2),
  voucher_code VARCHAR(50) REFERENCES voucher(code)
);

CREATE TABLE payment_line (
  payment_line_id SERIAL PRIMARY KEY,
  payment_id INT REFERENCES payment(payment_id),
  menu_id INT REFERENCES menu_item(menu_id),
  qty INT,
  price_each DECIMAL(12,2)
);
```

---

# 10) Saran implementasi & prioritas kerja untuk tugasmu (mommy tips)

1. **Buat universal `person` dulu** → ini memudahkan generalization.
2. **Buat lookup tables** (`status`, `payment_method`, `category`) → memudahkan perubahan UI/logic.
3. **Pisahkan transaksi** (`payment` + `payment_line`) → penting untuk laporan/akuntansi.
4. **Tangani member sebagai tabel terpisah** (promo, diskon) agar fleksibel.
5. **Buat constraint & indeks**: FK, unique (member_number), index pada foreign keys untuk performa.
6. **ER Diagram visual**: setelah skemanya oke, gambarkan ERD (draw.io / diagrams.net / dbdiagram.io).

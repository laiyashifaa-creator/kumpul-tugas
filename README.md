# Monolith vs Microservices dengan Flask Python

Praktikum membandingkan dua arsitektur pada toko buku sederhana (katalog buku dan pesanan):

- **Monolith**: semua fitur dalam satu file dan satu proses (port 5000).
- **Microservices**: dipecah menjadi **Book Service** (port 5001) dan **Order Service** (port 5002) yang berkomunikasi lewat HTTP.

Berdasarkan modul *Hands-on: Arsitektur Monolith vs Microservices dengan Flask Python*.

## Struktur Project

```text
monolith-microservices/
├── monolith_app.py
└── microservice/
    ├── book_service.py
    └── order_service.py
```

| File | Fungsi | Port |
|------|--------|------|
| `monolith_app.py` | Fitur buku dan pesanan dalam satu aplikasi | 5000 |
| `microservice/book_service.py` | Book Service: `GET /books`, `GET /books/<id>` | 5001 |
| `microservice/order_service.py` | Order Service: `POST /orders`, memanggil Book Service lewat HTTP | 5002 |

Data disimpan di memori (variabel Python), jadi hilang saat aplikasi dihentikan.

## Prasyarat

- Python terinstal (praktikum ini berjalan dengan Python 3.14.3).
- Terminal PowerShell (perintah di bawah memakai PowerShell; modul memakai `curl`).

Pasang pustaka yang dibutuhkan:

```powershell
pip install Flask requests
```

## Langkah Kerja

### Bagian 1: Aplikasi Monolith

**1. Buat file `monolith_app.py`** dengan isi berikut:

```python
from flask import Flask, jsonify, request

app = Flask(__name__)

# Database bohongan (In-memory)
books = [{"id": 1, "title": "Belajar Flask", "stock": 5}]
orders = []

# --- FITUR BUKU ---
@app.route('/books', methods=['GET'])
def get_books():
    return jsonify(books)

# --- FITUR PESANAN ---
@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    book_id = data.get('book_id')

    # Cek stok buku
    for b in books:
        if b['id'] == book_id and b['stock'] > 0:
            b['stock'] -= 1

            order = {
                "id": len(orders) + 1,
                "book_id": book_id,
                "status": "berhasil"
            }

            orders.append(order)
            return jsonify(order), 201

    return jsonify({
        "error": "Buku tidak ditemukan atau stok habis"
    }), 400


if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

**2. Jalankan aplikasi** dari folder project:

```powershell
python monolith_app.py
```

Server berjalan di `http://127.0.0.1:5000`.

**3. Cek daftar buku** (buka terminal baru):

```powershell
curl http://localhost:5000/books
```

Hasil yang diharapkan: status `200 OK` dengan data buku `Belajar Flask` (stok 5).

**4. Buat pesanan:**

```powershell
Invoke-WebRequest -Uri "http://localhost:5000/orders" `
  -Method POST `
  -ContentType "application/json" `
  -Body '{"book_id":1}'
```

Hasil yang diharapkan: status `201 CREATED` dengan isi `{"book_id": 1, "id": 1, "status": "berhasil"}`.

> Alternatif sesuai modul (terminal yang mendukung `curl` penuh):
> `curl -X POST -H "Content-Type: application/json" -d '{"book_id": 1}' http://localhost:5000/orders`

Hentikan aplikasi dengan `Ctrl+C` sebelum lanjut ke Bagian 2.

### Bagian 2: Aplikasi Microservices

Buat folder `microservice`, lalu buat dua file di dalamnya.

**1. Buat file `microservice/book_service.py`:**

```python
from flask import Flask, jsonify

app = Flask(__name__)

# Database khusus Book Service
books = [
    {
        "id": 1,
        "title": "Belajar Flask",
        "stock": 5
    }
]

@app.route('/books', methods=['GET'])
def get_books():
    return jsonify(books)

@app.route('/books/<int:book_id>', methods=['GET'])
def get_book(book_id):
    for b in books:
        if b['id'] == book_id:
            return jsonify(b)

    return jsonify({"error": "Not found"}), 404


if __name__ == '__main__':
    app.run(port=5001, debug=True)
```

**2. Buat file `microservice/order_service.py`:**

```python
from flask import Flask, jsonify, request
import requests

app = Flask(__name__)

orders = []

BOOK_SERVICE_URL = "http://localhost:5001"


@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    book_id = data.get('book_id')

    # Tanya Book Service
    try:
        response = requests.get(
            f"{BOOK_SERVICE_URL}/books/{book_id}"
        )

        if response.status_code == 200:
            book_data = response.json()

            if book_data['stock'] > 0:
                order = {
                    "id": len(orders) + 1,
                    "book_id": book_id,
                    "status": "berhasil"
                }

                orders.append(order)

                return jsonify(order), 201

        return jsonify({
            "error": "Buku tidak tersedia"
        }), 400

    except requests.exceptions.ConnectionError:
        return jsonify({
            "error": "Book Service sedang down!"
        }), 500


if __name__ == '__main__':
    app.run(port=5002, debug=True)
```

**3. Terminal 1: jalankan Book Service**

```powershell
cd microservice
python book_service.py
```

Berjalan di `http://127.0.0.1:5001`.

**4. Terminal 2: jalankan Order Service**

```powershell
cd microservice
python order_service.py
```

Berjalan di `http://127.0.0.1:5002`.

**5. Terminal 3: buat pesanan ke Order Service (port 5002)**

```powershell
curl.exe -X POST -H "Content-Type: application/json" --data-raw '{\"book_id\":1}' http://localhost:5002/orders
```

Hasil yang diharapkan:

```json
{
  "book_id": 1,
  "id": 1,
  "status": "berhasil"
}
```

Pada terminal Book Service akan tampak log permintaan `GET /books/1` dengan status 200. Itu bukti Order Service menanyakan data buku ke Book Service lewat HTTP.

### Eksperimen Kegagalan (Fault Isolation)

**1.** Hentikan Book Service: tekan `Ctrl+C` di Terminal 1.

**2.** Kirim ulang pesanan ke Order Service:

```powershell
$body = '{"book_id":1}'
Invoke-RestMethod -Uri "http://localhost:5002/orders" -Method POST -ContentType "application/json" -Body $body
```

**Hasil yang diamati:** Order Service tetap berjalan dan membalas pesan error:

```json
{"error": "Book Service sedang down!"}
```

PowerShell juga menampilkan informasi exception karena respons berupa error. Order Service tidak ikut mati meskipun Book Service berhenti; inilah yang disebut *fault isolation*.

## Ringkasan Perbedaan yang Teramati

| Aspek | Monolith | Microservices |
|-------|----------|---------------|
| Jumlah file program | 1 | 2 |
| Proses dan port | 1 server, port 5000 | 2 server, port 5001 dan 5002 |
| Pesanan memeriksa buku | Membaca variabel `books` langsung | HTTP request ke Book Service |
| Saat Book Service dihentikan | Tidak ada layanan terpisah (tidak diuji) | Order Service membalas "Book Service sedang down!" |

Latensi, scale-up, dan skenario crash pada monolith **tidak diuji** pada praktikum ini.


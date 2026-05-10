# EduResolve AI

EduResolve AI adalah aplikasi customer support berbasis AI untuk platform EdTech. Sistem ini membantu mahasiswa mengirim keluhan atau pertanyaan, lalu membantu tim customer support memprioritaskan tiket, memahami konteks masalah, membalas percakapan, dan melihat ringkasan analitik dari keluhan yang masuk.

Project ini terdiri dari dua aplikasi utama:

- `frontend/`: aplikasi web Next.js untuk halaman publik, autentikasi, dashboard mahasiswa, dan dashboard customer support.
- `backend/`: REST API berbasis Go + Gin yang menangani autentikasi, role guard, penyimpanan percakapan, analisis AI, saran balasan, dan analytics.

## Tujuan Project

Masalah utama yang diselesaikan project ini adalah antrean support di platform pembelajaran online. Tidak semua keluhan punya urgensi yang sama: masalah pembayaran, akses materi, error tugas/ujian, atau kendala teknis bisa membutuhkan prioritas berbeda.

EduResolve AI membuat proses itu lebih terstruktur:

1. Mahasiswa mengirim masalah dari dashboard.
2. Backend langsung meminta Gemini menganalisis isi keluhan.
3. Hasil analisis disimpan bersama tiket di Firestore.
4. Customer support melihat daftar tiket yang sudah diurutkan berdasarkan prioritas AI.
5. Support dapat membuka detail percakapan, melihat ringkasan AI, meminta saran balasan, lalu membalas mahasiswa.
6. Dashboard analytics membaca data percakapan untuk menampilkan distribusi isu, rata-rata prioritas, dan jumlah tiket harian.

## Tech Stack

- Frontend: Next.js 16, React 19, TypeScript, Tailwind CSS, Firebase Web SDK.
- Backend: Go, Gin, Firebase Admin SDK, Firestore, Gemini API.
- Auth: Firebase Authentication.
- Database: Cloud Firestore.
- AI: Gemini `gemini-2.5-flash`.

## Peran User

Sistem memakai role yang disimpan di collection `users` Firestore.

- `customer`: mahasiswa atau learner. Role ini bisa mengirim keluhan, melihat daftar percakapan miliknya sendiri, membuka detail percakapan miliknya, dan membalas customer support.
- `customer_support`: agent support. Role ini bisa melihat semua percakapan, mengurutkan tiket, membuka detail tiket, meminta saran balasan AI, membalas mahasiswa, dan melihat analytics.

Setiap request yang masuk ke endpoint terproteksi harus membawa Firebase ID token di header:

```http
Authorization: Bearer <firebase_id_token>
```

Backend memverifikasi token tersebut dengan Firebase Admin SDK, lalu membaca role user dari Firestore.

## Alur Sistem

### 1. Registrasi dan Login

Frontend menggunakan Firebase Authentication untuk login/register dengan email/password atau Google.

Saat register:

1. User membuat akun melalui Firebase Auth.
2. Frontend mengambil Firebase ID token.
3. Frontend mengirim token, nama, email, dan role ke `POST /api/v1/auth/register`.
4. Backend memverifikasi token.
5. Backend menyimpan profil user ke collection `users` di Firestore.

Saat login:

1. User login melalui Firebase Auth.
2. Frontend mengambil Firebase ID token.
3. Frontend mengirim token ke `POST /api/v1/auth/login`.
4. Backend memverifikasi token dan mengambil profil user dari Firestore.
5. Frontend mengarahkan user ke dashboard sesuai role.

### 2. Mahasiswa Mengirim Keluhan

Keluhan dibuat dari halaman `frontend/app/customer/dashboard/ask/page.tsx`.

Alurnya:

1. Mahasiswa memilih kategori masalah dan menulis detail keluhan.
2. Frontend mengirim request ke `POST /api/v1/student/complaints`.
3. Backend menjalankan `AuthMiddleware` untuk memverifikasi Firebase token.
4. Backend menjalankan `RequireRole("customer")` agar hanya mahasiswa yang bisa membuat tiket.
5. Backend memanggil `AIService.ProcessComplaint`.
6. `AIService` mengirim prompt ke Gemini untuk menghasilkan JSON berisi:
   - `summary`
   - `category`
   - `priority_score`
   - `reason`
   - `sentiment`
   - `is_processed`
7. Backend membuat dokumen baru di collection `conversations`.
8. Response berisi `ticket_id` dikirim kembali ke frontend.

Jika analisis AI gagal, backend tetap membuat tiket dengan fallback analysis:

- `priority_score`: `5`
- `category`: `Uncategorized`
- `summary`: `AI Analysis Failed`
- `is_processed`: `false`

### 3. Penyimpanan Percakapan

Data utama sistem ada di collection `conversations`.

Struktur data percakapan:

```ts
{
  id: string,
  student_id: string,
  student_name: string,
  student_email: string,
  last_message: string,
  status: "open" | "in_progress" | "resolved",
  messages: [
    {
      sender: "student" | "support",
      text: string,
      timestamp: Date
    }
  ],
  ai_analysis: {
    summary: string,
    category: string,
    priority_score: number,
    reason: string,
    sentiment: string,
    is_processed: boolean
  },
  created_at: Date,
  updated_at: Date
}
```

`messages` menyimpan histori chat. `last_message`, `status`, dan `updated_at` dipakai untuk tampilan daftar tiket dan sorting.

### 4. Mahasiswa Melihat dan Membalas Tiket

Mahasiswa hanya bisa melihat tiket miliknya sendiri.

Alurnya:

1. Frontend meminta daftar tiket ke `GET /api/v1/student/conversations`.
2. Backend membaca `user_id` dari token Firebase.
3. Backend query Firestore dengan filter `student_id == user_id`.
4. Backend mengurutkan hasil berdasarkan `updated_at` terbaru.
5. Saat membuka detail, frontend memanggil `GET /api/v1/student/conversations/:id`.
6. Backend memastikan `student_id` pada dokumen sama dengan user yang sedang login.
7. Jika mahasiswa membalas, frontend memanggil `POST /api/v1/student/conversations/:id/reply`.
8. Backend menambahkan message baru ke array `messages` dengan `firestore.ArrayUnion`.

### 5. Customer Support Mengelola Inbox

Dashboard support berada di `frontend/app/agent/dashboard/page.tsx`.

Alurnya:

1. Frontend memanggil `GET /api/v1/conversations`.
2. Backend memverifikasi token dan role `customer_support`.
3. Backend mengambil semua percakapan dari Firestore.
4. Default sorting memakai `ai_analysis.priority_score` descending, sehingga tiket paling urgent muncul di atas.
5. Support bisa mengganti sorting ke `updated_at`.
6. Dashboard menghitung statistik lokal seperti total open, in progress, resolved, critical, dan rata-rata priority.

Endpoint ini menerima query:

```http
GET /api/v1/conversations?sort_by=priority_score&order=desc
GET /api/v1/conversations?sort_by=updated_at&order=desc
```

### 6. Detail Percakapan dan Saran Balasan AI

Saat support membuka detail percakapan:

1. Frontend memanggil `GET /api/v1/conversations/:id`.
2. Backend mengambil dokumen percakapan dari Firestore.
3. Frontend menampilkan chat history dan hasil `ai_analysis`.

Untuk saran balasan:

1. Support menekan tombol generate AI.
2. Frontend memanggil `GET /api/v1/conversations/:id/suggestions`.
3. Backend mengambil percakapan dari Firestore.
4. Backend mengirim `last_message` ke Gemini melalui `AIService.GenerateSuggestions`.
5. Gemini diminta menghasilkan dua draft balasan dalam bahasa Indonesia:
   - formal
   - empathetic
6. Frontend menampilkan saran tersebut agar support bisa memilih dan mengeditnya sebelum dikirim.

Saat support membalas:

1. Frontend mengirim `POST /api/v1/conversations/:id/reply`.
2. Backend menambahkan message dengan sender `support`.
3. Backend memperbarui:
   - `messages`
   - `last_message`
   - `updated_at`
   - `status` menjadi `in_progress`

### 7. Analytics

Endpoint analytics tersedia untuk role `customer_support`.

```http
GET /api/v1/analytics/overview
```

Backend membaca semua dokumen `conversations`, lalu menghitung:

- distribusi kategori isu dari `ai_analysis.category`
- rata-rata `ai_analysis.priority_score`
- jumlah tiket harian selama 7 hari terakhir berdasarkan `updated_at`

Data ini dipakai untuk dashboard manajerial agar tim support bisa melihat pola masalah yang paling sering muncul.

## Endpoint Utama

Base path backend:

```http
/api/v1
```

Auth:

- `POST /auth/register`
- `POST /auth/login`

Mahasiswa:

- `POST /student/complaints`
- `GET /student/conversations`
- `GET /student/conversations/:id`
- `POST /student/conversations/:id/reply`

Customer support:

- `GET /conversations`
- `GET /conversations/:id`
- `GET /conversations/:id/suggestions`
- `POST /conversations/:id/reply`

Analytics:

- `GET /analytics/overview`

## Struktur Folder

```txt
.
|-- api_contract.json              # Referensi kontrak API project
|-- backend/
|   |-- cmd/api/main.go            # Entry point REST API
|   |-- internal/config/           # Konfigurasi environment
|   |-- internal/handler/          # Handler HTTP
|   |-- internal/middleware/       # Auth dan role middleware
|   |-- internal/model/            # Model data Firestore/API
|   |-- internal/service/          # Business logic AI dan analytics
|   `-- pkg/
|       |-- ai/                    # Inisialisasi Gemini
|       `-- firebase/              # Inisialisasi Firebase Admin
`-- frontend/
    |-- app/                       # Next.js App Router
    |-- app/(auth)/                # Login dan register
    |-- app/customer/dashboard/    # Dashboard mahasiswa
    |-- app/agent/dashboard/       # Dashboard support
    |-- app/context/AuthContext.tsx
    |-- lib/api/                   # Client API ke backend
    `-- lib/firebase/              # Firebase Web SDK config
```

## Environment Variables

Backend membutuhkan file `.env` di folder `backend/` atau environment system:

```env
FIREBASE_SERVICE_ACCOUNT_PATH=/path/to/service-account.json
GEMINI_API_KEY=your_gemini_api_key
```

Frontend membutuhkan environment Next.js:

```env
NEXT_PUBLIC_API_URL=http://localhost:8080/api/v1
NEXT_PUBLIC_FIREBASE_API_KEY=...
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=...
NEXT_PUBLIC_FIREBASE_PROJECT_ID=...
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=...
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=...
NEXT_PUBLIC_FIREBASE_APP_ID=...
```

Catatan: sebagian file frontend masih memiliki fallback URL lama `http://localhost:8081/api/v1`, sedangkan backend saat ini berjalan di `:8080`. Gunakan `NEXT_PUBLIC_API_URL=http://localhost:8080/api/v1` agar frontend mengarah ke backend lokal yang benar.

## Cara Menjalankan Lokal

Jalankan backend:

```bash
cd backend
go run ./cmd/api
```

Backend akan berjalan di:

```txt
http://localhost:8080
```

Jalankan frontend:

```bash
cd frontend
npm install
npm run dev
```

Frontend akan berjalan di:

```txt
http://localhost:3000
```

## Ringkasan Alur End-to-End

```txt
User login/register
    |
    v
Firebase Auth menerbitkan ID token
    |
    v
Frontend mengirim token ke backend
    |
    v
Backend verifikasi token dan role via Firebase Admin + Firestore
    |
    v
Mahasiswa mengirim keluhan
    |
    v
Backend memanggil Gemini untuk analisis keluhan
    |
    v
Backend menyimpan conversation + AI analysis ke Firestore
    |
    v
Support melihat inbox yang diprioritaskan oleh AI
    |
    v
Support membuka detail, meminta saran balasan AI, lalu membalas
    |
    v
Mahasiswa melihat balasan di dashboard
    |
    v
Analytics membaca data conversations untuk insight operasional
```

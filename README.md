# Semantic Search & LLM Application: FAQ E-commerce

## 1. Problem Statement

**Objective:** Tim customer service e-commerce menerima banyak pertanyaan repetitif
(pemesanan, pengiriman, pembayaran, pengembalian). Proyek ini membangun prototype asisten AI
yang mencari entri FAQ paling relevan secara semantik (walau kata-kata pelanggan berbeda)
lalu menyusun jawaban natural dengan LLM berdasarkan FAQ yang ditemukan.

**Knowledge base:** `data/ecommerce_faq_knowledge_base.csv`
- 59 entri, 8 kategori (Pemesanan, Pengiriman, Pembayaran, Pengembalian, Akun, Produk, Promo, Customer Service)
- Kolom: `faq_id`, `category`, `question`, `answer`
- Tidak ada nilai kosong. File asli tidak diubah.

**Batasan dan asumsi:**
- Sistem hanya menjawab berdasarkan FAQ yang tersedia. Jika informasi tidak ada, sistem menyatakannya.
- Satu entri FAQ = satu unit retrievable (tanpa chunking).

## 2. Alur Sistem

Ingest → Embed → Store → Retrieve → Generate

1. **Ingest:** CSV dimuat dengan Pandas. Kategori, pertanyaan, dan jawaban digabung menjadi satu teks per entri.
2. **Embed:** model `gemini-embedding-001`, 768 dimensi. Task type `RETRIEVAL_DOCUMENT` untuk FAQ
   dan `RETRIEVAL_QUERY` untuk pertanyaan pelanggan. Vektor dinormalisasi (L2).
3. **Store:** FAISS `IndexFlatIP` (inner product pada vektor ternormalisasi = cosine similarity).
   Jumlah vektor diverifikasi sama dengan jumlah entri (59). Index disimpan di `data/faq.index`.
4. **Retrieve:** query di-embed lalu dicari top-k (k=3) FAQ terdekat beserta skor kemiripan.
5. **Generate:** hasil retrieval disisipkan ke prompt terstruktur (Persona, Task, Context, Format)
   dan dikirim ke `gemini-2.5-flash` (temperature 0.2). Prompt menginstruksikan model menjawab
   hanya dari context dan mengakui jika informasi tidak tersedia.

## 3. Hasil Pengujian dan Analisis

| Tipe | Query | FAQ top-1 | Skor | Retrieval | Jawaban AI |
|---|---|---|---|---|---|
| Jelas ada #1 | Bagaimana cara membatalkan pesanan yang sudah dibuat? | Cara membatalkan pesanan | 0.782 | Akurat | Sesuai FAQ |
| Jelas ada #2 | Apakah saya bisa mengubah alamat pengiriman sesuai dengan yang saya inginkan? | Ubah alamat setelah pesanan dibuat | 0.791 | Akurat | Sesuai FAQ, tetapi dibuka dengan "Tentu" sebelum menyebut syarat |
| Parafrase #1 | Paket saya dimana, ko nda nyampe? | Paket belum sampai melebihi estimasi | 0.758 | Akurat | Relevan, tetapi menggabungkan dua FAQ sehingga aturan waktu agak tercampur |
| Parafrase #2 | Kalau barangnya rusak, uangnya bisa balik nggak? | Prosedur pengembalian barang tidak sesuai | 0.738 | Sebagian | Jujur terhadap context, tetapi tidak lengkap |
| Tidak ada di KB | Apakah kalian menjual tiket konser? | Pesanan reseller | 0.620 | Tidak relevan (wajar) | Menolak dengan benar |

**Analisis:**
- Semantic search menangkap makna, bukan hanya kata kunci: bahasa informal ("ko nda nyampe",
  "uangnya balik") tetap menemukan FAQ yang tepat.
- Jawaban pada 3 query pertama diverifikasi manual terhadap isi FAQ di CSV dan sesuai sumbernya.
- Pada Parafrase #2, ada dua FAQ refund di knowledge base yang tidak masuk top-3 (peringkat 8 dari
  hasil top-8). FAQ yang paling dekat maknanya ("paket diterima dalam kondisi rusak") ada di peringkat 7.
  Jadi LLM menjawab jujur sesuai context yang diterimanya, tetapi kegagalannya ada di tahap retrieval.
- Menaikkan k dari 3 ke 5 tidak memperbaiki jawaban. Konteks bertambah, tetapi FAQ yang relevan
  tetap belum masuk.
- Pada query di luar knowledge base, prompt berhasil mencegah jawaban yang dikarang.

## 4. Keterbatasan

- Skor kemiripan berdempetan (query "barang rusak": 0.738 sampai 0.700 pada top-8), sehingga urutan
  rapuh untuk query yang maksudnya gabungan.
- Skor query di luar knowledge base (0.620) hanya selisih sekitar 0.12 dari query yang ada jawabannya.
  Tidak ada ambang batas skor, sehingga pencegahan halusinasi sepenuhnya bergantung pada prompt.
- Top-k selalu mengambil FAQ terdekat walau tidak relevan, dan FAQ itu ikut masuk ke prompt.
- Model kadang menggabungkan beberapa FAQ dan menghilangkan detail (contoh: batas waktu 3 hari
  pada FAQ pengembalian).
- Pengujian hanya 5 query dan satu kasus di luar knowledge base, sehingga belum mewakili semua kasus.
- Ide perbaikan lanjutan (belum diuji): reranking, penulisan ulang query, dan pencarian hybrid.

## 5. Cara Menjalankan

```bash
git clone <url-repository>
cd <nama-folder-repository>
python -m venv .venv
.venv\Scripts\Activate.ps1        # Windows PowerShell (Mac/Linux: source .venv/bin/activate)
pip install -r requirements.txt
```

Buat file `.env` di folder utama:
```
GEMINI_API_KEY=isi_api_key_anda
```

Buka `notebook/semantic_search.ipynb`, pilih kernel `.venv`, lalu **Run All**.
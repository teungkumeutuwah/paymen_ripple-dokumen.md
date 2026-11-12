# BDT-GPI × Ripple Payments Direct

## Dokumen Teknis Perhitungan Biaya Transaksi, Pricing Engine, dan Integrasi Laravel

**Pemilik Infrastruktur:** PT Balai Digital Teknologi\
**Produk Infrastruktur:** BDT-GPI\
**Aplikasi / Use Case Awal:** Ovisito Pay\
**Provider pembayaran lintas negara:** Ripple Payments Direct (subject
to contract, onboarding, corridor enablement, and UAT)\
**Versi dokumen:** 1.0\
**Tanggal:** 12 November 2026\
**Status:** Technical design / implementation blueprint --- bukan
penawaran harga Ripple atau opini hukum.

------------------------------------------------------------------------

## 1. Tujuan Dokumen

Dokumen ini menetapkan rancangan teknis untuk:

1.  Memahami bagaimana biaya transaksi Ripple Payments Direct diperoleh.
2.  Menghitung total biaya pembayaran lintas negara secara transparan.
3.  Memisahkan biaya provider, biaya mitra lokal, biaya internal, pajak,
    dan margin.
4.  Menyimpan quote dan transaksi secara dapat diaudit.
5.  Mengintegrasikan quote dan payment flow ke backend Laravel.
6.  Menyiapkan rekonsiliasi, kontrol risiko, dan UAT sebelum produksi.

Dokumen ini berfokus pada **payout lintas negara** melalui Ripple
Payments Direct. Penerimaan pembayaran kartu, QR, virtual account, atau
metode collection di Indonesia merupakan alur berbeda dan tidak boleh
diasumsikan otomatis disediakan Ripple Payments Direct.

## 2. Ringkasan Eksekutif

### 2.1 Apakah Ripple memiliki satu tarif tetap per transaksi?

Tidak ada satu angka tarif publik yang dapat digunakan sebagai tarif
universal untuk semua transaksi. Biaya aktual bergantung pada koridor
negara, mata uang, payment rail, nilai transaksi, model funding, dan
pengaturan komersial akun BDT-GPI dengan Ripple.

Ripple Payments Direct menyediakan mekanisme **quote sebelum payment**.
Quote dapat memuat:

-   nominal sumber dan nominal tujuan;
-   nilai tukar yang disesuaikan (`adjustedExchangeRate`);
-   biaya (`fees`) dan rincian biaya jika tersedia;
-   pajak atas biaya provider jika berlaku (`taxes`);
-   waktu pembuatan dan kedaluwarsa quote;
-   payment rail yang tersedia.

Dengan demikian, sistem BDT-GPI sebaiknya tidak meng-hardcode biaya
Ripple. Sistem meminta quote aktual dari Ripple, menyimpan respons
mentah secara aman, kemudian menghitung biaya tambahan dan harga jual
berdasarkan kebijakan BDT-GPI.

### 2.2 Prinsip desain

-   **Ripple quote is the source of truth** untuk biaya dan kurs
    provider pada transaksi tertentu.
-   Jangan menganggap contoh angka pada dokumentasi API sebagai tarif
    komersial yang akan diterima BDT-GPI.
-   Jangan menganggap koridor MYR→IDR, SGD→IDR, atau USD→IDR tersedia
    sebelum dikonfirmasi di akun UAT/production dan kontrak Ripple.
-   BDT-GPI menyimpan komponen biaya secara terpisah, bukan hanya satu
    angka `total_fee`.
-   Setiap transaksi harus memiliki quote yang valid dan belum
    kedaluwarsa.
-   Jangan mengulang create-payment secara buta ketika terjadi timeout;
    lakukan inquiry/status check terlebih dahulu.
-   Aktivitas yang termasuk layanan sistem pembayaran harus ditinjau
    bersama penasihat hukum dan/atau Bank Indonesia. BDT-GPI tidak boleh
    melakukan custody, transfer, atau settlement dana tanpa dasar izin
    dan struktur mitra yang sesuai.

## 3. Ruang Lingkup dan Batasan

### Termasuk

-   Ripple Payments Direct API quote flow.
-   Kalkulasi biaya dan margin.
-   Model data dan penyimpanan quote.
-   Kontrak API internal BDT-GPI.
-   Desain integrasi Laravel.
-   Ledger, audit trail, idempotency, webhook, dan rekonsiliasi.
-   Simulasi biaya transaksi.

### Tidak otomatis termasuk

-   Tarif komersial Ripple yang dinegosiasikan.
-   Konfirmasi koridor Indonesia atau payment rail tertentu.
-   Akses production Ripple.
-   Ketersediaan collection/payin lokal di Indonesia.
-   Lisensi PJP/PIP atau izin lain.
-   Keputusan akuntansi dan pajak final.
-   Custody atau pengelolaan dana pelanggan oleh BDT.

Semua hal di atas harus dikonfirmasi berdasarkan dokumen resmi, kontrak,
dan hasil onboarding.

## 4. Terminologi

  -----------------------------------------------------------------------
  Istilah                             Definisi
  ----------------------------------- -----------------------------------
  Source amount                       Jumlah dana dalam mata uang asal
                                      yang dibayarkan/didanai.

  Destination amount                  Jumlah dana dalam mata uang tujuan
                                      yang diterima beneficiary.

  Source currency                     Mata uang asal, misalnya MYR atau
                                      USD.

  Destination currency                Mata uang tujuan, misalnya IDR.

  Corridor                            Jalur pembayaran antarnegara dan
                                      mata uang.

  Payment rail                        Metode/jaringan payout yang
                                      digunakan di negara tujuan.

  Quote                               Penawaran biaya, kurs, dan hasil
                                      pembayaran untuk transaksi
                                      tertentu.

  Quote collection                    Kumpulan quote yang dikembalikan
                                      untuk permintaan quote.

  Provider fee                        Biaya yang ditampilkan Ripple dalam
                                      respons quote.

  Local partner fee                   Biaya bank/mitra payout/mitra
                                      berizin di sisi lokal, bila
                                      terpisah.

  BDT service fee                     Biaya layanan yang ditetapkan
                                      BDT-GPI sesuai kontrak dan aturan
                                      yang berlaku.

  FX margin                           Margin kurs yang ditambahkan pada
                                      harga jual, jika diperbolehkan dan
                                      diungkapkan.

  Prefunding                          Dana yang ditempatkan terlebih
                                      dahulu pada ledger/balance
                                      provider.

  Credit funding                      Pembayaran yang diselesaikan
                                      melalui invoice/credit arrangement.

  JIT funding                         Dana dikirim ke rekening/ledger
                                      provider dalam batas waktu tertentu
                                      setelah payment dibuat.
  -----------------------------------------------------------------------

## 5. Cara Kerja Biaya Ripple Payments Direct

### 5.1 Alur quote ke payment

Alur normal:

1.  BDT-GPI menerima permintaan transfer.
2.  BDT-GPI memvalidasi negara, mata uang, nominal, identitas
    originator, beneficiary, dan instrumen tujuan.
3.  BDT-GPI meminta quote collection ke Ripple.
4.  Ripple mengembalikan quote yang berisi kurs, biaya, nominal
    sumber/tujuan, rail, dan masa berlaku.
5.  BDT-GPI menghitung biaya tambahan dan menampilkan total biaya kepada
    pemohon.
6.  Pemohon menyetujui quote.
7.  BDT-GPI membuat payment dengan quote yang dipilih sebelum
    kedaluwarsa.
8.  BDT-GPI memonitor status payment melalui API/webhook.
9.  BDT-GPI melakukan rekonsiliasi dengan ledger/provider statement.

### 5.2 Endpoint quote

Dokumentasi Payments Direct yang dijadikan acuan menyediakan endpoint:

``` http
POST /v3/quotes/quote-collection
```

Contoh request konseptual:

``` json
{
  "quoteAmount": 10000,
  "quoteAmountType": "SOURCE_AMOUNT",
  "sourceCurrency": "USD",
  "destinationCurrency": "MXN",
  "sourceCountry": "US",
  "destinationCountry": "MX",
  "payinCategory": "PRE_FUNDING"
}
```

Contoh di atas hanya contoh format dari dokumentasi. Bukan bukti bahwa
USD→MXN atau parameter yang sama telah diaktifkan untuk akun BDT-GPI.
Gunakan schema OpenAPI tenant dan versi API yang disetujui Ripple.

### 5.3 Elemen penting respons quote

Contoh struktur ringkas:

``` json
{
  "quoteCollectionId": "collection-id",
  "quotes": [
    {
      "quoteId": "quote-id",
      "quoteStatus": "ACTIVE",
      "quoteAmountType": "SOURCE_AMOUNT",
      "sourceAmount": 10000,
      "destinationAmount": 204533.30,
      "sourceCurrency": "USD",
      "destinationCurrency": "MXN",
      "paymentRail": "SPEI",
      "payinCategory": "PRE_FUNDING",
      "adjustedExchangeRate": {
        "adjustedRate": 20.4136
      },
      "fees": [
        {
          "totalFee": 14,
          "feeCurrency": "USD"
        }
      ],
      "createdAt": "2025-11-13T22:44:34.711Z",
      "expiresAt": "2025-11-13T22:59:34.767Z"
    }
  ]
}
```

Nilai di atas merupakan contoh dokumentasi, **bukan tarif Ripple untuk
BDT-GPI**.

### 5.4 Field yang wajib diproses

  -----------------------------------------------------------------------
  Field                               Perlakuan BDT-GPI
  ----------------------------------- -----------------------------------
  `quoteId`                           Simpan sebagai referensi quote
                                      provider dan gunakan sesuai
                                      instruksi API saat create payment.

  `quoteStatus`                       Hanya quote `ACTIVE` yang dapat
                                      dipertimbangkan untuk payment.

  `sourceAmount`                      Simpan sebagai nominal sumber dari
                                      provider.

  `destinationAmount`                 Simpan sebagai nominal yang akan
                                      diterima beneficiary menurut quote.

  `sourceCurrency`                    Simpan sebagai ISO 4217.

  `destinationCurrency`               Simpan sebagai ISO 4217.

  `paymentRail`                       Simpan rail yang dipilih provider.

  `adjustedExchangeRate`              Simpan kurs provider dan presisi
                                      aslinya.

  `fees`                              Simpan total fee, currency, dan
                                      breakdown bila ada.

  `taxes`                             Simpan pajak atas service fee jika
                                      dikembalikan.

  `createdAt`                         Simpan timestamp provider.

  `expiresAt`                         Gunakan untuk menolak quote yang
                                      kedaluwarsa.
  -----------------------------------------------------------------------

**Catatan penting:** dokumentasi menyebut quote ID digunakan dalam
pembuatan payment dan harus diperlakukan konsisten. Implementasi final
harus mengikuti versi OpenAPI tenant, bukan asumsi dari contoh kode.

## 6. Model Komponen Biaya BDT-GPI

BDT-GPI perlu menyimpan setiap komponen biaya secara terpisah.

### 6.1 Komponen biaya

  -----------------------------------------------------------------------
  Komponen                Sumber                  Cara memperoleh
  ----------------------- ----------------------- -----------------------
  Ripple provider fee     Ripple                  Quote API / kontrak

  Ripple fee tax          Ripple / aturan pajak   Field `taxes` pada
                          terkait                 quote bila tersedia

  FX conversion cost      Provider / harga quote  Perbedaan nilai kurs
                                                  dan struktur harga yang
                                                  dikonfirmasi

  Local payout fee        Mitra lokal             Kontrak/quote mitra,
                                                  bila tidak sudah
                                                  termasuk

  Bank/intermediary fee   Bank/mitra              Kontrak atau hasil
                                                  settlement

  BDT service fee         BDT-GPI                 Pricing policy yang
                                                  disetujui

  FX markup               BDT-GPI                 Pricing policy, jika
                                                  diperbolehkan dan
                                                  transparan

  Customer tax            Aturan pajak            Kajian pajak dan
                                                  konfigurasi produk
  -----------------------------------------------------------------------

Jangan menambahkan biaya lokal secara terpisah jika biaya tersebut sudah
termasuk dalam quote Ripple. Ini dapat menyebabkan double charging.

### 6.2 Formula dasar

Gunakan formula berikut sebagai model internal. Sesuaikan dengan
definisi nominal dan currency yang dikembalikan API.

``` text
provider_fee       = sum(quote.fees[].totalFee)
provider_fee_tax   = sum(quote.taxes[].totalTaxes)
local_partner_fee  = fee dari mitra lokal yang belum termasuk quote
bdt_service_fee    = biaya layanan BDT-GPI
other_fee          = biaya lain yang sah dan diungkapkan

total_customer_fee =
    provider_fee
  + provider_fee_tax
  + local_partner_fee
  + bdt_service_fee
  + other_fee
```

Semua komponen harus dikonversi ke satu `fee_currency` sebelum
dijumlahkan. Jika komponen biaya memiliki mata uang berbeda, gunakan
kurs konversi yang tercatat, timestamp kurs, dan metode pembulatan yang
konsisten.

### 6.3 Jangan mencampur fee dan FX

Biaya eksplisit (`fees`) berbeda dari kurs konversi
(`adjustedExchangeRate`). BDT-GPI harus menyimpan keduanya terpisah.

Jika BDT-GPI mengenakan FX markup, tampilkan dan simpan sebagai komponen
terpisah. Jangan menyamarkannya sebagai Ripple fee.

### 6.4 Quote berdasarkan source atau destination amount

Ada dua pola utama:

-   `SOURCE_AMOUNT`: pengirim menentukan jumlah yang akan dibayar.
-   `DESTINATION_AMOUNT`: pengirim menentukan jumlah yang harus diterima
    beneficiary.

Gunakan `DESTINATION_AMOUNT` ketika aplikasi menjanjikan nominal
penerimaan yang pasti kepada penerima. Gunakan `SOURCE_AMOUNT` ketika
nominal sumber yang menjadi batas utama.

Sistem harus menyimpan `quoteAmountType` dan tidak boleh menafsirkan
`sourceAmount` dan `destinationAmount` secara tertukar.

## 7. Simulasi Biaya --- Bukan Tarif Resmi

Bagian ini hanya untuk menguji model kalkulasi dan desain tampilan.
Angka berikut **tidak berasal dari penawaran resmi Ripple**.

### 7.1 Simulasi nominal Rp10.000.000

  Komponen                                                 Nilai simulasi
  ------------------------------------------------------ ----------------
  Ripple fee                                                     Rp25.000
  Pajak atas Ripple fee                                               Rp0
  Mitra payout lokal                                             Rp10.000
  BDT service fee                                                Rp15.000
  Biaya lain                                                          Rp0
  **Total biaya pelanggan**                                  **Rp50.000**
  Nilai transfer pokok                                       Rp10.000.000
  Total debit pelanggan (jika fee ditagihkan terpisah)       Rp10.050.000

Pada contoh ini, total biaya pelanggan adalah 0,50% dari nilai transfer.
Ini hanya contoh aritmetika, bukan rekomendasi harga jual atau perkiraan
harga Ripple.

### 7.2 Simulasi beberapa nominal

Dengan asumsi biaya simulasi tetap Rp50.000:

    Nominal transfer   Total biaya   Persentase efektif
  ------------------ ------------- --------------------
         Rp1.000.000      Rp50.000                5,00%
         Rp5.000.000      Rp50.000                1,00%
        Rp10.000.000      Rp50.000                0,50%
        Rp25.000.000      Rp50.000                0,20%
        Rp50.000.000      Rp50.000                0,10%

Tabel menunjukkan mengapa pricing sebaiknya tidak hanya menggunakan
fixed fee. BDT-GPI dapat mempertimbangkan gabungan fixed fee dan
percentage fee setelah biaya aktual, kontrak, regulasi, dan
willingness-to-pay diketahui.

### 7.3 Model harga internal yang dapat dikonfigurasi

Contoh formula:

``` text
bdt_service_fee =
    max(minimum_fee, source_amount * service_fee_rate)
```

Contoh nilai konfigurasi untuk sandbox internal (bukan harga final):

``` text
minimum_fee       = IDR 10,000
service_fee_rate  = 0.15%
maximum_fee       = IDR 100,000
```

Kemudian:

``` text
bdt_service_fee =
    min(maximum_fee,
        max(minimum_fee, source_amount * service_fee_rate))
```

Semua tarif harus berada dalam tabel konfigurasi berversi, bukan
tertanam di source code.

## 8. Arsitektur Sistem

``` text
Client / Ovisito Pay
        |
        v
BDT-GPI API Gateway
        |
        v
Payment Orchestrator (Laravel)
        |
        +---- Pricing Engine
        |
        +---- Quote Service
        |         |
        |         v
        |     Ripple Adapter
        |         |
        |         v
        |     Ripple Payments Direct API
        |
        +---- Compliance / Risk Controls
        |
        +---- Double-entry Ledger
        |
        +---- Webhook Processor
        |
        +---- Reconciliation Jobs
        |
        v
BDT-GPI Database (separate connection/database)
```

### 8.1 Prinsip arsitektur

-   Laravel modular monolith untuk MVP.
-   Ripple ditempatkan di adapter provider, bukan tersebar di seluruh
    aplikasi.
-   Internal API BDT-GPI tidak mengekspos payload Ripple secara
    langsung.
-   Database pembayaran dipisahkan dari database booking Ovisito.
-   Semua perubahan status dan nilai uang memiliki audit trail.
-   Gunakan decimal fixed precision untuk nominal; jangan gunakan
    floating point untuk uang.
-   Simpan timestamp dalam UTC dan tampilkan sesuai zona waktu pengguna.

## 9. Desain Database

Nama koneksi contoh: `bdt_payment`. Pastikan konfigurasi koneksi
tersedia pada `config/database.php`.

### 9.1 `payment_quotes`

Menyimpan quote dari provider dan hasil kalkulasi internal.

  ---------------------------------------------------------------------------
  Kolom                       Tipe konseptual         Keterangan
  --------------------------- ----------------------- -----------------------
  id                          UUID                    Primary key internal

  provider_code               varchar                 Contoh `RIPPLE`

  provider_quote_id           varchar                 ID quote provider

  quote_collection_id         varchar nullable        ID collection

  transaction_id              UUID nullable           Transaksi internal
                                                      terkait

  quote_status                varchar                 ACTIVE / EXPIRED / USED
                                                      / REJECTED

  quote_amount_type           varchar                 SOURCE_AMOUNT /
                                                      DESTINATION_AMOUNT

  source_amount               decimal(24,8)           Nominal sumber

  source_currency             char(3)                 ISO 4217

  destination_amount          decimal(24,8)           Nominal tujuan

  destination_currency        char(3)                 ISO 4217

  adjusted_exchange_rate      decimal(24,12)          Kurs provider

  payment_rail                varchar nullable        Rail yang dikutip

  provider_fee_amount         decimal(24,8)           Total fee provider

  provider_fee_currency       char(3) nullable        Mata uang fee

  provider_fee_tax_amount     decimal(24,8) nullable  Pajak fee provider

  provider_fee_tax_currency   char(3) nullable        Mata uang pajak

  local_partner_fee           decimal(24,8)           Biaya lokal terpisah

  bdt_service_fee             decimal(24,8)           Biaya BDT

  other_fee                   decimal(24,8)           Komponen lainnya

  total_customer_fee          decimal(24,8)           Total fee yang
                                                      ditampilkan

  fee_currency                char(3)                 Mata uang total fee

  customer_debit_amount       decimal(24,8)           Total debit yang harus
                                                      dibayar

  raw_response                json                    Respons provider yang
                                                      telah disanitasi

  created_at_provider         timestamp nullable      Waktu dari provider

  expires_at_provider         timestamp nullable      Kedaluwarsa quote

  accepted_at                 timestamp nullable      Persetujuan pelanggan

  created_at / updated_at     timestamp               Timestamp sistem
  ---------------------------------------------------------------------------

Tambahkan unique constraint yang sesuai pada kombinasi `provider_code`
dan `provider_quote_id`. Pastikan aturan retensi data dan perlindungan
PII diterapkan.

### 9.2 `payment_transactions`

Menyimpan transaksi utama.

Kolom penting:

-   `id` UUID primary key;
-   `public_reference` unique;
-   `provider_code`;
-   `provider_payment_id`;
-   `quote_id`;
-   `customer_id` atau referensi customer;
-   `originator_identity_id`;
-   `beneficiary_identity_id`;
-   `source_amount`, `source_currency`;
-   `destination_amount`, `destination_currency`;
-   `total_customer_fee`, `fee_currency`;
-   `status`;
-   `idempotency_key`;
-   `failure_code`, `failure_message`;
-   `submitted_at`, `completed_at`;
-   timestamps.

### 9.3 `payment_provider_events`

Menyimpan event webhook/provider.

Kolom penting:

-   `provider_code`;
-   `provider_event_id`;
-   `provider_payment_id`;
-   `event_type`;
-   `payload`;
-   `signature_valid`;
-   `processed_at`;
-   `processing_status`.

Gunakan unique constraint pada event ID provider bila tersedia. Jika
tidak tersedia, buat strategi deduplikasi berdasarkan identifier dan
payload hash yang disetujui.

### 9.4 `payment_idempotency`

Menyimpan request idempotency:

-   `idempotency_key`;
-   `client_id`;
-   `endpoint`;
-   `request_hash`;
-   `response_snapshot`;
-   `status`;
-   `expires_at`.

Unique key harus mempertimbangkan client dan endpoint. Request dengan
key sama tetapi body berbeda harus ditolak.

### 9.5 `ledger_accounts` dan `ledger_entries`

Ledger harus double-entry, immutable, dan mendukung currency. Jangan
mengubah atau menghapus jurnal yang sudah diposting; koreksi dilakukan
melalui reversal/adjustment entry.

Minimal simpan:

-   account code;
-   currency;
-   debit/credit;
-   amount;
-   transaction reference;
-   journal batch/reference;
-   posting timestamp;
-   source event;
-   metadata audit.

Setiap journal harus balance per currency. Jangan menjumlahkan debit MYR
dengan kredit IDR dalam satu pemeriksaan balance tanpa akun FX/clearing
yang tepat.

## 10. Kontrak API Internal BDT-GPI

### 10.1 Membuat quote

``` http
POST /api/v1/quotes
Authorization: Bearer <BDT_ACCESS_TOKEN>
Idempotency-Key: <unique-key>
Content-Type: application/json
```

Request internal:

``` json
{
  "client_reference": "OVI-ORDER-2026-00001",
  "quote_amount": "10000000.00",
  "quote_amount_type": "DESTINATION_AMOUNT",
  "source_currency": "MYR",
  "destination_currency": "IDR",
  "source_country": "MY",
  "destination_country": "ID",
  "funding_model": "PRE_FUNDING",
  "beneficiary_id": "beneficiary-uuid"
}
```

Response internal (contoh):

``` json
{
  "quote_reference": "bdt-quote-uuid",
  "status": "ACTIVE",
  "source_amount": "xxx",
  "source_currency": "MYR",
  "destination_amount": "10000000.00",
  "destination_currency": "IDR",
  "exchange_rate": "xxx",
  "fees": {
    "provider_fee": "xxx",
    "provider_fee_currency": "MYR",
    "provider_fee_tax": "xxx",
    "local_partner_fee": "xxx",
    "bdt_service_fee": "xxx",
    "total_customer_fee": "xxx",
    "fee_currency": "MYR"
  },
  "customer_debit_amount": "xxx",
  "expires_at": "2026-11-12T12:00:00Z"
}
```

`xxx` adalah placeholder. API produksi harus mengembalikan angka hasil
quote aktual dan konversi yang dapat dijelaskan.

### 10.2 Menyetujui dan membuat payment

``` http
POST /api/v1/payments
Authorization: Bearer <BDT_ACCESS_TOKEN>
Idempotency-Key: <unique-key>
Content-Type: application/json
```

``` json
{
  "quote_reference": "bdt-quote-uuid",
  "client_reference": "OVI-ORDER-2026-00001",
  "accept_terms": true
}
```

Validasi sebelum create payment:

1.  Quote milik client yang sama.
2.  Quote masih ACTIVE dan belum expired.
3.  Quote belum pernah digunakan.
4.  Nominal, mata uang, beneficiary, dan rail tidak berubah.
5.  Persetujuan pelanggan tercatat.
6.  Limit, screening, dan funding checks lolos.
7.  Idempotency key belum digunakan untuk request berbeda.

### 10.3 Mendapatkan status

``` http
GET /api/v1/payments/{payment_reference}
```

Status internal yang disarankan:

-   `CREATED`
-   `QUOTED`
-   `AWAITING_APPROVAL`
-   `APPROVED`
-   `SUBMITTED`
-   `PROCESSING`
-   `AWAITING_FUNDING`
-   `COMPLETED`
-   `FAILED`
-   `CANCELLED`
-   `UNKNOWN`
-   `REQUIRES_REVIEW`

Status mapping harus disesuaikan dengan lifecycle resmi Ripple API.

## 11. Laravel Implementation Blueprint

### 11.1 Struktur folder

``` text
app/
  Modules/
    Gpi/
      Contracts/
        PaymentProviderInterface.php
      DTO/
        QuoteRequestData.php
        QuoteResultData.php
        PaymentResultData.php
      Enums/
        PaymentStatus.php
      Models/
        PaymentTransaction.php
        PaymentQuote.php
        ProviderEvent.php
      Providers/
        Ripple/
          RippleClient.php
          RippleQuoteService.php
          RipplePaymentService.php
          RippleWebhookVerifier.php
          RippleMapper.php
      Services/
        PricingEngine.php
        QuoteOrchestrator.php
        PaymentOrchestrator.php
        LedgerService.php
        ReconciliationService.php
      Http/
        Controllers/
        Requests/
        Resources/
```

### 11.2 Database connection

Contoh `config/database.php`:

``` php
'connections' => [
    // ...
    'bdt_payment' => [
        'driver' => 'mysql',
        'host' => env('BDT_PAYMENT_DB_HOST'),
        'port' => env('BDT_PAYMENT_DB_PORT', '3306'),
        'database' => env('BDT_PAYMENT_DB_DATABASE'),
        'username' => env('BDT_PAYMENT_DB_USERNAME'),
        'password' => env('BDT_PAYMENT_DB_PASSWORD'),
        'charset' => 'utf8mb4',
        'collation' => 'utf8mb4_unicode_ci',
        'prefix' => '',
        'strict' => true,
        'engine' => null,
    ],
],
```

Model:

``` php
class PaymentQuote extends Model
{
    protected $connection = 'bdt_payment';

    protected $casts = [
        'raw_response' => 'array',
        'expires_at_provider' => 'immutable_datetime',
        'created_at_provider' => 'immutable_datetime',
        'accepted_at' => 'immutable_datetime',
    ];
}
```

### 11.3 Pricing engine

Pseudocode:

``` php
final class PricingEngine
{
    public function calculate(
        ProviderQuoteData $quote,
        PricingPolicy $policy,
        ?LocalPartnerCost $localCost
    ): PricingResult {
        // 1. Normalize all fees to the configured fee currency.
        // 2. Do not count local costs already included by provider.
        // 3. Calculate BDT service fee from a versioned policy.
        // 4. Calculate taxes only according to approved tax rules.
        // 5. Return a complete itemized breakdown.
    }
}
```

Jangan memakai `float` untuk nilai uang. Gunakan decimal library,
integer minor units bila sesuai, atau database DECIMAL dengan aturan
pembulatan eksplisit.

### 11.4 Ripple adapter

Adapter minimal menyediakan:

``` php
interface PaymentProviderInterface
{
    public function createQuote(QuoteRequestData $request): QuoteResultData;

    public function createPayment(
        string $providerQuoteId,
        PaymentSubmissionData $request
    ): PaymentResultData;

    public function getPayment(string $providerPaymentId): PaymentResultData;
}
```

`RippleClient` menangani:

-   OAuth2 access token;
-   token cache dan expiry;
-   timeout dan retry policy;
-   TLS verification;
-   request correlation ID;
-   sanitasi log;
-   mapping error;
-   secret rotation;
-   environment UAT/production.

Endpoint, scope, audience, request schema, dan response schema harus
diambil dari OpenAPI resmi yang berlaku pada tenant Ripple.

## 12. Pengelolaan Quote dan Payment

### 12.1 Quote expiry

-   Simpan `expiresAt` dari Ripple.
-   Jangan menerima quote setelah expiry.
-   Jangan memperpanjang quote secara lokal.
-   Jika quote expired, minta quote baru dan hitung ulang semua fee.
-   Jika harga berubah, minta persetujuan ulang pelanggan sebelum
    payment.

### 12.2 Timeout saat create payment

Jika request create payment timeout:

1.  Tandai transaksi `UNKNOWN`, bukan langsung `FAILED`.
2.  Simpan request ID, idempotency key, quote ID, dan waktu pengiriman.
3.  Lakukan inquiry/status check ke Ripple.
4.  Jika payment ditemukan, lanjutkan monitoring.
5.  Jika status belum dapat dipastikan, masuk antrean retry/inquiry
    terkontrol.
6.  Jangan membuat payment baru sampai hasil request pertama dipastikan.

### 12.3 Webhook

-   Verifikasi signature sesuai mekanisme resmi provider.
-   Tolak payload yang tidak valid.
-   Deduplicate event.
-   Simpan payload mentah yang sudah disanitasi.
-   Proses event secara asynchronous.
-   Pastikan transisi status valid.
-   Jangan mempercayai urutan event tanpa memeriksa status final
    provider.
-   Sediakan dead-letter queue dan mekanisme replay yang diaudit.

## 13. Ledger dan Perlakuan Dana

Ledger harus mencerminkan peran hukum BDT-GPI dalam alur dana. Contoh
akun dapat berbeda tergantung apakah BDT hanya penyedia teknologi, agen,
atau memiliki peran lain yang diizinkan.

Jangan menganggap BDT boleh menahan dana pelanggan atau memiliki saldo
settlement hanya karena secara teknis dapat membuat akun ledger.

### 13.1 Contoh akun konseptual

-   `CUSTOMER_FUNDS_CLEARING`
-   `PROVIDER_PREFUND_ASSET`
-   `PROVIDER_PAYABLE`
-   `MERCHANT_PAYABLE`
-   `BDT_SERVICE_REVENUE`
-   `PROVIDER_FEE_EXPENSE`
-   `LOCAL_PARTNER_FEE_EXPENSE`
-   `FX_CLEARING`
-   `REFUND_PAYABLE`

Daftar ini bukan instruksi jurnal akuntansi final. Chart of accounts dan
jurnal harus disetujui akuntan serta penasihat hukum berdasarkan kontrak
dan custody/fund flow aktual.

### 13.2 Kontrol wajib

-   Double-entry per currency.
-   Journal immutable.
-   Reversal untuk koreksi.
-   Rekonsiliasi harian.
-   Tidak ada saldo negatif tanpa limit/approval.
-   Pemisahan dana operasional perusahaan dan dana pihak lain.
-   Maker-checker untuk adjustment manual.
-   Audit trail untuk setiap perubahan.

## 14. Security, Compliance, dan Risk

### 14.1 Keamanan teknis

-   Simpan client secret di secret manager, bukan repository.
-   Enkripsi PII dan instrumen keuangan saat tersimpan.
-   Redact nomor rekening, identitas, token, dan credential dari log.
-   Terapkan least-privilege pada service account.
-   Rotasi secret dan kunci enkripsi.
-   Rate limiting dan IP/network policy jika tersedia.
-   TLS untuk semua komunikasi.
-   Backup terenkripsi dan uji pemulihan.
-   Audit log append-only.
-   Vulnerability assessment sebelum produksi.

### 14.2 Compliance

Sebelum live, verifikasi:

-   klasifikasi aktivitas BDT-GPI berdasarkan peraturan Bank Indonesia
    yang berlaku;
-   apakah BDT hanya technology provider atau menjalankan aktivitas
    pembayaran;
-   kelayakan badan hukum PT perorangan untuk struktur dan perjanjian
    yang direncanakan;
-   kewajiban PJP/PIP atau kewajiban lain yang relevan;
-   peran Ripple dan mitra lokal berizin;
-   KYC/KYB, AML/CFT, sanctions screening, dan transaction monitoring;
-   perlindungan data pribadi dan transfer data lintas negara;
-   aturan perlindungan konsumen, pengungkapan biaya, refund, dan
    pengaduan;
-   pajak atas service fee dan transaksi terkait.

Jangan meluncurkan alur transfer dana sebelum struktur legal dan
perizinan dikonfirmasi oleh penasihat hukum yang memahami sistem
pembayaran Indonesia dan pihak berwenang yang relevan.

## 15. Rekonsiliasi

Rekonsiliasi minimum harian:

1.  Internal `payment_transactions`.
2.  Ripple payment status dan ledger/transaction statement.
3.  Bank/mitra lokal statement.
4.  BDT-GPI double-entry ledger.
5.  Settlement dan outstanding payable.

Kategori exception:

-   payment internal ada, provider tidak ada;
-   provider payment ada, internal tidak ada;
-   nominal berbeda;
-   currency berbeda;
-   fee berbeda;
-   status berbeda;
-   settlement terlambat;
-   duplicate payment;
-   refund/return tidak tercatat.

Setiap exception memiliki owner, severity, due date, dan resolution
note.

## 16. UAT Test Plan

  -----------------------------------------------------------------------
  ID                      Skenario                Hasil yang diharapkan
  ----------------------- ----------------------- -----------------------
  UAT-001                 Quote valid             Quote tersimpan, fee
                                                  breakdown terbentuk

  UAT-002                 Quote expired           Payment ditolak; quote
                                                  baru diminta

  UAT-003                 Corridor tidak aktif    Error terkontrol; tidak
                                                  ada payment

  UAT-004                 Currency tidak didukung Validasi gagal sebelum
                                                  submit

  UAT-005                 Provider fee currency   Konversi fee tercatat
                          berbeda                 dan dapat diaudit

  UAT-006                 Fee tax tersedia        Tax disimpan terpisah
                                                  dari principal

  UAT-007                 Duplicate idempotency   Tidak terjadi duplicate
                          key                     payment

  UAT-008                 Same key, different     Request ditolak
                          payload                 

  UAT-009                 Create payment timeout  Status UNKNOWN; inquiry
                                                  sebelum retry

  UAT-010                 Invalid webhook         Event ditolak dan
                          signature               dicatat

  UAT-011                 Duplicate webhook       Event hanya diproses
                                                  sekali

  UAT-012                 Partial/late status     Monitoring dan
                          update                  reconciliation
                                                  menangani

  UAT-013                 Provider fee berubah    Customer approval
                          saat quote ulang        diminta ulang

  UAT-014                 Ledger currency         Journal gagal diposting
                          imbalance               

  UAT-015                 Refund/return           Status dan jurnal
                                                  reversal benar
  -----------------------------------------------------------------------

UAT harus menggunakan test identities, test beneficiaries, dan data yang
disetujui Ripple. Jangan menggunakan data pribadi atau dana riil tanpa
izin dan prosedur yang tepat.

## 17. Dashboard Operasional

Dashboard BDT-GPI setidaknya menampilkan:

-   total quote dan payment;
-   success/failure rate;
-   nilai transaksi per currency dan corridor;
-   provider fee rata-rata dan median;
-   total fee pelanggan;
-   pendapatan service fee BDT;
-   FX margin jika digunakan;
-   biaya mitra lokal;
-   outstanding funding;
-   transaksi UNKNOWN;
-   quote expired;
-   webhook gagal;
-   settlement pending;
-   reconciliation exception.

Filter minimum: tanggal, provider, corridor, currency,
client/application, payment rail, status.

## 18. Pricing Governance

Setiap perubahan harga harus memiliki:

-   `pricing_policy_id`;
-   versi kebijakan;
-   tanggal mulai berlaku;
-   client/produk yang berlaku;
-   currency;
-   fixed fee;
-   percentage fee;
-   minimum dan maksimum;
-   aturan FX markup;
-   aturan pembulatan;
-   tax treatment;
-   approver;
-   alasan perubahan.

Jangan mengubah policy lama untuk transaksi yang sudah di-quote. Simpan
snapshot pricing policy pada setiap quote agar perhitungan historis
dapat direproduksi.

## 19. Checklist Komersial kepada Ripple

Sebelum menyimpulkan biaya per transaksi, minta konfirmasi tertulis:

1.  Apakah BDT-GPI dapat onboard sebagai direct customer atau harus
    melalui sponsor/partner?
2.  Apakah entitas PT perorangan dapat diterima untuk proses KYB dan
    kontrak?
3.  Apakah use case B2B, B2C, dan payout ke rekening Indonesia
    diizinkan?
4.  Apakah koridor MYR→IDR, SGD→IDR, dan USD→IDR tersedia untuk akun
    yang diajukan?
5.  Payment rail apa yang tersedia untuk setiap corridor?
6.  Apakah biaya ditampilkan lengkap melalui quote API?
7.  Apakah ada fixed fee, percentage fee, minimum fee, monthly fee,
    setup fee, atau volume commitment?
8.  Apakah ada biaya prefunding, funding transfer, withdrawal, return,
    cancellation, atau failed payment?
9.  Bagaimana FX rate ditentukan dan apakah terdapat markup?
10. Apakah pajak atas service fee dikembalikan dalam quote?
11. Apakah ada minimum transaction volume atau prefunding balance?
12. Berapa waktu settlement dan cut-off?
13. Bagaimana refund, return, recall, dan dispute diproses?
14. Apakah biaya lokal sudah termasuk dalam quote atau ditagihkan
    terpisah?
15. Apakah UAT dan production menggunakan pricing yang sama?
16. Bagaimana notifikasi jika pricing atau corridor berubah?

**Hasil yang harus diminta:** rate card/fee schedule resmi, contoh quote
dari akun UAT untuk setiap corridor, SLA, funding terms, dan kontrak
final.

## 20. Rencana Implementasi 12 Minggu

  -----------------------------------------------------------------------
  Minggu                  Pekerjaan               Output
  ----------------------- ----------------------- -----------------------
  1--2                    Legal scope, partner    Go/no-go legal dan
                          mapping, Ripple         komersial
                          onboarding/KYB          

  3--4                    Database, transaction   Core BDT-GPI
                          core, ledger foundation 

  5--6                    Ripple OAuth, quote     Quote flow UAT
                          adapter, schema mapping 

  7--8                    Pricing engine, payment End-to-end payment UAT
                          orchestration, webhook  

  9--10                   Security,               Control readiness
                          reconciliation,         
                          operational dashboard   

  11                      UAT scenarios dan       UAT report
                          exception handling      

  12                      Review legal,           Production readiness
                          commercial, operational decision
                          readiness               
  -----------------------------------------------------------------------

Timeline ini indikatif dan bergantung pada onboarding, akses API,
aktivasi corridor, serta keputusan legal.

## 21. Keputusan Teknis yang Direkomendasikan

1.  Bangun BDT-GPI sebagai payment orchestration layer yang
    provider-agnostic.
2.  Jadikan Ripple adapter pertama, bukan ketergantungan tunggal.
3.  Gunakan quote API untuk biaya dan kurs transaksi aktual.
4.  Simpan semua fee itemized dan quote snapshot.
5.  Pisahkan database payment dari database booking Ovisito.
6.  Gunakan idempotency, webhook verification, status inquiry, dan
    reconciliation sejak MVP.
7.  Jangan membangun asumsi tarif sebelum memperoleh quote UAT dan rate
    card resmi.
8.  Jangan memindahkan atau menahan dana sebelum struktur izin dan mitra
    berizin jelas.

## 22. Sumber Resmi dan Referensi

Dokumentasi berikut menjadi titik awal dan harus diperiksa ulang ketika
implementasi dimulai karena versi API dapat berubah.

-   Ripple Payments Direct overview:
    https://docs.ripple.com/products/payments-direct-2/introduction/ripple-payments-direct
-   Quotes concept:
    https://docs.ripple.com/products/payments-direct-2/introduction/concepts/quotes
-   Fetch a quote collection:
    https://docs.ripple.com/products/payments-direct-2/api-docs/developer-guides/fetch-a-quote-collection
-   Create a payment:
    https://docs.ripple.com/products/payments-direct-2/api-docs/developer-guides/create-a-payment
-   Getting started / OpenAPI and SDK:
    https://docs.ripple.com/products/payments-direct-2/api-docs/getting-started
-   Payments Direct documentation index:
    https://docs.ripple.com/products/payments-direct-2

### Catatan sumber

Dokumentasi API menjelaskan struktur teknis quote, fee, taxes, expiry,
funding category, dan alur payment. Dokumentasi publik tersebut **tidak
menetapkan satu harga komersial universal untuk semua pelanggan
Ripple**. Harga final BDT-GPI harus berasal dari perjanjian komersial
Ripple dan quote yang dihasilkan untuk akun, corridor, nominal, dan rail
yang benar.

------------------------------------------------------------------------

## Lampiran A --- Contoh tampilan breakdown biaya

Contoh berikut hanya untuk menggambarkan UI; angka harus berasal dari
quote provider dan pricing policy aktif.

``` text
Ringkasan Transfer
Nominal diterima beneficiary        Rp10.000.000
Kurs provider                        [hasil quote]
Biaya Ripple                         [hasil quote]
Pajak biaya provider                 [jika berlaku]
Biaya mitra payout                   [jika terpisah]
Biaya layanan BDT-GPI                 [pricing policy]
---------------------------------------------------
Total biaya                          [hasil kalkulasi]
Total yang harus dibayar             [hasil kalkulasi]
Quote berlaku sampai                 [expiresAt]
```

## Lampiran B --- Aturan presisi dan pembulatan

-   Simpan nominal sesuai presisi mata uang dan schema provider.
-   Jangan membulatkan kurs sebelum kalkulasi final.
-   Gunakan aturan pembulatan yang konsisten dan terdokumentasi.
-   Terapkan aturan ISO 4217 untuk jumlah digit minor currency, dengan
    pengecualian hanya jika kontrak/API secara eksplisit menetapkan
    lain.
-   Simpan nilai asli provider dan nilai hasil kalkulasi internal secara
    terpisah.
-   Simpan formula dan versi policy yang digunakan.

## Lampiran C --- Definisi Done untuk MVP

MVP belum dapat dinyatakan siap produksi sampai:

-   [ ] Legal assessment dan struktur izin/mitra disetujui.
-   [ ] Ripple onboarding dan kontrak disetujui.
-   [ ] Corridor dan rail yang digunakan aktif pada akun.
-   [ ] Rate card dan biaya aktual telah diterima.
-   [ ] Quote API berhasil diuji.
-   [ ] Payment create dan inquiry berhasil diuji.
-   [ ] Timeout dan duplicate request aman.
-   [ ] Webhook signature dan deduplication berjalan.
-   [ ] Ledger balance per currency teruji.
-   [ ] Rekonsiliasi end-to-end selesai.
-   [ ] Security review dan incident response tersedia.
-   [ ] Customer fee disclosure, refund, dan complaint handling
    disetujui.
-   [ ] Production readiness ditandatangani pemilik bisnis, engineering,
    compliance, dan mitra terkait.

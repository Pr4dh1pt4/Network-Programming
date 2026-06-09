# Network-Programming

# UNO Online Multiplayer

Game kartu **UNO** multiplayer berbasis jaringan, dibangun dengan **Python**, **Pygame**, dan **Socket Programming (TCP)**. Proyek mata kuliah Pemrograman Jaringan — Institut Teknologi Sepuluh Nopember (ITS).

Server bersifat **authoritative**: seluruh aturan permainan, validasi, dan state diproses di server, sementara client hanya merender state dan mengirim input. Pendekatan ini menjaga konsistensi state antar pemain dan mencegah cheat.

---

## Fitur

**Wajib**
- Real-time game state synchronization
- Room system & matchmaking otomatis
- Minimal 2 pemain per match (mendukung 2–4)
- Reconnect handling (grace period)
- Indikator ping/latency
- Logging aktivitas player
- Anti invalid-packet validation

**Tambahan**
- Leaderboard, ranking, statistik pemain, riwayat match
- Top Global Player
- Spectator mode (pemenang otomatis jadi penonton, tetap di room)
- Replay match sederhana

---

## Tech Stack

### Inti (wajib sesuai ketentuan project)

| Komponen | Teknologi | Keterangan |
|---|---|---|
| Bahasa | **Python 3.10+** | Wajib. Versi 3.10+ untuk `match-case` & type hints modern. |
| Game engine | **Pygame** (`pygame-ce`) | Render papan, kartu, animasi, input. `pygame-ce` lebih aktif di-maintain. |
| Networking | **`socket` (built-in) + TCP** | Inti penilaian. Protokol & layer jaringan dibangun sendiri. |
| Concurrency | **`threading` + `queue` (built-in)** | Satu thread per client; mudah di-demo & di-debug. Alternatif: `asyncio`. |

> **Kenapa TCP?** UNO bersifat turn-based dan diskret. Yang dibutuhkan adalah *reliable, ordered delivery* — kehilangan satu paket aksi (mis. "main kartu") merusak konsistensi state semua pemain. Jaminan TCP lebih penting daripada keunggulan latency UDP, yang lebih cocok untuk game *fast-paced*.

### Data & Persistensi

| Komponen | Teknologi | Keterangan |
|---|---|---|
| Database (dev) | **`sqlite3` (built-in)** | Tanpa setup, satu file. Untuk akun, leaderboard, match history, replay. |
| Database (opsional) | **PostgreSQL** via `psycopg2-binary` | Untuk demo/produksi lebih serius. |
| Password hashing | **`bcrypt`** / `argon2-cffi` | Wajib — password tidak boleh disimpan plaintext. |
| Serialisasi paket | **`json` (built-in)** | Format packet komunikasi jaringan. |
| Framing paket | **`struct` (built-in)** | Length-prefix framing (4 byte panjang + payload JSON). |

### Pendukung & Quality

| Komponen | Teknologi | Keterangan |
|---|---|---|
| Logging | **`logging` (built-in)** | Logging aktivitas player (fitur wajib). |
| Testing | **`pytest`** | Uji GameEngine, validator, leaderboard. |
| Validasi struktur | **`dataclasses` (built-in)** / `pydantic` | Definisi & validasi struktur packet. |

### Sudah built-in di Python (tidak perlu install)
`socket` · `threading` · `asyncio` · `queue` · `json` · `struct` · `sqlite3` · `logging` · `dataclasses`

---

## Dependencies

`requirements.txt`:

```text
pygame-ce>=2.4
bcrypt>=4.1
pytest>=8.0
# opsional:
# psycopg2-binary>=2.9   # jika pakai PostgreSQL
# pydantic>=2.6          # jika ingin validasi packet berbasis model
```

> Catatan rubrik: ketentuan membolehkan library socket (SocketServer, Twisted, PodSixNet), tetapi rubrik menegaskan **implementasi Pemrograman Jaringan utama harus sendiri**. Karena itu proyek ini memakai modul `socket` standar dan membangun sendiri layer protokol (framing, validasi, routing paket), bukan framework game-networking jadi.

---

## Arsitektur

Arsitektur **3-tier** dengan server authoritative:

```
Tier 1 — Presentation (Client)    : Pygame UI, input handler, renderer, ping indicator
Tier 2 — Application (Server)     : socket server, packet validator, room manager,
                                    matchmaker, game engine, leaderboard, reconnect, logger
Tier 3 — Data                     : database (users, leaderboard, matches), replay & log files
```

Komunikasi: **JSON over TCP** dengan length-prefix framing (4 byte big-endian panjang payload, lalu payload JSON UTF-8).

---

## Struktur Folder

```text
uno-online/
├── README.md
├── requirements.txt
├── config.py                 # host, port, konstanta game
├── server/                   # entry point & logika server
│   ├── main_server.py
│   ├── socket_server.py
│   ├── packet/               # protocol & validator
│   ├── core/                 # game_engine, deck, card, room, matchmaker, reconnect
│   ├── services/             # auth, leaderboard, replay
│   ├── db/                   # database, schema.sql, models
│   └── utils/                # logger
├── client/                   # aplikasi Pygame
│   ├── main_client.py
│   ├── network/              # client_network, ping_monitor
│   ├── scenes/               # login, lobby, room, game, spectator, result, leaderboard
│   ├── ui/                   # widgets, renderer
│   └── assets/               # images, fonts
├── shared/                   # constants & packet_types bersama
├── data/                     # uno.db, logs/, replays/
└── tests/                    # test_game_engine, test_validator, test_leaderboard
```

---

## Cara Menjalankan

### 1. Persiapan

```bash
# clone repo
git clone <repo-url>
cd uno-online

# buat virtual environment
python -m venv venv
# Windows
venv\Scripts\activate
# Linux/Mac
source venv/bin/activate

# install dependencies
pip install -r requirements.txt
```

### 2. Inisialisasi database

```bash
python -m server.db.database --init
```

### 3. Jalankan server

```bash
python -m server.main_server
# server listen di 0.0.0.0:5555 (atur di config.py)
```

### 4. Jalankan client (di tiap pemain)

```bash
python -m client.main_client
# atur host server di config.py atau via argumen
```

---

## Sistem Poin & Rank

| Hasil | Poin |
|---|---|
| Menang | **+25** |
| Kalah | **−10** (poin tidak < 0) |

| Rank | Rentang Poin |
|---|---|
| Bronze | 0 – 999 |
| Silver | 1000 – 1499 |
| Gold | 1500 – 1999 |
| Platinum | 2000+ |

`win_rate = total_win / total_match`

---

## Testing

```bash
pytest tests/ -v
```

Skenario stabilitas yang diuji: client putus bersamaan, reconnect saat giliran, spam invalid packet, dan banyak room paralel.

---

## Tim

Proyek mata kuliah Pemrograman Jaringan — ITS.

## Lisensi

Untuk keperluan akademik.

---
title: "uv: Satu Tool untuk Semua Kebutuhan Python Dev"
date: 2026-06-13
description: "uv dari Astral bukan sekadar pengganti pip yang lebih cepat — ia menyatukan seluruh toolchain Python (pip, pyenv, virtualenv, poetry, pipx) menjadi satu alat tunggal."
tags: ["uv", "python", "packaging", "developer-tools", "productivity"]
categories: ["dev-tools"]
draft: false
showHero: false
showTableOfContents: true
showReadingTime: true
---

Kalau kamu developer Python, setup environment sebelum mulai kerja biasanya terasa seperti ritual: install pyenv, set Python version, buat virtualenv, install pip-tools, compile requirements, baru bisa `pip install`. Belum termasuk kalau pakai Poetry atau PDM.

**uv** hadir untuk merombak itu semua.

## Apa itu uv?

uv adalah package manager dan project manager untuk Python, dibuat oleh **Astral** — tim yang sama di balik [Ruff](https://github.com/astral-sh/ruff), linter Python super cepat yang sudah banyak dipakai. Ditulis dalam Rust, diluncurkan Februari 2024, dan berkembang besar-besaran pada Agustus 2024 menjadi "unified Python packaging."

Per Juni 2026: versi `0.11.21`, 86.3k GitHub stars, lisensi MIT/Apache-2.0.

Klaim Astral sendiri: *"a single tool to replace pip, pip-tools, pipx, poetry, pyenv, twine, virtualenv, and more."*

## Seberapa Cepat?

Ini bukan klaim marketing biasa. Dari benchmark dari Astral (dengan catatan: hasil bervariasi tergantung OS, filesystem, dan dependency set):

- **8-10x lebih cepat dari pip** tanpa cache
- **80-115x lebih cepat dengan warm cache**

Untuk resolusi dependency Trio dengan warm cache:
- uv: `0.01s`
- pip-compile: `1.56s`  
- Poetry: `0.60s`

Instalasinya:
- uv: `0.06s`
- pip-sync: `4.63s`
- PDM: `1.90s`

Kecepatan ini datang dari implementasi Rust, global cache yang efisien, dan filesystem copy yang optimal. Di CI/CD, perbedaannya sangat terasa.

## Apa yang Digantikan uv?

| Tool Lama | Pengganti uv |
|-----------|-------------|
| `pip install` | `uv pip install` |
| `pip-compile` | `uv pip compile` |
| `pyenv install` | `uv python install` |
| `python -m venv` | `uv venv` |
| `pipx run` | `uvx` / `uv tool run` |
| Poetry / PDM | `uv init` + `uv add` + `uv sync` |

Catatan penting: ini bukan drop-in replacement sempurna untuk setiap edge case. Untuk proyek modern, coverage-nya sangat baik. Untuk legacy packaging (`.egg` distributions, dsb.), tetap perlu pip fallback.

## Workflow Dasar

### Setup proyek baru

```bash
# Install Python versi spesifik
uv python install 3.12

# Buat proyek baru
uv init my-project
cd my-project

# Tambah dependency
uv add httpx fastapi

# Import dari requirements.txt yang sudah ada
uv add -r requirements.txt
```

Setelah `uv init`, kamu dapat `pyproject.toml` dan `uv.lock` — lockfile universal yang konsisten di semua OS.

### Jalankan kode tanpa aktivasi venv

```bash
# uv otomatis pastikan environment up-to-date
uv run python main.py
uv run pytest
uv run --with httpx python script.py  # pakai package tambahan tanpa install permanen
```

Tidak perlu `source .venv/bin/activate` lagi.

### Sync environment dari lockfile

```bash
# Sync exact dari uv.lock (hapus package yang tidak ada di lockfile)
uv sync

# Update lockfile
uv lock
```

### Jalankan tools sekali pakai (pengganti pipx)

```bash
# Tanpa install permanen
uvx ruff check .
uvx black .

# Install global tool
uv tool install ruff
```

## Migrasi dari pip + pyenv

**Opsi 1: Low-friction (coba dulu)**

Langsung ganti `pip` dengan `uv pip` — tidak perlu ubah project structure:
```bash
uv pip install -r requirements.txt
uv pip install httpx
```

Langsung dapat speed boost, zero risk.

**Opsi 2: Full migration**

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Buat proyek uv dari folder yang sudah ada
uv init --bare  # tanpa buat file hello.py
uv add -r requirements.txt  # import existing deps
```

Setelah ini kamu punya `pyproject.toml` dan `uv.lock` — project siap untuk workflow modern.

## Siapa yang Sudah Pakai?

Bukan sekadar hype developer individual:

- **Apache Airflow** — sudah punya `uv.lock` di repo, docs kontribusi pakai `uv sync`, GitHub Actions CI pakai `astral-sh/setup-uv`
- **LangChain / LangGraph** — contributing docs arahkan kontributor pakai `uv venv` dan `uv sync --all-groups`
- [Hynek Schlawack](https://hynek.me/articles/docker-uv/) (penulis Python practices yang dikenal luas) menulis tentang production Docker containers dengan uv

Untuk CI/CD, ada `astral-sh/setup-uv` di GitHub Actions dengan 789 stars — ekosistem tooling-nya sudah solid.

## Catatan Sebelum Switch

1. **Benchmark adalah vendor benchmark** — hasilnya bagus, tapi validate di environment kamu sendiri
2. **Legacy packaging** — kalau ada `.egg` distributions atau packaging yang sangat tua, test dulu
3. **Python distributions** — uv pakai `python-build-standalone` dari Astral, bukan binary resmi CPython. Untuk sebagian besar use case, identik; untuk edge case tertentu, mungkin berbeda
4. **uv.lock format** — berbeda dari `requirements.txt` atau `poetry.lock`, tidak interchangeable

## Kesimpulan

uv bukan sekadar pip yang lebih cepat. Ia menyederhanakan seluruh setup toolchain Python — dari Python version management, virtualenv, dependency resolution, lockfile, hingga tool execution — menjadi satu command family yang konsisten.

Untuk proyek baru: mulai dengan uv langsung. Untuk proyek yang sudah ada: mulai dengan `uv pip` sebagai drop-in replacement, evaluasi pengalaman kamu, lalu migrasi penuh kalau hasilnya bagus.

---

*Sumber utama: [docs.astral.sh/uv](https://docs.astral.sh/uv), [github.com/astral-sh/uv](https://github.com/astral-sh/uv), [astral.sh/blog/uv](https://astral.sh/blog/uv)*

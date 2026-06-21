---
title: "Setup Environment Python Dev Modern: pyenv + uv + VS Code"
date: 2026-06-21
description: "Panduan praktis menyusun environment Python dev yang cepat dan rapi: pyenv untuk version management, uv untuk dependency & virtualenv, VS Code untuk editor — lengkap dengan konfigurasi siap pakai."
tags: ["python", "pyenv", "uv", "vscode", "developer-tools", "productivity"]
categories: ["dev-tools"]
draft: false
showHero: false
showTableOfContents: true
showReadingTime: true
---

Tiap kali mulai project Python baru, ada satu pertanyaan yang selalu muncul: versi Python yang mana, dan environment-nya diisolasi pakai apa? Dulu jawabannya berantakan — system Python, virtualenv manual, kadang conda, kadang pip biasa. Sekarang kombinasi yang paling masuk akal ada tiga: **pyenv** untuk version Python, **uv** untuk dependency dan virtualenv, **VS Code** sebagai editor yang menyatukan semuanya. Tulisan ini menjabarkan setup-nya dari nol sampai siap kerja.

## Kenapa Tiga Tool Ini?

Tiap tool punya tanggung jawab yang jelas, tidak tumpang tindih:

- **pyenv** — mengelola *banyak versi* interpreter Python di satu mesin, dan ganti versi per-project tanpa bersinggungan dengan Python sistem.
- **uv** — mengelola dependency, virtualenv, dan lockfile project, jauh lebih cepat dari pip/poetry. (Sudah pernah dibahas lebih detail di [postingan uv sebelumnya](/posts/uv-python-package-manager/).)
- **VS Code** — editor dengan extension Python yang otomatis mendeteksi virtualenv dan interpreter yang sedang dipakai.

Pembagian ini penting: uv sendiri sebenarnya bisa men-download Python build standalone tanpa pyenv. Tapi kalau kamu sudah punya banyak project lama yang bergantung pada pyenv, atau butuh kontrol penuh atas versi sistem-wide (`pyenv global`) di luar konteks satu project, pyenv tetap relevan sebagai layer paling bawah.

## 1. Install pyenv

Di macOS/Linux, cara paling gampang lewat installer resmi:

```bash
curl https://pyenv.run | bash
```

Tambahkan ke `~/.bashrc` atau `~/.zshrc`:

```bash
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
```

Reload shell, lalu cek:

```bash
pyenv --version
```

Install versi Python yang dibutuhkan:

```bash
pyenv install 3.12.7
pyenv install 3.13.0
```

Set versi default sistem, dan versi khusus untuk direktori project:

```bash
pyenv global 3.12.7

cd ~/projects/my-api
pyenv local 3.13.0   # bikin file .python-version di folder ini
```

`pyenv local` menulis file `.python-version` yang dibaca otomatis tiap kali kamu masuk direktori itu (asal `pyenv init -` ter-load di shell). Ini yang membuat banyak project dengan versi Python berbeda bisa hidup berdampingan tanpa drama.

## 2. Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Verifikasi:

```bash
uv --version
```

uv akan otomatis memakai interpreter Python yang aktif di `PATH` — yang berarti, kalau pyenv sudah men-set versi yang benar lewat `.python-version`, uv mengikuti tanpa konfigurasi tambahan.

### Bikin Project Baru

```bash
mkdir my-api && cd my-api
pyenv local 3.12.7
uv init
```

`uv init` membuat `pyproject.toml`, `.python-version` (uv punya versi sendiri, biasanya konsisten dengan yang di-set pyenv), dan struktur project minimal.

### Tambah Dependency

```bash
uv add fastapi uvicorn
uv add --dev pytest ruff
```

uv otomatis membuat virtualenv `.venv/` di root project dan mengisi `uv.lock` — lockfile yang deterministik, mirip `package-lock.json` di dunia Node.

### Jalankan Tanpa Manual Activate

```bash
uv run uvicorn main:app --reload
```

`uv run` mengaktifkan virtualenv secara implisit untuk satu command, jadi kamu tidak perlu lagi ingat `source .venv/bin/activate` tiap buka terminal baru. Kalau memang mau activate manual:

```bash
source .venv/bin/activate
```

## 3. Konfigurasi VS Code

Install extension wajib:

- **Python** (Microsoft) — `ms-python.python`
- **Pylance** — bahasa server, biasanya ikut terinstall otomatis dengan extension Python
- **Ruff** (opsional, kalau pakai ruff untuk lint/format) — `charliermarsh.ruff`

### Set Interpreter

Buka Command Palette (`Cmd/Ctrl+Shift+P`) → **Python: Select Interpreter** → pilih yang mengarah ke `.venv/bin/python` di project kamu. VS Code biasanya sudah mendeteksi `.venv` otomatis kalau folder itu ada di root workspace.

### settings.json Project-Level

Taruh di `.vscode/settings.json` supaya konsisten untuk semua kontributor:

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["tests"],
  "editor.formatOnSave": true,
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.codeActionsOnSave": {
      "source.organizeImports": "explicit",
      "source.fixAll": "explicit"
    }
  }
}
```

Ini bikin tiap kali project di-clone ulang, siapapun yang buka di VS Code langsung dapat interpreter yang benar, format-on-save, dan lint aktif — tanpa setup manual berulang.

## 4. Alur Kerja Lengkap (Project Baru)

Ringkasan urutan dari nol:

```bash
mkdir my-project && cd my-project
pyenv local 3.12.7          # pin versi Python
uv init                      # bikin pyproject.toml + venv
uv add fastapi sqlalchemy    # dependency runtime
uv add --dev pytest ruff mypy
code .                        # buka VS Code, pilih interpreter .venv
```

Lima command, kurang dari satu menit, dan project sudah punya: versi Python yang terkunci, dependency yang ter-lock, virtualenv terisolasi, dan editor yang langsung paham semuanya.

## 5. Perangkap Umum

**pyenv shims tidak terdeteksi setelah install** — biasanya karena `eval "$(pyenv init -)"` belum dimuat ulang. Restart terminal, atau jalankan `exec $SHELL`.

**uv pakai Python sistem, bukan versi pyenv** — pastikan `.python-version` ada di root project dan pyenv shim ada lebih dulu di `PATH` dibanding `uv`'s bundled toolchain. Cek dengan `which python` sebelum `uv run`.

**VS Code masih nunjuk ke virtualenv lama** — kalau baru pindah/ganti `.venv`, reload window (`Cmd/Ctrl+Shift+P` → **Developer: Reload Window**) supaya Pylance re-index interpreter yang baru.

**Lockfile konflik di git** — `uv.lock` itu generated file, tapi tetap di-commit (best practice, sama seperti `package-lock.json`). Kalau conflict, jangan merge manual — jalankan `uv lock` ulang lalu commit hasilnya.

## 6. Testing di Beberapa Versi Python

Kalau project kamu maintain library yang harus jalan di Python 3.10–3.13, pyenv berguna untuk install semua versi sekaligus:

```bash
pyenv install 3.10.15
pyenv install 3.11.10
pyenv install 3.12.7
pyenv install 3.13.0
```

uv juga punya kemampuan menjalankan test matrix tanpa tool tambahan seperti tox, lewat `uv run --python`:

```bash
for v in 3.10 3.11 3.12 3.13; do
  uv run --python $v pytest
done
```

Tiap iterasi, uv bikin (atau reuse) virtualenv terpisah untuk versi tersebut, install dependency dari `uv.lock`, lalu jalankan test. Tidak perlu konfigurasi tox.ini kalau kebutuhannya cuma run test matrix simpel.

## 7. Integrasi CI (GitHub Actions)

uv punya action resmi yang bikin setup CI singkat — tidak perlu pyenv sama sekali di CI karena runner sudah punya Python, dan uv bisa pakai itu langsung:

```yaml
name: test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv python install ${{ matrix.python-version }}
      - run: uv sync --all-extras --dev
      - run: uv run pytest
      - run: uv run ruff check .
```

Catatan: di CI, pakai `uv python install` (bukan pyenv) karena uv bisa download build Python standalone sendiri tanpa overhead pyenv. pyenv tetap lebih cocok untuk *local dev machine* yang sudah punya banyak project lama bergantung padanya.

## 8. Kapan Masih Perlu Conda?

Kombinasi pyenv + uv menutup hampir semua kasus pure-Python. Tapi conda (atau mamba) masih punya tempat kalau:

- Dependency-nya bukan cuma Python package, tapi juga binary non-Python (CUDA toolkit versi spesifik, library C/Fortran besar seperti GDAL) yang lebih gampang di-resolve lewat conda channel.
- Tim data science kamu sudah established di ekosistem conda dan migrasi tidak sepadan dengan effort-nya.

Untuk web service, CLI tool, atau library Python biasa — pyenv + uv sudah lebih dari cukup, dan jauh lebih cepat.

## FAQ Singkat

**Apa pyenv masih relevan kalau uv bisa install Python sendiri?**
Untuk project baru, sering tidak terlalu dibutuhkan — `uv python install 3.12` saja cukup. Tapi kalau kamu butuh ganti versi Python *global* di shell (di luar konteks satu project, misal untuk script one-off atau tool lain yang bukan dikelola uv), `pyenv global` masih cara yang lebih natural. Banyak setup existing juga sudah terlanjur bergantung pada pyenv, jadi migrasi penuh ke uv-only belum tentu sepadan.

**Perlu virtualenvwrapper atau pyenv-virtualenv juga?**
Tidak. uv sudah menggantikan keduanya. `uv venv` dan `uv run` cukup untuk seluruh siklus virtualenv — bikin, aktivasi implisit, dan jalanin command di dalamnya.

**Bagaimana kalau tim sebagian masih pakai pip biasa?**
uv tetap kompatibel: `uv pip install -r requirements.txt` jalan seperti pip biasa, hanya jauh lebih cepat karena cache global dan resolver yang ditulis di Rust. Migrasi bisa gradual — mulai dari mengganti command pip dengan `uv pip`, baru pindah ke `pyproject.toml` + `uv add` kalau sudah siap.

**Apakah `.venv` harus di-gitignore?**
Ya, selalu. Yang di-commit adalah `pyproject.toml` dan `uv.lock` — bukan folder virtualenv-nya. Siapapun yang clone repo tinggal jalankan `uv sync` untuk reproduce environment yang identik dari lockfile.

## Kesimpulan

pyenv, uv, dan VS Code menyelesaikan tiga masalah yang berbeda dan saling melengkapi: versi interpreter, dependency/virtualenv, dan editor experience. Kombinasi ini menggantikan setup lama yang biasanya butuh conda, pip-tools, virtualenvwrapper, dan beberapa plugin VS Code terpisah — sekarang cukup tiga tool dengan konfigurasi yang jelas tanggung jawabnya masing-masing.

Kalau project kamu masih pakai pip + venv manual, migrasi ke uv biasanya cukup satu sore: `uv init`, lalu `uv add` semua dependency dari `requirements.txt` lama.

---

*Pernah kena masalah environment Python yang lain? Share di komentar.*

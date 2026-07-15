# Ganti Tombol Ikon → Teks Inggris (Start/Stop + Pause/Resume)

## Tujuan
Ikon/simbol SVG pada tombol Start/Stop dan Pause/Resume terlihat jelek. Ganti
menjadi **teks bahasa Inggris**. Selain itu:
- Kedua tombol **berukuran sama** (lebar sama rata).
- **Jarak antar tombol cukup lebar**.

File yang diubah: **hanya** `C:\Users\okky\Documents\CODE\kytiapomo\index.html`
(single-file app, CSS + JS inline).

## Keputusan Desain
- Label teks (uppercase; `.btn` sudah `text-transform:uppercase`):
  - Tombol kiri: **START** (idle) / **STOP** (running atau paused).
  - Tombol kanan: **PAUSE** (running) / **RESUME** (paused); **disabled** saat idle.
- Kedua tombol lebar sama: `flex: 1 1 0` (flex-basis 0 → berbagi lebar sama rata),
  bukan lagi satu melar + satu kotak.
- Jarak antar tombol: **24px** (dari sebelumnya 12px).
- Lebar total baris tetap `min(var(--action-size), 85%)` seperti sekarang.
- Ikon SVG (constants `ICON_PLAY`/`ICON_PAUSE`/`ICON_STOP`) dan aturan `.btn svg`
  dihapus (jadi dead code setelah perubahan).

## Perubahan

### 1. Markup — index.html:1145-1150
Ganti class kedua tombol menjadi `btn btn-equal` (hapus `btn-action`/`btn-square`).
Boleh mengisi teks awal langsung (akan tetap disinkronkan `setButtonUi()`).
```html
<div class="controls" aria-label="Timer controls">
  <div class="controls-row">
    <button id="startPauseBtn" class="btn btn-equal" type="button" aria-label="Start timer">START</button>
    <button id="pauseBtn" class="btn btn-equal" type="button" aria-label="Pause timer" disabled>PAUSE</button>
  </div>
</div>
```

### 2. CSS — index.html:817-855
- Ubah `.controls-row` gap `12px` → `24px`.
- Hapus blok `.btn-action` dan `.btn-square`; ganti dengan `.btn-equal`.
- Hapus blok `.btn svg` (tidak dipakai lagi).
- Pertahankan `.btn:disabled` dan `.btn:disabled:hover`.

Menjadi kira-kira:
```css
.controls-row {
  display: flex;
  gap: 24px;                 /* jarak lebih lebar */
  align-items: center;
  justify-content: center;
  width: min(var(--action-size), 85%);
  margin: 0 auto;
}
.btn-equal {
  flex: 1 1 0;               /* kedua tombol lebar sama */
  min-width: 0;              /* override min-width lama pada .btn */
  display: grid;
  place-items: center;       /* teks center */
}
.btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  box-shadow: none;
}
.btn:disabled:hover {
  background: rgba(0, 180, 216, 0.08);
  color: var(--color-accent);
  box-shadow: none;
}
/* (blok .btn svg dihapus) */
```

### 3. CSS responsif — index.html:505-513
Ganti override `.btn-action`/`.btn-square` dengan `.btn-equal` agar tetap satu baris
& lebar sama di layar kecil:
```css
.controls-row {
  width: 100%;
}
.btn-equal {
  width: auto;   /* batalkan .btn { width:100% } dari media query, biar flex yang atur */
}
```
(Hapus baris `.btn-action { width:auto }` dan `.btn-square { width: var(--button-height) }`.)

### 4. JS — hapus konstanta ikon (index.html:1847-1849)
Hapus `ICON_PLAY`, `ICON_PAUSE`, `ICON_STOP`.

### 5. JS — `setButtonUi()` pakai teks (index.html:1851-1866)
Ganti `innerHTML = ICON_*` menjadi `textContent` label:
```js
function setButtonUi() {
  const running = timerState === "running";
  const paused = timerState === "paused";

  // Tombol kiri: START saat idle, STOP saat running/paused
  el.startPauseBtn.textContent = (running || paused) ? "STOP" : "START";
  el.startPauseBtn.setAttribute("aria-label", (running || paused) ? "Stop timer" : "Start timer");

  // Tombol kanan: PAUSE saat running, RESUME saat paused, disabled saat idle
  el.pauseBtn.disabled = !(running || paused);
  el.pauseBtn.textContent = paused ? "RESUME" : "PAUSE";
  el.pauseBtn.setAttribute("aria-label", paused ? "Resume timer" : "Pause timer");

  el.runningHint.textContent = running ? "Running…" : (paused ? "Paused" : "");
}
```

## Perilaku yang tetap (tidak diubah)
- Logika state idle/running/paused, handler klik Start/Stop & Pause/Resume,
  auto-cycle, fix double-rAF, dan guard settings-save (`timerState === "idle"`)
  tidak berubah.

## Validasi (buka index.html di browser)
1. Saat load: dua tombol berdampingan **lebar sama**, jarak jelas (24px), teks
   "START" dan "PAUSE" (PAUSE tampak redup/disabled).
2. Klik START → teks kiri jadi "STOP", kanan aktif "PAUSE", hint "Running…".
3. Klik PAUSE → countdown berhenti, teks kanan jadi "RESUME", hint "Paused".
4. Klik RESUME → lanjut, teks kembali "PAUSE".
5. Klik STOP → reset, tombol jadi "START" + "PAUSE" (disabled).
6. Cek layar <460px: kedua tombol tetap satu baris, lebar sama, teks tidak terpotong.
7. Cek tidak ada refereni `ICON_PLAY/PAUSE/STOP` yang tersisa (tidak ada error console).

## Catatan
- Butuh agent implementasi (mode yang bisa edit source). Plan mode tidak mengubah source.

# Auto-start Rest / Long Rest (Siklus Penuh Otomatis)

## Konteks
Aplikasi Pomodoro single-file (`index.html`). Saat ini saat timer focus habis,
`onTimerEnded()` (index.html:1899) memang pindah tab ke Rest/Long Rest secara
otomatis, tapi **tidak** menyalakan timer-nya (diset `timerState = "idle"`, tombol
jadi START, user harus klik manual).

Permintaan: setelah focus habis, timer Rest/Long Rest **langsung nyala otomatis**,
dan siklus terus berputar (focus → rest/longRest → focus → rest → ... → longRest → focus ...)
tanpa perlu klik tombol. Keputusan user:
- Arah: **siklus penuh otomatis** (termasuk rest habis → focus berikutnya auto nyala).
- Timing: **langsung setelah alarm** (rest mulai hitung mundur saat focus = 0, alarm bunyi bersamaan).
- Toggle: **tidak ada**, selalu aktif.

Logika lanjutan sudah ada di `nextTabAfterEnd()` (index.html:1929):
- setelah pomodoro: jika `sessionCounts.pomodoro % settings.cycle === 0` → longRest, else rest.
- setelah rest/longRest → pomodoro.
Jadi tidak perlu mengubah logika pemilihan tab, hanya cara `onTimerEnded` menyalakan timer berikutnya.

## Perubahan

### 1. `onTimerEnded()` — index.html:1899-1925
Ganti bagian auto-advance agar menyalakan timer berikutnya (bukan idle):

Saat ini:
```js
const next = nextTabAfterEnd(currentTab);
timerState = "idle";
setTabSelected(next);
setTimerForTab(next);
setButtonUi();
moveUnderlineToSelected();
```

Menjadi:
```js
const next = nextTabAfterEnd(currentTab);
setTabSelected(next);
setTimerForTab(next);        // reset timeRemaining & durasi tab berikutnya
timerState = "running";      // langsung nyala (full auto-cycle)
setButtonUi();               // tombol → STOP
moveUnderlineToSelected();
startTimer();                // mulai hitung mundur (alarm focus masih bunyi bersamaan)
```

Catatan:
- Urutan penting: `setTimerForTab(next)` harus dipanggil sebelum `startTimer()`
  agar `timeRemaining`/`durationSeconds` sudah sesuai tab baru. `startTimer()`
  membaca `timeRemaining` global (index.html:1879).
- Alarm (loop `beep` via setTimeout ~1.4s di index.html:1906) tetap berjalan
  berdampingan dengan countdown rest — sesuai pilihan "langsung setelah alarm".

### 2. Komentar spec — index.html:1748-1754
Perbarui deskripsi spec v1.0 agar mencerminkan auto-cycle:
- "- Timer ends -> ring flashes, beep, session counter increments" tetap.
- "- Auto-advance: ... -> auto-start next session (full automatic cycle)".

## Perilaku yang tetap (tidak diubah)
- Tombol **STOP** (`startPauseBtn`) tetap menghentikan siklus: saat running, klik STOP
  memanggil `stopTimer()` + `timerState = "idle"` + `setTimerForTab(currentTab)`
  (index.html:1976-1982). User bisa memutus siklus kapan saja.
- **Restart** (`restartConfirmBtn`, index.html:2021) tetap mengembalikan ke idle
  (panggil `switchTab("pomodoro")`); ini reset eksplisit, jadi tidak auto-start.
  (Konsisten: user sengaja me-reset.)
- Background tab: `setInterval` 250ms (index.html:1885) tetap memicu `refreshFromClock`
  → `onTimerEnded` → auto-start berikutnya, sehingga siklus jalan walau tab tersembunyi.
- Settings: `DURATIONS` diperbarui saat save; siklus berikutnya pakai durasi baru.
  Validasi settings (min 1) tidak berubah.

## Validasi
Buka `index.html` di browser, uji via console Dev (`window.DEV`):
1. `DEV.setTime(3)` lalu biarkan habis → pastikan tab otomatis pindah ke Rest DAN
   timer Rest langsung berjalan (tombol berubah jadi STOP, countdown jalan).
2. Biarkan Rest habis → otomatis pindah ke Pomodoro & langsung jalan.
3. Ulangi hingga `sessionCounts.pomodoro` mencapai kelipatan `cycle` (default 4)
   → pastikan setelah pomodoro ke-4, tab otomatis pindah ke **Long Rest** & langsung jalan.
4. Saat sedang running, klik STOP → siklus berhenti, tombol jadi START, tab tetap di
   posisi sekarang (reset ke durasi penuh). Klik START → lanjut manual.
5. Cek counter `focusCount`/`breakCount`/`longRestCount` bertambah tiap sesi otomatis.
6. (Opsional) reload halaman di tengah siklus → timer mulai idle (expected; tidak ada
   resume-on-reload).

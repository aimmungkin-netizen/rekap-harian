<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Kalkulator Multiple Taruhan 2D (Multi-Pasaran + Pola A–T + Backup)</title>
<style>
  body { font-family: Arial, sans-serif; margin: 25px; background: #f9f9f9; color: #333; }
  h2 { color: #0077cc; }
  select, input { padding: 6px; margin-top: 4px; border-radius: 5px; border: 1px solid #aaa; }
  button { padding: 8px 14px; border: none; border-radius: 5px; cursor: pointer; font-size: 14px; margin: 5px 5px 5px 0; }
  button:hover { opacity: 0.95; }
  .primary { background: #0077cc; color: white; }
  .danger { background: #cc0000; color: white; }
  .success { background: #009933; color: white; }
  .container { background: #fff; padding: 15px; border-radius: 10px; box-shadow: 0 2px 6px rgba(0,0,0,0.1); margin-bottom: 20px; }
  table { border-collapse: collapse; width: 100%; margin-top: 10px; }
  th, td { border: 1px solid #ccc; padding: 6px; text-align: center; }
  th { background: #e0f0ff; }
  .summary { margin-top: 10px; background: #eef8ee; padding: 10px; border-radius: 8px; }
  .controls-row { display:flex; gap:8px; align-items:center; flex-wrap:wrap; margin-top:8px; }
  .pattern-row { display:flex; gap:6px; flex-wrap:wrap; margin-top:10px; }
  .pattern-btn { width:36px; height:36px; border-radius:6px; border:1px solid #999; background:#fff; cursor:pointer; }
  .pattern-btn.active { background:#0077cc; color:#fff; border-color:#005fa3; }
  .pattern-label { margin-left:8px; font-weight:bold; }
  .small { font-size:13px; color:#666; }
</style>
</head>
<body>

<h2>Kalkulator Multiple Taruhan 2D — Multi Pasaran</h2>

<div class="container">
  <label>Pilih Pasaran:</label>
  <select id="pasaranSelect" onchange="gantiPasaran()">
    <option>HAWAI</option>
    <option>TOKYO</option>
    <option>PAPUA</option>
    <option>MEXICO</option>
    <option>HOCIMINH</option>
    <option>CAMBODIA</option>
    <option>BUSAN</option>
    <option>SIDNEY</option>
    <option>SIDNEY LOTTO</option>
    <option>HANOI</option>
    <option>CHINA</option>
    <option>KOREA</option>
    <option>JAPAN</option>
    <option>SINGAPORE</option>
    <option>KA</option>
    <option>BOJA</option>
    <option>JEJU</option>
    <option>PCSO</option>
    <option>PENANG</option>
    <option>TAIWAN</option>
    <option>HONGKONG</option>
    <option>HONGKONG LOTTO</option>
  </select>

  <!-- Pola A-T -->
  <div style="margin-top:8px;">
    <div class="small">Pilih Pola (disimpan per pasaran):</div>
    <div id="patternRow" class="pattern-row" aria-hidden="false"></div>
    <div class="pattern-label">Pola aktif: <span id="activePatternLabel">A</span></div>
  </div>
</div>

<div class="container">
  <label>Jumlah nomor dipasang (M):</label>
  <input type="number" id="jumlahNomor" value="10" min="1" max="99">
  
  <label>Payout ratio:</label>
  <input type="number" id="payout" value="100">
  
  <label>Target keuntungan per menang:</label>
  <input type="number" id="target" value="50000">
  
  <label>Minimal taruhan per nomor:</label>
  <input type="number" id="minTaruhan" value="100" step="100">
  
  <p>Total kekalahan (otomatis): <input id="kerugian" value="0" readonly></p>
  <p>Total modal kumulatif: <input id="totalModal" value="0" readonly></p>

  <div class="controls-row">
    <button class="primary" onclick="hitung()">Hitung Multiple</button>
    <button class="success" onclick="tambahKalah()">Tambah Kekalahan</button>
    <button class="success" onclick="tambahMenang()">Menang Hari Ini</button>
    <button onclick="simulasi()">Simulasi 10 Hari</button>
    <button class="danger" onclick="resetPasaran()">Reset Pasaran</button>
    <!-- Export / Import buttons placed next to Reset as requested -->
    <button onclick="exportData()">Ekspor Data</button>
    <button onclick="triggerImport()">Impor Data</button>
    <input type="file" id="importFile" accept=".json" style="display:none" onchange="importData(event)">
  </div>
</div>

<div class="container" id="hasil"></div>

<div class="container">
  <h3>Histori Harian (pada pola aktif)</h3>
  <table id="tabelHistori">
    <tr><th>Hari</th><th>Status</th><th>Modal</th><th>Total Kekalahan</th><th>Total Modal</th></tr>
  </table>
  <div class="summary" id="ringkasan"></div>
</div>

<script>
/*
  New storage structure (backwards-compatible migration included):

  localStorage key: "dataMultiPasaran"
  Stored value: {
    "<PASARAN_NAME>": {
      // for backward compatibility, existing top-level keys (totalKalah, totalModal, histori, modalTerakhir)
      // will be migrated into patterns.A
      patterns: {
        "A": { totalKalah, totalModal, histori: [...], modalTerakhir },
        "B": { ... },
        ...
        "T": { ... }
      },
      activePattern: "A"  // current active pattern for this pasaran
    },
    ...
  }

  lastActivePasaran stored in localStorage as before.
*/

const PATTERNS = Array.from({length:20}, (_,i) => String.fromCharCode(65 + i)); // A..T

let dataSemuaPasaran = JSON.parse(localStorage.getItem("dataMultiPasaran") || "{}");
let pasaranAktif = localStorage.getItem("lastActivePasaran") || document.getElementById("pasaranSelect").value;

// Ensure every pasaran exists and migrate old structure if present
function ensureAllPasaranExist() {
  const sel = document.getElementById("pasaranSelect");
  for (let i = 0; i < sel.options.length; i++) {
    const name = sel.options[i].text;
    if (!dataSemuaPasaran[name]) {
      dataSemuaPasaran[name] = createEmptyPasaran();
    } else {
      // if existing structure is old (no patterns), migrate into patterns.A
      const entry = dataSemuaPasaran[name];
      if (!entry.patterns) {
        // old keys: totalKalah, totalModal, histori, modalTerakhir
        const patternsObj = {};
        PATTERNS.forEach(p => patternsObj[p] = createEmptyPattern());
        // migrate existing data into A
        patternsObj['A'].totalKalah = entry.totalKalah || 0;
        patternsObj['A'].totalModal = entry.totalModal || 0;
        patternsObj['A'].histori = entry.histori || [];
        patternsObj['A'].modalTerakhir = entry.modalTerakhir || 0;
        // replace entry
        dataSemuaPasaran[name] = { patterns: patternsObj, activePattern: entry.activePattern || 'A' };
      } else {
        // ensure all patterns present
        PATTERNS.forEach(p => {
          if (!entry.patterns[p]) entry.patterns[p] = createEmptyPattern();
        });
        if (!entry.activePattern) entry.activePattern = 'A';
      }
    }
  }
}

function createEmptyPattern() {
  return { totalKalah: 0, totalModal: 0, histori: [], modalTerakhir: 0 };
}

ensureAllPasaranExist();
if (!dataSemuaPasaran[pasaranAktif]) pasaranAktif = document.getElementById("pasaranSelect").value;

function simpanData() {
  localStorage.setItem("dataMultiPasaran", JSON.stringify(dataSemuaPasaran));
  localStorage.setItem("lastActivePasaran", pasaranAktif);
}

function gantiPasaran() {
  pasaranAktif = document.getElementById("pasaranSelect").value;
  if (!dataSemuaPasaran[pasaranAktif]) dataSemuaPasaran[pasaranAktif] = { patterns: {}, activePattern: 'A' };
  // ensure patterns exist
  PATTERNS.forEach(p => { if (!dataSemuaPasaran[pasaranAktif].patterns[p]) dataSemuaPasaran[pasaranAktif].patterns[p] = createEmptyPattern(); });
  // set pattern row UI
  renderPatternButtons();
  simpanData();
  tampilkanData();
}

/* Pattern UI */
function renderPatternButtons() {
  const row = document.getElementById('patternRow');
  row.innerHTML = '';
  const entry = dataSemuaPasaran[pasaranAktif];
  const active = entry.activePattern || 'A';
  PATTERNS.forEach(p => {
    const btn = document.createElement('button');
    btn.textContent = p;
    btn.className = 'pattern-btn' + (p === active ? ' active' : '');
    btn.onclick = () => selectPattern(p);
    row.appendChild(btn);
  });
  document.getElementById('activePatternLabel').textContent = entry.activePattern || 'A';
}

/* Select a pattern (A..T) for current pasaran */
function selectPattern(p) {
  if (!PATTERNS.includes(p)) return;
  dataSemuaPasaran[pasaranAktif].activePattern = p;
  // ensure pattern exists
  if (!dataSemuaPasaran[pasaranAktif].patterns[p]) dataSemuaPasaran[pasaranAktif].patterns[p] = createEmptyPattern();
  simpanData();
  renderPatternButtons();
  tampilkanData();
}

/* Helper to get currently-active pattern object for current pasaran */
function currentPatternObj() {
  const entry = dataSemuaPasaran[pasaranAktif];
  const p = entry.activePattern || 'A';
  if (!entry.patterns[p]) entry.patterns[p] = createEmptyPattern();
  return entry.patterns[p];
}

/* Main functions (adapted to work per-pattern) */
function hitung() {
  const f = parseInt(document.getElementById("payout").value, 10);
  const T = parseInt(document.getElementById("target").value, 10);
  const M = parseInt(document.getElementById("jumlahNomor").value, 10);
  const entry = dataSemuaPasaran[pasaranAktif];
  const patternObj = currentPatternObj();
  const L = patternObj.totalKalah || 0;
  const minBet = parseInt(document.getElementById("minTaruhan").value, 10);

  if (isNaN(f) || isNaN(T) || isNaN(M) || isNaN(minBet)) {
    alert("Masukkan nilai numerik yang valid.");
    return;
  }
  if (M >= f) {
    alert("Jumlah nomor tidak boleh ≥ payout ratio!");
    return;
  }

  let s = (T + L) / (f - M);
  s = Math.ceil(s / minBet) * minBet;
  const totalModal = M * s;
  const totalMenang = f * s;
  const net = totalMenang - totalModal;

  patternObj.modalTerakhir = totalModal;
  // save back
  entry.patterns[entry.activePattern] = patternObj;
  dataSemuaPasaran[pasaranAktif] = entry;
  simpanData();

  document.getElementById("hasil").innerHTML = `
    <p>Pasaran: <strong>${pasaranAktif}</strong> &nbsp;|&nbsp; Pola: <strong>${entry.activePattern}</strong></p>
    <p>Stake ideal per nomor: <strong>Rp${s.toLocaleString()}</strong></p>
    <p>Total modal hari ini: <strong>Rp${totalModal.toLocaleString()}</strong></p>
    <p>Total menang (1 nomor kena): <strong>Rp${totalMenang.toLocaleString()}</strong></p>
    <p><strong>Net hasil:</strong> Rp${net.toLocaleString()}</p>
  `;

  tampilkanData();
}

function tambahKalah() {
  const entry = dataSemuaPasaran[pasaranAktif];
  const patternObj = currentPatternObj();
  if (!patternObj.modalTerakhir || patternObj.modalTerakhir === 0) return alert("Hitung dulu sebelum menambah kekalahan!");
  patternObj.totalKalah = (patternObj.totalKalah || 0) + patternObj.modalTerakhir;
  patternObj.totalModal = (patternObj.totalModal || 0) + patternObj.modalTerakhir;
  patternObj.histori.push({ hari: (patternObj.histori ? patternObj.histori.length : 0) + 1, status: "Kalah", modal: patternObj.modalTerakhir, totalKalah: patternObj.totalKalah, totalModal: patternObj.totalModal });
  // write back
  entry.patterns[entry.activePattern] = patternObj;
  dataSemuaPasaran[pasaranAktif] = entry;
  simpanData();
  tampilkanData();
  alert("Total kekalahan (pola " + entry.activePattern + ") diperbarui: Rp" + patternObj.totalKalah.toLocaleString());
}

function tambahMenang() {
  const entry = dataSemuaPasaran[pasaranAktif];
  const patternObj = currentPatternObj();
  if (!patternObj.modalTerakhir || patternObj.modalTerakhir === 0) return alert("Hitung dulu sebelum mencatat kemenangan!");
  const f = parseInt(document.getElementById("payout").value, 10);
  const M = parseInt(document.getElementById("jumlahNomor").value, 10);
  const s = patternObj.modalTerakhir / M;
  const totalMenang = f * s;
  const net = totalMenang - patternObj.modalTerakhir;
  patternObj.totalKalah = Math.max(0, (patternObj.totalKalah || 0) - net);
  patternObj.totalModal = (patternObj.totalModal || 0) + patternObj.modalTerakhir;
  patternObj.histori.push({ hari: (patternObj.histori ? patternObj.histori.length : 0) + 1, status: "Menang", modal: patternObj.modalTerakhir, totalKalah: patternObj.totalKalah, totalModal: patternObj.totalModal });
  // reset only this pattern's running losses
  patternObj.modalTerakhir = 0;
  // write back
  entry.patterns[entry.activePattern] = patternObj;
  dataSemuaPasaran[pasaranAktif] = entry;
  simpanData();
  tampilkanData();
  alert("Menang tercatat untuk pola " + entry.activePattern + ". Total kekalahan pola kini: Rp" + patternObj.totalKalah.toLocaleString());
}

/* Display only the histori for the active pattern (user requested "papan ketik mengikuti lainnya") */
function tampilkanData() {
  const entry = dataSemuaPasaran[pasaranAktif];
  const pattern = entry.activePattern || 'A';
  const patternObj = entry.patterns[pattern] || createEmptyPattern();
  document.getElementById("kerugian").value = patternObj.totalKalah || 0;
  document.getElementById("totalModal").value = patternObj.totalModal || 0;

  const table = document.getElementById("tabelHistori");
  table.innerHTML = "<tr><th>Hari</th><th>Status</th><th>Modal</th><th>Total Kekalahan</th><th>Total Modal</th></tr>";
  (patternObj.histori || []).forEach(r => {
    const tr = document.createElement("tr");
    tr.innerHTML = `<td>${r.hari}</td><td>${r.status}</td><td>Rp${r.modal.toLocaleString()}</td><td>Rp${r.totalKalah.toLocaleString()}</td><td>Rp${r.totalModal.toLocaleString()}</td>`;
    tr.style.background = r.status === "Menang" ? "#d0ffd0" : "#ffd0d0";
    table.appendChild(tr);
  });

  const menang = (patternObj.histori || []).filter(x => x.status === "Menang").length;
  const kalah = (patternObj.histori || []).filter(x => x.status === "Kalah").length;
  const rataModal = (patternObj.histori && patternObj.histori.length) ? Math.round((patternObj.totalModal || 0) / patternObj.histori.length) : 0;

  document.getElementById("ringkasan").innerHTML = `
    Pasaran Aktif: <strong>${pasaranAktif}</strong> &nbsp;|&nbsp; Pola: <strong>${pattern}</strong><br>
    Total Hari (pola aktif): ${patternObj.histori ? patternObj.histori.length : 0} | Menang: ${menang} | Kalah: ${kalah}<br>
    Rata-rata Modal per Hari (pola): Rp${rataModal.toLocaleString()}<br>
    Total Modal Kumulatif (pola): Rp${(patternObj.totalModal || 0).toLocaleString()}<br>
    Total Kekalahan Saat Ini (pola): Rp${(patternObj.totalKalah || 0).toLocaleString()}
  `;
}

/* Simulasi tetap bekerja per pola (menggunakan L per pola) */
function simulasi() {
  const f = parseInt(document.getElementById("payout").value, 10);
  const T = parseInt(document.getElementById("target").value, 10);
  const M = parseInt(document.getElementById("jumlahNomor").value, 10);
  const minBet = parseInt(document.getElementById("minTaruhan").value, 10);
  let L = 0;
  let log = "";

  for (let i = 1; i <= 10; i++) {
    let s = (T + L) / (f - M);
    s = Math.ceil(s / minBet) * minBet;
    const totalModal = M * s;
    const totalMenang = f * s;
    const net = totalMenang - totalModal;
    const menang = (i === 10);
    if (menang) { L = Math.max(0, L - net); } else { L += totalModal; }
    log += `<tr><td>${i}</td><td>${menang ? "Menang" : "Kalah"}</td><td>Rp${totalModal.toLocaleString()}</td><td>Rp${L.toLocaleString()}</td></tr>`;
  }

  document.getElementById("hasil").innerHTML = `
    <h4>Simulasi 10 Hari (Menang di hari ke-10) — untuk pola aktif</h4>
    <table><tr><th>Hari</th><th>Status</th><th>Modal</th><th>Total Kekalahan</th></tr>${log}</table>
  `;
}

/* Reset pasaran aktif (hanya data pasaran itu) -> Reset only current pattern's data? 
   User previously specified Reset clears data for pasaran; we keep same behavior but make it clear:
   - Reset Pasaran: resets ALL patterns for that pasaran (as previous).
   - If user wants to reset only pattern, they can switch pattern and press Reset - it will reset all patterns.
   To keep backward compatibility, Reset clears all patterns for that pasaran (same as before).
*/
function resetPasaran() {
  if (!confirm("Reset semua data untuk pasaran ini? Ini akan menghapus histori dan saldo untuk pasaran aktif dan semua pola pada pasaran ini.")) return;
  // reset all patterns for this pasaran
  const entry = dataSemuaPasaran[pasaranAktif];
  entry.patterns = {};
  PATTERNS.forEach(p => entry.patterns[p] = createEmptyPattern());
  entry.activePattern = 'A';
  dataSemuaPasaran[pasaranAktif] = entry;
  simpanData();
  renderPatternButtons();
  tampilkanData();
}

/* Export: ambil semua data and lastActivePasaran */
function exportData() {
  try {
    const payload = {
      exportedAt: new Date().toISOString(),
      lastActivePasaran: pasaranAktif,
      dataMultiPasaran: dataSemuaPasaran
    };
    const blob = new Blob([JSON.stringify(payload, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "backup_togel.json";
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
    alert("File backup_togel.json berhasil dibuat dan diunduh.");
  } catch (e) {
    alert("Gagal mengekspor data: " + e.message);
  }
}

/* Import: trigger file input */
function triggerImport() {
  document.getElementById("importFile").value = "";
  document.getElementById("importFile").click();
}

/* Import: baca file JSON, validasi, dan tulis ke localStorage */
function importData(event) {
  const file = event.target.files[0];
  if (!file) return;
  if (!confirm("Impor data akan menimpa data lokal saat ini untuk semua pasaran dan pola. Lanjutkan?")) return;

  const reader = new FileReader();
  reader.onload = function(e) {
    try {
      const parsed = JSON.parse(e.target.result);
      // Basic validation
      if (!parsed || typeof parsed !== "object" || !parsed.dataMultiPasaran) {
        throw new Error("Format file tidak valid (tidak ditemukan dataMultiPasaran).");
      }
      // Overwrite local data
      dataSemuaPasaran = parsed.dataMultiPasaran;
      // Ensure all pasaran exist and patterns present
      ensureAllPasaranExist();
      // Restore last active pasaran if available and present
      if (parsed.lastActivePasaran && dataSemuaPasaran[parsed.lastActivePasaran]) {
        pasaranAktif = parsed.lastActivePasaran;
      } else {
        pasaranAktif = document.getElementById("pasaranSelect").value;
      }
      // Save to localStorage
      simpanData();
      // Update dropdown selection to restored pasaranAktif
      document.getElementById("pasaranSelect").value = pasaranAktif;
      renderPatternButtons();
      tampilkanData();
      alert("Data berhasil diimpor dan dipulihkan.");
    } catch (err) {
      alert("Gagal mengimpor: " + err.message);
    }
  };
  reader.onerror = function() {
    alert("Gagal membaca file impor.");
  };
  reader.readAsText(file);
}

/* On initial load, set dropdown to last active pasaran if available */
window.addEventListener("load", () => {
  const sel = document.getElementById("pasaranSelect");
  if (pasaranAktif) {
    // If pasaranAktif exists in options, set it; otherwise keep default
    let found = false;
    for (let i = 0; i < sel.options.length; i++) {
      if (sel.options[i].text === pasaranAktif) {
        sel.selectedIndex = i;
        found = true;
        break;
      }
    }
    if (!found) pasaranAktif = sel.value;
  } else {
    pasaranAktif = sel.value;
  }
  // ensure patterns exist for active pasaran
  ensureAllPasaranExist();
  // set pattern UI
  renderPatternButtons();
  simpanData();
  tampilkanData();
});
</script>

</body>
</html>

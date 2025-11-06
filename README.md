<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Kalkulator Multiple Taruhan 2D (Multi-Pasaran + Backup)</title>
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
  <h3>Histori Harian</h3>
  <table id="tabelHistori">
    <tr><th>Hari</th><th>Status</th><th>Modal</th><th>Total Kekalahan</th><th>Total Modal</th></tr>
  </table>
  <div class="summary" id="ringkasan"></div>
</div>

<script>
/*
  Storage structure:
  localStorage key: "dataMultiPasaran"
  Stored value: {
    "<PASARAN_NAME>": {
      totalKalah: number,
      totalModal: number,
      histori: [ { hari, status, modal, totalKalah, totalModal } , ... ],
      modalTerakhir: number
    },
    ...
  }
  Also store "lastActivePasaran" for convenience on export/import.
*/

let dataSemuaPasaran = JSON.parse(localStorage.getItem("dataMultiPasaran") || "{}");
let pasaranAktif = localStorage.getItem("lastActivePasaran") || document.getElementById("pasaranSelect").value;

// Initialize missing pasaran entries
function ensureAllPasaranExist() {
  const sel = document.getElementById("pasaranSelect");
  for (let i = 0; i < sel.options.length; i++) {
    const name = sel.options[i].text;
    if (!dataSemuaPasaran[name]) {
      dataSemuaPasaran[name] = { totalKalah: 0, totalModal: 0, histori: [], modalTerakhir: 0 };
    }
  }
}
ensureAllPasaranExist();
if (!dataSemuaPasaran[pasaranAktif]) pasaranAktif = document.getElementById("pasaranSelect").value;

function simpanData() {
  localStorage.setItem("dataMultiPasaran", JSON.stringify(dataSemuaPasaran));
  localStorage.setItem("lastActivePasaran", pasaranAktif);
}

function gantiPasaran() {
  pasaranAktif = document.getElementById("pasaranSelect").value;
  if (!dataSemuaPasaran[pasaranAktif]) {
    dataSemuaPasaran[pasaranAktif] = { totalKalah: 0, totalModal: 0, histori: [], modalTerakhir: 0 };
  }
  simpanData();
  tampilkanData();
}

function hitung() {
  const f = parseInt(document.getElementById("payout").value, 10);
  const T = parseInt(document.getElementById("target").value, 10);
  const M = parseInt(document.getElementById("jumlahNomor").value, 10);
  const d = dataSemuaPasaran[pasaranAktif];
  const L = d.totalKalah || 0;
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

  d.modalTerakhir = totalModal;
  dataSemuaPasaran[pasaranAktif] = d;
  simpanData();

  document.getElementById("hasil").innerHTML = `
    <p>Stake ideal per nomor: <strong>Rp${s.toLocaleString()}</strong></p>
    <p>Total modal hari ini: <strong>Rp${totalModal.toLocaleString()}</strong></p>
    <p>Total menang (1 nomor kena): <strong>Rp${totalMenang.toLocaleString()}</strong></p>
    <p><strong>Net hasil:</strong> Rp${net.toLocaleString()}</p>
  `;

  tampilkanData();
}

function tambahKalah() {
  const d = dataSemuaPasaran[pasaranAktif];
  if (!d.modalTerakhir || d.modalTerakhir === 0) return alert("Hitung dulu sebelum menambah kekalahan!");
  d.totalKalah = (d.totalKalah || 0) + d.modalTerakhir;
  d.totalModal = (d.totalModal || 0) + d.modalTerakhir;
  d.histori.push({ hari: d.histori.length + 1, status: "Kalah", modal: d.modalTerakhir, totalKalah: d.totalKalah, totalModal: d.totalModal });
  dataSemuaPasaran[pasaranAktif] = d;
  simpanData();
  tampilkanData();
  alert("Total kekalahan diperbarui: Rp" + d.totalKalah.toLocaleString());
}

function tambahMenang() {
  const d = dataSemuaPasaran[pasaranAktif];
  if (!d.modalTerakhir || d.modalTerakhir === 0) return alert("Hitung dulu sebelum mencatat kemenangan!");
  const f = parseInt(document.getElementById("payout").value, 10);
  const M = parseInt(document.getElementById("jumlahNomor").value, 10);
  const s = d.modalTerakhir / M;
  const totalMenang = f * s;
  const net = totalMenang - d.modalTerakhir;
  d.totalKalah = Math.max(0, (d.totalKalah || 0) - net);
  d.totalModal = (d.totalModal || 0) + d.modalTerakhir;
  d.histori.push({ hari: d.histori.length + 1, status: "Menang", modal: d.modalTerakhir, totalKalah: d.totalKalah, totalModal: d.totalModal });
  dataSemuaPasaran[pasaranAktif] = d;
  simpanData();
  tampilkanData();
  alert("Menang tercatat. Total kekalahan kini: Rp" + d.totalKalah.toLocaleString());
}

function tampilkanData() {
  const d = dataSemuaPasaran[pasaranAktif];
  document.getElementById("kerugian").value = d.totalKalah || 0;
  document.getElementById("totalModal").value = d.totalModal || 0;

  const table = document.getElementById("tabelHistori");
  table.innerHTML = "<tr><th>Hari</th><th>Status</th><th>Modal</th><th>Total Kekalahan</th><th>Total Modal</th></tr>";
  (d.histori || []).forEach(r => {
    const tr = document.createElement("tr");
    tr.innerHTML = `<td>${r.hari}</td><td>${r.status}</td><td>Rp${r.modal.toLocaleString()}</td><td>Rp${r.totalKalah.toLocaleString()}</td><td>Rp${r.totalModal.toLocaleString()}</td>`;
    tr.style.background = r.status === "Menang" ? "#d0ffd0" : "#ffd0d0";
    table.appendChild(tr);
  });

  const menang = (d.histori || []).filter(x => x.status === "Menang").length;
  const kalah = (d.histori || []).filter(x => x.status === "Kalah").length;
  const rataModal = (d.histori && d.histori.length) ? Math.round((d.totalModal || 0) / d.histori.length) : 0;

  document.getElementById("ringkasan").innerHTML = `
    Pasaran Aktif: <strong>${pasaranAktif}</strong><br>
    Total Hari: ${d.histori ? d.histori.length : 0} | Menang: ${menang} | Kalah: ${kalah}<br>
    Rata-rata Modal per Hari: Rp${rataModal.toLocaleString()}<br>
    Total Modal Kumulatif: Rp${(d.totalModal || 0).toLocaleString()}<br>
    Total Kekalahan Saat Ini: Rp${(d.totalKalah || 0).toLocaleString()}
  `;
}

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
    <h4>Simulasi 10 Hari (Menang di hari ke-10)</h4>
    <table><tr><th>Hari</th><th>Status</th><th>Modal</th><th>Total Kekalahan</th></tr>${log}</table>
  `;
}

/* Reset pasaran aktif (hanya data pasaran itu) */
function resetPasaran() {
  if (!confirm("Reset semua data untuk pasaran ini? Ini akan menghapus histori dan saldo untuk pasaran aktif.")) return;
  dataSemuaPasaran[pasaranAktif] = { totalKalah: 0, totalModal: 0, histori: [], modalTerakhir: 0 };
  simpanData();
  tampilkanData();
}

/* Export: ambil semua data dan pasaran aktif terakhir, buat file JSON dan download */
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
  if (!confirm("Impor data akan menimpa data lokal saat ini untuk semua pasaran. Lanjutkan?")) return;

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
      // Ensure missing pasaran from dropdown are created (so dropdown options remain valid)
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
  simpanData();
  tampilkanData();
});
</script>

</body>
</html>

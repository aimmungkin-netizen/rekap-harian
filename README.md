<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Kalkulator Multiple Taruhan 2D (Multi-Pasaran + Backup) — Final</title>
<style>
  body { font-family: Arial, sans-serif; margin: 20px; background: #f9f9f9; color: #333; }
  h2 { color: #0077cc; margin-bottom: 8px; }
  select, input, textarea { padding: 6px; margin-top: 4px; border-radius: 5px; border: 1px solid #aaa; font-size: 14px; }
  textarea { resize: vertical; }
  button { padding: 8px 12px; border: none; border-radius: 5px; cursor: pointer; font-size: 14px; margin: 5px 6px 5px 0; }
  button:hover { opacity: 0.95; }
  .primary { background: #0077cc; color: white; }
  .danger { background: #cc0000; color: white; }
  .success { background: #009933; color: white; }
  .container { background: #fff; padding: 14px; border-radius: 10px; box-shadow: 0 2px 6px rgba(0,0,0,0.08); margin-bottom: 18px; }
  table { border-collapse: collapse; width: 100%; margin-top: 10px; font-size: 13px; }
  th, td { border: 1px solid #ddd; padding: 8px; text-align: center; }
  th { background: #e8f6ff; }
  .summary { margin-top: 10px; background: #eef8ee; padding: 10px; border-radius: 8px; }
  .controls-row { display:flex; gap:8px; align-items:center; flex-wrap:wrap; margin-top:8px; }
  #statusHariIni { color:#009933; font-weight:700; margin-top:10px; }
  .small { font-size:13px; color:#555; }
</style>
</head>
<body>

<h2>Kalkulator Multiple Taruhan 2D — Multi Pasaran</h2>

<div class="container">
  <label class="small">Pilih Pasaran:</label><br>
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
    <option>KAMBOJA</option>
    <option>JEJU</option>
    <option>PCSO</option>
    <option>PENANG</option>
    <option>TAIWAN</option>
    <option>HONGKONG</option>
    <option>HONGKONG LOTTO</option>
  </select>
</div>

<div class="container">
  <label class="small">Jumlah nomor dipasang (M):</label><br>
  <input type="number" id="jumlahNomor" value="10" min="1" max="99">
  
  <label class="small" style="margin-left:10px">Payout ratio:</label><br>
  <input type="number" id="payout" value="100" style="margin-right:10px">
  
  <label class="small">Target keuntungan per menang:</label><br>
  <input type="number" id="target" value="50000">
  
  <label class="small" style="margin-left:10px">Minimal taruhan per nomor:</label><br>
  <input type="number" id="minTaruhan" value="100" step="100">
  
  <p class="small">Total kekalahan (otomatis): <input id="kerugian" value="0" readonly style="border:none; background:transparent;"></p>
  <p class="small">Total modal kumulatif: <input id="totalModal" value="0" readonly style="border:none; background:transparent;"></p>

  <div class="controls-row">
    <button class="primary" onclick="hitung()">Hitung Multiple</button>
    <button class="success" onclick="tambahKalah()">Tambah Kekalahan</button>
    <button class="success" onclick="tambahMenang()">Menang Hari Ini</button>
    <button onclick="simulasi()">Simulasi 10 Hari</button>
    <button class="danger" onclick="resetPasaran()">Reset Pasaran</button>
    <button onclick="batalkanHariTerakhir()">Batalkan Hari Terakhir</button>
    <button onclick="exportData()">Ekspor Data</button>
    <button onclick="triggerImport()">Impor Data</button>
    <input type="file" id="importFile" accept=".json" style="display:none" onchange="importData(event)">
  </div>

  <p id="statusHariIni"></p>
</div>

<div class="container" id="hasil"></div>

<div class="container">
  <h3>Histori Harian</h3>
  <table id="tabelHistori">
    <tr><th>Hari</th><th>Tanggal</th><th>Status</th><th>Modal</th><th>Total Kekalahan</th><th>Total Modal</th></tr>
  </table>
  <div class="summary" id="ringkasan"></div>
</div>

<!-- Textarea catatan per pasaran: akan ditaruh paling bawah -->
<div class="container">
  <h3>Catatan Keluaran (per pasaran)</h3>
  <textarea id="catatanPasaran" rows="8" style="width:100%"></textarea>
  <div style="margin-top:8px">
    <button onclick="simpanCatatan()" class="primary">Simpan</button>
    <button onclick="hapusCatatan()" class="danger">Hapus</button>
    <button onclick="salinCatatan()" class="success">Salin</button>
  </div>
  <p class="small" style="margin-top:8px">Catatan disimpan per pasaran dan akan muncul otomatis saat Anda mengganti pasaran.</p>
</div>

<script>
/*
  Struktur disimpan di localStorage key "dataMultiPasaran"
  Per pasaran:
  {
    totalKalah: number,
    totalModal: number,
    histori: [ { hari, tanggal, status, modal, totalKalah, totalModal }, ... ],
    modalTerakhir: number,
    lastActionDate: "YYYY-MM-DD",  // tanggal terakhir ada aksi
    catatan: "..."                 // textarea per pasaran
  }
*/

let dataSemuaPasaran = JSON.parse(localStorage.getItem("dataMultiPasaran") || "{}");
let pasaranAktif = localStorage.getItem("lastActivePasaran") || document.getElementById("pasaranSelect").value;

function ensureAllPasaranExist() {
  const sel = document.getElementById("pasaranSelect");
  for (let i = 0; i < sel.options.length; i++) {
    const name = sel.options[i].text;
    if (!dataSemuaPasaran[name]) {
      dataSemuaPasaran[name] = {
        totalKalah: 0,
        totalModal: 0,
        histori: [],
        modalTerakhir: 0,
        lastActionDate: "",
        catatan: ""
      };
    } else {
      // ensure keys exist for older data
      const d = dataSemuaPasaran[name];
      if (typeof d.totalKalah === "undefined") d.totalKalah = 0;
      if (typeof d.totalModal === "undefined") d.totalModal = 0;
      if (!Array.isArray(d.histori)) d.histori = [];
      if (typeof d.modalTerakhir === "undefined") d.modalTerakhir = 0;
      if (typeof d.lastActionDate === "undefined") d.lastActionDate = "";
      if (typeof d.catatan === "undefined") d.catatan = "";
    }
  }
}
ensureAllPasaranExist();

function simpanData() {
  localStorage.setItem("dataMultiPasaran", JSON.stringify(dataSemuaPasaran));
  localStorage.setItem("lastActivePasaran", pasaranAktif);
}

function gantiPasaran() {
  pasaranAktif = document.getElementById("pasaranSelect").value;
  if (!dataSemuaPasaran[pasaranAktif]) {
    dataSemuaPasaran[pasaranAktif] = { totalKalah: 0, totalModal: 0, histori: [], modalTerakhir: 0, lastActionDate: "", catatan: "" };
  }
  simpanData();
  tampilkanData();
}

/* Helper: tanggal ISO (YYYY-MM-DD) untuk perbandingan dan format lokal untuk tampilan */
function tanggalISO(d) {
  return d.toISOString().split("T")[0];
}
function tanggalLokalSekarang() {
  return new Date().toLocaleDateString("id-ID");
}
function hariIniISO() {
  return tanggalISO(new Date());
}

/* Update indikator status hari ini (hijau) */
function updateStatusHariIni(status) {
  const el = document.getElementById("statusHariIni");
  if (!status) el.innerText = "";
  else el.innerText = `📅 Hari ini sudah dicatat: ${status}`;
}

/* Perhitungan utama (tidak dirubah logika) */
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

/* Tambah Kekalahan dengan confirm + pembatas 1x per hari */
function tambahKalah() {
  const d = dataSemuaPasaran[pasaranAktif];
  const todayISO = hariIniISO();
  if (d.lastActionDate === todayISO) return alert("Hari ini sudah dicatat!");
  if (!confirm("Yakin ingin menandai hari ini sebagai KALAH?")) return;
  if (!d.modalTerakhir || d.modalTerakhir === 0) return alert("Hitung dulu sebelum menambah kekalahan!");

  d.totalKalah = (d.totalKalah || 0) + d.modalTerakhir;
  d.totalModal = (d.totalModal || 0) + d.modalTerakhir;
  d.lastActionDate = todayISO;
  d.histori.push({
    hari: d.histori.length + 1,
    tanggal: tanggalLokalSekarang(),
    status: "Kalah",
    modal: d.modalTerakhir,
    totalKalah: d.totalKalah,
    totalModal: d.totalModal
  });
  dataSemuaPasaran[pasaranAktif] = d;
  simpanData();
  tampilkanData();
  updateStatusHariIni("KALAH");
}

/* Tambah Menang dengan confirm + pembatas 1x per hari */
function tambahMenang() {
  const d = dataSemuaPasaran[pasaranAktif];
  const todayISO = hariIniISO();
  if (d.lastActionDate === todayISO) return alert("Hari ini sudah dicatat!");
  if (!confirm("Yakin ingin menandai hari ini sebagai MENANG?")) return;
  if (!d.modalTerakhir || d.modalTerakhir === 0) return alert("Hitung dulu sebelum mencatat kemenangan!");

  const f = parseInt(document.getElementById("payout").value, 10);
  const M = parseInt(document.getElementById("jumlahNomor").value, 10);
  const s = d.modalTerakhir / M;
  const totalMenang = f * s;
  const net = totalMenang - d.modalTerakhir;
  d.totalKalah = Math.max(0, (d.totalKalah || 0) - net);
  d.totalModal = (d.totalModal || 0) + d.modalTerakhir;
  d.lastActionDate = todayISO;
  d.histori.push({
    hari: d.histori.length + 1,
    tanggal: tanggalLokalSekarang(),
    status: "Menang",
    modal: d.modalTerakhir,
    totalKalah: d.totalKalah,
    totalModal: d.totalModal
  });
  dataSemuaPasaran[pasaranAktif] = d;
  simpanData();
  tampilkanData();
  updateStatusHariIni("MENANG");
}

/* Batalkan Hari Terakhir (undo 1 langkah) */
function batalkanHariTerakhir() {
  const d = dataSemuaPasaran[pasaranAktif];
  if (!d.histori || d.histori.length === 0) return alert("Belum ada histori untuk dibatalkan.");
  if (!confirm("Batalkan (hapus) entri hari terakhir? Aksi ini hanya menghapus 1 entri terakhir.")) return;

  // Buang entri terakhir
  d.histori.pop();

  // Rehitung totalKalah & totalModal dari histori yang tersisa (lebih aman daripada mencoba rollback langsung)
  let totalKalah = 0;
  let totalModal = 0;
  for (const r of d.histori) {
    totalModal += r.modal;
    totalKalah = r.totalKalah; // setiap r.totalKalah sudah merepresentasikan kumulatif pada titik itu
  }
  // Jika histori kosong, reset totals ke 0
  if (d.histori.length === 0) {
    totalKalah = 0;
    totalModal = 0;
  } else {
    // pastikan konsistensi: gunakan nilai totalModal terakhir dari objek terakhir histori jika ada
    totalModal = d.histori[d.histori.length - 1].totalModal || totalModal;
    totalKalah = d.histori[d.histori.length - 1].totalKalah || totalKalah;
  }

  d.totalKalah = totalKalah;
  d.totalModal = totalModal;

  // Perbarui lastActionDate ke tanggal entri terakhir jika ada, atau kosongkan
  if (d.histori.length > 0) {
    // ambil tanggal terakhir histori sebagai ISO untuk lastActionDate
    const lastTanggalLocal = d.histori[d.histori.length - 1].tanggal; // format 'dd/mm/yyyy'
    // coba parse dd/mm/yyyy menjadi ISO simple
    const parts = lastTanggalLocal.split('/');
    if (parts.length === 3) {
      const iso = `${parts[2]}-${parts[1].padStart(2,'0')}-${parts[0].padStart(2,'0')}`;
      d.lastActionDate = iso;
    } else {
      d.lastActionDate = "";
    }
  } else {
    d.lastActionDate = "";
  }

  // If modalTerakhir still valid keep it; do not modify modalTerakhir as calculation code assumes user calls Hitung() first.
  dataSemuaPasaran[pasaranAktif] = d;
  simpanData();
  tampilkanData();
  alert("Entri hari terakhir berhasil dibatalkan.");
}

/* Tampilkan data & histori (menambahkan kolom Tanggal) */
function tampilkanData() {
  const d = dataSemuaPasaran[pasaranAktif];
  document.getElementById("kerugian").value = d.totalKalah || 0;
  document.getElementById("totalModal").value = d.totalModal || 0;

  const table = document.getElementById("tabelHistori");
  table.innerHTML = "<tr><th>Hari</th><th>Tanggal</th><th>Status</th><th>Modal</th><th>Total Kekalahan</th><th>Total Modal</th></tr>";
  (d.histori || []).forEach(r => {
    const tr = document.createElement("tr");
    tr.innerHTML = `<td>${r.hari}</td><td>${r.tanggal || ""}</td><td>${r.status}</td><td>Rp${r.modal.toLocaleString()}</td><td>Rp${r.totalKalah.toLocaleString()}</td><td>Rp${r.totalModal.toLocaleString()}</td>`;
    tr.style.background = r.status === "Menang" ? "#dfffe0" : "#ffe0e0";
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

  // Update status indikator (hijau) jika lastActionDate adalah hari ini
  if (d.lastActionDate === hariIniISO()) {
    // gunakan status dari histori terakhir jika ada
    const lastStatus = (d.histori && d.histori.length) ? d.histori[d.histori.length - 1].status.toUpperCase() : "";
    updateStatusHariIni(lastStatus);
  } else {
    updateStatusHariIni();
  }

  // Load catatan pasaran ke textarea
  document.getElementById("catatanPasaran").value = d.catatan || "";
}

/* Simulasi (tidak diubah logikanya) */
function simulasi() {
  const f = parseInt(document.getElementById("payout").value, 10);
  const T = parseInt(document.getElementById("target").value, 10);
  const M = parseInt(document.getElementById("jumlahNomor").value, 10);
  const minBet = parseInt(document.getElementById("minTaruhan").value, 10);
  let L = 0, log = "";
  for (let i = 1; i <= 10; i++) {
    let s = (T + L) / (f - M);
    s = Math.ceil(s / minBet) * minBet;
    const totalModal = M * s, totalMenang = f * s, net = totalMenang - totalModal;
    const menang = (i === 10);
    if (menang) L = Math.max(0, L - net); else L += totalModal;
    log += `<tr><td>${i}</td><td>${menang ? "Menang" : "Kalah"}</td><td>Rp${totalModal.toLocaleString()}</td><td>Rp${L.toLocaleString()}</td></tr>`;
  }
  document.getElementById("hasil").innerHTML = `<h4>Simulasi 10 Hari (Menang di hari ke-10)</h4><table><tr><th>Hari</th><th>Status</th><th>Modal</th><th>Total Kekalahan</th></tr>${log}</table>`;
}

/* Reset pasaran aktif (tetap konfirmasi) */
function resetPasaran() {
  if (!confirm("Reset semua data untuk pasaran ini? Ini akan menghapus histori dan saldo untuk pasaran aktif.")) return;
  dataSemuaPasaran[pasaranAktif] = { totalKalah: 0, totalModal: 0, histori: [], modalTerakhir: 0, lastActionDate: "", catatan: "" };
  simpanData();
  tampilkanData();
}

/* Export / Import (termasuk semua field baru) */
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
function triggerImport() { document.getElementById("importFile").value = ""; document.getElementById("importFile").click(); }
function importData(event) {
  const file = event.target.files[0];
  if (!file) return;
  if (!confirm("Impor data akan menimpa data lokal saat ini untuk semua pasaran. Lanjutkan?")) return;
  const reader = new FileReader();
  reader.onload = function(e) {
    try {
      const parsed = JSON.parse(e.target.result);
      if (!parsed || typeof parsed !== "object" || !parsed.dataMultiPasaran) {
        throw new Error("Format file tidak valid (tidak ditemukan dataMultiPasaran).");
      }
      dataSemuaPasaran = parsed.dataMultiPasaran;
      ensureAllPasaranExist();
      if (parsed.lastActivePasaran && dataSemuaPasaran[parsed.lastActivePasaran]) {
        pasaranAktif = parsed.lastActivePasaran;
      } else {
        pasaranAktif = document.getElementById("pasaranSelect").value;
      }
      simpanData();
      document.getElementById("pasaranSelect").value = pasaranAktif;
      tampilkanData();
      alert("Data berhasil diimpor dan dipulihkan.");
    } catch (err) {
      alert("Gagal mengimpor: " + err.message);
    }
  };
  reader.onerror = function() { alert("Gagal membaca file impor."); };
  reader.readAsText(file);
}

/* Catatan per pasaran: simpan / hapus / salin */
function simpanCatatan() {
  const teks = document.getElementById("catatanPasaran").value;
  dataSemuaPasaran[pasaranAktif].catatan = teks;
  simpanData();
  alert("Catatan tersimpan untuk pasaran " + pasaranAktif + ".");
}
function hapusCatatan() {
  if (!confirm("Hapus catatan pasaran ini?")) return;
  dataSemuaPasaran[pasaranAktif].catatan = "";
  simpanData();
  document.getElementById("catatanPasaran").value = "";
  alert("Catatan dihapus.");
}
function salinCatatan() {
  const teks = document.getElementById("catatanPasaran").value;
  if (!teks) return alert("Tidak ada teks untuk disalin.");
  navigator.clipboard?.writeText(teks).then(() => {
    alert("Catatan disalin ke clipboard.");
  }).catch(() => {
    // fallback
    const ta = document.createElement("textarea");
    ta.value = teks; document.body.appendChild(ta); ta.select();
    try { document.execCommand("copy"); alert("Catatan disalin ke clipboard."); } catch (e) { alert("Gagal menyalin."); }
    ta.remove();
  });
}

/* Inisialisasi on page load */
window.addEventListener("load", () => {
  ensureAllPasaranExist();
  const sel = document.getElementById("pasaranSelect");
  let found = false;
  for (let i = 0; i < sel.options.length; i++) {
    if (sel.options[i].text === pasaranAktif) { sel.selectedIndex = i; found = true; break; }
  }
  if (!found) pasaranAktif = sel.value;
  simpanData();
  tampilkanData();
});
</script>

</body>
</html>

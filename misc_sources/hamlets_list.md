# Sortable & Filterable Table of All Hamlets in Umbria

*From the OpenStreetMap database.*

You can filter by **name** and **municipality**, and sort each column by clicking the headers. Coordinates link to Google Maps.

If the place you want isn't here, it might be found in this [list from the 1971 census](https://lipari.istat.it/digibib/Censimenti%20popolazione/censpop1971/IST0007188VOLIII_frazioni_geografiche/IST0004590CP1971_fasc10_UMBRIA+OCRottimizz.pdf#page=11).

---

<div id="filters">
  <label>Name: <input type="text" id="nameFilter"></label>
  <label>Municipality: <input type="text" id="muniFilter"></label>
</div>

<div id="datasource">
  <label><input type="radio" name="source" value="hamlets" checked> Modern Hamlets (OSM)</label>
  <label><input type="radio" name="source" value="stradario"> Stradario Umbria 2025</label>
  <label><input type="radio" name="source" value="papal"> Papal States 1836</label>
</div>

<table id="sortable">
  <thead>
    <tr></tr>
  </thead>
  <tbody></tbody>
</table>

<style>
#sortable {
  border-collapse: collapse;
  width: 100%;
  margin-top: 10px;
  font-family: Arial, sans-serif;
  font-size: 14px;
}
#sortable th, #sortable td {
  border: 1px solid #ccc;
  padding: 4px 6px;
  text-align: left;
}
#sortable th {
  cursor: pointer;
  background: #f2f2f2;
}
#sortable tr:nth-child(even) {
  background: #fafafa;
}
#sortable tr:hover {
  background: #f1f7ff;
}
#filters input {
  margin: 5px;
  padding: 4px;
}
</style>

<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<script>
const sources = {
  hamlets: {
    file: "../docs/hamlets.csv",
    headers: ["name","place","population","muni_name","coord"],
    muniIndex: 3,
    renderRow: row => `
      <td>${row.name}</td>
      <td>${row.place}</td>
      <td>${row.population}</td>
      <td>${row.muni_name}</td>
      <td><a href="https://www.google.com/maps?q=${row.lat},${row.lon}" target="_blank">
          ${row.lat}, ${row.lon}</a></td>
    `
  },
  papal: {
    file: "../docs/papal_pops_1836.csv",
    headers: ["name","type","comune","governo","distretto","legazione","diocesi","population"],
    muniIndex: 2,
    renderRow: row => `
      <td>${row.name}</td>
      <td>${row.type}</td>
      <td>${row.comune}</td>
      <td>${row.governo}</td>
      <td>${row.distretto}</td>
      <td>${row.legazione}</td>
      <td>${row.diocesi}</td>
      <td>${row.population}</td>
    `
  },
  stradario: {
    file: "../docs/stradarioUmbria20250903.csv",
    headers: ["Street Name","ID","Locality","Municipality"],
    muniIndex: 3,
    renderRow: row => `
      <td>${row.name}</td>
      <td>${row.ID}</td>
      <td>${row.locality}</td>
      <td>${row.municipality}</td>
    `
  }
};

let currentSource = "hamlets";
let bufferedRows = [];
let renderedCount = 0;
let parseComplete = false;
const PAGE_SIZE = 200;

function loadTable(sourceKey) {
  currentSource = sourceKey;
  bufferedRows = [];
  renderedCount = 0;
  parseComplete = false;

  const source = sources[sourceKey];
  const theadRow = document.querySelector("#sortable thead tr");
  const tbody = document.querySelector("#sortable tbody");

  // Clear existing
  theadRow.innerHTML = "";
  tbody.innerHTML = "";

  // Build headers
  source.headers.forEach(h => {
    const th = document.createElement("th");
    th.textContent = h;
    theadRow.appendChild(th);
  });

  // Parse CSV streaming
  Papa.parse(source.file, {
    download: true,
    header: true,
    skipEmptyLines: true,
    comments: "#",
    step: function(row) {
      const r = row.data;
      if (!r.name) return;
      bufferedRows.push(r);
      if (bufferedRows.length >= PAGE_SIZE) {
        flushRows();
      }
    },
    complete: function() {
      parseComplete = true;
      if (bufferedRows.length > 0) flushRows();
      enableSorting();
      enableFiltering();
    },
    error: err => console.error("Error loading CSV:", err)
  });
}

function flushRows() {
  const source = sources[currentSource];
  const tbody = document.querySelector("#sortable tbody");

  if (bufferedRows.length === 0) return;

  const chunk = bufferedRows.splice(0, PAGE_SIZE);
  chunk.forEach(r => {
    const tr = document.createElement("tr");
    tr.innerHTML = source.renderRow(r);
    tbody.appendChild(tr);
  });
  renderedCount += chunk.length;
}

// Infinite scroll
window.addEventListener("scroll", () => {
  if (window.innerHeight + window.scrollY >= document.body.offsetHeight - 200) {
    if (bufferedRows.length > 0) {
      flushRows();
    } else if (parseComplete) {
      // no more data -> disable listener (optional)
      // window.removeEventListener("scroll", arguments.callee);
    }
  }
});

// Sorting
function enableSorting() {
  document.querySelectorAll("th").forEach((th, i) => {
    th.onclick = () => {
      const tbody = th.closest("table").querySelector("tbody");
      const rows = Array.from(tbody.querySelectorAll("tr"));
      const asc = !th.classList.contains("asc");

      rows.sort((a, b) => {
        const A = a.children[i].textContent.trim();
        const B = b.children[i].textContent.trim();
        const nA = parseFloat(A.replace(/,/g, ""));
        const nB = parseFloat(B.replace(/,/g, ""));
        if (!isNaN(nA) && !isNaN(nB)) return asc ? nA - nB : nB - nA;
        return asc ? A.localeCompare(B) : B.localeCompare(A);
      });

      rows.forEach(r => tbody.appendChild(r));
      th.parentElement.querySelectorAll("th").forEach(x => x.classList.remove("asc","desc"));
      th.classList.add(asc ? "asc" : "desc");
    };
  });
}

// Filtering with debounce
function enableFiltering() {
  const nf = document.getElementById("nameFilter");
  const mf = document.getElementById("muniFilter");
  const muniIndex = sources[currentSource].muniIndex;

  let debounce;
  function filter() {
    clearTimeout(debounce);
    debounce = setTimeout(() => {
      const n = nf.value.toLowerCase();
      const m = mf.value.toLowerCase();
      document.querySelectorAll("#sortable tbody tr").forEach(r => {
        const name = (r.children[0]?.textContent || "").toLowerCase();
        const muni = (r.children[muniIndex]?.textContent || "").toLowerCase();
        r.style.display = (name.includes(n) && muni.includes(m)) ? "" : "none";
      });
    }, 200);
  }
  nf.oninput = filter;
  mf.oninput = filter;
}

// === Init ===
document.querySelectorAll("input[name=source]").forEach(radio => {
  radio.addEventListener("change", e => loadTable(e.target.value));
});
loadTable("hamlets");
</script>

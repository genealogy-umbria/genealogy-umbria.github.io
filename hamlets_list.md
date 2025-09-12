# Sortable & Filterable Table of All Hamlets in Umbria

*From the OpenStreetMap database.*

You can filter by **name** and **municipality**, and sort each column by clicking the headers. Coordinates link to Google Maps.

If the place you want isn't here, it might be found in this [list from the 1971 census](https://lipari.istat.it/digibib/Censimenti%20popolazione/censpop1971/IST0007188VOLIII_frazioni_geografiche/IST0004590CP1971_fasc10_UMBRIA+OCRottimizz.pdf#page=11).

---

<div id="filters">
  <label>Name: <input type="text" id="nameFilter"></label>
  <label>Municipality: <input type="text" id="muniFilter"></label>
</div>

<table id="sortable">
  <thead>
    <tr>
      <th>name</th>
      <th>place</th>
      <th>population</th>
      <th>muni_name</th>
      <th>coord</th>
    </tr>
  </thead>
  <tbody></tbody>
</table>

<style>
/* Inline style, scoped to #sortable */
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
// Load CSV and build table
Papa.parse("docs/hamlets.csv", {
  download: true,
  header: true,
  skipEmptyLines: true,
  complete: function(results) {
    const tbody = document.querySelector("#sortable tbody");

    results.data.forEach(row => {
      if (!row.name) return;

      const tr = document.createElement("tr");
      tr.innerHTML = `
        <td>${row.name}</td>
        <td>${row.place}</td>
        <td>${row.population}</td>
        <td>${row.muni_name}</td>
        <td><a href="https://www.google.com/maps?q=${row.lat},${row.lon}" target="_blank">
            ${row.lat}, ${row.lon}</a></td>
      `;
      tbody.appendChild(tr);
    });

    enableSorting();
    enableFiltering();
  },
  error: function(err) {
    console.error("Error loading CSV:", err);
  }
});

// Sorting
function enableSorting() {
  document.querySelectorAll("th").forEach((th, i) => {
    th.addEventListener("click", () => {
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
      th.parentElement.querySelectorAll("th").forEach(x => x.classList.remove("asc", "desc"));
      th.classList.add(asc ? "asc" : "desc");
    });
  });
}

// Filtering
function enableFiltering() {
  const nf = document.getElementById("nameFilter");
  const mf = document.getElementById("muniFilter");

  function filter() {
    const n = nf.value.toLowerCase();
    const m = mf.value.toLowerCase();

    document.querySelectorAll("#sortable tbody tr").forEach(r => {
      const name = r.children[0].textContent.toLowerCase();
      const muni = r.children[3].textContent.toLowerCase();
      r.style.display = (name.includes(n) && muni.includes(m)) ? "" : "none";
    });
  }
  nf.addEventListener("input", filter);
  mf.addEventListener("input", filter);
}
</script>

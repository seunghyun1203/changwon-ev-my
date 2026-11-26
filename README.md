# changwon-ev-my
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>창원시 전기차 충전 실측 검증 및 증설·운영 대시보드</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    body { font-family: 'Malgun Gothic', sans-serif; margin: 24px; background: #f8fafc; color: #1e293b; }
    .container { max-width: 1180px; margin: 0 auto; }
    .card { background: #fff; border-radius: 12px; padding: 20px; margin-bottom: 20px; box-shadow: 0 2px 8px rgba(0,0,0,0.06); }
    .summary-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 12px; margin-bottom: 16px; }
    .kpi { background: #f1f5f9; border-radius: 10px; padding: 14px; }
    .kpi-label { font-size: 12px; color: #475569; }
    .kpi-val { font-size: 20px; font-weight: bold; margin-top: 4px; }
    .controls { display: flex; flex-wrap: wrap; gap: 16px; margin-bottom: 16px; align-items: center; }
    select { padding: 6px 12px; border-radius: 6px; border: 1px solid #cbd5e1; font-size: 14px; }
    table { width: 100%; border-collapse: collapse; font-size: 13px; }
    th, td { border: 1px solid #e2e8f0; padding: 8px 10px; text-align: left; }
    th { background: #f1f5f9; }
    .tag { padding: 3px 8px; border-radius: 999px; font-size: 12px; font-weight: bold; display: inline-block; }
    .expand { background: #fecaca; color: #991b1b; }
    .operate { background: #fed7aa; color: #9a3412; }
    .monitor { background: #dcfce7; color: #166534; }
  </style>
</head>
<body>
<div class="container">
  <h1>창원시 전기차 충전 실측 검증 및 증설·운영 검토</h1>

  <div class="card">
    <div class="controls">
      <label>
        <strong>행정구 필터:</strong>
        <select id="districtFilter" onchange="render()">
          <option value="ALL">전체 구</option>
          <option value="성산구">성산구</option>
          <option value="의창구">의창구</option>
          <option value="진해구">진해구</option>
          <option value="마산합포구">마산합포구</option>
          <option value="마산회원구">마산회원구</option>
        </select>
      </label>
      <label>
        <strong>검토 구분:</strong>
        <select id="actionFilter" onchange="render()">
          <option value="ALL">전체</option>
          <option value="증설·고출력 검토">증설·고출력 검토 (점유율 ≥ 75%)</option>
          <option value="운영 개선 우선">운영 개선 우선 (55% ≤ 점유율 < 75%)</option>
          <option value="현행 유지·모니터링">현행 유지·모니터링 (점유율 < 55%)</option>
        </select>
      </label>
    </div>

    <div class="summary-grid" id="kpiBox"></div>
    <canvas id="compareChart" height="100"></canvas>
  </div>

  <div class="card">
    <h2>구간별 실측 검증 및 증설·운영 판단표</h2>
    <div style="overflow-x:auto;">
      <table>
        <thead>
          <tr>
            <th>행정구</th>
            <th>요일</th>
            <th>시간대</th>
            <th>기준 환산(kWh)</th>
            <th>실측 충전량(kWh)</th>
            <th>오차율(%)</th>
            <th>실측 점유율</th>
            <th>검토 방향</th>
          </tr>
        </thead>
        <tbody id="resultBody"></tbody>
      </table>
    </div>
  </div>
</div>

<script>
let mergedRows = [];
let chart = null;

function parseCSV(text) {
  const lines = text.trim().split(/\r?\n/);
  const headers = lines[0].split(",").map(h => h.trim());
  return lines.slice(1).filter(Boolean).map(line => {
    const cols = line.split(",");
    const obj = {};
    headers.forEach((h, i) => obj[h] = (cols[i] ?? "").trim());
    return obj;
  });
}

function classifyAction(occ) {
  if (occ >= 0.75) return { label: "증설·고출력 검토", cls: "expand" };
  if (occ >= 0.55) return { label: "운영 개선 우선", cls: "operate" };
  return { label: "현행 유지·모니터링", cls: "monitor" };
}

async function init() {
  const [baseRes, actualRes] = await Promise.all([
    fetch("data/changwon_ev_dataset.csv"),
    fetch("data/actual_charging.csv")
  ]);
  const baseRows = parseCSV(await baseRes.text());
  const actualRows = parseCSV(await actualRes.text());

  const actualMap = new Map();
  actualRows.forEach(a => {
    actualMap.set(`${a.district}|${a.day_group}|${a.time_slot}`, a);
  });

  const pairs = [];
  baseRows.forEach(b => {
    const a = actualMap.get(`${b.district}|${b.day_group}|${b.time_slot}`);
    if (a && a.actual_kwh !== "" && a.occupancy_rate !== "") {
      pairs.push({
        district: b.district,
        day_group: b.day_group,
        time_slot: b.time_slot,
        demand_index: Number(b.demand_index),
        actual_kwh: Number(a.actual_kwh),
        occupancy_rate: Number(a.occupancy_rate)
      });
    }
  });

  const sumActual = pairs.reduce((s, r) => s + r.actual_kwh, 0);
  const sumIndex = pairs.reduce((s, r) => s + r.demand_index, 0) || 1;
  const scale = sumActual / sumIndex;

  mergedRows = pairs.map(r => {
    const scaledModel = r.demand_index * scale;
    const ape = r.actual_kwh > 0 ? Math.abs(r.actual_kwh - scaledModel) / r.actual_kwh * 100 : 0;
    const action = classifyAction(r.occupancy_rate);
    return { ...r, scaledModel, ape, action };
  });

  render();
}

function render() {
  const dist = document.getElementById("districtFilter").value;
  const act = document.getElementById("actionFilter").value;

  const rows = mergedRows.filter(r =>
    (dist === "ALL" || r.district === dist) &&
    (act === "ALL" || r.action.label === act)
  );

  const avgApe = rows.length ? (rows.reduce((s, r) => s + r.ape, 0) / rows.length).toFixed(1) : "-";
  const avgOcc = rows.length ? (rows.reduce((s, r) => s + r.occupancy_rate, 0) / rows.length * 100).toFixed(1) : "-";
  const expandCount = rows.filter(r => r.action.label === "증설·고출력 검토").length;

  document.getElementById("kpiBox").innerHTML = `
    <div class="kpi"><div class="kpi-label">검증 대상 구간</div><div class="kpi-val">${rows.length}건</div></div>
    <div class="kpi"><div class="kpi-label">평균 절대백분율오차(MAPE)</div><div class="kpi-val">${avgApe}%</div></div>
    <div class="kpi"><div class="kpi-label">평균 실측 점유율</div><div class="kpi-val">${avgOcc}%</div></div>
    <div class="kpi"><div class="kpi-label">증설·고출력 검토 구간</div><div class="kpi-val">${expandCount}건</div></div>
  `;

  document.getElementById("resultBody").innerHTML = rows.map(r => `
    <tr>
      <td>${r.district}</td>
      <td>${r.day_group}</td>
      <td>${r.time_slot}</td>
      <td>${r.scaledModel.toFixed(1)}</td>
      <td>${r.actual_kwh.toFixed(1)}</td>
      <td>${r.ape.toFixed(1)}%</td>
      <td>${(r.occupancy_rate * 100).toFixed(1)}%</td>
      <td><span class="tag ${r.action.cls}">${r.action.label}</span></td>
    </tr>
  `).join("");

  const labels = rows.map(r => `${r.district} ${r.time_slot}`);
  const ctx = document.getElementById("compareChart").getContext("2d");
  if (chart) chart.destroy();
  chart = new Chart(ctx, {
    type: "bar",
    data: {
      labels,
      datasets: [
        { label: "기준 환산치 (kWh)", data: rows.map(r => Number(r.scaledModel.toFixed(1))), backgroundColor: "#94a3b8" },
        { label: "실측 충전량 (kWh)", data: rows.map(r => r.actual_kwh), backgroundColor: "#2563eb" }
      ]
    },
    options: { responsive: true }
  });
}

init();
</script>
</body>
</html>

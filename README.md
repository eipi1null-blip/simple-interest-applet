const principalInput = document.querySelector("#principal");
const rateInput = document.querySelector("#rate");
const yearsInput = document.querySelector("#years");
const modeSelect = document.querySelector("#mode");
const resetButton = document.querySelector("#reset");

const principalValue = document.querySelector("#principalValue");
const rateValue = document.querySelector("#rateValue");
const yearsValue = document.querySelector("#yearsValue");
const modeValue = document.querySelector("#modeValue");
const formula = document.querySelector("#formula");
const interestLine = document.querySelector("#interestLine");
const balanceStack = document.querySelector("#balanceStack");
const baseAmount = document.querySelector("#baseAmount");
const interestStrip = document.querySelector("#interestStrip");
const rows = document.querySelector("#rows");
const canvas = document.querySelector("#chart");
const ctx = canvas.getContext("2d");

const moneyFormatter = new Intl.NumberFormat("en-US", {
  maximumFractionDigits: 0
});

function money(value) {
  return moneyFormatter.format(Math.round(value));
}

function getState() {
  const principal = Number(principalInput.value);
  const rate = Number(rateInput.value) / 100;
  const years = Number(yearsInput.value);
  const maxYears = Number(yearsInput.max);
  const yearlyInterest = principal * rate;
  return { principal, rate, years, maxYears, yearlyInterest };
}

function balanceAt({ principal, yearlyInterest }, year) {
  return principal + yearlyInterest * year;
}

function createCoin(className = "") {
  const coin = document.createElement("span");
  coin.className = `coin ${className}`.trim();
  return coin;
}

function renderStack(container, coinCount, className = "", animated = false) {
  container.innerHTML = "";
  for (let i = 0; i < coinCount; i += 1) {
    container.appendChild(createCoin(`${className} ${animated ? "new-coin" : ""}`));
  }
}

function renderScene(state) {
  const { principal, years, yearlyInterest } = state;
  const principalCoins = 8;
  const interestCoinsPerYear = Math.max(1, Math.round((yearlyInterest / principal) * 20));

  balanceStack.innerHTML = "";
  for (let i = 0; i < principalCoins; i += 1) {
    balanceStack.appendChild(createCoin("principal-coin"));
  }
  for (let year = 1; year <= years; year += 1) {
    for (let i = 0; i < interestCoinsPerYear; i += 1) {
      const animated = year === years;
      balanceStack.appendChild(createCoin(`interest-coin ${animated ? "new-coin" : ""}`));
    }
  }

  baseAmount.textContent = `A = ${money(balanceAt(state, years))}`;
  interestStrip.innerHTML = "";

  if (years === 0) {
    const card = document.createElement("article");
    card.className = "growth-card";
    card.innerHTML = `<strong>Starting amount</strong><span>Only the principal is shown: ${money(principal)}.</span>`;
    interestStrip.appendChild(card);
    return;
  }

  const visibleYears = Math.min(years, 6);
  for (let year = 1; year <= visibleYears; year += 1) {
    const card = document.createElement("article");
    card.className = "growth-card";
    card.innerHTML = `
      <strong>Year ${year}</strong>
      <span>add ${money(yearlyInterest)} interest</span>
      <span>balance: ${money(balanceAt(state, year))}</span>
    `;
    interestStrip.appendChild(card);
  }

  if (years > visibleYears) {
    const more = document.createElement("article");
    more.className = "growth-card";
    more.innerHTML = `<strong>More years</strong><span>${years - visibleYears} more equal interest layers are included in the stack.</span>`;
    interestStrip.appendChild(more);
  }
}

function renderTable(state) {
  rows.innerHTML = "";
  for (let year = 0; year <= state.years; year += 1) {
    const interest = year === 0 ? 0 : state.yearlyInterest;
    const totalInterest = state.yearlyInterest * year;
    const balance = balanceAt(state, year);
    const tr = document.createElement("tr");
    tr.innerHTML = `
      <td>${year}</td>
      <td>${year === 0 ? "-" : money(interest)}</td>
      <td>${money(totalInterest)}</td>
      <td>${money(balance)}</td>
    `;
    rows.appendChild(tr);
  }
}

function drawChart(state) {
  const rect = canvas.getBoundingClientRect();
  const dpr = window.devicePixelRatio || 1;
  canvas.width = Math.round(rect.width * dpr);
  canvas.height = Math.round(rect.height * dpr);
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);

  const width = rect.width;
  const height = rect.height;
  const pad = { left: 62, right: 28, top: 28, bottom: 48 };
  const plotW = width - pad.left - pad.right;
  const plotH = height - pad.top - pad.bottom;
  const maxBalance = balanceAt(state, state.maxYears);
  const maxY = Math.ceil(maxBalance / 500) * 500;
  const minY = 0;

  const xFor = (year) => pad.left + (year / state.maxYears) * plotW;
  const yFor = (amount) => pad.top + (1 - (amount - minY) / (maxY - minY)) * plotH;

  ctx.clearRect(0, 0, width, height);
  ctx.fillStyle = "#ffffff";
  ctx.fillRect(0, 0, width, height);

  ctx.strokeStyle = "#dfe7f1";
  ctx.lineWidth = 1;
  ctx.fillStyle = "#66758a";
  ctx.font = "12px system-ui, sans-serif";
  ctx.textAlign = "right";
  ctx.textBaseline = "middle";

  for (let i = 0; i <= 5; i += 1) {
    const value = minY + ((maxY - minY) / 5) * i;
    const y = yFor(value);
    ctx.beginPath();
    ctx.moveTo(pad.left, y);
    ctx.lineTo(width - pad.right, y);
    ctx.stroke();
    ctx.fillText(money(value), pad.left - 10, y);
  }

  ctx.strokeStyle = "#9aa8ba";
  ctx.lineWidth = 1.5;
  ctx.beginPath();
  ctx.moveTo(pad.left, pad.top);
  ctx.lineTo(pad.left, height - pad.bottom);
  ctx.lineTo(width - pad.right, height - pad.bottom);
  ctx.stroke();

  ctx.textAlign = "center";
  ctx.textBaseline = "top";
  for (let year = 0; year <= state.maxYears; year += 1) {
    if (state.maxYears > 8 && year % 2 === 1) continue;
    ctx.fillText(year, xFor(year), height - pad.bottom + 14);
  }

  const futurePoints = [];
  const points = [];
  for (let year = 0; year <= state.maxYears; year += 1) {
    const point = { x: year, y: balanceAt(state, year) };
    if (year <= state.years) points.push(point);
    if (year >= state.years) futurePoints.push(point);
  }

  drawDashedLine(futurePoints, xFor, yFor, "rgba(35, 118, 185, 0.22)", 3);
  drawLine(points, xFor, yFor, "#2376b9", 4);
  drawPoints(points, xFor, yFor, "#174f7c");

  if (modeSelect.value === "yearly") {
    drawInterestBars(state, xFor, yFor);
  }

  ctx.textAlign = "left";
  ctx.textBaseline = "middle";
  ctx.fillStyle = "#174f7c";
  ctx.font = "700 13px system-ui, sans-serif";
  ctx.fillText("Balance grows by a constant amount", pad.left + 10, pad.top + 14);
}

function drawInterestBars(state, xFor, yFor) {
  ctx.save();
  ctx.strokeStyle = "rgba(114, 184, 75, 0.65)";
  ctx.lineWidth = 2;
  for (let year = 1; year <= state.years; year += 1) {
    const x = xFor(year);
    const y1 = yFor(balanceAt(state, year - 1));
    const y2 = yFor(balanceAt(state, year));
    ctx.beginPath();
    ctx.moveTo(x + 7, y1);
    ctx.lineTo(x + 7, y2);
    ctx.stroke();
  }
  ctx.restore();
}

function drawLine(points, xFor, yFor, color, width) {
  if (points.length === 0) return;
  ctx.strokeStyle = color;
  ctx.lineWidth = width;
  ctx.lineJoin = "round";
  ctx.lineCap = "round";
  ctx.beginPath();
  points.forEach((point, index) => {
    const x = xFor(point.x);
    const y = yFor(point.y);
    if (index === 0) ctx.moveTo(x, y);
    else ctx.lineTo(x, y);
  });
  ctx.stroke();
}

function drawDashedLine(points, xFor, yFor, color, width) {
  ctx.save();
  ctx.setLineDash([8, 9]);
  drawLine(points, xFor, yFor, color, width);
  ctx.restore();
}

function drawPoints(points, xFor, yFor, color) {
  ctx.fillStyle = color;
  points.forEach((point) => {
    ctx.beginPath();
    ctx.arc(xFor(point.x), yFor(point.y), 5, 0, Math.PI * 2);
    ctx.fill();
  });
}

function render() {
  const state = getState();
  const ratePercent = state.rate * 100;
  principalValue.textContent = money(state.principal);
  rateValue.textContent = `${ratePercent.toFixed(ratePercent % 1 === 0 ? 0 : 1)}%`;
  yearsValue.textContent = state.years;
  modeValue.textContent = modeSelect.value;
  formula.textContent = `A = ${money(state.principal)}(1 + ${state.rate.toFixed(3)}t)`;
  interestLine.textContent = `Interest each year: ${money(state.yearlyInterest)}`;

  renderScene(state);
  renderTable(state);
  drawChart(state);
}

[principalInput, rateInput, yearsInput, modeSelect].forEach((control) => {
  control.addEventListener("input", render);
});

resetButton.addEventListener("click", () => {
  principalInput.value = 4000;
  rateInput.value = 4;
  yearsInput.value = 5;
  modeSelect.value = "yearly";
  render();
});

window.addEventListener("resize", render);
render();

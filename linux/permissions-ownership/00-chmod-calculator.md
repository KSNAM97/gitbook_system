# Chmod 계산기

Obsidian dataviewjs 기반 인터랙티브 chmod 권한 계산기 위젯 원본. GitBook에서는 스크립트가 실행되지 않으므로 로직 참고용으로만 사용한다.

```dataviewjs
// ===== 상태 =====
const state = {
  owner: { r: true,  w: true,  x: true  }, // 7
  group: { r: true,  w: false, x: true  }, // 5
  other: { r: true,  w: false, x: true  }, // 5
};

const labels = {
  owner: "소유자 (Owner)",
  group: "그룹 (Group)",
  other: "기타 (Public)",
};

const perms = [
  { key: "r", name: "읽기(Read)",  val: 4, color: "#10b981" },
  { key: "w", name: "쓰기(Write)", val: 2, color: "#f59e0b" },
  { key: "x", name: "실행(Exec)",  val: 1, color: "#3b82f6" },
];

// ===== 계산 함수 =====
const calcOctal = (g) => (g.r?4:0) + (g.w?2:0) + (g.x?1:0);
const calcSymbolic = (g) => (g.r?"r":"-") + (g.w?"w":"-") + (g.x?"x":"-");

// ===== 스타일 =====
const style = document.createElement("style");
style.textContent = `
  .chmod-wrap { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; font-family: system-ui; }
  .chmod-card { border: 1px solid #e5e7eb; border-radius: 12px; padding: 16px; background: var(--background-primary); }
  .chmod-card h3 { margin: 0 0 12px; display: flex; justify-content: space-between; align-items: center; font-size: 15px; }
  .chmod-octal { font-size: 22px; font-weight: 700; color: #a855f7; }
  .chmod-perms { display: flex; gap: 8px; }
  .chmod-btn {
    flex: 1; padding: 10px; border: 2px solid #e5e7eb; border-radius: 8px;
    background: transparent; cursor: pointer; text-align: center;
    font-size: 12px; transition: all .15s;
  }
  .chmod-btn.active { border-width: 2px; background: rgba(0,0,0,0.02); }
  .chmod-btn small { display: block; opacity: .6; margin-top: 2px; }
  .chmod-result {
    grid-column: 2; grid-row: 1 / span 3;
    border: 2px solid #d946ef; border-radius: 12px; padding: 20px;
    background: linear-gradient(180deg, #fdf4ff, #fae8ff);
    text-align: center; align-self: start;
  }
  .chmod-result .big { font-size: 64px; font-weight: 800; color: #6b21a8; margin: 8px 0; }
  .chmod-result input {
    font-size: 40px; text-align: center; width: 140px; border: none;
    background: transparent; color: #6b21a8; font-weight: 800;
  }
  .chmod-symbolic { font-family: monospace; font-size: 28px; padding: 12px; 
    border: 1px solid #e5e7eb; border-radius: 8px; text-align: center; margin-top: 12px; }
  .chmod-cmd { background: #0f172a; color: #e2e8f0; padding: 12px; border-radius: 8px;
    font-family: monospace; margin-top: 12px; font-size: 13px; }
`;
dv.container.appendChild(style);

// ===== 렌더 =====
const wrap = dv.container.createEl("div", { cls: "chmod-wrap" });

function render() {
  wrap.empty();

  // 좌측 3개 카드
  for (const groupKey of ["owner", "group", "other"]) {
    const g = state[groupKey];
    const card = wrap.createEl("div", { cls: "chmod-card" });
    const h = card.createEl("h3");
    h.createSpan({ text: labels[groupKey] });
    h.createEl("span", { text: calcOctal(g), cls: "chmod-octal" });

    const row = card.createEl("div", { cls: "chmod-perms" });
    for (const p of perms) {
      const btn = row.createEl("button", { cls: "chmod-btn" });
      const active = g[p.key];
      if (active) {
        btn.classList.add("active");
        btn.style.borderColor = p.color;
        btn.style.color = p.color;
      }
      btn.innerHTML = `${p.name}<small>(${p.val})</small>`;
      btn.onclick = () => { g[p.key] = !g[p.key]; render(); };
    }
  }

  // 우측 결과 패널
  const result = wrap.createEl("div", { cls: "chmod-result" });
  result.createEl("div", { text: "리눅스 8진수 값", 
    attr: { style: "color:#a21caf;font-weight:600;" } });

  const octal = `${calcOctal(state.owner)}${calcOctal(state.group)}${calcOctal(state.other)}`;
  const input = result.createEl("input", { attr: { value: octal, maxlength: 3 } });
  input.addEventListener("input", (e) => {
    const v = e.target.value.replace(/[^0-7]/g, "").padEnd(3, "0").slice(0, 3);
    const digits = v.split("").map(Number);
    ["owner","group","other"].forEach((k, i) => {
      state[k].r = !!(digits[i] & 4);
      state[k].w = !!(digits[i] & 2);
      state[k].x = !!(digits[i] & 1);
    });
    render();
  });

  result.createEl("div", { text: "* 숫자를 직접 입력하여 역계산할 수 있습니다.",
    attr: { style: "font-size:11px;opacity:.6;margin-top:8px;" } });

  const sym = calcSymbolic(state.owner) + calcSymbolic(state.group) + calcSymbolic(state.other);
  const symBox = wrap.createEl("div", { cls: "chmod-card" });
  symBox.createEl("div", { text: "기호 표기법 (Symbolic Notation)",
    attr: { style: "font-size:13px;margin-bottom:8px;" } });
  symBox.createEl("div", { text: sym, cls: "chmod-symbolic" });

  const cmdBox = wrap.createEl("div", { cls: "chmod-card" });
  cmdBox.createEl("div", { text: "터미널 명령어",
    attr: { style: "font-size:13px;margin-bottom:8px;" } });
  cmdBox.createEl("div", { text: `$ chmod ${octal} file.txt`, cls: "chmod-cmd" });
}

render();
```

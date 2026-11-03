# Скрипт графика для `use_figma`

Рисует редактируемый график нативными векторами и текстом Navigo на холсте `canvas`
(макет `m-canvas`, подготовлен скриптом E из `slides-figma-build`). Цвета привязаны к переменным
`Manila Slides`; в файле без них скрипт подставит те же HEX. Один вызов — один график. Менять только блок `// INPUT`.

Типы: `column` (столбцы), `bar` (горизонтальные полосы, для длинных подписей и рейтингов),
`line` (динамика, до 3 серий), `donut` (доля целого, до 5 сегментов), `combo` (доля столбцами +
1–2 линии абсолютных значений — для истории «доля растёт при растущем потоке»).

Для `combo`: `SERIES[0]` — доля в процентах (высота столбца от 0 до 100 %, крупная подпись внутри),
`SERIES[1..2]` — абсолютные значения линиями в своём масштабе; оси Y нет, значения подписаны прямо
на графике, поэтому правило «не больше одной оси» не нарушается. У серии можно задать
`dashed: true` (для «всего» — серой пунктирной линией). `POINT_LABELS` — подписи над точками
(«80 из 182 задач»).

```js
// INPUT
const CANVAS_ID = "27:5186";
const TYPE = "column";                 // column | bar | line | donut
const LABELS = ["2022", "2023", "2024", "2025"];
const SERIES = [                       // для column/bar/donut — ровно одна серия
  { name: "Выручка", values: [1.0, 1.5, 1.9, 2.4] }
];
const HIGHLIGHT = [3];                 // индексы точек, выделенных жёлтым акцентом; [] — без акцента
const UNIT = " млрд ₽";                // добавляется к подписям значений
const DECIMALS = 1;
const SOURCE = "Источник: управленческая отчётность";  // "" — без подписи
const CENTER_LABEL = "";               // только donut: крупный текст в центре, например "62%"
const POINT_LABELS = [];               // только combo: подписи над точками по категориям
// END INPUT

const hex = h => ({ r: parseInt(h.slice(1, 3), 16) / 255, g: parseInt(h.slice(3, 5), 16) / 255, b: parseInt(h.slice(5, 7), 16) / 255 });
const VARS = Object.fromEntries((await figma.variables.getLocalVariablesAsync('COLOR')).map(v => [v.name, v]));
const tok = (name, fb) => { const p = { type: 'SOLID', color: hex(fb) }; return VARS[name] ? figma.variables.setBoundVariableForPaint(p, 'color', VARS[name]) : p; };
const C = { text: tok('text/primary', '#2b2d33'), muted: tok('text/secondary', '#535661'), grid: tok('line/grid', '#a5d6d6'), accent: tok('chart/accent', '#fee600'),
  white: tok('surface/card', '#f9f9fa'), track: tok('surface/card-soft', '#e1f2f2'), soft: tok('chart/muted', '#a5d6d6'),
  series: [tok('chart/primary', '#36a0a6'), tok('chart/secondary', '#8fcccc'), tok('chart/dark', '#336363'), tok('chart/muted', '#a5d6d6'), tok('text/muted', '#a3a7b5')] };
const F = { family: 'Navigo', style: 'Regular' };
await figma.loadFontAsync(F);
const canvas = await figma.getNodeByIdAsync(CANVAS_ID);
for (const k of [...canvas.children]) k.remove();
const W = canvas.width, H = canvas.height;
const fmt = v => v.toFixed(DECIMALS).replace('.', ',');
function rect(x, y, w, h, color, radii) {
  const n = figma.createRectangle(); canvas.appendChild(n); n.resize(Math.max(1, w), Math.max(1, h));
  n.fills = [color];
  if (radii) { n.topLeftRadius = radii[0]; n.topRightRadius = radii[1]; n.bottomRightRadius = radii[2]; n.bottomLeftRadius = radii[3]; }
  n.x = x; n.y = y; return n;
}
function text(s, size, color, x, y, w, align) {
  const t = figma.createText(); canvas.appendChild(t); t.fontName = F; t.fontSize = size; t.characters = s;
  t.fills = [color];
  if (w) { t.textAutoResize = 'HEIGHT'; t.resize(w, t.height); }
  if (align) t.textAlignHorizontal = align;
  t.x = x; t.y = y; return t;
}
const colorAt = i => HIGHLIGHT.includes(i) ? C.accent : C.series[0];
const footer = SOURCE ? 60 : 0;
if (SOURCE) text(SOURCE, 24, C.muted, 0, H - 36, 1200);

if (TYPE === 'column' || TYPE === 'line') {
  const all = SERIES.flatMap(s => s.values);
  const rawMax = Math.max(...all), minV = Math.min(0, ...all);
  const rough = (rawMax - minV) / 4, mag = Math.pow(10, Math.floor(Math.log10(rough || 1)));
  const step = [1, 2, 2.5, 5, 10].map(k => k * mag).find(s => s >= rough);
  const max = Math.ceil(rawMax * 1.08 / step) * step, ticks = Math.round((max - minV) / step);
  const tickFmt = v => (Number.isInteger(step) ? String(Math.round(v)) : v.toFixed(step < 1 ? 1 : 0)).replace('.', ',');
  const padL = 110, padT = 70, padB = 90 + footer, plotW = W - padL - 20, plotH = H - padT - padB;
  const yOf = v => padT + plotH - plotH * (v - minV) / (max - minV);
  for (let i = 0; i <= ticks; i++) { const v = minV + step * i; rect(padL, yOf(v), plotW, 2, C.grid); text(tickFmt(v), 24, C.muted, 0, yOf(v) - 16, padL - 20, 'RIGHT'); }
  const slotW = plotW / LABELS.length;
  LABELS.forEach((lab, i) => text(lab, 28, C.muted, padL + slotW * i, padT + plotH + 20, slotW, 'CENTER'));
  if (TYPE === 'column') {
    const vals = SERIES[0].values, bw = Math.min(200, slotW * 0.55);
    vals.forEach((v, i) => {
      const x = padL + slotW * i + (slotW - bw) / 2, y = yOf(v);
      rect(x, y, bw, yOf(minV) - y, colorAt(i), [16, 16, 0, 0]);
      text(fmt(v) + UNIT, 30, C.text, x - 80, y - 50, bw + 160, 'CENTER');
    });
  } else {
    SERIES.forEach((s, si) => {
      const pts = s.values.map((v, i) => [padL + slotW * i + slotW / 2, yOf(v)]);
      const vec = figma.createVector(); canvas.appendChild(vec);
      vec.vectorPaths = [{ windingRule: 'NONE', data: pts.map((p, i) => (i ? 'L ' : 'M ') + p[0].toFixed(1) + ' ' + p[1].toFixed(1)).join(' ') }];
      vec.strokes = [si === 0 && HIGHLIGHT.length === 0 ? C.series[0] : C.series[si]];
      vec.strokeWeight = 6; vec.strokeJoin = 'ROUND'; vec.strokeCap = 'ROUND'; vec.fills = [];
      pts.forEach(([x, y], i) => {
        const hl = si === 0 && HIGHLIGHT.includes(i);
        const d = figma.createEllipse(); canvas.appendChild(d); const r = hl ? 16 : 10; d.resize(r * 2, r * 2);
        d.fills = [hl ? C.accent : C.series[si]]; d.strokes = [C.white]; d.strokeWeight = 4;
        d.x = x - r; d.y = y - r;
        if (SERIES.length === 1 || hl) text(fmt(s.values[i]) + UNIT, 28, C.text, x - 120, y - 60, 240, 'CENTER');
      });
    });
    if (SERIES.length > 1) { let lx = padL; SERIES.forEach((s, si) => { rect(lx, 10, 28, 28, C.series[si], [8, 8, 8, 8]); const t = text(s.name, 28, C.text, lx + 44, 6); lx += 44 + t.width + 56; }); }
  }
}

if (TYPE === 'bar') {
  const vals = SERIES[0].values, max = Math.max(...vals) * 1.15;
  const labW = 520, padT = 20, rowH = (H - padT - footer - 20) / vals.length, bh = Math.min(64, rowH * 0.6), plotW = W - labW - 260;
  vals.forEach((v, i) => {
    const y = padT + rowH * i + (rowH - bh) / 2;
    text(LABELS[i], 30, C.text, 0, y + bh / 2 - 20, labW - 30, 'RIGHT');
    const w = plotW * v / max;
    rect(labW, y, w, bh, colorAt(i), [0, 12, 12, 0]);
    text(fmt(v) + UNIT, 30, C.text, labW + w + 20, y + bh / 2 - 20, 240);
  });
}

if (TYPE === 'donut') {
  const vals = SERIES[0].values, sum = vals.reduce((a, b) => a + b, 0);
  const D = Math.min(H - footer - 40, 760), cx = 40, cy = (H - footer - D) / 2;
  let a0 = -Math.PI / 2;
  vals.forEach((v, i) => {
    const a1 = a0 + 2 * Math.PI * v / sum;
    const e = figma.createEllipse(); canvas.appendChild(e); e.resize(D, D);
    e.arcData = { startingAngle: a0, endingAngle: a1 - 0.012, innerRadius: 0.62 };
    e.fills = [HIGHLIGHT.includes(i) ? C.accent : C.series[i % C.series.length]];
    e.x = cx; e.y = cy; a0 = a1;
  });
  if (CENTER_LABEL) text(CENTER_LABEL, 114, C.text, cx, cy + D / 2 - 70, D, 'CENTER');
  const lx = cx + D + 120, rowH = 110, ly = (H - footer - rowH * vals.length) / 2;
  vals.forEach((v, i) => {
    const y = ly + rowH * i;
    rect(lx, y + 8, 36, 36, HIGHLIGHT.includes(i) ? C.accent : C.series[i % C.series.length], [10, 10, 10, 10]);
    text(LABELS[i], 32, C.text, lx + 60, y, 700);
    text(fmt(100 * v / sum) + '%' + (UNIT ? ' · ' + fmt(v) + UNIT : ''), 24, C.muted, lx + 60, y + 46, 700);
  });
}

if (TYPE === 'combo') {
  const share = SERIES[0].values, lines = SERIES.slice(1), n = LABELS.length;
  const legendH = 56, padT = 60, padB = 60 + legendH + footer, plotH = H - padT - padB;
  const slotW = W / n, bw = slotW - 48;
  const lineColors = [C.muted, C.series[2]];
  rect(0, padT + plotH, W, 2, C.grid);
  share.forEach((v, i) => {
    const x = slotW * i + 24, bh = plotH * v / 100;
    rect(x, padT, bw, plotH, C.track, [16, 16, 0, 0]);
    rect(x, padT + plotH - bh, bw, bh, HIGHLIGHT.includes(i) ? C.accent : C.soft, [16, 16, 0, 0]);
    text(fmt(v) + UNIT, 72, C.text, x, padT + plotH - 110, bw, 'CENTER');
    text(LABELS[i], 24, C.muted, x, padT + plotH + 14, bw, 'CENTER');
  });
  const lmax = Math.max(...lines.flatMap(s => s.values)) * 1.15;
  const yOf = v => padT + plotH - plotH * v / lmax;
  const tops = Array(n).fill(Infinity);
  for (let si = 0; si < lines.length; si++) {
    const s = lines[si], col = lineColors[si % lineColors.length];
    const pts = s.values.map((v, i) => [slotW * i + slotW / 2, yOf(v)]);
    const vec = figma.createVector(); canvas.appendChild(vec);
    vec.vectorPaths = [{ windingRule: 'NONE', data: pts.map((p, i) => (i ? 'L ' : 'M ') + p[0].toFixed(1) + ' ' + p[1].toFixed(1)).join(' ') }];
    vec.strokes = [col]; vec.strokeWeight = 5; vec.strokeJoin = 'ROUND'; vec.strokeCap = 'ROUND'; vec.fills = [];
    if (s.dashed) vec.dashPattern = [16, 12];
    pts.forEach(([x, y], i) => {
      const d = figma.createEllipse(); canvas.appendChild(d); d.resize(28, 28);
      d.fills = [col]; d.strokes = [C.white]; d.strokeWeight = 4;
      d.x = x - 14; d.y = y - 14; tops[i] = Math.min(tops[i], y);
    });
  }
  POINT_LABELS.forEach((s, i) => { if (s) text(s, 28, C.text, slotW * i, Math.max(padT + 8, tops[i] - 56), slotW, 'CENTER'); });
  let lx = 0; const ly = H - footer - legendH + 16;
  [{ name: SERIES[0].name, color: C.soft, box: true }, ...lines.map((s, si) => ({ name: s.name, color: lineColors[si % lineColors.length] }))].forEach(it => {
    if (it.box) rect(lx, ly, 40, 24, it.color, [6, 6, 6, 6]); else rect(lx, ly + 10, 40, 5, it.color);
    const t = text(it.name, 24, C.text, lx + 52, ly - 2); lx += 52 + t.width + 56;
  });
}

const kids = canvas.children;
const outside = kids.filter(n => n.x < -2 || n.y < -2 || n.x + n.width > W + 2 || n.y + n.height > H + 2).map(n => n.name);
return { canvasId: canvas.id, nodes: kids.length, outside, createdNodeIds: kids.map(n => n.id) };
```

`outside` должен быть пустым. Если нет — укоротить подписи (`LABELS`, `UNIT`) или уменьшить
число точек, затем перезапустить: скрипт сам очищает холст перед рисованием.

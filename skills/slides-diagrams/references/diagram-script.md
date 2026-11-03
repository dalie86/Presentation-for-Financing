# Скрипт схемы для `use_figma`

Рисует редактируемую схему нативными фигурами на холсте `canvas` (макет `m-canvas`, скрипт E из `slides-figma-build`).
Использовать только когда ни один макет раздела «Схемы», «Списки» или «Шаги» не подходит.
В Figma Slides нет коннекторов, поэтому стрелки — отдельные линии; при правке схемы вручную их
двигают вместе с карточками.

Типы:
- `flow` — цепочка из 2–6 этапов слева направо со стрелками (процесс, «было → сделали → работает»).
  У узла может быть `callout` — жёлтая плашка с цитатой или правилом внутри карточки. `LOOP` рисует
  обратную стрелку под карточками для истории «ходим по кругу»;
- `matrix` — матрица 2×2 с подписями осей (приоритизация, позиционирование);
- `hub` — центральный элемент и 3–6 связанных с ним блоков (экосистема, участники);
- `system-numbers` — сверху 2–4 карточки механизма, снизу 2–3 крупных числа. Для темы, где спикер
  рассказывает и как устроено, и чего достигли (так авторы слили два слайда Work OS в один).

```js
// INPUT
const CANVAS_ID = "27:5300";
const TYPE = "flow";                     // flow | matrix | hub | system-numbers
const NODES = [                          // flow: этапы по порядку; matrix: 4 квадранта (ЛВ, ПВ, ЛН, ПН); hub: [центр, ...блоки]; system-numbers: карточки механизма
  { title: "Ревью человеком", text: "Тесты проверяли номер, но не весь документ", callout: "Сравнивай с ожидаемым результатом целиком" },
  { title: "Новое правило", text: "Исправили то, что агент читает в каждой задаче", callout: "Подготовь весь результат заранее" },
  { title: "Правило работает", text: "Новая задача: агент сам проверил результат" }
];
const ACCENT = [2];                      // индексы узлов с жёлтым акцентом
const LOOP = null;                       // только flow: { from: 4, to: 1, label: "И снова по кругу" } — индексы узлов
const NUMBERS = [];                      // только system-numbers: [{ value: "22", text: "инициативы сотрудников" }, ...]
const AXES = { x: "Сложность →", y: "Ценность →" };  // только matrix
const SOURCE = "";
// END INPUT

const hex = h => ({ r: parseInt(h.slice(1, 3), 16) / 255, g: parseInt(h.slice(3, 5), 16) / 255, b: parseInt(h.slice(5, 7), 16) / 255 });
const VARS = Object.fromEntries((await figma.variables.getLocalVariablesAsync('COLOR')).map(v => [v.name, v]));
const tok = (name, fb) => { const p = { type: 'SOLID', color: hex(fb) }; return VARS[name] ? figma.variables.setBoundVariableForPaint(p, 'color', VARS[name]) : p; };
const C = { text: tok('text/primary', '#2b2d33'), muted: tok('text/secondary', '#535661'), card: tok('surface/card', '#f9f9fa'), white: tok('surface/card', '#f9f9fa'), accent: tok('surface/accent', '#fee600'), line: tok('line/connector', '#36a0a6') };
const F = { family: 'Navigo', style: 'Regular' };
await figma.loadFontAsync(F);
const canvas = await figma.getNodeByIdAsync(CANVAS_ID);
for (const k of [...canvas.children]) k.remove();
const W = canvas.width, H = canvas.height - (SOURCE ? 60 : 0);
function txt(parent, s, size, color) {
  const t = figma.createText(); parent.appendChild(t); t.fontName = F; t.fontSize = size; t.characters = s;
  t.fills = [color]; t.layoutSizingHorizontal = 'FILL'; t.textAutoResize = 'HEIGHT'; return t;
}
function card(x, y, w, h, node, i, ts) {
  const acc = ACCENT.includes(i);
  const f = figma.createAutoLayout('VERTICAL', { name: 'node ' + (i + 1), itemSpacing: 16 });
  canvas.appendChild(f);
  f.paddingTop = f.paddingBottom = 40; f.paddingLeft = f.paddingRight = 36;
  f.cornerRadius = 32; f.fills = [acc ? C.accent : C.card];
  f.primaryAxisSizingMode = 'FIXED'; f.counterAxisSizingMode = 'FIXED'; f.resize(w, h);
  if (node.title) txt(f, node.title, ts || 44, C.text);
  if (node.text) txt(f, node.text, 30, C.text);
  if (node.callout) {
    const c = figma.createAutoLayout('VERTICAL', { name: 'callout' });
    f.appendChild(c); c.layoutSizingHorizontal = 'FILL';
    c.paddingTop = c.paddingBottom = 20; c.paddingLeft = c.paddingRight = 24; c.cornerRadius = 16;
    c.fills = [acc ? C.white : C.accent];
    txt(c, node.callout, 28, C.text);
  }
  f.x = x; f.y = y; return f;
}
async function path(pts) {
  const v = figma.createVector(); canvas.appendChild(v);
  const ox = Math.min(...pts.map(p => p[0])), oy = Math.min(...pts.map(p => p[1]));
  await v.setVectorNetworkAsync({
    vertices: pts.map((p, i) => i === pts.length - 1 ? { x: p[0] - ox, y: p[1] - oy, strokeCap: 'ARROW_LINES' } : { x: p[0] - ox, y: p[1] - oy }),
    segments: pts.slice(1).map((_, i) => ({ start: i, end: i + 1 })), regions: []
  });
  v.strokes = [C.line]; v.strokeWeight = 4; v.fills = []; v.name = 'arrow';
  v.x = ox; v.y = oy; return v;
}
function label(s, size, color, x, y, w, align) {
  const t = figma.createText(); canvas.appendChild(t); t.fontName = F; t.fontSize = size; t.characters = s;
  t.fills = [color]; if (w) { t.textAutoResize = 'HEIGHT'; t.resize(w, t.height); }
  if (align) t.textAlignHorizontal = align; t.x = x; t.y = y; return t;
}
if (SOURCE) label(SOURCE, 24, C.muted, 0, canvas.height - 36, 1200);

if (TYPE === 'flow') {
  const n = NODES.length, gap = n >= 5 ? 64 : 96, ts = n >= 5 ? 38 : 44, w = (W - gap * (n - 1)) / n;
  const h = LOOP ? 420 : Math.min(n >= 5 ? 360 : 460, H - 80), y = LOOP ? 120 : (H - h) / 2;   // проверено: 4 узла, плашка и возврат
  for (let i = 0; i < n; i++) {
    const x = i * (w + gap); card(x, y, w, h, NODES[i], i, ts);
    if (i < n - 1) await path([[x + w + 12, y + h / 2], [x + w + gap - 12, y + h / 2]]);
  }
  if (LOOP) {
    const cx = i => i * (w + gap) + w / 2, yb = y + h + 16, yl = y + h + 110;
    await path([[cx(LOOP.from), yb], [cx(LOOP.from), yl], [cx(LOOP.to), yl], [cx(LOOP.to), yb]]);
    if (LOOP.label) { const l = Math.min(cx(LOOP.from), cx(LOOP.to)), r = Math.max(cx(LOOP.from), cx(LOOP.to)); label(LOOP.label, 30, C.text, l, yl + 24, r - l, 'CENTER'); }
  }
}
if (TYPE === 'matrix') {
  const padL = 90, padB = 80, gap = 24, cw = (W - padL - gap) / 2, ch = (H - padB - gap) / 2;
  NODES.slice(0, 4).forEach((node, i) => card(padL + (i % 2) * (cw + gap), (i < 2 ? 0 : ch + gap), cw, ch, node, i));
  label(AXES.x, 28, C.muted, padL, H - 50, W - padL, 'CENTER');
  const yl = label(AXES.y, 28, C.muted, 0, 0, H - padB, 'CENTER'); yl.rotation = 90; yl.x = 10; yl.y = H - padB;
}
if (TYPE === 'hub') {
  const [center, ...rest] = NODES, cw = 460, chh = 240, cx = (W - cw) / 2, cy = (H - chh) / 2;
  const n = rest.length, bw = 420, bh = 200, rx = (W - bw) / 2 - 20, ry = (H - bh) / 2 - 10;
  for (let i = 0; i < n; i++) {
    const node = rest[i], a = -Math.PI / 2 + 2 * Math.PI * i / n;
    const ca = Math.cos(a), sa = Math.sin(a);
    const bcx = W / 2 + rx * ca, bcy = H / 2 + ry * sa;
    const sx = W / 2 + ca * (cw / 2 + 20), sy = H / 2 + sa * (chh / 2 + 20);
    const ex = bcx - ca * (bw / 2 + 20), ey = bcy - sa * (bh / 2 + 20);
    if ((ex - sx) * ca + (ey - sy) * sa > 20) await path([[sx, sy], [ex, ey]]);
    card(bcx - bw / 2, bcy - bh / 2, bw, bh, node, i + 1);
  }
  card(cx, cy, cw, chh, center, 0);
}
if (TYPE === 'system-numbers') {
  const n = NODES.length, gap = 24, w = (W - gap * (n - 1)) / n, h = 330;   // проверено: 4 карточки, заголовок в 2 строки + текст в 3
  NODES.forEach((node, i) => card(i * (w + gap), 0, w, h, node, i, 36));
  const m = NUMBERS.length, nw = (W - 48 * (m - 1)) / m, ny = h + 80;
  NUMBERS.forEach((num, i) => {
    const x = i * (nw + 48);
    const v = label(num.value, 200, C.text, x, ny); v.name = 'number ' + (i + 1);
    label(num.text, 32, C.text, x, ny + v.height + 8, Math.min(nw, 560));
  });
}
const kids = canvas.children;
const outside = kids.filter(k => k.x < -2 || k.y < -2 || k.x + k.width > W + 2 || k.y + k.height > canvas.height + 2).map(k => k.name);
const overflow = kids.filter(k => k.type === 'FRAME' && k.name.startsWith('node')).filter(f => f.children.reduce((a, c) => a + c.height, 0) + 16 * (f.children.length - 1) + 80 > f.height + 1).map(f => f.name);
return { canvasId: canvas.id, nodes: kids.length, outside, overflow, createdNodeIds: kids.map(k => k.id) };
```

После вызова обязательно сделать скриншот слайда: у схем чаще всего ломается перенос текста
в карточках. `overflow` — карточки, где текст не влез по высоте. Если текст не влез — сократить
`title` до 1–3 слов, `text` до 60 символов, `callout` до 50 символов.

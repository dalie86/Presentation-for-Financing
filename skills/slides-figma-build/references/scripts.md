# Скрипты сборки для `use_figma`

Скрипты проверены на шаблоне «Манила 30». Копировать дословно, менять только блок `// INPUT`.
Каждый вызов `use_figma` передаёт `fileKey`, `sessionId` и `skillNames: "figma-use,figma-use-slides"`
(или с префиксом `resource:`, если навыки Figma загружены как MCP-ресурсы).

Порядок для каждой партии слайдов: **A → B → C**. Клонирование и инвентарь не объединять в один
вызов: у только что склонированного слайда вложенные слои инстансов ещё недоступны.

---

## A. Создать раздел и склонировать макеты

```js
// INPUT
const SECTION_NAME = "Название презентации — черновик";
const ROW_ID = null;               // id существующего раздела, чтобы дописать в него; иначе null
const LAYOUT_IDS = ["7:18", "7:1211", "10:3208"]; // id макетов из layouts.md в порядке колоды, повторы разрешены
// END INPUT

let row = ROW_ID ? await figma.getNodeByIdAsync(ROW_ID) : null;
if (!row) { row = figma.createSlideRow(); row.name = SECTION_NAME; }
const sources = await Promise.all(LAYOUT_IDS.map(id => figma.getNodeByIdAsync(id)));
const missing = LAYOUT_IDS.filter((id, i) => !sources[i] || sources[i].type !== 'SLIDE');
if (missing.length) return { error: "layout not found", missing };
const created = [];
for (const src of sources) { const s = src.clone(); row.appendChild(s); created.push(s.id); }
return { rowId: row.id, rowName: row.name, createdSlideIds: created };
```

Не использовать `figma.getSlideGrid` / `figma.setSlideGrid`: после удаления слайдов в той же
сессии `getSlideGrid` падает, а `setSlideGrid` требует передать все слайды файла. Раздел и его
слайды читать через дерево (`SLIDE_GRID.children` → `SLIDE_ROW.children`). Имена слайдов задать
нельзя — Figma Slides нумерует их сама, поэтому слайды искать по id. Найти раздел по имени:
`figma.currentPage.children.find(n => n.type === 'SLIDE_GRID').children.find(r => r.name === '…')`.

---

## B. Инвентарь слайда

Возвращает для каждого слайда:
- `header` — вариант библиотечной шапки и её тексты (заголовок, описание);
- `groups` — контейнеры auto layout с повторяющимися элементами (карточки, строки, шаги) и сколько
  в них элементов; по id контейнера скрипт C меняет их число;
- `items` — инстансы компонентов шаблона и их текущие свойства (тексты, варианты, булевы);
- `slots` — свободные тексты вне компонентов (крупные заголовки титулов, пояснения), размеченные
  `slot:NN` сверху вниз и слева направо, с кеглем, шириной и оценкой вместимости.

Ответ ограничен 20 КБ: для 8+ слайдов делить на несколько вызовов.

```js
// INPUT
const SLIDE_IDS = ["27:5082", "27:5117"];
// END INPUT

const SKIP = n => n.name === 'Header' || n.name === 'Label' || n.name === 'logo' || n.name.startsWith('Имиджи');
const SEP = ['Divider', 'Arrow', 'Arrow slot', 'Bullet'];
const pad = n => String(n).padStart(2, '0');
const out = {};
for (const id of SLIDE_IDS) {
  const slide = await figma.getNodeByIdAsync(id);
  if (!slide) { out[id] = { error: 'not found' }; continue; }
  const sb = slide.absoluteBoundingBox;
  const h = slide.findOne(n => n.type === 'INSTANCE' && n.name === 'Header');
  const header = h ? { variant: (await h.getMainComponentAsync()).name, texts: h.findAll(n => n.type === 'TEXT').map(t => t.characters) } : null;
  const items = [], groups = {}, loose = [];
  const walk = node => { for (const c of node.children || []) {
    if (c.type === 'INSTANCE') {
      if (SKIP(c)) continue;
      const props = {};
      for (const [k, v] of Object.entries(c.componentProperties)) props[k.split('#')[0]] = v.value;
      if (!SEP.includes(c.name)) items.push({ id: c.id, comp: c.name, in: c.parent.type === 'SLIDE' ? null : c.parent.id, props });
    } else if (c.type === 'TEXT') { if (c.visible) loose.push(c); }
    else if ('children' in c) walk(c);
    if (c.parent.type !== 'SLIDE' && c.parent.layoutMode && c.parent.layoutMode !== 'NONE' && !SEP.includes(c.name) && !SKIP(c)) {
      const g = groups[c.parent.id] || (groups[c.parent.id] = { name: c.parent.name, kinds: {} });
      g.kinds[c.name] = (g.kinds[c.name] || 0) + 1;
    }
  } };
  walk(slide);
  for (const [gid, g] of Object.entries(groups)) if (Object.values(g.kinds).every(n => n < 2)) delete groups[gid];
  const tagged = loose.length && loose.every(t => /^slot:\d+$/.test(t.name));
  let slots = loose;
  if (!tagged) {
    slots = loose.map(t => ({ t, b: t.absoluteBoundingBox }))
      .sort((p, q) => (Math.round((p.b.y - sb.y) / 24) - Math.round((q.b.y - sb.y) / 24)) || (p.b.x - q.b.x)).map(o => o.t);
    slots.forEach((t, i) => { t.name = 'slot:' + pad(i + 1); });
  } else slots.sort((a, b) => a.name.localeCompare(b.name));
  out[id] = { header, groups, items, slots: slots.map(t => {
    const fs = t.fontSize === figma.mixed ? 32 : t.fontSize;
    const lh = t.lineHeight !== figma.mixed && t.lineHeight.unit === 'PIXELS' ? t.lineHeight.value : fs * 1.25;
    const lines = Math.max(1, Math.round(t.height / lh)), cpl = Math.floor(t.width / (fs * 0.55));
    return { slot: Number(t.name.slice(5)), text: t.characters.replace(/\s+/g, ' ').slice(0, 60), fs, w: Math.round(t.width), lines, maxChars: cpl * Math.max(lines, 1) };
  }) };
}
return out;
```

Вместимость свойств компонентов — в `layouts.md` (столбец «Что заполнять»).

---

## C. Заполнить слайд

На слайд — объект с четырьмя необязательными частями. Порядок применения: `count` → `items` →
`header` → `slots`.

- `count` — `{ "<id контейнера из groups>": N }`: оставить N элементов. Лишние удаляются с конца
  (вместе с разделителем `Divider` / стрелкой перед ними), недостающие дублируются с последнего.
- `items` — `{ "<id инстанса>" | "<id контейнера>#<номер с 0>": { "<свойство>": значение } }`.
  Свойства — по-русски, как в инвентаре: «Заголовок», «Описание», «Утверждение», «Подпись»,
  «Число», «Текст», «Номер»; варианты — `Style`, `State`, `Type`, `Color`; булевы — «Иконка».
  Номер считается среди элементов контейнера без разделителей, после `count`.
- `header` — `[заголовок, описание]`; `null` в позиции — не трогать. Только для шапок `Content` и
  `Header + description`; шапку `Title` (титул, Q&A, финал) не менять.
- `slots` — `{ "<номер>": "текст" | "__hide__" | null }` для свободных текстов из инвентаря.

Если меняли `count` у контейнера из фреймов со свободными текстами (`m-list-numbered`,
`m-list-2`, `m-roadmap`), после C снова вызвать B: номера слотов сдвинутся.

```js
// INPUT
const FILL = {
  "27:5117": {
    count: { "27:5130": 3 },
    items: { "27:5130#0": { "Заголовок": "Проблема", "Описание": "…" }, "27:5130#2": { "Style": "Accent" } },
    header: ["Заголовок слайда", null],
    slots: { "1": "Крупный тезис" }
  }
};
// END INPUT

const SEP = ['Divider', 'Arrow', 'Arrow slot'];
await Promise.all(['Regular', 'Medium', 'Bold'].map(style => figma.loadFontAsync({ family: 'Navigo', style })));
const linesOf = t => { const fs = t.fontSize === figma.mixed ? 32 : t.fontSize; const lh = t.lineHeight !== figma.mixed && t.lineHeight.unit === 'PIXELS' ? t.lineHeight.value : fs * 1.25; return Math.max(1, Math.round(t.height / lh)); };
const mainKids = box => { const ks = box.children.filter(c => !SEP.includes(c.name)); const cnt = {}; ks.forEach(k => cnt[k.name] = (cnt[k.name] || 0) + 1); const main = Object.entries(cnt).sort((a, b) => b[1] - a[1])[0]; return main ? ks.filter(k => k.name === main[0]) : []; };
const report = {}, mutated = [];
for (const [sid, spec] of Object.entries(FILL)) {
  const slide = await figma.getNodeByIdAsync(sid); const r = { items: {}, slots: {}, errors: [] };
  for (const [cid, n] of Object.entries(spec.count || {})) {
    const box = await figma.getNodeByIdAsync(cid); if (!box) { r.errors.push('no container ' + cid); continue; }
    let ks = mainKids(box);
    while (ks.length > n && ks.length > 1) { const last = ks[ks.length - 1]; const i = box.children.indexOf(last); const prev = box.children[i - 1]; last.remove(); if (prev && SEP.includes(prev.name)) prev.remove(); ks = mainKids(box); }
    while (ks.length < n) { const last = ks[ks.length - 1]; const i = box.children.indexOf(last); const prev = box.children[i - 1];
      let at = i + 1; if (prev && SEP.includes(prev.name)) { const s = prev.clone(); box.insertChild(at, s); at++; }
      const c = last.clone(); box.insertChild(at, c); ks = mainKids(box); }
    r.count = Object.assign(r.count || {}, { [cid]: mainKids(box).length });
  }
  for (const [key, props] of Object.entries(spec.items || {})) {
    let node;
    if (key.includes('#')) { const [cid, idx] = key.split('#'); const box = await figma.getNodeByIdAsync(cid); node = box ? mainKids(box)[Number(idx)] : null; }
    else node = await figma.getNodeByIdAsync(key);
    if (!node || node.type !== 'INSTANCE') { r.errors.push('no instance ' + key); continue; }
    const defs = node.componentProperties, set = {};
    for (const [name, v] of Object.entries(props)) { const k = Object.keys(defs).find(d => d.split('#')[0] === name); if (k) set[k] = v; else r.errors.push(key + ': no property ' + name); }
    node.setProperties(set); mutated.push(node.id);
    const lines = {};
    for (const t of node.findAll(n => n.type === 'TEXT' && n.componentPropertyReferences && n.componentPropertyReferences.characters)) lines[t.componentPropertyReferences.characters.split('#')[0]] = linesOf(t);
    r.items[key] = lines;
  }
  if (spec.header) {
    const h = slide.findOne(n => n.type === 'INSTANCE' && n.name === 'Header');
    if (h) { const ts = h.findAll(n => n.type === 'TEXT'); spec.header.forEach((v, i) => { if (v != null && ts[i]) ts[i].characters = v; }); r.header = ts.map(linesOf); mutated.push(h.id); }
  }
  if (spec.slots) {
    const by = {}; for (const t of slide.findAll(n => n.type === 'TEXT' && /^slot:\d+$/.test(n.name))) by[Number(t.name.slice(5))] = t;
    for (const [k, v] of Object.entries(spec.slots)) { const t = by[Number(k)]; if (v == null) continue; if (!t) { r.errors.push('no slot ' + k); continue; }
      if (v === '__hide__') { t.visible = false; r.slots[k] = 'hidden'; } else { t.characters = v; r.slots[k] = linesOf(t); } mutated.push(t.id); }
  }
  report[sid] = r;
}
return { report, mutatedNodeIds: mutated };
```

В ответе — фактическое число строк каждого текста. Больше, чем допускает `layouts.md`, — текст не
влез: сократить (`slides-text-fit`) и записать тем же скриптом. Повторный вызов безопасен.

---

## D. Заметки докладчика

```js
// INPUT
const NOTES = { "27:5117": "- Главная мысль: …\n- Переход: …" };
// END INPUT
const done = [];
for (const [id, md] of Object.entries(NOTES)) { const s = await figma.getNodeByIdAsync(id); s.speakerNotes = md; done.push(id); }
return { mutatedNodeIds: done };
```

---

## E. Холст под график или схему

Клонировать `m-canvas` (`14:2596`) или `m-canvas-desc` (`14:2610`) скриптом A, затем этим скриптом
(отдельный вызов) задать шапку и очистить холст. Возвращает id фрейма `canvas`, в который рисуют
скрипты навыков `slides-charts` и `slides-diagrams`.

```js
// INPUT
const SLIDE_ID = "27:5154";
const TITLE = "Выручка выросла в 2,4 раза за три года";
const SUBTITLE = null;             // только для m-canvas-desc: «Млрд ₽, 2022–2025»
// END INPUT
await figma.loadFontAsync({ family: 'Navigo', style: 'Regular' });
const slide = await figma.getNodeByIdAsync(SLIDE_ID);
const h = slide.findOne(n => n.type === 'INSTANCE' && n.name === 'Header');
const canvas = slide.findOne(n => n.type === 'FRAME' && n.name === 'canvas');
if (!h || !canvas) return { error: 'not an m-canvas layout' };
const ts = h.findAll(n => n.type === 'TEXT'); ts[0].characters = TITLE; if (SUBTITLE && ts[1]) ts[1].characters = SUBTITLE;
for (const k of [...canvas.children]) k.remove();
return { canvasId: canvas.id, size: [canvas.width, canvas.height], mutatedNodeIds: [h.id, canvas.id] };
```

Координаты внутри холста абсолютные, от левого верхнего угла.

---

## F. Удалить свой черновой раздел

Только по прямой просьбе пользователя и только раздел, созданный в этой сессии (его `rowId`).

```js
// INPUT
const ROW_ID = "27:4991";
// END INPUT
const row = await figma.getNodeByIdAsync(ROW_ID);
const removed = row.children.map(s => s.id);
for (const s of [...row.children]) s.remove();
return { removedSlideIds: removed }; // опустевший раздел Figma удаляет сам
```

---

## G. Переставить слайд внутри раздела

```js
// INPUT
const ROW_ID = "28:4962";
const SLIDE_ID = "35:5003";
const INDEX = 7;            // новая позиция (0 — первый слайд раздела)
// END INPUT
const row = await figma.getNodeByIdAsync(ROW_ID);
const s = await figma.getNodeByIdAsync(SLIDE_ID);
row.insertChild(INDEX, s);
return { order: row.children.map(c => c.id) };
```

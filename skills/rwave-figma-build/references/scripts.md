# Скрипты сборки для `use_figma`

Все скрипты проверены на шаблоне R'WAVE. Копировать дословно, менять только блок `// INPUT`.
Каждый вызов `use_figma` передаёт `fileKey`, `sessionId` и `skillNames: "figma-use,figma-use-slides"`
(или с префиксом `resource:`, если навыки Figma загружены как MCP-ресурсы).

Порядок вызовов для каждой партии слайдов строго такой: **A → B → C**. Клонирование и разметку
нельзя объединять в один вызов: у только что склонированного слайда вложенные слои инстансов
(в т. ч. заголовок слайда в компоненте `Header`) ещё недоступны, и заголовок «пропадает» из разметки.

---

## A. Создать раздел и склонировать макеты

```js
// INPUT
const SECTION_NAME = "Название презентации — черновик";
const ROW_ID = null;               // id существующего раздела, чтобы дописать в него; иначе null
const LAYOUT_IDS = ["11:1714", "11:11341", "11:25191"]; // id макетов в порядке колоды, повторы разрешены
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

Не использовать `figma.getSlideGrid` / `figma.setSlideGrid`: после удаления слайдов в той же сессии
`getSlideGrid` падает с ошибкой «The node with id … does not exist», а `setSlideGrid` требует передать
все слайды файла. Раздел и его слайды читать через дерево узлов (`SLIDE_GRID.children` → `SLIDE_ROW.children`),
клоны размещать через `row.appendChild(clone)`. Найти раздел по имени:
`figma.currentPage.children.find(n => n.type === 'SLIDE_GRID').children.filter(n => n.type === 'SLIDE_ROW')`.

---

## B. Разметить поля и получить инвентарь слотов

Идемпотентен: размечает только неразмеченные клоны (визуальный порядок: сверху вниз по центру поля,
затем слева направо), уже размеченные просто перечисляет. Возвращает для каждого слота текущий текст
(на свежем клоне — подсказку макета), кегль, ширину и оценку вместимости. Ответ ограничен 20 КБ:
для 10+ слайдов делить на несколько вызовов или сокращать вывод.

```js
// INPUT
const SLIDE_IDS = ["27:5082", "27:5117"];
// END INPUT

const pad = n => String(n).padStart(2, '0');
const hintLines = s => /одну строку|одной строк/i.test(s) ? 1 : /три строк/i.test(s) ? 3 : /две строк|двух строк|или две|строка или две/i.test(s) ? 2 : null;
const out = {};
for (const id of SLIDE_IDS) {
  const slide = await figma.getNodeByIdAsync(id);
  const sb = slide.absoluteBoundingBox;
  let texts = slide.findAllWithCriteria({ types: ['TEXT'] }).filter(t => t.visible);
  const allTagged = texts.length && texts.every(t => /^slot:\d+$/.test(t.name));
  if (!allTagged) {
    texts = texts.map(t => ({ t, b: t.absoluteBoundingBox }))
      .sort((p, q) => (Math.round((p.b.y + p.b.height / 2 - sb.y) / 24) - Math.round((q.b.y + q.b.height / 2 - sb.y) / 24)) || (p.b.x - q.b.x))
      .map(o => o.t);
    texts.forEach((t, i) => { t.name = 'slot:' + pad(i + 1); });
  } else {
    texts.sort((a, b) => a.name.localeCompare(b.name));
  }
  out[id] = texts.map(t => {
    const fs = t.fontSize === figma.mixed ? 32 : t.fontSize;
    const ph = t.characters.replace(/\s+/g, ' ').trim();
    const autoW = t.textAutoResize === 'WIDTH_AND_HEIGHT';
    const lines = autoW ? 1 : (hintLines(ph) || Math.max(1, Math.round(t.height / (fs * 1.25))));
    const cpl = Math.floor(t.width / (fs * 0.55));
    return { slot: Number(t.name.slice(5)), text: ph.slice(0, 60), fs, w: Math.round(t.width), lines, maxChars: autoW ? Math.max(ph.length, 4) : cpl * lines, autoWidth: autoW };
  });
}
return out;
```

Поле с `autoWidth: true` — короткая метка (цифра, дата, тег): длиннее подсказки не писать,
иначе текст уедет за край контейнера и обрежется.

---

## C. Заполнить слоты

Значение `null` или отсутствующий ключ — слот не трогать (номера шагов, цифры в кружках).
Значение `"__hide__"` — скрыть слот; если слот единственный текст в своей плашке (например,
«Тег — опционально»), скрывается вся плашка. После заполнения возвращает фактическое число строк.

```js
// INPUT
const FILL = {
  "27:5117": { "1": "Харнесс берёт рутину на себя", "2": "Структура истории", "5": "__hide__" }
};
// END INPUT

const report = {};
const mutated = [];
for (const [slideId, values] of Object.entries(FILL)) {
  const slide = await figma.getNodeByIdAsync(slideId);
  const bySlot = {};
  for (const t of slide.findAllWithCriteria({ types: ['TEXT'] })) {
    const m = /^slot:(\d+)$/.exec(t.name); if (m) bySlot[Number(m[1])] = t;
  }
  const targets = Object.entries(values).filter(([, v]) => v !== null && v !== undefined).map(([k, v]) => [bySlot[Number(k)], v, k]);
  const lost = targets.filter(([t]) => !t).map(([, , k]) => k);
  const fonts = new Set();
  for (const [t] of targets) if (t) for (const s of t.getStyledTextSegments(['fontName'])) fonts.add(JSON.stringify(s.fontName));
  await Promise.all([...fonts].map(f => figma.loadFontAsync(JSON.parse(f))));
  const res = [];
  for (const [t, v, k] of targets) {
    if (!t) continue;
    if (v === "__hide__") {
      const p = t.parent;
      if (p && p.type === 'FRAME' && p.children.length === 1) p.visible = false; else t.visible = false;
      res.push({ slot: Number(k), hidden: true }); mutated.push(t.id); continue;
    }
    t.characters = v;
    const fs = t.fontSize === figma.mixed ? 32 : t.fontSize;
    const lh = t.lineHeight !== figma.mixed && t.lineHeight.unit === 'PIXELS' ? t.lineHeight.value : fs * 1.25;
    res.push({ slot: Number(k), lines: Math.max(1, Math.round(t.height / lh)) });
    mutated.push(t.id);
  }
  report[slideId] = { filled: res, missingSlots: lost };
}
return { report, mutatedNodeIds: mutated };
```

Если `lines` больше `lines` из инвентаря — текст не влез: сократить (навык `rwave-text-fit`) и
перезаписать тот же слот этим же скриптом. Поля размечены, повторная запись безопасна. У крупных
заголовков с фиксированной высотой `lines` бывает завышен — сверять скриншотом.

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

## E. Пустой слайд под график или схему

Клонировать макет `content-canvas` (см. `rwave-template` → layouts.md) скриптом A, затем
этим скриптом (отдельный вызов) задать заголовок и очистить область контента. Возвращает id
холста `canvas`, в который рисуют скрипты из навыков `rwave-charts` и `rwave-diagrams`.

```js
// INPUT
const SLIDE_ID = "27:5154";
const TITLE = "Выручка выросла в 2,4 раза за три года";
// END INPUT
const slide = await figma.getNodeByIdAsync(SLIDE_ID);
const frame = slide.children[0];
const header = frame.children.find(n => n.name === 'Header');
const content = frame.children.find(n => n.name === 'Content');
if (!header || !content) return { error: "not a content-canvas layout", kids: frame.children.map(n => n.name) };
const title = header.findAllWithCriteria({ types: ['TEXT'] })[0];
await Promise.all(title.getStyledTextSegments(['fontName']).map(s => figma.loadFontAsync(s.fontName)));
title.characters = TITLE; title.name = 'slot:01';
for (const k of [...content.children]) k.remove();
const canvas = figma.createFrame();
content.appendChild(canvas);
if (content.layoutMode !== 'NONE') { canvas.layoutSizingHorizontal = 'FILL'; canvas.layoutSizingVertical = 'FILL'; }
else { canvas.resize(1840, 880); canvas.x = 40; canvas.y = 0; }
canvas.name = 'canvas'; canvas.fills = []; canvas.clipsContent = false;
return { canvasId: canvas.id, size: [canvas.width, canvas.height], mutatedNodeIds: [title.id, canvas.id] };
```

Холст 1840×880, отступы задаёт сам макет. Координаты внутри холста абсолютные.

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

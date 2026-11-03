# Скрипт снятия пар для `use_figma`

Только чтение. Работает в файлах Figma Design и Figma Slides. Возвращает для каждой пары тексты
сгенерированного и финального слайда: `y|кегль|текст`, сверху вниз. Ответ ограничен 20 КБ — для
больших наборов вызывать частями через `FROM` / `TO`.

```js
// INPUT
const MODE = "frame";            // "frame" — фрейм-сравнение слева/справа; "list" — явные пары
const CONTAINER_ID = "1:2";      // для MODE = "frame": фрейм, где лежат слайды 1920×1080
const PAIRS = [];                // для MODE = "list": [["id сгенерированного", "id финального"], ...]
const FROM = 0, TO = 7;          // какие пары вернуть в этом вызове
// END INPUT

let pairs = [];
if (MODE === "frame") {
  const root = await figma.getNodeByIdAsync(CONTAINER_ID);
  const frames = root.children.filter(n => (n.type === 'FRAME' || n.type === 'SLIDE') && Math.round(n.width) === 1920);
  const mid = root.width / 2;
  const left = frames.filter(f => f.x + f.width / 2 < mid), right = frames.filter(f => f.x + f.width / 2 >= mid);
  pairs = left.map(l => [l, right.find(r => Math.abs(r.y - l.y) < 600) || null]);
} else {
  for (const [g, f] of PAIRS) pairs.push([await figma.getNodeByIdAsync(g), f ? await figma.getNodeByIdAsync(f) : null]);
}
function texts(f) {
  if (!f) return null;
  const fb = f.absoluteBoundingBox;
  return f.findAllWithCriteria({ types: ['TEXT'] }).filter(t => {
    if (!t.visible) return false;
    for (let p = t.parent; p && p !== f; p = p.parent) if (p.visible === false) return false;
    const b = t.absoluteBoundingBox;
    return b && b.x < fb.x + fb.width && b.y < fb.y + fb.height && b.x + b.width > fb.x && b.y + b.height > fb.y;
  }).map(t => { const b = t.absoluteBoundingBox; return { y: Math.round(b.y - fb.y), x: Math.round(b.x - fb.x), fs: t.fontSize === figma.mixed ? 'mx' : Math.round(t.fontSize), s: t.characters.replace(/\s+/g, ' ').trim() }; })
    .filter(o => o.s && !/^\d{1,2}$/.test(o.s))
    .sort((a, b) => a.y - b.y || a.x - b.x)
    .map(o => `${o.y}|${o.fs}|${o.s.slice(0, 200)}`);
}
const used = new Set(pairs.map(p => p[1] && p[1].id));
return {
  total: pairs.length,
  unpairedFinal: MODE === "frame" ? (await figma.getNodeByIdAsync(CONTAINER_ID)).children.filter(n => Math.round(n.width) === 1920 && !pairs.some(p => p[0] === n) && !used.has(n.id)).map(n => n.id) : [],
  pairs: pairs.slice(FROM, TO).map(([g, f]) => ({ gen: g.id, final: f ? f.id : null, g: texts(g), f: texts(f) }))
};
```

Пара без финального слайда (`final: null`) — слайд удалили или слили с соседним. Финальный слайд
без пары в `unpairedFinal` — слайд добавил автор. Слияние проверить по скриншотам и записать в
`pairs.json` вручную (`"merged": ["id1", "id2"]`).

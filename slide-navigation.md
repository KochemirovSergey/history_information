# Slide Navigation — как устроено переключение

## Общая идея

В проекте два независимых механизма перехода между слайдами:

1. **CSS translateY** — обычная вертикальная прокрутка между обычными слайдами
2. **Morph-анимация** — плавный морфинг SVG-элементов между двумя состояниями одного и того же слайда (без перемотки контейнера)

Слайды 0 и 1 физически живут в **одном DOM-узле** (`#s0`). Переход между ними — это морфинг SVG, а не CSS-прокрутка. Слайды 2+ — отдельные DOM-узлы, переключаются через `translateY`.

---

## HTML-структура

```html
<div id="wrap">                          <!-- overflow:hidden, 100vw×100vh -->
  <div id="container">                   <!-- flex-column, transition: translateY -->

    <div class="slide" id="s0">          <!-- слайды 0 и 1 живут здесь -->
      <div id="tzone"></div>             <!-- SVG рендерится сюда -->
    </div>

    <div class="slide ph">...</div>      <!-- слайд 2 -->
    <div class="slide ph">...</div>      <!-- слайд 3 -->
    <div class="slide ph">...</div>      <!-- слайд 4 -->

  </div>
</div>
```

`.slide` — `width:100vw; height:100vh; flex-shrink:0`

`#container` — `display:flex; flex-direction:column; transition: transform .75s cubic-bezier(.77,0,.175,1)`

---

## Навигация: функция `go(n)`

```js
const TOTAL = 5;   // всего слайдов (0..4)
let cur = 0;       // текущий индекс

function go(n) {
  if (n < 0 || n >= TOTAL) return;
  const prev = cur; cur = n;

  // обновить dot-индикаторы
  document.querySelectorAll('.dot').forEach((d, i) =>
    d.classList.toggle('on', i === n)
  );

  // переход 0 ↔ 1: только морфинг, без CSS-прокрутки
  if (prev <= 1 && n <= 1) {
    animateMorph(n === 1 ? 1 : 0);
    return;
  }

  // переход к слайду 2+: CSS translateY
  // cur 0 и 1 → translateY(0),  cur 2 → translateY(-100vh), cur 3 → -200vh ...
  cont.style.transform = `translateY(-${Math.max(n - 1, 0) * 100}vh)`;

  // если прыгаем к слайду 1 или 0 из дальнего слайда — установить morph вручную
  if (n === 1) { morphT = 1; applyMorph(1); }
  if (n === 0) { morphT = 0; applyMorph(0); }
}
```

### Маппинг cur → translateY

| cur | translateY |
|-----|-----------|
| 0   | 0         |
| 1   | 0         |
| 2   | -100vh    |
| 3   | -200vh    |
| 4   | -300vh    |

Слайды 0 и 1 оба при `translateY(0)` — контейнер не двигается, морф отрабатывает внутри `#s0`.

---

## Ввод пользователя

```js
// кнопки ↑ ↓
document.getElementById('bup').onclick = () => go(cur - 1);
document.getElementById('bdn').onclick = () => go(cur + 1);

// колесо мыши с блокировкой (900мс между переходами)
let wlock = false;
window.addEventListener('wheel', e => {
  if (wlock) return;
  go(e.deltaY > 0 ? cur + 1 : cur - 1);
  wlock = true;
  setTimeout(() => wlock = false, 900);
}, { passive: true });

// клавиатура
window.addEventListener('keydown', e => {
  if (e.key === 'ArrowDown') go(cur + 1);
  if (e.key === 'ArrowUp')   go(cur - 1);
});
```

---

## Morph-анимация (переход 0 ↔ 1)

`morphT` — число от 0 до 1: `0` = состояние А, `1` = состояние Б.

### Запуск

```js
let morphT = 0, morphAnimId = null;

function animateMorph(target) {
  if (morphAnimId) cancelAnimationFrame(morphAnimId);
  const from = morphT, to = target, dur = 800, t0 = performance.now();

  function frame(now) {
    const raw = Math.min((now - t0) / dur, 1);
    morphT = lerp(from, to, easeIO(raw));
    applyMorph(morphT);
    if (raw < 1) morphAnimId = requestAnimationFrame(frame);
    else { morphT = to; applyMorph(to); morphAnimId = null; }
  }
  morphAnimId = requestAnimationFrame(frame);
}
```

### Easing и lerp

```js
function lerp(a, b, t) { return a + (b - a) * t; }
function easeIO(t) { return t < 0.5 ? 2*t*t : -1 + (4 - 2*t)*t; }  // ease-in-out квадратичный
```

### Применение морфа

`applyMorph(t)` вызывается на каждом кадре и обновляет SVG-атрибуты через `lerp`:

```js
function applyMorph(t) {
  // морфируемые элементы — каждый хранит состояния tx (from) и cx (to)
  mPeriods.forEach(m => {
    m.rect.setAttribute('x',            lerp(m.tx.rx,  m.cx.rx,  t));
    m.rect.setAttribute('y',            lerp(m.tx.ry,  m.cx.ry,  t));
    m.rect.setAttribute('width',        lerp(m.tx.rw,  m.cx.rw,  t));
    m.rect.setAttribute('height',       lerp(m.tx.rh,  m.cx.rh,  t));
    m.rect.setAttribute('fill-opacity', lerp(m.tx.rfo, m.cx.rfo, t));
    // ... другие атрибуты
  });

  // элементы только состояния А — исчезают
  mFadeOut.forEach(e => e.setAttribute('opacity', lerp(1, 0, t)));

  // элементы только состояния Б — появляются
  mColCells.forEach(c => {
    c.rect.setAttribute('fill-opacity', lerp(0, c.tfo, t));
    c.textEls.forEach(te => te.setAttribute('fill-opacity', lerp(0, c.tto, t)));
  });
}
```

### Структура данных элемента

При рендере каждый морфируемый элемент сохраняется с двумя наборами координат:

```js
mPeriods.push({
  rect,        // ссылка на SVG-элемент
  nameEl,
  divider,
  tx: {        // состояние А (timeline): начальные значения атрибутов
    rx, ry, rw, rh, rfo,
    nx, ny, nfs, nanchor
  },
  cx: {        // состояние Б (columns): целевые значения атрибутов
    rx, ry, rw, rh, rfo,
    nx, ny, nfs, nanchor
  }
});
```

`applyMorph(t)` интерполирует между `tx.*` и `cx.*`.

---

## Как перенести в другой проект

### Минимальный чеклист

1. **HTML**: контейнер `flex-column` с `overflow:hidden` на обёртке, слайды `100vw×100vh` с `flex-shrink:0`
2. **CSS**: `transition: transform .75s cubic-bezier(.77,0,.175,1)` на контейнере
3. **JS**: переменные `cur`, `TOTAL`; функции `go()`, `animateMorph()`, `applyMorph()`, `lerp()`, `easeIO()`
4. **Массивы состояний**: для каждого морфируемого элемента — сохранить ссылку на DOM-узел + объект `{from, to}` по каждому анимируемому атрибуту
5. **Рендер**: вызвать `applyMorph(morphT)` после каждого ре-рендера чтобы восстановить текущее состояние

### Что можно менять

- `dur = 800` — длительность морфа в мс
- `easeIO` — можно заменить на любую easing-функцию `f(t) → t`
- `900` в `wlock` — пауза между прокрутками колесом
- `.75s cubic-bezier(.77,0,.175,1)` — CSS-переход для обычных слайдов

### Важный нюанс с text-anchor

SVG `text-anchor` — дискретный атрибут, его нельзя интерполировать числом. В этом проекте решено переключением в середине анимации:

```js
m.nameEl.setAttribute('text-anchor', t >= 0.5 ? 'middle' : m.tx.nanchor);
```

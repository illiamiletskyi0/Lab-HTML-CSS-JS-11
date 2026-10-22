# ЛР №11 - дослідницька частина (§4). ShowFinder і TVmaze API

**Тема:** AI-агент для інтеграції REST API: Fetch API, Promise та async/await, аналіз документації, стани завантаження й помилок, debugging через Network.

**Дата:** 22.10.2026

**AI-агент:** DeepSeek V4.1 Flash 1017 у Freebuff.

## 1. Таблиця 13 експериментів

| # | Що досліджено | Мій аналіз | Фактичний результат (Console і Network) | Пояснення (якщо не збіглося) |
| --- | --- | --- | --- | --- |
| 1 | `fetch` і статус 404 | Думав, що `fetch` відхилить проміс на 404. Насправді він відхиляє лише на збій мережі, скасування чи CORS, а HTTP-статус - це дані: `ok` false, `status` 404, у тілі JSON з описом помилки | `false 404`; `{name: "Not Found", code: 0, status: 404}`; у Console червоний рядок браузера. Network: один запит → **404**, `application/json; charset=UTF-8`, 1,09 с; `Uncaught` немає | - |
| 2 | Тіло читається один раз; HTML замість JSON | Тіло - одноразовий потік: перше читання дасть назву, друге - `TypeError`, а спроба розібрати HTML як JSON - `SyntaxError` на першому символі `<` | `Game of Thrones`; `вдруге: TypeError` (body stream already read); `HTML: SyntaxError Unexpected token '<', "<!doctype "...`. Network: `/shows/82` → 200 (`application/json`), `experiments.html` → 200 (`text/html`) | Дві різні причини: `TypeError` - тіло прочитали двічі, `SyntaxError` - сервер віддав HTML замість JSON |
| 3 | `forEach`, `for…of`, `Promise.all` | `forEach` не чекає промісів, тому одразу буде 0, а через 2 с - 3. Послідовний `for…of` - сума трьох затримок, `Promise.all` - найдовша з них; чекав різницю приблизно втричі | `forEach: 0`; `через 2 с: 3`; `for…of: 1972 мс`; `Promise.all: 381 мс`. Network: `for…of` - 1233, 411, 320 мс сходинкою; `Promise.all` - старт 66 399–66 400 мс, 80–378 мс паралельно | Я чекав ~3×, вийшло **5,2×**: перший послідовний запит платив установлення з’єднання (1233 мс), наступні були «на гарячому» каналі |
| 4 | `return` і `return await` у `try` | `catch` навколо `return fetch(...)` не спрацює: `try` завершується до відхилення поверненого проміса. З `return await` помилка виникає всередині `try` і ловиться | `поза load: TypeError` - варіант `return fetch(url)`; з `return await` - `catch у load2: TypeError` і `null`. Network: два запити до `.invalid` failed, `net::ERR_NAME_NOT_RESOLVED`; `Uncaught` немає, бо `.catch` стоїть зовні | - |
| 5 | `Promise.all` і `allSettled` | `all` відхилиться на першій помилці й відкине решту, навіть якщо ті відповіді вже прийшли. `allSettled` не відхиляється ніколи й дає статус по кожному запиту | `all: TypeError`; `allSettled: ["fulfilled","rejected","fulfilled"]`; статуси `[200, TypeError, 200]`. Network: два вдалі запити → **200**, але їхні результати втрачено | - |
| 6 | `AbortController` | Скасування до відповіді - `AbortError`. Скасування після заголовків не обнуляє `Response`: `status` лишиться 200, але тіло вже не прочитається. У Network такі запити позначені `(canceled)` | `до відповіді: AbortError`; `після відповіді: 200`; `json(): AbortError` (The user aborted a request). Network: `search/shows` → `(canceled)`; `shows/139` → 200, потім обрив читання тіла | Скасування - це не помилка: для користувача воно не має виглядати як «Не вдалося» |
| 7 | Заголовки відповіді | Скрипт бачить лише CORS-safelisted заголовки (`Cache-Control`, `Content-Type`, `Expires`, `Last-Modified`, `Pragma`). `Date`, `Server`, `Retry-After` приховані, доки сервер не додасть `Access-Control-Expose-Headers` | `application/json; charset=UTF-8`; `public, max-age=3600`; `date: null`; ключів 2 - `cache-control`, `content-type`. Network: у відповіді є `server`, `date`, `vary`, `access-control-allow-origin`, але скрипт їх не бачить | TVmaze не надсилає `Access-Control-Expose-Headers`, тому `Retry-After` на 429 теж буде недоступний |
| 8 | HTTP-кеш | `max-age=3600` → браузер зберігає відповідь на годину, другий запит не йде в мережу. `cache: 'no-store'` не читає й не зберігає кеш, тому третій іде в мережу. При повторі експерименту перший запит теж прийде з кешу | `перший: 779 мс`; `другий: 6 мс`; `no-store: 281 мс`, усі 200. Network: три рядки → 200, у другого **Size 0 B** - байтів не передано | - |
| 9 | Переадресація `/lookup/shows` | `fetch` сам іде за 301, тому клієнт бачить `status` 200, `redirected` true, а `url` - фінальну адресу. Статус 301 бачить лише браузер | `200 true https://api.tvmaze.com/shows/82`; `Game of Thrones`. Network: `lookup/shows?imdb=tt0944947` → **301** (`location: /shows/82`), `shows/82` → **200** | Тому перевіряти в коді треба `redirected`, а не `status === 301` - такої умови ніколи не спрацює |
| 10 | Рядок запиту | `&` у рядку запиту - роздільник параметрів, тому `Law & Order` без кодування розривається на `q=Law ` і окремий параметр ` Order`. `URLSearchParams` кодує значення сам | `Law  Array(2)` - параметрів два; з `URLSearchParams` - `Law & Order ?q=Law+%26+Order`; перша адреса - `?q=Law%20&%20Order`. Network: запитів немає, я лише будував адреси | API отримав би «Law » замість «Law & Order», тобто шукав би не те |
| 11 | Два домени TVmaze | `api.tvmaze.com` і `www.tvmaze.com` - різні origin (протокол + домен + порт). Запит на інший origin без `Access-Control-Allow-Origin` блокує браузер, причому блокує читання відповіді, а не надсилання | `api: 200`; CORS-помилка «No 'Access-Control-Allow-Origin' header is present»; `www: TypeError Failed to fetch`. Network: `www` → **301** → **200** (HTML), рядок позначений failed | Запит дійшов і відповідь прийшла - інакше браузер не знав би, якого заголовка бракує |
| 12 | Preflight через `Content-Type` | `Content-Type: application/json` не входить до safelisted значень, тому браузер спершу надсилає `OPTIONS`. TVmaze на нього відповідає 405 без CORS-заголовків → `TypeError`. `Accept` safelisted, preflight не потрібен | `Content-Type: TypeError Failed to fetch`; `Accept: 200`. Network: `OPTIONS /shows/82` → **405** (`allow: GET, HEAD`), `GET` → failed, `GET` з `Accept` → **200** | «All endpoints are CORS enabled» стосується простих запитів: сам `GET` із `Content-Type` до сервера навіть не дійшов |
| 13 | `mode: 'no-cors'` | `no-cors` не виправляє CORS, а посилює обмеження: відповідь стає opaque - `type` opaque, `status` 0, `ok` false, тіло порожнє | `opaque 0 false`; `""`. Network: `www.tvmaze.com/shows/82` → **301**, `game-of-thrones` → **200** (HTML), але тіло для скрипта порожнє | Помилка зникає разом із даними, тому це не рішення |

## 2. Деталі по кожному експерименту

### §4.1 Fetch API і Promise

#### 1. fetch і статус 404

```js
const response = await fetch('https://api.tvmaze.com/shows/999999999');
console.log(response.ok, response.status);
console.log(await response.json());
```

```text
Failed to load resource: the server responded with a status of 404 ()
false 404
{code: 0, message: '', name: 'Not Found', status: 404}
```

Network: `/shows/999999999` → 404, `application/json; charset=UTF-8`, 1,09 с.

**Висновок.** 404 не відхиляє проміс: тіло прочиталося, помилка прийшла як звичайні дані. Тому перевіряю `response.ok`.

#### 2. Тіло відповіді читається один раз

```js
const response = await fetch('https://api.tvmaze.com/shows/82');
console.log((await response.json()).name);
try { await response.json(); } catch (error) { console.log('вдруге:', error.name); }
const page = await fetch('experiments.html');
try { await page.json(); } catch (error) { console.log('HTML:', error.name, error.message); }
```

```text
Game of Thrones
вдруге: TypeError
HTML: SyntaxError Unexpected token '<', "<!doctype "... is not valid JSON
```

Network: `/shows/82` → 200; `experiments.html` → 200, `text/html`.

**Висновок.** `TypeError` - тіло прочитали двічі; `SyntaxError` - сервер віддав HTML. Це різні діагнози.

#### 3. forEach, for…of і Promise.all

```js
const ids = [82, 139, 33320];
const load = async (id) => (await fetch(`https://api.tvmaze.com/shows/${id}/cast`, { cache: 'no-store' })).json();
const counts = [];
ids.forEach(async (id) => counts.push((await load(id)).length));
console.log('forEach:', counts.length);
await new Promise((resolve) => setTimeout(resolve, 2000));
console.log('через 2 с:', counts.length);
let start = performance.now();
for (const id of ids) await load(id);
console.log('for…of:', Math.round(performance.now() - start), 'мс');
start = performance.now();
await Promise.all(ids.map(load));
console.log('Promise.all:', Math.round(performance.now() - start), 'мс');
```

```text
forEach: 0
через 2 с: 3
for…of: 1972 мс
Promise.all: 381 мс
```

Network: `for…of` - 1233, 411, 320 мс поспіль; `Promise.all` - старт 66 399–66 400 мс, 80–378 мс.

**Висновок.** `forEach` не чекає на проміси. Послідовний варіант - сума затримок, паралельний - найдовша з них.

#### 4. return і return await у try

```js
async function load(url) {
  try { return fetch(url); } catch (error) { console.log('catch у load:', error.name); return null; }
}
const result = await load('https://api.tvmaze.invalid/shows/82').catch((error) => `поза load: ${error.name}`);
console.log(result);
```

```text
поза load: TypeError
```

Network: два запити до `.invalid` → failed, `net::ERR_NAME_NOT_RESOLVED`.

**Висновок.** `try` завершується до відхилення, тому `return fetch(...)` не ловить збій мережі. Якщо написати `return await fetch(...)`, спрацює `catch у load: TypeError` і функція поверне `null`.

#### 5. Promise.all і Promise.allSettled

```js
const urls = ['https://api.tvmaze.com/shows/82', 'https://api.tvmaze.invalid/shows/1', 'https://api.tvmaze.com/shows/139'];
try {
  const responses = await Promise.all(urls.map((url) => fetch(url)));
  console.log('all:', responses.map((response) => response.status));
} catch (error) { console.log('all:', error.name); }
const settled = await Promise.allSettled(urls.map((url) => fetch(url)));
console.log('allSettled:', settled.map((result) => result.status));
```

```text
all: TypeError
allSettled: (3) ['fulfilled', 'rejected', 'fulfilled']
```

Network: `shows/82` → 200, `.invalid` → failed, `shows/139` → 200.

**Висновок.** `all` відкидає все при одній помилці; `allSettled` дає результат по кожному запиту.

#### 6. AbortController

```js
const early = new AbortController();
const request = fetch('https://api.tvmaze.com/search/shows?q=girls', { signal: early.signal });
early.abort();
try { await request; } catch (error) { console.log('до відповіді:', error.name); }
const late = new AbortController();
const response = await fetch('https://api.tvmaze.com/shows/139', { signal: late.signal });
late.abort();
console.log('після відповіді:', response.status);
try { await response.json(); } catch (error) { console.log('json():', error.name); }
```

```text
до відповіді: AbortError
після відповіді: 200
json(): AbortError
```

Network: обидва запити `(canceled)`, `net::ERR_ABORTED`.

**Висновок.** Скасування - не помилка: після заголовків `status` лишається 200, але тіло вже не читається.

### §4.2 HTTP: статуси, заголовки, кеш

#### 7. Заголовки відповіді

```js
const response = await fetch('https://api.tvmaze.com/shows/82');
console.log(response.headers.get('content-type'));
console.log(response.headers.get('cache-control'));
console.log(response.headers.get('date'));
console.log([...response.headers.keys()]);
```

```text
application/json; charset=UTF-8
public, max-age=3600
null
(2) ['cache-control', 'content-type']
```

Network: у Response Headers є `server`, `date`, `vary`, `access-control-allow-origin`, але скрипт їх не бачить.

**Висновок.** Скрипт бачить лише два safelisted заголовки, решту браузер приховує.

#### 8. HTTP-кеш

Щоб бачити час, я обгорнув три запити у вимірювання через `performance.now()`.

```js
const run = async (label, init) => {
  const start = performance.now();
  const response = await fetch('https://api.tvmaze.com/shows/1', init);
  await response.text();
  console.log(label + ':', Math.round(performance.now() - start), 'мс, статус', response.status);
};
await run('перший');
await run('другий');
await run('no-store', { cache: 'no-store' });
```

```text
перший: 779 мс, статус 200
другий: 6 мс, статус 200
no-store: 281 мс, статус 200
```

Network: три рядки → 200, у другого Size 0 B - байтів не передано.

**Висновок.** `max-age=3600` працює: другий запит із кешу, `no-store` іде в мережу.

#### 9. Переадресація

```js
const response = await fetch('https://api.tvmaze.com/lookup/shows?imdb=tt0944947');
console.log(response.status, response.redirected, response.url);
console.log((await response.json()).name);
```

```text
200 true https://api.tvmaze.com/shows/82
Game of Thrones
```

Network: `lookup` → 301 (`location: /shows/82`), `shows/82` → 200.

**Висновок.** 301 бачить браузер, а не скрипт; у коді є лише `redirected`.

#### 10. Рядок запиту

```js
const query = 'Law & Order';
const plain = new URL(`https://api.tvmaze.com/search/shows?q=${query}`);
const encoded = new URL(`https://api.tvmaze.com/search/shows?${new URLSearchParams({ q: query })}`);
console.log(plain.searchParams.get('q'), [...plain.searchParams.keys()]);
console.log(encoded.searchParams.get('q'), encoded.search);
```

```text
Law  (2) ['q', ' Order']
Law & Order ?q=Law+%26+Order
```

Network: запитів немає - я лише будував адреси.

**Висновок.** Без кодування `&` рве запит на два параметри; `URLSearchParams` кодує сам.

### §4.3 CORS

#### 11. Два домени TVmaze

```js
const api = await fetch('https://api.tvmaze.com/shows/82');
console.log('api:', api.status);
try {
  const site = await fetch('https://www.tvmaze.com/shows/82');
  console.log('www:', site.status);
} catch (error) { console.log('www:', error.name, error.message); }
```

```text
api: 200
Access to fetch at 'https://www.tvmaze.com/shows/82' from origin 'http://127.0.0.1:8099' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource. If an opaque response serves your needs, set the request's mode to 'no-cors' to fetch the resource with CORS disabled.
www: TypeError Failed to fetch
```

Network: `www` → 301 → 200 (HTML), рядок failed.

**Висновок.** Запит дійшов і відповідь прийшла, але браузер не віддав її скрипту.

#### 12. Preflight

```js
try {
  const response = await fetch('https://api.tvmaze.com/shows/82', { headers: { 'Content-Type': 'application/json' } });
  console.log('Content-Type:', response.status);
} catch (error) { console.log('Content-Type:', error.name, error.message); }
const response = await fetch('https://api.tvmaze.com/shows/82', { headers: { Accept: 'application/json' } });
console.log('Accept:', response.status);
```

```text
Access to fetch at 'https://api.tvmaze.com/shows/82' from origin 'http://127.0.0.1:8099' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource. If an opaque response serves your needs, set the request's mode to 'no-cors' to fetch the resource with CORS disabled.
Content-Type: TypeError Failed to fetch
Accept: 200
```

Network: `OPTIONS` → 405, `GET` → failed, `GET` з `Accept` → 200.

**Висновок.** `Content-Type` робить запит preflight-ним, а TVmaze на `OPTIONS` віддає 405.

#### 13. mode: 'no-cors'

```js
const response = await fetch('https://www.tvmaze.com/shows/82', { mode: 'no-cors' });
console.log(response.type, response.status, response.ok);
console.log(JSON.stringify(await response.text()));
```

```text
opaque 0 false
""
```

Network: `www` → 301 → 200 (HTML), тіло порожнє.

**Висновок.** Помилки немає, але й даних немає - тому `no-cors` не рішення.

## 3. Що я побачив у панелі Network (§4.4)

| Що дивився | Експерименти | Результат |
| --- | --- | --- |
| Status | 1, 6, 9, 12 | `404`; `(canceled)`; `301` + `200`; `OPTIONS 405` |
| Size | 8 | другий запит - 0 B і 6 мс замість 779 мс |
| Waterfall | 3 | `forEach` і `Promise.all` - одночасно, `for…of` - поспіль |
| Headers → Response | 7 | `server` і `date` є, але скрипт їх не бачить |
| Initiator | усі | тип `fetch` зі сторінки `experiments.html` |
| Preserve log | 9, 12 | видно обидва рядки переадресації та `OPTIONS` + `GET` |

## 4. Конспект API (§4.5)

Конспект окремим файлом: [API.md](API.md). Кожен рядок я перевірив запитом:

| Знахідка | Адреса |
| --- | --- |
| `embed[]=episodes&embed[]=cast` віддає серіал, 73 епізоди й 42 актори одним запитом | `/shows/82?embed[]=episodes&embed[]=cast` |
| Порожній пошук - це `200` і `[]`, не 404 | `/search/shows?q=zzzzqqqxyz` |
| Неіснуючий id - `404` з JSON `Not Found` | `/shows/999999999` |
| `image`, `rating.average`, `summary`, `premiered` бувають `null` | `/shows/5083`, `/shows/5003`, `/shows/5215` |
| Документація обіцяє «all endpoints are CORS enabled», але `OPTIONS` повертає `405` | `OPTIONS /shows/82` |
| `Retry-After` скрипт не бачить: немає `Access-Control-Expose-Headers`; 429 відтворити не вдалося - 30 паралельних запитів дали 200 | `/search/shows?q=girls` |

## 5. Помилки AI-агентів (§4.6)

| Помилка | Доказ | Що ламає |
| --- | --- | --- |
| Не перевіряє `response.ok` | екс. 1 | 404 і 500 стають «даними» |
| Додає `Content-Type` до GET | екс. 12 | запит блокується preflight-ом |
| Не скасовує застарілі запити | екс. 3, 6 | стара відповідь перемальовує нову |
| Не кодує параметри пошуку | екс. 10 | «Law & Order» шукає «Law » |
| Запитує серіал, епізоди й акторів поспіль | екс. 3, API.md | три затримки замість однієї |
| Лишає стан завантаження після помилки | екс. 4, 5 | екран назавжди «Завантаження…» |
| Показує `AbortError` як помилку | екс. 6 | «Не вдалося…» під час звичайного пошуку |
| «Виправляє» CORS через `no-cors` | екс. 11, 13 | помилка зникає разом із даними |

## 6. Висновок

Я зрозумів, що більшість збоїв у застосунку - не проблема мережі, а неправильне читання того, що повернув браузер. `fetch` не падає на 404 і 500: статус треба перевіряти через `response.ok`, інакше JSON помилки стане результатом пошуку. Тіло читається один раз, а `SyntaxError` на `<` означає, що сервер віддав HTML замість JSON.

Незалежні запити варто робити паралельно, `Promise.all` нещадний до часткових даних (для них є `allSettled`), а `forEach` з async не чекає результатів. `return fetch(...)` у `try` не ловить збій мережі - ловить лише `return await`. Скасування - це окремий стан, а не помилка: `AbortError` не має показуватися користувачу.

Кеш браузера вже працює завдяки `max-age=3600`, тому повторне відкриття серіалу може бути миттєвим. Переадресацію 301 клієнт не бачить - тільки `redirected`. Частину заголовків (`date`, `server`, `Retry-After`) скрипт не отримує, тому паузу на 429 треба рахувати самому.

CORS - це дозвіл сервера й заборона браузера. Запит на `www.tvmaze.com` доходить, але відповідь не читається; `Content-Type` у GET робить запит preflight-ним, а TVmaze на `OPTIONS` відповідає 405; `no-cors` не лікує проблему, а віддає порожню відповідь.

І головне: документацію треба перевіряти запитом. `embed` справді економить два запити, порожній пошук - це `200` і `[]`, частина полів буває `null`, а обіцяні 20 запитів за 10 секунд - мінімум, а не точний ліміт.

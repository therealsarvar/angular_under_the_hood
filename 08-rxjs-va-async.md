# Bo'lim 8: RxJS va Async

> RxJS — Angular'ning **async qoni**. HttpClient, Router event'lari, Forms'ning
> `valueChanges`/`statusChanges`, `EventEmitter` — bularning hammasi ostida
> **Observable** yotadi. Bu bo'lim Observable'ning ichki mexanizmini (Observer →
> Subscriber zanjiri, Subscription va teardown), operator va `pipe()` qanday
> ishlashini, flattening operatorlar (`switchMap`/`mergeMap`/`concatMap`/`exhaustMap`)
> farqini, Subject turlarini, `async` pipe'ning kaput ostidagi `markForCheck`
> chaqiruvini, `takeUntilDestroyed` bilan memory-leak'ni yopishni va RxJS ↔
> signals interop'ni **kaput ostidan** ko'rsatadi.
>
> Alohida savolga alohida javob: **nega Angular signals kelgan bo'lsa ham RxJS'dan
> voz kecha olmayapti?** Signal — *sinxron holat* (state) uchun; RxJS — *vaqt
> bo'ylab oqadigan event'lar* (streams), cancellation, backpressure va murakkab
> async orkestratsiya uchun. Bu ikki model raqib emas, bir-birini to'ldiradi —
> shuning uchun Angular ikkalasini birga qo'llab-quvvatlaydi (`toSignal`/`toObservable`).

---

## Mundarija
- [Observable nima: stream, lazy, cold vs hot](#observable-nima-stream-lazy-cold-vs-hot)
- [Nega Angular RxJS'dan voz kecha olmaydi](#nega-angular-rxjsdan-voz-kecha-olmaydi)
- [Observable ichki mexanizmi: Observer, Subscriber, Subscription](#observable-ichki-mexanizmi-observer-subscriber-subscription)
- [Observable yaratish: creation operatorlari](#observable-yaratish-creation-operatorlari)
- [pipe() va operatorlar mexanizmi](#pipe-va-operatorlar-mexanizmi)
- [Transformation va flattening operatorlari](#transformation-va-flattening-operatorlari)
- [Filtering va utility operatorlari](#filtering-va-utility-operatorlari)
- [Combination operatorlari](#combination-operatorlari)
- [Subjects va multicasting](#subjects-va-multicasting)
- [share va shareReplay: cold'ni hot qilish](#share-va-sharereplay-coldni-hot-qilish)
- [Error handling patterns](#error-handling-patterns)
- [Async pipe: under the hood](#async-pipe-under-the-hood)
- [takeUntilDestroyed va subscription management](#takeuntildestroyed-va-subscription-management)
- [RxJS va Signals interop](#rxjs-va-signals-interop)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Observable nima: stream, lazy, cold vs hot

### Nazariya

**Observable** — bu vaqt bo'ylab **0, 1 yoki cheksiz** qiymat yetkazib beradigan
**lazy** (dangasa) manba. Uni "vaqt o'qi bo'ylab massiv" deb tasavvur qilish
mumkin: `Array` bir zumda barcha qiymatlarni saqlaydi, `Observable` esa ularni
**vaqt bo'ylab** (asinxron ham) uzatadi.

`Promise` bilan farqi tubdan:

- **Promise** — bitta qiymat, **eager** (yaratilishi bilan ishga tushadi),
  **bekor qilib bo'lmaydi** (uncancellable).
- **Observable** — ko'p qiymat, **lazy** (faqat `subscribe` qilinganda ishga
  tushadi), **cancellable** (`unsubscribe` bilan to'xtatiladi).

Observable **uch xil signal** yuboradi (Observer kontrakti):

- **`next(value)`** — navbatdagi qiymat (0..∞ marta).
- **`error(err)`** — xato; **terminal** — stream tugaydi, boshqa `next` yo'q.
- **`complete()`** — muvaffaqiyatli yakun; **terminal** — boshqa `next` yo'q.

**Terminal grammatikasi:** `next* (error | complete)?` — ya'ni istalgancha
`next`, so'ng **eng ko'pi bilan bitta** `error` yoki `complete`. Terminal
signal'dan keyin Observable hech nima yubormaydi.

**Cold vs Hot** — RxJS'dagi eng muhim tushunchalardan biri:

- **Cold Observable** — har bir `subscribe` uchun **yangi** producer yaratadi.
  Har bir subscriber o'zining mustaqil oqimini oladi (masalan, har `subscribe`da
  yangi HTTP so'rov ketadi). `HttpClient`, `of`, `interval` — cold.
- **Hot Observable** — producer **subscriber'lardan tashqarida** yashaydi;
  barcha subscriber bir xil oqimni **baham ko'radi** (shared). `Subject`,
  `fromEvent`, `share()` natijasi — hot. Kech qo'shilgan subscriber undan
  oldingi qiymatlarni o'tkazib yuborishi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

Observable — bu aslida **funksiya o'rami**. Uning yuragi — `subscribe` chaqirilganda
ishga tushadigan `subscribe` funksiyasi (`_subscribe`). RxJS manba kodida
(`rxjs/src/internal/Observable.ts`) taxminan shunday:

```ts
export class Observable<T> {
  constructor(subscribe?: (subscriber: Subscriber<T>) => TeardownLogic) {
    if (subscribe) this._subscribe = subscribe;
  }
  // Har subscribe — _subscribe funksiyasini QAYTADAN chaqiradi
  subscribe(observer): Subscription { /* ... */ }
}
```

**Cold bo'lishining sababi shu:** `subscribe` har chaqirilganda `_subscribe`
funksiyasi **qaytadan** ishga tushadi. Agar shu funksiya ichida `fetch` yoki
`setInterval` bo'lsa — har subscriber uchun **yangi** `fetch`/`setInterval`
yaratiladi. Producer subscription ichida yaratilgani uchun cold.

**Hot bo'lish** — producer'ni `subscribe` funksiyasidan **tashqariga** chiqarish
demakdir. `Subject`da producer (`subject.next(...)`) tashqarida joylashgan;
`subscribe` faqat subscriber'ni ichki ro'yxatga qo'shadi. Shu sabab hamma bir
manbani baham ko'radi.

Bu **lazy** modelning natijasi: Observable yaratilishi hech narsa qilmaydi —
faqat `subscribe` "vilkani rozetkaga" ulaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
import { Observable, of } from 'rxjs';

// Qo'lda Observable yaratish — lazy ekanini ko'rsatadi
const source$ = new Observable<number>((subscriber) => {
  console.log('Producer ishga tushdi'); // faqat subscribe'da chiqadi
  subscriber.next(1);
  subscriber.next(2);
  subscriber.complete();
  // complete'dan keyingi next() e'tiborsiz qoladi:
  subscriber.next(3); // hech qachon yetib bormaydi
});

console.log('Hali subscribe qilinmadi'); // "Producer ishga tushdi" hali chiqmagan

source$.subscribe({
  next: (v) => console.log('next:', v),
  error: (e) => console.error('error:', e),
  complete: () => console.log('complete'),
});
// Chiqish: Producer ishga tushdi → next:1 → next:2 → complete
```

```ts
// Cold: har subscribe alohida producer
import { interval } from 'rxjs';
import { take } from 'rxjs/operators';

const cold$ = interval(1000).pipe(take(3));

cold$.subscribe((v) => console.log('A:', v)); // A: 0, A: 1, A: 2
setTimeout(() => {
  // 2s kech qo'shilgan subscriber ham 0 dan boshlaydi — chunki cold
  cold$.subscribe((v) => console.log('B:', v)); // B: 0, B: 1, B: 2
}, 2000);
```

</details>

---

## Nega Angular RxJS'dan voz kecha olmaydi

### Nazariya

Angular 16+ da **signals** kelgach, ko'pchilik "RxJS endi keraksiz" deb o'yladi.
Bu **noto'g'ri**. Ikkalasi **turli muammolarni** yechadi:

| Xususiyat | Signal | Observable (RxJS) |
|-----------|--------|-------------------|
| Model | **Value** (sinxron holat) | **Stream** (vaqt bo'ylab event'lar) |
| Push/Pull | Pull (o'qiganda qiymat) | Push (kelganda beradi) |
| Vaqt (time) | Vaqt tushunchasi yo'q | Vaqt birinchi darajali (debounce, throttle) |
| Cancellation | Yo'q (kerak emas) | Bor (`switchMap`, `unsubscribe`) |
| Async orkestratsiya | Cheklangan | Kuchli (combineLatest, forkJoin, retry) |
| Ko'p qiymat | Faqat oxirgi qiymat | Butun oqim (history bilan) |

**Angular yadrosining o'zi RxJS'ga bog'langan** — bu texnik sabab:

- **`HttpClient`** har so'rovni `Observable` qaytaradi (`http.get<T>()`).
- **`Router`** — `router.events` Observable; `ActivatedRoute.params`,
  `queryParams`, `data` — hammasi Observable.
- **Reactive Forms** — `form.valueChanges`, `control.statusChanges` — Observable.
- **`EventEmitter`** — bu aslida `Subject`'ning subklassi (`@Output` ostida).

**Signal nima uchun stream'ni almashtira olmaydi?** Signal faqat **oxirgi qiymatni**
saqlaydi — u "glitch-free, memoized state". Lekin quyidagilar signal'da
tabiiy ifodalanmaydi:

- **Event oqimi** (har bosishni, har keypress'ni ushlash) — signal faqat oxirgi
  qiymatni ko'radi, oraliq event'lar yo'qoladi.
- **Time-based** operatsiyalar: `debounceTime`, `throttleTime`, `auditTime`.
- **Cancellation** semantikasi: `switchMap` — eski so'rovni bekor qilib yangisiga
  o'tish (search-as-you-type uchun kritik).
- **Murakkab kombinatsiya**: `forkJoin` (bir nechta so'rovni kutish), `retry`
  (xatoda qayta urinish), `concatMap` (navbatga qo'yish).

**Xulosa:** signals — **state uchun**, RxJS — **events va async oqimlar uchun**.
Angular jamoasi rasman shu pozitsiyani tutadi: signals RxJS'ni almashtirmaydi,
`toSignal`/`toObservable` orqali **ko'prik** quriladi. RxJS'ni to'liq tashlab
yuborish — HttpClient, Router va Forms'dan voz kechish demakdir.

<details>
<summary><strong>Under the Hood</strong></summary>

`EventEmitter`ning `@angular/core` ichidagi ta'rifi (soddalashtirilgan) shuni
tasdiqlaydi:

```ts
// @angular/core — EventEmitter aslida Subject
export class EventEmitter<T> extends Subject<T> {
  emit(value?: T) { super.next(value); }
  subscribe(next?, error?, complete?): Subscription { /* ... */ }
}
```

Ya'ni har bir `@Output()` (yoki `output()`) — ostida RxJS `Subject`. `emit()`
chaqirilganda `Subject.next()` ishlaydi. Bu Angular komponent event modeli
**tubdan** RxJS ustiga qurilganini ko'rsatadi.

`HttpClient` ham shunday: `HttpClient.request()` har chaqiruvda `new Observable`
qaytaradi va `subscribe` bo'lgandagina `HttpBackend` orqali haqiqiy `XMLHttpRequest`/
`fetch` yuboradi. `unsubscribe` qilinsa — so'rov **bekor qilinadi** (`xhr.abort()`).
Bu cancellation'ni signal bera olmaydi — shuning uchun `switchMap` bilan
search-as-you-type Angular'da RxJS talab qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// Angular yadrosi RxJS'ga bog'langanini ko'rsatuvchi tipik komponent
import { Component, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { ActivatedRoute, Router } from '@angular/router';
import { FormControl } from '@angular/forms';
import { switchMap, debounceTime, distinctUntilChanged } from 'rxjs/operators';

@Component({
  selector: 'app-search',
  template: `<input [formControl]="query" placeholder="Qidiruv..." />`,
})
export class Search {
  private http = inject(HttpClient);
  private route = inject(ActivatedRoute);

  query = new FormControl('');

  // Forms → RxJS: valueChanges Observable
  results$ = this.query.valueChanges.pipe(
    debounceTime(300),          // faqat RxJS bera oladi (time)
    distinctUntilChanged(),     // bir xil qiymatni tashlab yuborish
    switchMap((q) =>            // eski so'rovni bekor qil, yangisiga o't
      this.http.get(`/api/search?q=${q}`) // HttpClient → Observable
    ),
  );

  // Router → RxJS: route parametrlari Observable
  id$ = this.route.params;
}
```

Bu misolni **faqat signal bilan** yozib bo'lmaydi: `debounceTime` (vaqt),
`switchMap` (cancellation) va `HttpClient` (Observable qaytaradi) — uchalasi ham
RxJS'ni talab qiladi.

</details>

---

## Observable ichki mexanizmi: Observer, Subscriber, Subscription

### Nazariya

RxJS'da to'rtta asosiy rol bor:

- **Observer** — `{ next, error, complete }` uchligiga ega oddiy obyekt.
  Sen `subscribe`ga beradigan "tinglovchi".
- **Subscriber** — RxJS Observer'ni **o'rab oladi** (`SafeSubscriber`) va grammatikani
  himoya qiladi: `complete`/`error`dan keyin `next`larni **bloklaydi**, `isStopped`
  flag'ini qo'yadi, xatolarni ushlaydi. Subscriber — bu **himoyalangan Observer**.
- **Subscription** — ishlab turgan bajarilishni ifodalaydi; `unsubscribe()` metodi
  bilan resurslarni tozalaydi (teardown). Subscription'lar **daraxt** hosil qiladi:
  ota-`unsubscribe` bolalarni ham to'xtatadi.
- **Teardown logic** — Observable ichida `return () => {...}` qaytarilgan funksiya;
  `unsubscribe`da chaqiriladi (masalan `clearInterval`, `removeEventListener`).

Operatorlar zanjiri aslida **Subscriber'lar zanjiri**: `source$.pipe(map, filter)`
`subscribe` qilinsa, subscriber'lar teskari yo'nalishda ulanadi — pastdagi
(final) subscriber yuqoriga (source'gacha) subscribe qiladi, qiymat esa
yuqoridan pastga oqadi.

<details>
<summary><strong>Under the Hood</strong></summary>

`subscribe` chaqirilganda nima bo'ladi (soddalashtirilgan oqim):

1. Bering Observer (`{next, error, complete}`) yoki funksiya `SafeSubscriber`ga
   o'raladi. `SafeSubscriber extends Subscriber extends Subscription`.
2. Subscriber'da `isStopped = false`. Har `next` `isStopped`ni tekshiradi —
   `true` bo'lsa qiymat **e'tiborsiz** qoladi. Bu terminal grammatikani majburlaydi.
3. Observable'ning `_subscribe(subscriber)` funksiyasi ishga tushadi va
   **TeardownLogic** (funksiya yoki Subscription) qaytaradi.
4. Bu teardown Subscriber'ga `add()` qilinadi. `unsubscribe()` chaqirilganda
   barcha teardown'lar chaqiriladi va `closed = true` bo'ladi.

**Subscription daraxti:** operator zanjirida har bir bosqich o'z subscriber'ini
otasiga `add` qiladi. Final `unsubscribe` **butun daraxtni** tozalaydi — shuning
uchun bitta `subscription.unsubscribe()` `interval`, HTTP so'rov va event
listener'larni bir vaqtda to'xtatadi.

`error` va `complete` **avtomatik unsubscribe** qiladi: terminal signal kelganda
Subscriber `unsubscribe()`ni o'zi chaqiradi — shuning uchun `complete` bo'ladigan
oqimlar (`http.get`, `of`, `take(n)`) memory leak bermaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
import { Observable } from 'rxjs';

// Teardown logic — unsubscribe'da resurs tozalash
const timer$ = new Observable<number>((subscriber) => {
  let count = 0;
  const id = setInterval(() => subscriber.next(count++), 1000);

  // TeardownLogic — unsubscribe yoki complete'da ishlaydi
  return () => {
    clearInterval(id);
    console.log('Teardown: interval tozalandi');
  };
});

const sub = timer$.subscribe((v) => console.log(v)); // 0, 1, 2, ...

// 3.5s dan keyin to'xtatamiz → teardown ishlaydi, clearInterval chaqiriladi
setTimeout(() => sub.unsubscribe(), 3500);
```

```ts
// Subscription daraxti: bitta unsubscribe hammasini to'xtatadi
import { interval, Subscription } from 'rxjs';

const sub = new Subscription();
sub.add(interval(1000).subscribe((v) => console.log('A:', v)));
sub.add(interval(1500).subscribe((v) => console.log('B:', v)));

// Bitta chaqiruv — ikkala interval ham to'xtaydi
setTimeout(() => sub.unsubscribe(), 5000);
```

</details>

---

## Observable yaratish: creation operatorlari

### Nazariya

**Creation operatorlar** — noldan Observable yaratadigan funksiyalar (`rxjs`dan
import qilinadi, `pipe` ichida emas). Eng muhimlari:

- **`of(...values)`** — berilgan qiymatlarni ketma-ket, sinxron uzatadi, keyin
  `complete`. `of(1, 2, 3)`.
- **`from(input)`** — massiv, `Promise`, iterable yoki boshqa Observable'ni
  Observable'ga aylantiradi. `from(fetch(...))`, `from([1, 2, 3])`.
- **`fromEvent(target, name)`** — DOM/Node event'ni Observable'ga (hot). 
  `fromEvent(button, 'click')`.
- **`interval(ms)`** — har `ms` da `0, 1, 2, ...` (cheksiz, cold).
- **`timer(delay, period?)`** — `delay`dan keyin bir marta, `period` berilsa
  keyin muntazam.
- **`EMPTY`** — hech nima yubormay darhol `complete` (fallback uchun).
- **`throwError(() => err)`** — darhol `error` yuboradi.
- **`defer(factory)`** — subscribe paytida **yangi** Observable yaratadi (har
  subscriber uchun yangi holat; lazy `of` kabi).
- **`NEVER`** — hech qachon hech nima yubormaydi (test uchun).

<details>
<summary><strong>Under the Hood</strong></summary>

`of` va `from` — ular **`scheduler`siz** sinxron ishlaydi. `of(1,2,3).subscribe(...)`
chaqirilishi bilan uch `next` va `complete` **darhol**, `subscribe` qaytishidan
oldin yuboriladi. Shuning uchun `of` "asinxron kutish" bermaydi.

`defer` muhim naql: `of`, `throwError` ba'zan **eager**'ga o'xshab tuyuladi
(masalan `of(Date.now())` — vaqt `of` yaratilganda "muzlab qoladi"). `defer`
factory'ni **subscribe paytida** chaqiradi, shuning uchun `defer(() => of(Date.now()))`
har subscriber uchun yangi vaqtni beradi. Bu cold semantikani qo'lda tiklaydi.

`interval` va `timer` ostida **`asyncScheduler`** (`setInterval`/`setTimeout`)
yotadi. Ular producer'ni teardown'da (`clearInterval`) tozalaydi.

`fromEvent` **hot**: u `addEventListener` chaqiradi va teardown'da
`removeEventListener` qiladi. Event'lar subscriber bor-yo'qligidan qat'i nazar
sodir bo'ladi — shuning uchun hot.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
import { of, from, fromEvent, interval, timer, EMPTY, throwError, defer, NEVER } from 'rxjs';
import { take } from 'rxjs/operators';

of(1, 2, 3).subscribe(console.log);              // 1, 2, 3 (sinxron)
from([10, 20, 30]).subscribe(console.log);        // 10, 20, 30
from(Promise.resolve('ok')).subscribe(console.log); // ok (mikrotask)

// interval + take — cheksizni cheklaymiz
interval(500).pipe(take(3)).subscribe(console.log); // 0, 1, 2 keyin complete

// timer(delay, period): 2s kutadi, keyin har 1s
timer(2000, 1000).pipe(take(2)).subscribe(console.log); // 0 (2s da), 1 (3s da)

// throwError va EMPTY — error handling'da fallback sifatida
throwError(() => new Error('xato')).subscribe({ error: (e) => console.error(e.message) });
EMPTY.subscribe({ complete: () => console.log('darhol complete') });
```

```ts
// defer: har subscribe uchun YANGI qiymat
import { defer, of } from 'rxjs';

const now$ = defer(() => of(Date.now())); // subscribe paytida hisoblanadi

now$.subscribe((t) => console.log('A:', t));
setTimeout(() => now$.subscribe((t) => console.log('B:', t)), 100);
// A va B turli timestamp — chunki defer har safar qayta hisoblaydi
```

```ts
// fromEvent — hot Observable (DOM event)
import { fromEvent } from 'rxjs';
import { map, throttleTime } from 'rxjs/operators';

const clicks$ = fromEvent<MouseEvent>(document, 'click').pipe(
  throttleTime(1000),          // 1s da bir marta
  map((e) => ({ x: e.clientX, y: e.clientY })),
);
const sub = clicks$.subscribe(console.log);
// Tozalash: sub.unsubscribe() → removeEventListener chaqiriladi
```

</details>

---

## pipe() va operatorlar mexanizmi

### Nazariya

**Operator** — Observable'ni oladi va **yangi** Observable qaytaradigan funksiya.
RxJS 6+ da operatorlar **pipeable** (lettable): ular `.pipe()` metodi ichida
kompozitsiya qilinadi, prototype'ga bog'lanmaydi. Bu **tree-shaking**ni mumkin
qiladi — ishlatilmagan operatorlar bundle'ga tushmaydi.

`pipe()` — bu matematik funksiya kompozitsiyasi: `source.pipe(f, g, h)` aslida
`h(g(f(source)))` degani. Har bir operator (`OperatorFunction<T, R>`) — bu
`(source: Observable<T>) => Observable<R>` tipidagi funksiya.

Operatorni ikki qismga ajratamiz:

- **Pipeable operator** — `map`, `filter`, `switchMap` — `pipe()` ichida,
  mavjud Observable'ni transformatsiya qiladi.
- **Creation operator** — `of`, `from`, `interval` — noldan yaratadi, `pipe`siz.

<details>
<summary><strong>Under the Hood</strong></summary>

`pipe`ning implementatsiyasi juda oddiy — funksiyalarni chapdan o'ngga yig'adi:

```ts
// rxjs/src/internal/util/pipe.ts (soddalashtirilgan)
export function pipeFromArray(fns) {
  if (fns.length === 0) return (x) => x;
  if (fns.length === 1) return fns[0];
  return (input) => fns.reduce((prev, fn) => fn(prev), input);
}
```

Ya'ni `pipe` **hech qanday sehrsiz** — u operatorlarni ketma-ket qo'llaydi.
Har operator `source`ni oladi, `new Observable` qaytaradi.

Har bir zamonaviy operator ichida **`operate`** yordamchisi ishlatiladi. `map`
manba kodi (soddalashtirilgan):

```ts
export function map(project) {
  return operate((source, subscriber) => {
    let index = 0;
    source.subscribe(
      createOperatorSubscriber(subscriber, (value) => {
        // Har next uchun project'ni qo'llaymiz va pastga uzatamiz
        subscriber.next(project(value, index++));
      })
    );
  });
}
```

Diqqat qiling: `map` **yangi Subscriber** yaratadi (`createOperatorSubscriber`),
u manbadan qiymat oladi, `project` qo'llaydi va **pastdagi** subscriber'ga
uzatadi. Xato va complete avtomatik pastga o'tadi. Shuning uchun operator
zanjiri = subscriber zanjiri: qiymat yuqoridan pastga, subscribe esa pastdan
yuqoriga.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
import { of } from 'rxjs';
import { map, filter } from 'rxjs/operators';

// pipe = funksiya kompozitsiyasi
of(1, 2, 3, 4, 5).pipe(
  filter((n) => n % 2 === 1),   // 1, 3, 5
  map((n) => n * 10),           // 10, 30, 50
).subscribe(console.log);       // 10, 30, 50
```

```ts
// O'zingizning custom pipeable operatoringiz — bu shunchaki funksiya
import { Observable, OperatorFunction } from 'rxjs';

function logEach<T>(label: string): OperatorFunction<T, T> {
  return (source: Observable<T>) =>
    new Observable<T>((subscriber) =>
      source.subscribe({
        next: (v) => {
          console.log(`[${label}]`, v);
          subscriber.next(v); // o'zgartirmay pastga uzatamiz
        },
        error: (e) => subscriber.error(e),
        complete: () => subscriber.complete(),
      })
    );
}

of('a', 'b').pipe(logEach('debug')).subscribe();
// [debug] a → [debug] b
```

</details>

---

## Transformation va flattening operatorlari

### Nazariya

**Transformation** — qiymatni o'zgartirish yoki bir Observable'ni boshqasiga
"tekislash" (flatten). Bu bo'lim RxJS'dagi **eng muhim** va eng ko'p adashtiradigan
qismi.

Oddiy transformatsiya:

- **`map(fn)`** — har qiymatni o'zgartiradi (`Array.map` kabi).
- **`scan(fn, seed)`** — akkumulyator, har oraliq natijani chiqaradi (`reduce`,
  lekin har qadamda emit qiladi).
- **`reduce(fn, seed)`** — faqat oxirida bitta yig'ilgan qiymat.
- **`pairwise()`** — [oldingi, hozirgi] juftlik.

**Flattening (higher-order) operatorlar** — bular qiymatni **yangi Observable**'ga
aylantiradi va uni "tekislaydi". To'rttasining farqi — **concurrency strategiyasi**da:

| Operator | Strategiya | Qachon |
|----------|-----------|--------|
| **`switchMap`** | Yangi kelsa — eskini **bekor** qiladi | Search, latest wins (typeahead) |
| **`mergeMap`** | Hammasini **parallel** ishlatadi | Mustaqil so'rovlar, tartib muhim emas |
| **`concatMap`** | **Navbat**ga qo'yadi, ketma-ket | Tartib muhim (yozuv tartibi) |
| **`exhaustMap`** | Ishlab turganda yangisini **e'tiborsiz** | Login tugmasi (double-click himoyasi) |

Buni yodlash uchun: **switch** = almashtir, **merge** = birlashtir (parallel),
**concat** = ulash (navbat), **exhaust** = charchat (band bo'lsa rad et).

<details>
<summary><strong>Under the Hood</strong></summary>

To'rttasi ham ichida **`mergeInternals`** yordamchisiga tayanadi, farq — 
**`concurrent`** parametri va **bekor qilish** logikasida:

- **`mergeMap`** — `concurrent = Infinity`. Har outer qiymat uchun yangi inner
  Observable'ga subscribe qiladi va hammasi bir vaqtda faol. `mergeMap(fn, concurrency)`
  ikkinchi argument bilan parallel limitini cheklash mumkin.
- **`concatMap`** = `mergeMap(fn, 1)` — concurrency 1. Ichki buffer'ga qo'yadi,
  oldingi inner `complete` bo'lgach navbatdagisiga o'tadi.
- **`switchMap`** — har yangi outer qiymatda **oldingi inner subscription'ni
  `unsubscribe`** qiladi (bekor qiladi), keyin yangisiga subscribe qiladi. HTTP'da
  bu `xhr.abort()`ni keltirib chiqaradi.
- **`exhaustMap`** — inner faol bo'lsa (`hasInner = true`), yangi outer qiymatlarni
  **tashlab yuboradi**. Inner `complete` bo'lgachgina yangi outer qabul qilinadi.

**Nega switchMap search uchun kritik:** foydalanuvchi "ang" → "angu" → "angul"
yozganda, "ang" va "angu" so'rovlari bekor qilinadi, faqat oxirgi "angul" natijasi
qoladi. Bu **race condition**'ni (eski javob kech kelib yangisini bosib ketishi)
oldini oladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
import { fromEvent, of, interval } from 'rxjs';
import { map, scan, switchMap, mergeMap, concatMap, exhaustMap, delay, take } from 'rxjs/operators';

// map + scan
of(1, 2, 3).pipe(map((n) => n * 2)).subscribe(console.log);       // 2, 4, 6
of(1, 2, 3).pipe(scan((acc, n) => acc + n, 0)).subscribe(console.log); // 1, 3, 6
```

```ts
// switchMap: typeahead — eski so'rovni bekor qiladi
import { inject, Component } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { FormControl } from '@angular/forms';
import { debounceTime, distinctUntilChanged, switchMap, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

@Component({ selector: 'app-typeahead', template: `<input [formControl]="q" />` })
export class Typeahead {
  private http = inject(HttpClient);
  q = new FormControl('');

  results$ = this.q.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap((term) =>
      this.http.get<string[]>(`/api/search?q=${term}`).pipe(
        catchError(() => of([])), // xatoda bo'sh ro'yxat, oqim tirik qoladi
      )
    ),
  );
}
```

```ts
// concatMap vs mergeMap: tartib
import { of } from 'rxjs';
import { concatMap, mergeMap, delay } from 'rxjs/operators';

// concatMap — TARTIB saqlanadi (navbat): 1, 2, 3
of(1, 2, 3).pipe(
  concatMap((n) => of(n).pipe(delay(400 - n * 100))),
).subscribe((v) => console.log('concat:', v)); // 1, 2, 3

// mergeMap — tez tugagani birinchi chiqadi: 3, 2, 1
of(1, 2, 3).pipe(
  mergeMap((n) => of(n).pipe(delay(400 - n * 100))),
).subscribe((v) => console.log('merge:', v)); // 3, 2, 1
```

```ts
// exhaustMap: login tugmasini double-click'dan himoya
import { fromEvent, of } from 'rxjs';
import { exhaustMap, delay } from 'rxjs/operators';

const loginBtn = document.querySelector('#login')!;
fromEvent(loginBtn, 'click').pipe(
  exhaustMap(() => of('login...').pipe(delay(2000))), // 2s davomida boshqa bosishlar rad
).subscribe(console.log);
```

</details>

---

## Filtering va utility operatorlari

### Nazariya

**Filtering** — qiymatlarni saralash yoki miqdorini cheklash:

- **`filter(pred)`** — shartga mos qiymatlarni o'tkazadi.
- **`take(n)`** — birinchi `n` ta qiymat, keyin `complete`.
- **`takeUntil(notifier$)`** — `notifier$` emit qilguncha, so'ng to'xtaydi
  (unsubscribe pattern uchun asosiy).
- **`takeWhile(pred, inclusive?)`** — shart `true` bo'lguncha.
- **`first(pred?)`** / **`last(pred?)`** — birinchi/oxirgi (bo'sh bo'lsa xato).
- **`skip(n)`** / **`skipWhile`** / **`skipUntil`** — o'tkazib yuborish.
- **`distinctUntilChanged(cmp?)`** — ketma-ket takrorlangan qiymatlarni tashlaydi.
- **`debounceTime(ms)`** — qiymat kelgach `ms` jim tursa emit qiladi (typing tugagach).
- **`throttleTime(ms)`** — `ms` oynasida faqat birinchi(scroll, resize).
- **`auditTime(ms)`** — `ms` oynasida oxirgi qiymat.

**Utility**: `tap` (yon effekt, oqimni o'zgartirmaydi), `delay`, `finalize`
(complete/error/unsubscribe'da chaqiriladi), `timeout`.

<details>
<summary><strong>Under the Hood</strong></summary>

**`debounceTime` vs `throttleTime`** — vaqt boshqaruvi bilan farqlanadi:

- `debounceTime(ms)` — har yangi qiymatda ichki taymer **qayta ishga tushadi**
  (`asyncScheduler`). Qiymat kelgach `ms` davomida **yangi qiymat kelmasa**, oxirgisi
  emit qilinadi. Tez-tez keladigan event'larda faqat "tinch" nuqtada chiqadi —
  shuning uchun search uchun ideal.
- `throttleTime(ms)` — birinchi qiymatni **darhol** chiqaradi, keyin `ms` davomida
  "yopiq" turadi. Doimiy oqimda **muntazam sur'at** beradi — scroll/mousemove uchun.

**`distinctUntilChanged`** ichida oxirgi emit qilingan qiymat saqlanadi va yangisi
`===` (yoki bergan comparator) bilan solishtiriladi. Diqqat: u faqat **ketma-ket**
takrorni tashlaydi, `1,1,2,1` → `1,2,1` (uchinchi 1 o'tadi).

**`takeUntil`** ichida ikki subscription bor: source va notifier. Notifier birinchi
`next` qilganda `takeUntil` **source'ni unsubscribe** qiladi va o'zi `complete`ni
uzatadi. Bu Angular'da `ngOnDestroy` bilan bog'lash uchun asosiy pattern edi
(hozir `takeUntilDestroyed` afzal).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
import { of, interval, fromEvent } from 'rxjs';
import {
  filter, take, distinctUntilChanged, debounceTime,
  throttleTime, tap, finalize, map,
} from 'rxjs/operators';

of(1, 2, 2, 3, 3, 3, 1).pipe(
  distinctUntilChanged(),  // ketma-ket takror tashlanadi
).subscribe(console.log);  // 1, 2, 3, 1

interval(300).pipe(
  filter((n) => n % 2 === 0), // juftlar
  take(3),                    // 3 ta, keyin complete
  tap((n) => console.log('tap:', n)), // yon effekt
  finalize(() => console.log('tugadi')), // complete'da
).subscribe((n) => console.log('val:', n));
```

```ts
// debounceTime — search input uchun
import { fromEvent } from 'rxjs';
import { map, debounceTime, distinctUntilChanged } from 'rxjs/operators';

const input = document.querySelector<HTMLInputElement>('#search')!;
fromEvent(input, 'input').pipe(
  map((e) => (e.target as HTMLInputElement).value),
  debounceTime(300),        // yozib tugagach 300ms
  distinctUntilChanged(),
).subscribe((v) => console.log('qidiruv:', v));
```

```ts
// throttleTime — scroll event (muntazam sur'at)
import { fromEvent } from 'rxjs';
import { throttleTime, map } from 'rxjs/operators';

fromEvent(window, 'scroll').pipe(
  throttleTime(200),                     // har 200ms da bir marta
  map(() => window.scrollY),
).subscribe((y) => console.log('scrollY:', y));
```

</details>

---

## Combination operatorlari

### Nazariya

**Combination** — bir nechta Observable'ni birlashtiradi. Har biri boshqa
holatda kerak:

- **`combineLatest([a$, b$])`** — **istalgan** manba emit qilganda, **hamma**ning
  oxirgi qiymatini birlashtiradi. Har manba **kamida bir marta** emit qilishi shart.
  (Form maydonlarini birga kuzatish.)
- **`forkJoin([a$, b$])`** — hamma **complete** bo'lganda, har birining **oxirgi**
  qiymatini beradi. `Promise.all` kabi. (Bir nechta HTTP so'rovni birga kutish.)
- **`zip([a$, b$])`** — indeks bo'yicha juftlaydi: a[0]+b[0], a[1]+b[1].
- **`merge(a$, b$)`** — ikkalasini bitta oqimga qo'shadi (interleave), tartibsiz.
- **`concat(a$, b$)`** — a$ complete bo'lgach, b$ni boshlaydi (ketma-ket).
- **`withLatestFrom(b$)`** — asosiy oqim emit qilganda b$ning oxirgisini qo'shadi.
- **`startWith(seed)`** — oqim boshiga qiymat qo'yadi (initial value).
- **`race(a$, b$)`** — birinchi emit qilgan g'olib.

<details>
<summary><strong>Under the Hood</strong></summary>

**`combineLatest` vs `forkJoin`** — ikkalasi ham ichki massivda oxirgi qiymatlarni
saqlaydi, farq — **qachon emit qilishida**:

- `combineLatest` — har manba emit qilganda, agar **hamma** kamida bir marta
  qiymat bergan bo'lsa, joriy kombinatsiyani chiqaradi. Har o'zgarishda yangi
  kombinatsiya. Manbalar cheksiz bo'lishi mumkin.
- `forkJoin` — faqat **hamma complete bo'lganda** bir marta chiqaradi. Agar biror
  manba complete bo'lmasa (masalan `interval`), `forkJoin` **hech qachon** emit
  qilmaydi. Agar biror manba **hech nima emit qilmay** complete bo'lsa, `forkJoin`
  ham hech nima emit qilmay complete bo'ladi (`emitError`ga ehtiyot bo'ling).

**`combineLatest` gotchasi:** dastlab hamma manba emit qilmaguncha **jim turadi**.
Agar biror manba hech qachon emit qilmasa, natija ham chiqmaydi — shuning uchun
`startWith` bilan initial qiymat berish ko'p ishlatiladi.

`forkJoin` HTTP uchun ideal, chunki `http.get` bitta qiymat berib complete bo'ladi
— bu `Promise.all` semantikasi bilan mos.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
import { combineLatest, forkJoin, zip, merge, concat, of, timer } from 'rxjs';
import { map, startWith } from 'rxjs/operators';

// forkJoin — bir nechta HTTP so'rovni birga kutish (Promise.all kabi)
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

const http = inject(HttpClient);
forkJoin({
  user: http.get('/api/user/1'),
  posts: http.get('/api/user/1/posts'),
  settings: http.get('/api/user/1/settings'),
}).subscribe(({ user, posts, settings }) => {
  console.log('Hammasi keldi:', user, posts, settings);
});
```

```ts
// combineLatest — ikki form maydonini birga kuzatish
import { FormControl } from '@angular/forms';
import { combineLatest } from 'rxjs';
import { map, startWith } from 'rxjs/operators';

const firstName = new FormControl('');
const lastName = new FormControl('');

const fullName$ = combineLatest([
  firstName.valueChanges.pipe(startWith('')),  // startWith — dastlabki emit
  lastName.valueChanges.pipe(startWith('')),
]).pipe(map(([f, l]) => `${f} ${l}`.trim()));

fullName$.subscribe((name) => console.log('Toʻliq ism:', name));
```

```ts
// zip vs merge vs concat
import { of, zip, merge, concat } from 'rxjs';
import { delay } from 'rxjs/operators';

zip(of('a', 'b'), of(1, 2)).subscribe(console.log); // ['a',1], ['b',2]

merge(of('x').pipe(delay(100)), of('y')).subscribe(console.log); // y (darhol), x (100ms)

concat(of(1, 2), of(3, 4)).subscribe(console.log); // 1, 2, 3, 4 (ketma-ket)
```

</details>

---

## Subjects va multicasting

### Nazariya

**Subject** — bir vaqtning o'zida ham **Observable**, ham **Observer**. Ya'ni
unga `subscribe` qilish ham, `next()`/`error()`/`complete()` chaqirish ham
mumkin. Subject — **hot** va **multicast**: bitta ichki producer, ko'p subscriber.

To'rtta turi bor:

- **`Subject`** — oddiy. Faqat subscribe qilinganidan **keyingi** qiymatlarni oladi.
  Oldingi qiymatlar yo'qoladi.
- **`BehaviorSubject(initial)`** — **joriy qiymat**ni saqlaydi. Yangi subscriber
  darhol **oxirgi** (yoki initial) qiymatni oladi. `.value` bilan sinxron o'qish
  mumkin. State/store uchun eng ko'p ishlatiladi.
- **`ReplaySubject(bufferSize?, windowTime?)`** — oxirgi `bufferSize` qiymatni
  **yodda saqlaydi** va yangi subscriber'ga qaytaradi (replay).
- **`AsyncSubject`** — faqat **oxirgi** qiymatni, faqat `complete` bo'lgandagina
  beradi.

**Multicast** — bitta manbani ko'p joyga ulash. `Subject` bilan cold Observable'ni
hot'ga aylantirish mumkin. Amalda buni `share()`/`shareReplay()` avtomatlashtiradi
(keyingi bo'lim).

<details>
<summary><strong>Under the Hood</strong></summary>

`Subject` ichida **`observers` massivi** (subscriber'lar ro'yxati) bor. `next(v)`
chaqirilganda u **ro'yxatdagi har bir** subscriber'ning `next(v)`ini chaqiradi —
shu sabab multicast. `subscribe` esa subscriber'ni ro'yxatga qo'shadi (`_subscribe`
ichida `this.observers.push(subscriber)`).

```ts
// Subject.next (soddalashtirilgan)
next(value: T) {
  if (this.closed) return;
  const { observers } = this;
  const copy = observers.slice(); // snapshot — reentrancy'dan himoya
  for (const observer of copy) observer.next(value);
}
```

**`BehaviorSubject`** — `Subject`ni kengaytiradi, `_value` maydonini saqlaydi.
`next(v)` `_value = v` qiladi. Yangi `_subscribe` esa subscriber'ni ro'yxatga
qo'shishdan **oldin** darhol `subscriber.next(this._value)` chaqiradi — shuning
uchun har yangi subscriber joriy qiymatni oladi. Bu signal'ga o'xshab tuyuladi
(current value + notification) — shuning uchun `toSignal` ostida ba'zan
`BehaviorSubject`-ga o'xshash mexanizm bo'ladi.

**`ReplaySubject`** — ichki buffer massivida (`_buffer`) oxirgi N qiymatni saqlaydi
va yangi subscriber'ga hammasini uzatadi. `windowTime` bilan vaqt bo'yicha ham
eskirtiriladi.

Muhim xavf: `Subject` **hot** bo'lgani uchun `complete`/`error`dan keyin **o'lik**
qoladi — yangi subscriber darhol `complete`/`error` oladi va hech qanday qiymat
yubormaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
import { Subject, BehaviorSubject, ReplaySubject, AsyncSubject } from 'rxjs';

// Subject — kech qo'shilgan oldingini ko'rmaydi
const s = new Subject<number>();
s.subscribe((v) => console.log('A:', v));
s.next(1); // A: 1
s.subscribe((v) => console.log('B:', v)); // B hali hech nima
s.next(2); // A: 2, B: 2

// BehaviorSubject — joriy qiymat bor
const b = new BehaviorSubject<number>(0);
b.subscribe((v) => console.log('B1:', v)); // B1: 0 (darhol initial)
b.next(10);                                 // B1: 10
b.subscribe((v) => console.log('B2:', v)); // B2: 10 (oxirgi qiymat)
console.log('joriy:', b.value);             // joriy: 10 (sinxron o'qish)

// ReplaySubject — oxirgi 2 qiymatni yodda saqlaydi
const r = new ReplaySubject<number>(2);
r.next(1); r.next(2); r.next(3);
r.subscribe((v) => console.log('R:', v)); // R: 2, R: 3 (oxirgi 2 replay)
```

```ts
// BehaviorSubject bilan minimal signal-siz "store" (legacy pattern)
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class CounterStore {
  private readonly _count = new BehaviorSubject<number>(0);
  readonly count$ = this._count.asObservable(); // faqat o'qish uchun

  increment(): void {
    this._count.next(this._count.value + 1);
  }
}
// Izoh: zamonaviy Angular'da bu odatda signal() bilan yoziladi.
// BehaviorSubject — RxJS interop kerak bo'lganda (masalan combineLatest bilan).
```

</details>

---

## share va shareReplay: cold'ni hot qilish

### Nazariya

Cold Observable har `subscribe` uchun ishni **qaytadan** bajaradi. Agar bitta HTTP
so'rovni ikki joyda ishlatsangiz — ikki so'rov ketadi. **Multicasting operatorlar**
bitta bajarilishni ko'p subscriber orasida **baham** qiladi:

- **`share()`** — manbani `Subject` orqali multicast qiladi. Birinchi subscriber
  manbaga ulaydi, oxirgisi ketganda uzadi (**ref-counting**). Kech qo'shilgan
  subscriber oldingi qiymatlarni **ko'rmaydi**.
- **`shareReplay({ bufferSize, refCount })`** — `ReplaySubject` orqali multicast +
  oxirgi `bufferSize` qiymatni yangi subscriber'ga **qaytaradi**. HTTP natijasini
  cache qilish uchun eng ko'p ishlatiladi.

**Eng muhim tuzoq:** `shareReplay()` ni `refCount` siz ishlatsangiz (default
`refCount: false`), oxirgi subscriber ketganda ham manbaga ulanish **uzilmaydi** —
`interval` yoki live stream'da bu **memory leak**. HTTP (bir marta complete
bo'ladigan) uchun muammo emas, lekin cheksiz oqimlar uchun `{ refCount: true }`
bering.

<details>
<summary><strong>Under the Hood</strong></summary>

`share()` ichida **`connectable`** + **ref-counting** yotadi. U manbani bitta
`Subject`ga ulaydi va faol subscriber sonini sanaydi:

- Subscriber soni `0 → 1` bo'lganda manbaga `subscribe` qiladi (connect).
- Subscriber soni `1 → 0` bo'lganda manbadan `unsubscribe` qiladi (disconnect),
  agar `resetOnRefCountZero: true` bo'lsa.

`share({ connector: () => new ReplaySubject(1) })` — bu aslida `shareReplay(1)`ning
asosi. `shareReplay` ichida `ReplaySubject` connector sifatida ishlatiladi, shuning
uchun kech kelgan subscriber buffer'dagi qiymatlarni oladi.

**`refCount` semantikasi:** `shareReplay({ bufferSize: 1, refCount: false })` (eski
default) — birinchi subscribe manbaga ulaydi va **hech qachon uzilmaydi** (buffer
saqlanib qoladi). `refCount: true` — oxirgi subscriber ketganda manbadan uziladi va
buffer tozalanadi. Angular 9.x+ / RxJS 6.4+ dan `refCount` opsiyasi mavjud;
HTTP cache uchun `refCount: true` xavfsizroq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
import { interval } from 'rxjs';
import { share, take, tap } from 'rxjs/operators';

// share'siz: har subscriber alohida "Producer" ishga tushiradi
const cold$ = interval(1000).pipe(
  take(3),
  tap((v) => console.log('Producer:', v)), // share bo'lsa bir marta chiqadi
);
const shared$ = cold$.pipe(share());

shared$.subscribe((v) => console.log('A:', v));
setTimeout(() => shared$.subscribe((v) => console.log('B:', v)), 1500);
// Producer bir marta ishlaydi; B kech qo'shilgani uchun 0 ni o'tkazib yuboradi
```

```ts
// shareReplay — HTTP natijasini cache qilish (ko'p component bir data ishlatsa)
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { shareReplay } from 'rxjs/operators';
import { Observable } from 'rxjs';

interface Config { apiUrl: string; }

@Injectable({ providedIn: 'root' })
export class ConfigService {
  private http = inject(HttpClient);

  // Bir marta so'raladi, natija cache'lanadi; keyingi subscriber'lar shu qiymatni oladi
  readonly config$: Observable<Config> = this.http.get<Config>('/api/config').pipe(
    shareReplay({ bufferSize: 1, refCount: true }),
  );
}
```

</details>

---

## Error handling patterns

### Nazariya

Observable'da xato **terminal** — u kelsa oqim o'ladi. Shuning uchun xatoni
**oqim ichida** ushlash muhim. Asosiy operatorlar:

- **`catchError((err, caught) => Observable)`** — xatoni ushlaydi va **yangi
  Observable** qaytaradi (fallback qiymat, `EMPTY`, yoki `throwError` bilan qayta
  otish). Bu `try/catch`ning oqim versiyasi.
- **`retry({ count, delay })`** — xatoda manbaga **qayta subscribe** qiladi
  (qaytadan HTTP so'rov). `delay` bilan orasida kutish (backoff).
- **`throwError(() => err)`** — dasturiy ravishda xato yaratish.
- **`finalize(fn)`** — complete, error yoki unsubscribe'da chaqiriladi (loader
  o'chirish uchun ideal — `finally` kabi).
- **`timeout(ms)`** — belgilangan vaqtda javob kelmasa xato otadi.
- **`EMPTY`** — `catchError`da fallback: xatoni "yutib", jimgina complete qilish.

**Muhim joylashuv:** `catchError`ni **inner** Observable ichida (`switchMap` ichida)
qo'yish — outer oqimni tirik saqlaydi. Agar `catchError`ni **tashqarida** qo'ysangiz,
bitta xato **butun** oqimni o'ldiradi (masalan search input butunlay to'xtaydi).

<details>
<summary><strong>Under the Hood</strong></summary>

`catchError` ichida yangi Subscriber yaratiladi. U manbadan **`error`** signalni
ushlaydi va pastga uzatish o'rniga `selector(err, caught)` funksiyasini chaqirib,
qaytgan Observable'ga **subscribe** qiladi. `caught` — bu asl manba (retry uchun
ishlatilishi mumkin).

```ts
// catchError (soddalashtirilgan g'oya)
source.subscribe(
  createOperatorSubscriber(subscriber,
    /* next */ undefined,
    /* complete */ undefined,
    (err) => {
      const result = selector(err, caught); // fallback Observable
      innerSub = result.subscribe(subscriber); // unga subscribe qilamiz
    })
);
```

**`retry`** aslida `catchError` + qayta subscribe: xato kelganda manbaga qaytadan
`subscribe` qiladi (`count` marta). `retry({ count, delay })` — `delay` `Observable`
qaytarsa (masalan `timer`), exponential backoff quriladi. HTTP'da bu **yangi so'rov**
demakdir — idempotent bo'lmagan (POST) so'rovlarda ehtiyot bo'ling.

**Terminal grammatikasi tufayli:** xato ushlanmasa, u zanjir bo'ylab pastga
`subscriber.error(err)` sifatida o'tadi va agar hech kim ushlamasa, global
`unhandled error` bo'ladi (RxJS uni `reportUnhandledError` bilan konsolga chiqaradi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
import { of, throwError, timer } from 'rxjs';
import { catchError, retry, finalize, timeout, map } from 'rxjs/operators';
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

const http = inject(HttpClient);

// To'liq error handling pipeline
http.get<User[]>('/api/users').pipe(
  timeout(5000),                          // 5s ichida javob bo'lmasa xato
  retry({ count: 3, delay: (err, i) => timer(1000 * 2 ** i) }), // exp backoff
  catchError((err) => {
    console.error('Yuklab bo\'lmadi:', err);
    return of([] as User[]);              // fallback: bo'sh ro'yxat
  }),
  finalize(() => console.log('Loader o\'chirildi')), // har holatda
).subscribe((users) => console.log(users));

interface User { id: number; name: string; }
```

```ts
// catchError JOYLASHUVI: inner vs outer
import { FormControl } from '@angular/forms';
import { switchMap, catchError } from 'rxjs/operators';
import { of } from 'rxjs';

const q = new FormControl('');

// ✅ TO'G'RI: catchError switchMap ICHIDA — outer oqim tirik qoladi
const good$ = q.valueChanges.pipe(
  switchMap((term) =>
    http.get(`/api/search?q=${term}`).pipe(
      catchError(() => of([])), // faqat shu so'rovni yutadi
    )
  ),
);

// ❌ NOTO'G'RI: catchError tashqarida — bitta xato butun oqimni o'ldiradi
const bad$ = q.valueChanges.pipe(
  switchMap((term) => http.get(`/api/search?q=${term}`)),
  catchError(() => of([])), // xatodan keyin valueChanges BOSHQA ishlamaydi
);
```

</details>

---

## Async pipe: under the hood

### Nazariya

**`async` pipe** (`| async`) — template'da Observable (yoki Promise)'ga
**avtomatik** subscribe qiladi, oxirgi qiymatni ko'rsatadi va komponent yo'q
qilinganda **avtomatik unsubscribe** qiladi. Bu Angular'dagi memory-leak'ni
oldini olishning eng oson yo'li.

Afzalliklari:

- **Manual subscribe/unsubscribe kerak emas** — leak xavfi yo'q.
- **`OnPush` bilan mukammal** — qiymat kelganda `markForCheck` chaqiradi.
- **Zoneless bilan mukammal** — Zone.js kerak emas.

Asosiy pattern — komponentda `foo$` ni yozib, template'da `{{ foo$ | async }}`
yoki `@if (foo$ | async; as foo)` bilan ishlatish.

<details>
<summary><strong>Under the Hood</strong></summary>

`AsyncPipe` (`@angular/common`) — bu **impure** pipe (`pure: false`), ya'ni u
har change detection tsiklida `transform` chaqiriladi. Ichida:

1. **Birinchi `transform(obs$)`da** — pipe obs$ga `subscribe` qiladi va
   `_latestValue`ni saqlaydi. Subscription pipe ichida saqlanadi.
2. **Qiymat kelganda** — pipe callback ichida `this._ref.markForCheck()` chaqiradi
   (`ChangeDetectorRef`). Bu komponentni "dirty" belgilaydi — **`OnPush` va zoneless
   rejimda ham** view yangilanadi. Aynan shu sabab `async` pipe `OnPush` bilan
   ideal ishlaydi.
3. **`transform` yana chaqirilganda** — agar bir xil obs$ bo'lsa, `_latestValue`ni
   qaytaradi (qayta subscribe qilmaydi). Agar **boshqa** obs$ berilsa — eskisini
   `unsubscribe` qilib yangisiga ulaydi.
4. **`ngOnDestroy`da** — pipe `_subscription.unsubscribe()` qiladi. Angular pipe'ning
   `ngOnDestroy`ini komponent yo'q qilinganda chaqiradi.

Ichki callback (`_updateLatestValue`) manba kodida taxminan:

```ts
private _updateLatestValue(async: any, value: Object): void {
  if (async === this._obj) {
    this._latestValue = value;
    this._ref!.markForCheck(); // ← OnPush/zoneless uchun kalit
  }
}
```

Shuning uchun `async` pipe — Zone.js'ga tayanmaydi; u to'g'ridan-to'g'ri
`markForCheck` orqali CD'ni xabardor qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// OnPush komponent + async pipe — leak yo'q, manual subscribe yo'q
import { Component, inject, ChangeDetectionStrategy } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

interface User { id: number; name: string; }

@Component({
  selector: 'app-users',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [AsyncPipe],
  template: `
    @if (users$ | async; as users) {
      <ul>
        @for (u of users; track u.id) {
          <li>{{ u.name }}</li>
        }
      </ul>
    } @else {
      <p>Yuklanmoqda...</p>
    }
  `,
})
export class Users {
  private http = inject(HttpClient);
  // Template subscribe qiladi; component destroy'da avtomatik unsubscribe
  users$: Observable<User[]> = this.http.get<User[]>('/api/users');
}
```

```html
<!-- Gotcha: bir obs$ni bir necha marta | async qilmang — har biri alohida subscribe -->
<!-- ❌ NOTO'G'RI: 3 ta subscribe = 3 ta HTTP so'rov -->
<p>{{ (user$ | async)?.name }}</p>
<p>{{ (user$ | async)?.email }}</p>

<!-- ✅ TO'G'RI: bir marta subscribe, `as` bilan qayta ishlatish -->
@if (user$ | async; as user) {
  <p>{{ user.name }}</p>
  <p>{{ user.email }}</p>
}
```

</details>

---

## takeUntilDestroyed va subscription management

### Nazariya

Agar template'da `async` ishlatmasangiz va TypeScript'da **qo'lda** `subscribe`
qilsangiz — **unsubscribe** qilishni unutmaslik kerak, aks holda **memory leak**
bo'ladi (component yo'q qilingach ham subscription tirik qoladi).

Zamonaviy yechim — **`takeUntilDestroyed()`** (`@angular/core/rxjs-interop`,
Angular 16+). U komponent/direktiva yo'q qilinganda oqimni avtomatik to'xtatadi:

- **Injection context ichida** (masalan field initializer, constructor) argumentsiz
  ishlatiladi — `DestroyRef`ni o'zi topadi.
- **Kontekstdan tashqarida** (masalan `ngOnInit` ichida) — `DestroyRef`ni
  `inject(DestroyRef)` bilan olib, `takeUntilDestroyed(this.destroyRef)` deb beriladi.

Alternativalar:

- **`async` pipe** — template'da subscribe qilsangiz, eng afzal.
- **`DestroyRef.onDestroy(fn)`** — qo'lda tozalash callback'i.
- **Eski pattern:** `takeUntil(this.destroy$)` + `ngOnDestroy`da `destroy$.next()`
  — legacy, endi `takeUntilDestroyed` afzal.

<details>
<summary><strong>Under the Hood</strong></summary>

`takeUntilDestroyed` ichida **`DestroyRef`** yotadi. `DestroyRef` — komponent yoki
direktiva `NodeInjector`iga bog'langan servis; komponent yo'q qilinganda
`onDestroy` callback'lari chaqiriladi.

`takeUntilDestroyed(destroyRef?)` quyidagini qiladi:

1. Agar `destroyRef` berilmasa, `inject(DestroyRef)` bilan **joriy injection
   context**dan oladi — shuning uchun uni field initializer'da (injection context
   faol) ishlatish mumkin, lekin oddiy metod ichida emas.
2. `destroyRef.onDestroy(() => subject.next())` ro'yxatdan o'tkazadi.
3. Ichida `takeUntil(destroyed$)` qo'llaydi — komponent destroy bo'lganda oqim
   `complete` bo'ladi va source **unsubscribe** qilinadi.

`DestroyRef` — Angular 16+ da qo'shilgan; u `LView`ning destroy hook'lariga
ulanadi. Komponent `LView`i yo'q qilinganda (`ɵɵdestroyView`) barcha `DestroyRef`
callback'lari ishlaydi. Shu tariqa RxJS lifecycle Angular komponent lifecycle'iga
bog'lanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// takeUntilDestroyed — injection context (field initializer) ichida
import { Component, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({ selector: 'app-clock', template: `{{ tick }}` })
export class Clock {
  tick = 0;

  // Field initializer — injection context faol, argumentsiz ishlaydi
  private sub = interval(1000)
    .pipe(takeUntilDestroyed()) // component destroy'da avtomatik to'xtaydi
    .subscribe((v) => (this.tick = v));
}
```

```ts
// takeUntilDestroyed — ngOnInit (kontekstdan tashqari) ichida
import { Component, inject, DestroyRef, OnInit } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({ selector: 'app-timer', template: `{{ count }}` })
export class Timer implements OnInit {
  private destroyRef = inject(DestroyRef); // kontekstda oldindan olamiz
  count = 0;

  ngOnInit(): void {
    interval(1000)
      .pipe(takeUntilDestroyed(this.destroyRef)) // DestroyRef qo'lda beriladi
      .subscribe((v) => (this.count = v));
  }
}
```

```ts
// Legacy pattern (eski kodda uchraydi) — takeUntil + destroy$ Subject
import { Component, OnDestroy } from '@angular/core';
import { Subject, interval } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Component({ selector: 'app-legacy', template: `` })
export class Legacy implements OnDestroy {
  private destroy$ = new Subject<void>();

  constructor() {
    interval(1000).pipe(takeUntil(this.destroy$)).subscribe();
  }
  ngOnDestroy(): void {
    this.destroy$.next();   // oqimlarni to'xtatadi
    this.destroy$.complete();
  }
}
```

</details>

---

## RxJS va Signals interop

### Nazariya

Signals (QISM 4) va RxJS bir loyihada birga yashaydi. `@angular/core/rxjs-interop`
ikki ko'prik beradi:

- **`toSignal(observable$, options?)`** — Observable'ni **signal**'ga aylantiradi.
  U avtomatik subscribe qiladi (injection context'da), oxirgi qiymatni signal
  sifatida beradi va destroy'da unsubscribe qiladi. Template'da `foo()` deb o'qiladi.
- **`toObservable(signal, options?)`** — signal'ni **Observable**'ga aylantiradi.
  Signal o'zgarganda `next` qiladi (ichida `effect` ishlatadi).

**`toSignal` opsiyalari:**

- `initialValue` — dastlabki qiymat (Observable hali emit qilmagunча).
- `requireSync: true` — manba **sinxron** birinchi qiymat berishini kafolatlaydi
  (masalan `BehaviorSubject`), shунда `initialValue` shart emas.

**Qoida:** state va template display uchun **signal** ishlating; async oqim,
time-based operatsiyalar, HTTP, cancellation uchun **RxJS**. Chegarada `toSignal`
bilan RxJS oqimini signal'ga aylantirib, template'ni sodda qiling. Yangi
Angular'da `resource()` / `rxResource()` (Angular 19+) async data uchun signal-native
yechim beradi (`<!-- TEKSHIRILSIN: rxResource stabilligini so'nggi versiyaga ko'ra -->`).

<details>
<summary><strong>Under the Hood</strong></summary>

**`toSignal`** ichida `computed`/`signal` va Observable subscription birlashtiriladi:

1. `toSignal` bir **`signal`** (yoki ichki `WritableSignal`) yaratadi.
2. Observable'ga `subscribe` qiladi; har `next` kelganda signal `.set(value)` qiladi
   — bu signal graf orqali consumer'larni (template, `computed`) xabardor qiladi.
3. `DestroyRef` orqali destroy'da unsubscribe qiladi (shuning uchun injection
   context talab qiladi, agar `manualCleanup: true` berilmasa).
4. `requireSync: false` (default) bo'lsa va `initialValue` yo'q bo'lsa, signal tipiga
   `undefined` qo'shiladi (`Signal<T | undefined>`) — chunki Observable hali emit
   qilmagan bo'lishi mumkin.

**`toObservable`** ichida **`effect`** yotadi. Signal o'qiladi; signal o'zgarganda
effect qayta ishga tushadi va ichki `ReplaySubject(1)`ga `next` qiladi. Shuning
uchun `toObservable` **glitch-free**: effect scheduling tufayli u sinxron emas,
CD bilan mos keladigan mikrotask'da emit qiladi.

Bu ikki yo'nalish signal grafi (producer-consumer) va RxJS Subscriber zanjirini
**bir-biriga ulaydi** — shu sabab Angular ikki modelni birga qo'llab-quvvatlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// toSignal — HTTP Observable'ni template'da signal sifatida ishlatish
import { Component, inject } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { HttpClient } from '@angular/common/http';

interface User { id: number; name: string; }

@Component({
  selector: 'app-profile',
  template: `
    @if (user(); as u) {
      <h2>{{ u.name }}</h2>
    } @else {
      <p>Yuklanmoqda...</p>
    }
  `,
})
export class Profile {
  private http = inject(HttpClient);
  // Observable → signal; async pipe kerak emas, template soddalashadi
  user = toSignal(this.http.get<User>('/api/user/1')); // Signal<User | undefined>
}
```

```ts
// toObservable — signal o'zgarishini RxJS operatorlar bilan qayta ishlash
import { Component, signal, inject } from '@angular/core';
import { toObservable, toSignal } from '@angular/core/rxjs-interop';
import { HttpClient } from '@angular/common/http';
import { debounceTime, distinctUntilChanged, switchMap } from 'rxjs/operators';

@Component({
  selector: 'app-search',
  template: `
    <input (input)="query.set($any($event.target).value)" />
    @for (r of results(); track r) { <p>{{ r }}</p> }
  `,
})
export class SearchSignal {
  private http = inject(HttpClient);
  query = signal('');

  // signal → Observable → RxJS operatorlar → signal
  results = toSignal(
    toObservable(this.query).pipe(
      debounceTime(300),          // time-based — RxJS kuchi
      distinctUntilChanged(),
      switchMap((q) => this.http.get<string[]>(`/api/search?q=${q}`)),
    ),
    { initialValue: [] as string[] },
  );
}
```

```ts
// requireSync — BehaviorSubject sinxron qiymat beradi, initialValue shart emas
import { Component } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { BehaviorSubject } from 'rxjs';

@Component({ selector: 'app-sync', template: `{{ count() }}` })
export class SyncExample {
  private count$ = new BehaviorSubject<number>(0);
  // requireSync — manba darhol qiymat beradi, Signal<number> (undefined'siz)
  count = toSignal(this.count$, { requireSync: true });
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: `subscribe` qilinmasa, Observable **hech narsa qilmaydi**

Observable lazy — `subscribe` qilinmasa producer ishga tushmaydi. Ko'p yangi
developer `http.get(...)` chaqirib, natija kelmasligidan hayron bo'ladi.

```ts
// ❌ HTTP so'rov KETMAYDI — subscribe yo'q
this.http.post('/api/save', data); // hech nima bo'lmaydi!

// ✅ subscribe (yoki template'da | async, yoki toSignal)
this.http.post('/api/save', data).subscribe();
```

**Nega:** `HttpClient` cold Observable qaytaradi; haqiqiy so'rov faqat `subscribe`
paytida yuboriladi (lazy modelning to'g'ridan-to'g'ri natijasi).

### Gotcha 2: `shareReplay` `refCount`siz — cheksiz oqimda leak

```ts
// ❌ interval hech qachon complete bo'lmaydi; refCount:false bo'lsa uzilmaydi
const leaky$ = interval(1000).pipe(shareReplay(1)); // refCount default false

// ✅ oxirgi subscriber ketganda uzilsin
const safe$ = interval(1000).pipe(shareReplay({ bufferSize: 1, refCount: true }));
```

**Nega:** `refCount: false` da birinchi subscribe manbaga ulanadi va **hech qachon**
uzilmaydi. HTTP (complete bo'ladi) uchun muammo yo'q, lekin `interval`/live oqim
uchun bu ishlayotgan interval'ni ushlab qoladi — memory/CPU leak.

### Gotcha 3: `catchError` noto'g'ri joyda — butun oqim o'ladi

Yuqorida ko'rdik: `catchError`ni `switchMap`dan **tashqarida** qo'ysangiz, bitta
xato outer oqimni (masalan `valueChanges`) butunlay to'xtatadi. Doim inner
Observable ichida ushlang.

```ts
// ✅ inner ichida — outer tirik
source$.pipe(switchMap((x) => inner(x).pipe(catchError(() => EMPTY))));
```

**Nega:** xato terminal signal; u qaysi Observable ichida ushlanmasa, o'sha
Observable **butunlay** o'ladi. Inner ichida ushlash faqat shu ichki oqimni
"tiklaydi", outer ta'sirlanmaydi.

### Gotcha 4: Bir template'da bir Observable'ni ko'p marta `| async`

```html
<!-- ❌ 3 ta subscribe = 3 ta HTTP so'rov -->
{{ (data$ | async)?.a }} {{ (data$ | async)?.b }} {{ (data$ | async)?.c }}
```

**Nega:** har `| async` **alohida** subscribe qiladi. Cold Observable (HTTP) uchun
bu bir necha so'rov demakdir. Yechim: `@if (data$ | async; as data)` bilan bir marta
subscribe qilib, `data`ni qayta ishlatish (yoki `shareReplay`).

### Gotcha 5: `forkJoin` — manba emit qilmay complete bo'lsa

```ts
// EMPTY hech nima emit qilmay complete bo'ladi
forkJoin([http.get('/a'), EMPTY]).subscribe({
  next: (v) => console.log(v),      // HECH QACHON chaqirilmaydi
  complete: () => console.log('done'), // darhol complete
});
```

**Nega:** `forkJoin` har manbadan **oxirgi** qiymatni kutadi. Agar biror manba
qiymat bermay complete bo'lsa, `forkJoin` butunlay qiymatsiz complete bo'ladi.
`combineLatest` ham har manba kamida bir marta emit qilmasa jim turadi.

### Gotcha 6: `takeUntilDestroyed` argumentsiz — kontekstdan tashqarida xato

```ts
// ❌ ngOnInit — injection context faol emas
ngOnInit() {
  interval(1000).pipe(takeUntilDestroyed()).subscribe(); // Runtime xato!
}
```

**Nega:** argumentsiz `takeUntilDestroyed()` `inject(DestroyRef)`ni chaqiradi, bu
faqat injection context (constructor, field initializer)da ishlaydi. `ngOnInit`da
`takeUntilDestroyed(this.destroyRef)` deb `DestroyRef`ni qo'lda bering.

---

## Common Mistakes

### ❌ Xato 1: Nested subscribe (subscribe ichida subscribe)

```ts
// ❌ NOTO'G'RI — callback hell, cancellation yo'q, leak xavfi
this.route.params.subscribe((params) => {
  this.http.get(`/api/user/${params['id']}`).subscribe((user) => {
    this.user = user;
  });
});

// ✅ TO'G'RI — flattening operator bilan tekislash
this.route.params.pipe(
  switchMap((params) => this.http.get<User>(`/api/user/${params['id']}`)),
  takeUntilDestroyed(this.destroyRef),
).subscribe((user) => (this.user = user));
```

**Nega:** nested subscribe eski so'rovni bekor qilmaydi (route o'zgarsa eski
so'rov davom etadi — race condition), tozalash murakkab. `switchMap` outer o'zgarganda
inner'ni **avtomatik bekor** qiladi va zanjir bitta subscribe bilan boshqariladi.

### ❌ Xato 2: Qo'lda subscribe qilib, unsubscribe qilmaslik

```ts
// ❌ NOTO'G'RI — component destroy bo'lsa ham interval ishlayveradi (leak)
ngOnInit() {
  interval(1000).subscribe((v) => (this.tick = v));
}

// ✅ TO'G'RI — takeUntilDestroyed yoki template'da async
private sub = interval(1000)
  .pipe(takeUntilDestroyed())
  .subscribe((v) => (this.tick = v));
```

**Nega:** `interval` hech qachon complete bo'lmaydi. Component yo'q qilinsa ham
subscription tirik qoladi, callback komponent state'iga tegishda davom etadi —
memory leak va notoʻgʻri xatti-harakat. `async` pipe yoki `takeUntilDestroyed`
lifecycle bilan bog'laydi.

### ❌ Xato 3: `switchMap` ni "write" (POST) operatsiyada ishlatish

```ts
// ❌ NOTO'G'RI — tez bosishda oldingi SAVE bekor qilinadi (ma'lumot yo'qolishi)
saveClicks$.pipe(switchMap(() => this.http.post('/api/save', data))).subscribe();

// ✅ TO'G'RI — har save yakunlansin: concatMap (navbat) yoki exhaustMap (himoya)
saveClicks$.pipe(concatMap(() => this.http.post('/api/save', data))).subscribe();
```

**Nega:** `switchMap` yangi emit kelganda eskisini **bekor** qiladi. Read (search)
uchun bu to'g'ri, lekin write uchun bekor qilingan POST server holatini noaniq
qoldiradi. Tartib muhim bo'lsa `concatMap`, double-submit'dan himoya kerak bo'lsa
`exhaustMap`.

### ❌ Xato 4: Async data'ni signal state bilan qo'lda sinxronlash

```ts
// ❌ NOTO'G'RI — subscribe ichida signal set, ortiqcha boilerplate + leak xavfi
users = signal<User[]>([]);
constructor() {
  this.http.get<User[]>('/api/users').subscribe((u) => this.users.set(u));
}

// ✅ TO'G'RI — toSignal to'g'ridan-to'g'ri ko'prik quradi
users = toSignal(this.http.get<User[]>('/api/users'), { initialValue: [] });
```

**Nega:** qo'lda `subscribe` + `set` — takrorlanuvchi kod va tozalashni talab qiladi.
`toSignal` subscription, unsubscribe va signal yangilanishini **avtomatik** boshqaradi
(`DestroyRef` orqali). Kamroq kod, leak yo'q.

### ❌ Xato 5: `debounceTime`ni `distinctUntilChanged`siz ishlatish (search)

```ts
// ❌ Kamchilik — bir xil so'zni qayta yozsa ham so'rov ketadi
q.valueChanges.pipe(debounceTime(300), switchMap((t) => search(t)));

// ✅ TO'G'RI — o'zgarmagan qiymatni tashlab yuborish
q.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(), // "abc" → "abcd" → "abc" bo'lsa oxirgi takror tashlanadi
  switchMap((t) => search(t)),
);
```

**Nega:** `debounceTime`siz har keypress so'rov beradi; `distinctUntilChanged`siz
esa foydalanuvchi bir qiymatga qaytsa (yoki fokus o'zgarsa) ortiqcha bir xil so'rov
ketadi. Ikkalasi birga — minimal, samarali so'rovlar.

---

## Amaliy Mashqlar

### Mashq 1: Debounced counter log (Oson)

**Savol:** Tugma bosilganda `count`ni oshiradigan komponent yozing. Foydalanuvchi
tez-tez bossa ham, oxirgi bosishdan **500ms** o'tgachgina konsolga joriy `count`ni
chiqaring. `fromEvent` yoki `Subject` + `debounceTime` ishlating va
`takeUntilDestroyed` bilan tozalang.

<details>
<summary>Javob</summary>

```ts
import { Component, signal } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { Subject } from 'rxjs';
import { debounceTime } from 'rxjs/operators';

@Component({
  selector: 'app-counter',
  template: `
    <button (click)="onClick()">Bosish</button>
    <p>Count: {{ count() }}</p>
  `,
})
export class Counter {
  count = signal(0);
  private clicks = new Subject<void>();

  constructor() {
    this.clicks.pipe(
      debounceTime(500),
      takeUntilDestroyed(),
    ).subscribe(() => console.log('Barqaror count:', this.count()));
  }

  onClick(): void {
    this.count.update((c) => c + 1);
    this.clicks.next();
  }
}
```

**Tushuntirish:** har bosishda `count` darhol oshadi (signal), lekin `clicks`
Subject `debounceTime(500)` orqali faqat "tinch" nuqtada log qiladi. `takeUntilDestroyed`
constructor (injection context) ichida argumentsiz ishlaydi va component destroy'da
subscription'ni yopadi.

</details>

### Mashq 2: Typeahead search switchMap bilan (O'rta)

**Savol:** `FormControl` input'idan foydalanib typeahead qidiruv yasang: 300ms
debounce, takroriy qiymatni tashlab yuborish, eski so'rovni bekor qilib yangisiga
o'tish (`switchMap`), va HTTP xatosida bo'sh ro'yxatga tushish (oqim o'lmasin).
Natijani template'da `async` pipe bilan ko'rsating.

<details>
<summary>Javob</summary>

```ts
import { Component, inject, ChangeDetectionStrategy } from '@angular/core';
import { AsyncPipe } from '@angular/common';
import { ReactiveFormsModule, FormControl } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { debounceTime, distinctUntilChanged, switchMap, catchError, filter } from 'rxjs/operators';

interface Repo { id: number; name: string; }

@Component({
  selector: 'app-repo-search',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [ReactiveFormsModule, AsyncPipe],
  template: `
    <input [formControl]="query" placeholder="GitHub repo..." />
    @if (results$ | async; as repos) {
      <ul>
        @for (r of repos; track r.id) { <li>{{ r.name }}</li> }
      </ul>
    }
  `,
})
export class RepoSearch {
  private http = inject(HttpClient);
  query = new FormControl('', { nonNullable: true });

  results$: Observable<Repo[]> = this.query.valueChanges.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    filter((q) => q.length >= 2),        // qisqa so'rovni tashlash
    switchMap((q) =>
      this.http.get<Repo[]>(`/api/search/repos?q=${q}`).pipe(
        catchError(() => of([])),        // xatoda bo'sh, oqim tirik
      )
    ),
  );
}
```

**Tushuntirish:** `debounceTime` + `distinctUntilChanged` ortiqcha so'rovlarni
kesadi. `switchMap` search-as-you-type uchun kritik — eski so'rov bekor qilinib
(HTTP abort), faqat oxirgi natija qoladi (race condition yo'q). `catchError`ni
`switchMap` **ichida** qo'yish outer `valueChanges`ni tirik saqlaydi. `async`
pipe subscribe/unsubscribe'ni boshqaradi.

</details>

### Mashq 3: Parallel yuklash + retry + signal (Qiyin)

**Savol:** Bitta `id` uchun **uch** HTTP so'rovni (`user`, `posts`, `comments`)
**parallel** yuklang (`forkJoin`), xato bo'lsa **2 marta** 1s oraliqda qayta
urining (`retry` backoff), umumiy javob **8s**dan oshsa `timeout` bering, va
yakuniy natijani `toSignal` orqali template'da ko'rsating. Yuklanish/xato holatini
ham ko'rsating.

<details>
<summary>Javob</summary>

```ts
import { Component, inject, computed } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { HttpClient } from '@angular/common/http';
import { forkJoin, of, timer } from 'rxjs';
import { retry, timeout, catchError, map, startWith } from 'rxjs/operators';

interface User { id: number; name: string; }
interface Post { id: number; title: string; }
interface Comment { id: number; body: string; }

type State =
  | { status: 'loading' }
  | { status: 'error'; error: string }
  | { status: 'ready'; user: User; posts: Post[]; comments: Comment[] };

@Component({
  selector: 'app-dashboard',
  template: `
    @switch (state().status) {
      @case ('loading') { <p>Yuklanmoqda...</p> }
      @case ('error') { <p class="err">Xato: {{ errorMsg() }}</p> }
      @case ('ready') {
        <h2>{{ user()?.name }}</h2>
        <p>{{ postCount() }} post, {{ commentCount() }} izoh</p>
      }
    }
  `,
})
export class Dashboard {
  private http = inject(HttpClient);
  private id = 1;

  private state$ = forkJoin({
    user: this.http.get<User>(`/api/user/${this.id}`),
    posts: this.http.get<Post[]>(`/api/user/${this.id}/posts`),
    comments: this.http.get<Comment[]>(`/api/user/${this.id}/comments`),
  }).pipe(
    timeout(8000),                                    // 8s limit
    retry({ count: 2, delay: () => timer(1000) }),     // 2 marta, 1s oraliq
    map((data) => ({ status: 'ready' as const, ...data })),
    catchError((err) =>
      of({ status: 'error' as const, error: err?.message ?? 'Nomaʼlum xato' })
    ),
    startWith({ status: 'loading' as const }),         // dastlab loading
  );

  state = toSignal(this.state$, { requireSync: true }); // startWith → sinxron

  // Derived signals — 'ready' holatida ma'lumotni ajratish
  user = computed(() => {
    const s = this.state();
    return s.status === 'ready' ? s.user : undefined;
  });
  postCount = computed(() => {
    const s = this.state();
    return s.status === 'ready' ? s.posts.length : 0;
  });
  commentCount = computed(() => {
    const s = this.state();
    return s.status === 'ready' ? s.comments.length : 0;
  });
  errorMsg = computed(() => {
    const s = this.state();
    return s.status === 'error' ? s.error : '';
  });
}
```

**Tushuntirish:** `forkJoin` uch so'rovni parallel yuklaydi va hammasi complete
bo'lgach bitta obyekt beradi (`Promise.all` semantikasi). `timeout(8000)` sekin
javobda xato otadi; `retry({ count: 2, delay })` xatoda 1s kutib qayta urinadi.
`catchError` xatoni `error` state'ga aylantiradi — oqim o'lmaydi. `startWith`
dastlabki `loading` state beradi, shu tufayli `requireSync: true` ishlaydi
(manba sinxron birinchi qiymatni beradi). `toSignal` + `computed` bilan template
signal-native bo'ladi va `async` pipe kerak emas. Diskriminatsiyalangan union
(`State`) type-safe holatlarni beradi.

</details>

---

## Xulosa

- **Observable** — lazy, cancellable, ko'p qiymatli stream; `Promise`dan farqli
  o'laroq vaqt bo'ylab 0..∞ qiymat beradi, `next*(error|complete)?` grammatikasi bilan.
- **Cold vs Hot** — cold har subscribe uchun yangi producer (HTTP, `of`, `interval`),
  hot esa umumiy producer (`Subject`, `fromEvent`, `share`); farq producer qayerda
  yaratilishida.
- **Nega RxJS kerak** — Angular yadrosi (HttpClient, Router, Forms, EventEmitter)
  RxJS ustiga qurilgan; signal state uchun, RxJS esa event oqimi, time va
  cancellation uchun — bir-birini almashtirmaydi.
- **Ichki mexanizm** — Observer → Subscriber (grammatika himoyasi) → Subscription
  daraxti + teardown; operator zanjiri = subscriber zanjiri, qiymat yuqoridan pastga.
- **Creation** — `of`/`from`/`fromEvent`/`interval`/`timer`/`defer`/`EMPTY`/`throwError`;
  `defer` har subscribe uchun yangi qiymat, `fromEvent` hot.
- **pipe()** — funksiya kompozitsiyasi (`fns.reduce`); pipeable operatorlar tree-shakable,
  har biri `(source) => new Observable`.
- **Flattening** — `switchMap` (bekor qiladi, search), `mergeMap` (parallel),
  `concatMap` (navbat), `exhaustMap` (band bo'lsa rad); tanlov concurrency strategiyasida.
- **Filtering/time** — `filter`/`take`/`takeUntil`/`distinctUntilChanged`;
  `debounceTime` (tinch nuqta) va `throttleTime` (muntazam sur'at) — vaqt boshqaruvi.
- **Combination** — `combineLatest` (har o'zgarishda oxirgilar), `forkJoin` (hammasi
  complete'da, `Promise.all`), `zip`/`merge`/`concat`/`withLatestFrom`/`startWith`.
- **Subjects** — `Subject` (multicast), `BehaviorSubject` (joriy qiymat + `.value`),
  `ReplaySubject` (buffer), `AsyncSubject` (oxirgi + complete); ostida `observers` massivi.
- **share/shareReplay** — cold'ni hot'ga, ref-counting bilan; `shareReplay` HTTP
  cache uchun, cheksiz oqimda `refCount: true` shart (aks holda leak).
- **Error handling** — `catchError` (inner ichida!), `retry({count, delay})` backoff,
  `finalize` (loader), `timeout`, `EMPTY` fallback; xato terminal.
- **async pipe** — impure pipe, `markForCheck` orqali OnPush/zoneless bilan ishlaydi,
  destroy'da avtomatik unsubscribe; bir obs$ni bir marta subscribe qiling.
- **takeUntilDestroyed** — `DestroyRef`ga bog'lanadi; injection context'da argumentsiz,
  aks holda `takeUntilDestroyed(destroyRef)`; qo'lda subscription leak'ini yopadi.
- **Interop** — `toSignal` (Observable → signal, auto-subscribe), `toObservable`
  (signal → Observable, `effect` ostida); chegarada ulab, state signal / oqim RxJS.

---

**Keyingi bo'lim:** [09-routing.md](09-routing.md) — Angular Router internals:
router konfiguratsiyasi va `provideRouter`, `Routes` daraxti, `RouterOutlet`,
navigatsiya lifecycle'i (`NavigationStart`→`NavigationEnd`), functional guards
(`CanActivateFn`, `CanMatchFn`), resolverlar, lazy loading (`loadComponent`/`loadChildren`),
`ActivatedRoute` va paramlar, route reuse strategy.

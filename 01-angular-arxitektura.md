# Bo'lim 1: Angular nima va qanday ishlaydi

> Angular — bu shunchaki "UI library" emas, balki **opinionated platforma**:
> komponent modeli, dependency injection, reaktivlik (signals), router, forms
> va compiler bir butun sifatida birga keladi. Bu bo'lim Angular ilovasi
> `main.ts` dan brauzerdagi DOM'gacha qanday yo'l bosib o'tishini — komponent
> nima, template nimaga compile bo'ladi, ilova qanday bootstrap bo'ladi,
> dependency injection va change detection qanday ishlashini **kaput ostidan**
> (under the hood) ko'rsatadi.
>
> Boshqa bo'limlar bitta mavzuni chuqur qaziydi; bu bo'lim esa **panoramik
> xarita**: har bir asosiy mexanizm (component, Ivy compiler, DI, change
> detection, signals, control flow, rendering) bir-biriga qanday ulanishini
> ko'rsatadi, keyingi QISM'larda har biri alohida chuqurlashtiriladi.

---

## Mundarija
- [Angular nima va nega kerak](#angular-nima-va-nega-kerak)
- [Angular qanaqa paketlardan tashkil topgan](#angular-qanaqa-paketlardan-tashkil-topgan)
- [Component: Angular ilovasining atomi](#component-angular-ilovasining-atomi)
- [Template va data binding turlari](#template-va-data-binding-turlari)
- [Bootstrap jarayoni: ilova qanday ishga tushadi](#bootstrap-jarayoni-ilova-qanday-ishga-tushadi)
- [Standalone arxitektura](#standalone-arxitektura)
- [Dependency Injection asoslari](#dependency-injection-asoslari)
- [Ivy compiler: AOT va JIT](#ivy-compiler-aot-va-jit)
- [Change Detection: view qanday yangilanadi](#change-detection-view-qanday-yangilanadi)
- [Signals: zamonaviy reaktivlik](#signals-zamonaviy-reaktivlik)
- [Yangi control flow: @if, @for, @switch, @defer](#yangi-control-flow-if-for-switch-defer)
- [Rendering pipeline va ilova hayot tsikli](#rendering-pipeline-va-ilova-hayot-tsikli)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Angular nima va nega kerak

### Nazariya

Angular — Google tomonidan saqlanadigan, **scalable** web ilovalar qurish uchun
to'liq freymvork (framework). Uni React yoki Vue kabi "kutubxona" (library)
deb atash to'liq to'g'ri emas: React faqat view qatlamini beradi, qolganini
(router, forms, HTTP, DI, build) o'zing yig'asan. Angular esa **batareyalari
ichida** (batteries-included) keladi — komponent modeli, dependency injection,
router (`@angular/router`), forms (`@angular/forms`), HTTP client
(`@angular/common/http`), SSR (`@angular/ssr`) va compiler bir ekotizim
sifatida birga ishlaydi.

Angular **nega** kerak degan savolga javob — u **katta jamoalar uchun
arxitekturani standartlashtiradi**:

- **Bir xil struktura.** Har bir Angular loyihasi bir xil ko'rinadi:
  component + template + DI. Yangi developer loyihaga tez kiradi.
- **TypeScript birinchi.** Angular TypeScript'ni asos qilib qurilgan — har
  bir API kuchli tiplangan, compiler template'lardagi xatolarni ham topadi
  (type-checking of templates).
- **Compile-time optimizatsiya.** Angular'ning **Ivy** compileri template'ni
  oddiy matn sifatida emas, balki ishga tushadigan JavaScript funksiyaga
  aylantiradi va ishlatilmayotgan kodni tashlab yuboradi (tree-shaking).
- **Reaktivlik built-in.** Angular 16+ dan boshlab **signals** — yengil,
  fine-grained reaktivlik modeli framework yadrosiga kiritilgan.

Bugungi (eng so'nggi stable, Angular 20+/v21+) Angular **standalone** va
**signals**'ga asoslangan: NgModule kerak emas, change detection Zone.js'siz
(**zoneless**) ishlay oladi.

> **Termin aniqligi:** "Framework vs library" farqi — *Inversion of Control*'da.
> Library'da sening koding library'ni chaqiradi; framework'da framework
> sening kodingni chaqiradi (lifecycle hooks, change detection, DI). Angular —
> framework, chunki u sening komponentlaringni qachon yaratish/yangilash/yo'q
> qilishni o'zi boshqaradi.

<details>
<summary><strong>Under the Hood</strong></summary>

Angular runtime'ining yuragi `@angular/core` paketidir. Ilova ishga
tushganda quyidagi obyektlar grafi quriladi:

- **`PlatformRef`** — brauzer platformasi darajasidagi yagona obyekt.
  Bitta sahifada bitta platforma bo'ladi.
- **`ApplicationRef`** — ishga tushgan ilovaning ildiz obyekti. U barcha
  "attached" view'lar ro'yxatini saqlaydi va change detection'ni
  (`ApplicationRef.tick()`) boshqaradi.
- **`EnvironmentInjector`** (ichki implementatsiyasi `R3Injector`) — ilova
  darajasidagi dependency injection konteyneri. Barcha `providedIn: 'root'`
  servislar shu yerda yashaydi.
- **Komponent daraxti** — ildiz komponentdan boshlanadi; har bir komponent
  instance'i runtime'da **`LView`** (Logical View) massivi sifatida, uning
  static "qolipi" esa **`TView`** (Template View) sifatida saqlanadi.

Angular "framework" bo'lgani uchun bu grafni **u o'zi quradi va boshqaradi**:
sen faqat `@Component` deklaratsiyasini yozasan, qolganini compiler + runtime
bajaradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// hello.ts — eng kichik to'liq Angular komponenti (standalone, default)
import { Component } from '@angular/core';

@Component({
  selector: 'app-hello',
  // template inline — alohida .html ham bo'lishi mumkin (templateUrl)
  template: `<h1>Salom, Angular!</h1>`,
})
export class Hello {}
```

```ts
// main.ts — ilovani ishga tushirish (bootstrap)
import { bootstrapApplication } from '@angular/platform-browser';
import { Hello } from './hello';

// Hech qanday NgModule yo'q — to'g'ridan-to'g'ri komponentni bootstrap qilamiz
bootstrapApplication(Hello).catch((err) => console.error(err));
```

```html
<!-- index.html — Angular shu element ichiga ilovani "joylaydi" -->
<body>
  <app-hello></app-hello>
</body>
```

Yuqoridagi uch fayl — to'liq ishlaydigan Angular ilovasi. `bootstrapApplication`
`<app-hello>` selektorini topib, uning ichiga komponent template'ini render
qiladi.

</details>

---

## Angular qanaqa paketlardan tashkil topgan

### Nazariya

Angular monolit emas — bir nechta `@angular/*` npm paketlaridan iborat. Har
biri aniq vazifa bajaradi va faqat keragini import qilasan (modularity +
tree-shaking). Asosiylari:

| Paket | Vazifasi |
|-------|----------|
| `@angular/core` | Component, DI, signals, change detection, lifecycle — yadro |
| `@angular/common` | `NgClass`, `DatePipe`, `@if`/`@for` runtime, `HttpClient` (`@angular/common/http`) |
| `@angular/compiler` | Ivy compiler — template'ni JS instruction'larga aylantiradi |
| `@angular/platform-browser` | Brauzerda render qilish, `bootstrapApplication` |
| `@angular/router` | Client-side routing, lazy loading, guards |
| `@angular/forms` | Reactive va template-driven forms |
| `@angular/ssr` | Server-side rendering, hydration |

Muhim nuqta: **`@angular/compiler` production bundle'ga (deyarli) tushmaydi.**
Sababi — **AOT (Ahead-of-Time)** compilation: template'lar build vaqtida JS'ga
o'giriladi, shuning uchun brauzerda compiler kerak emas. Bu bundle hajmini
keskin kichraytiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

Angular paketlari ikki "olamga" bo'linadi:

1. **Build-time (compiler):** `@angular/compiler` va `@angular/compiler-cli`
   (`ngtsc` — Angular'ning TypeScript compiler plugini). Bular `.ts`
   fayllaringizni va `@Component` dekoratorlarini o'qib, **`ComponentDef`**
   (`ɵcmp`) static maydonlarini va template funksiyalarini generatsiya
   qiladi. Bu jarayon `ng build` paytida bo'ladi.

2. **Runtime:** `@angular/core`'dagi `ɵɵ`-prefiksli **instruction'lar**
   (`ɵɵelementStart`, `ɵɵtext`, `ɵɵproperty`, `ɵɵadvance` ...). Generatsiya
   qilingan template funksiyalari aynan shu instruction'larni chaqiradi.
   `ɵ` (theta) prefiksi — "private/internal" degani; bu API'larni qo'lda
   yozmaysan, ularni compiler yozadi.

Tree-shaking shu yerda ishlaydi: agar ilovangiz `ɵɵstyleProp` instruction'ini
hech qayerda ishlatmasa (style binding yo'q bo'lsa), u bundle'ga umuman
tushmaydi. Ivy'ning "locality" prinsipi — har bir komponent o'ziga kerakli
instruction'lardangina foydalanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// Har bir feature alohida paketdan keladi — faqat keragini import qilasan
import { Component, signal, inject } from '@angular/core';
import { DatePipe } from '@angular/common';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-clock',
  imports: [DatePipe], // pipe ni shu komponentga "olib kiramiz"
  template: `<p>Hozir: {{ now() | date:'medium' }}</p>`,
})
export class Clock {
  private http = inject(HttpClient); // DI orqali HttpClient
  now = signal(new Date());
}
```

```ts
// package.json (qism) — paketlar versiyasi odatda bir xil bo'ladi
{
  "dependencies": {
    "@angular/core": "^21.0.0",
    "@angular/common": "^21.0.0",
    "@angular/platform-browser": "^21.0.0",
    "@angular/router": "^21.0.0"
  }
}
```

</details>

---

## Component: Angular ilovasining atomi

### Nazariya

Component — Angular ilovasining eng kichik qurilish bloki. Har bir component
uch narsani birlashtiradi:

1. **TypeScript class** — holat (state) va xulq-atvor (methods).
2. **Template** (HTML) — nima ko'rsatilishi.
3. **Styles** (CSS) — qanday ko'rinishi (encapsulated).

Bularni `@Component` dekoratori bog'laydi:

```ts
@Component({
  selector: 'app-counter',   // HTML tag nomi
  template: `...`,           // yoki templateUrl: './counter.html'
  styles: `...`,             // yoki styleUrl/styleUrls
})
export class Counter { /* class logikasi */ }
```

Angular 19+ dan boshlab **komponentlar default holatda `standalone`** —
`standalone: true` yozish shart emas, NgModule kerak emas. Boshqa komponent,
directive yoki pipe'ni ishlatish uchun ularni `imports` massiviga qo'shasan.

> **Legacy callout:** Eski kodda komponent `@NgModule`'ning `declarations`
> massiviga yozilardi va `standalone: false` (yoki `standalone` umuman
> yo'q) bo'lardi. Bu uslub Angular 14 gacha yagona yo'l edi va hozir
> **legacy** hisoblanadi. Yangi kodda standalone'dan foydalan.

<details>
<summary><strong>Under the Hood</strong></summary>

`@Component` dekoratori — bu **runtime'da hech narsa qilmaydigan marker**.
Asl ishni Ivy compiler bajaradi: u har bir `@Component` class'iga
**`ɵcmp`** nomli static maydon qo'shadi. Bu maydon — **`ComponentDef`**
obyekti. Uning ichida:

- `type` — komponent class'iga havola.
- `selectors` — `[['app-counter']]` ko'rinishidagi parse qilingan selektorlar.
- `decls` — template'da nechta "deklaratsiya" (DOM node, text) borligi.
- `vars` — binding'lar uchun zarur "slot" soni (change detection xotirasi).
- `template` — `(rf, ctx) => {...}` ko'rinishidagi **template funksiyasi**.
- `consts` — static atributlar/qiymatlar massivi.
- `inputs` / `outputs` — `input()`/`output()` metadata.
- `standalone` — `true`.
- `features`, `encapsulation`, `changeDetection` — qo'shimcha sozlamalar.

Runtime komponent instance'ini yaratganda `ɵcmp.template` funksiyasini
chaqiradi va natijani `LView` massiviga yozadi. Ya'ni: **dekorator =
metadata, `ɵcmp` = haqiqiy "ishlaydigan" ta'rif.**

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

Quyidagi komponent:

```ts
@Component({
  selector: 'app-greeting',
  template: `<h1>Salom {{ name }}</h1>`,
})
export class Greeting {
  name = 'Ali';
}
```

Ivy compiler tomonidan taxminan shunga o'giriladi (soddalashtirilgan,
aniq chiqindi versiyaga qarab biroz farq qiladi):

```js
export class Greeting {
  constructor() { this.name = 'Ali'; }
}

// Compiler qo'shgan static maydon — ComponentDef
Greeting.ɵfac = function Greeting_Factory(t) {
  return new (t || Greeting)();
};

Greeting.ɵcmp = ɵɵdefineComponent({
  type: Greeting,
  selectors: [['app-greeting']],
  standalone: true,
  decls: 2,   // <h1> elementi + uning ichidagi text node
  vars: 1,    // {{ name }} uchun bitta binding slot
  template: function Greeting_Template(rf, ctx) {
    if (rf & 1) {                 // RenderFlags.Create — bir marta ishlaydi
      ɵɵelementStart(0, 'h1');
      ɵɵtext(1);
      ɵɵelementEnd();
    }
    if (rf & 2) {                 // RenderFlags.Update — har CD'da ishlaydi
      ɵɵadvance(1);               // 1-slotga (text node) o't
      ɵɵtextInterpolate1('Salom ', ctx.name, '');
    }
  },
});
```

E'tibor ber: template ikki rejimda ishlaydi — **create** (`rf & 1`, DOM'ni
bir marta quradi) va **update** (`rf & 2`, har change detection'da binding'ni
yangilaydi). Bu Angular'ning eng muhim ichki g'oyalaridan biri.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// counter.ts — to'liq, ishlaydigan standalone komponent (signals bilan)
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <button (click)="decrement()">−</button>
    <span>{{ count() }}</span>
    <button (click)="increment()">+</button>
  `,
  styles: `
    span { font-weight: bold; margin: 0 8px; }
  `,
})
export class Counter {
  count = signal(0);

  increment(): void {
    this.count.update((c) => c + 1);
  }

  decrement(): void {
    this.count.update((c) => c - 1);
  }
}
```

```ts
// parent.ts — boshqa komponentni imports orqali ishlatish
import { Component } from '@angular/core';
import { Counter } from './counter';

@Component({
  selector: 'app-parent',
  imports: [Counter], // Counter'ni shu template'da ishlatish uchun
  template: `
    <h2>Mening hisoblagichim</h2>
    <app-counter />
  `,
})
export class Parent {}
```

</details>

---

## Template va data binding turlari

### Nazariya

Template — komponent class'i bilan DOM o'rtasidagi **deklarativ ko'prik**.
Angular'da to'rt asosiy binding turi bor:

1. **Interpolation** — `{{ expression }}` — qiymatni text sifatida chiqaradi.
2. **Property binding** — `[prop]="expr"` — DOM yoki komponent property'siga
   qiymat uzatadi (masalan `[disabled]="isBusy()"`).
3. **Event binding** — `(event)="handler()"` — DOM yoki komponent
   event'iga obuna bo'ladi (masalan `(click)="save()"`).
4. **Two-way binding** — `[(ngModel)]="value"` yoki signal `model()` bilan
   `[(value)]` — property + event birikmasi ("banana in a box").

Muhim farq: **attribute** (HTML matnidagi statik qiymat) va **property** (DOM
obyektidagi jonli xossa) — bir narsa emas. `[disabled]="false"` property'ni
`false` qiladi (tugma faol), lekin oddiy `disabled="false"` HTML atributi
hamon tugmani o'chirib qo'yadi, chunki attribute mavjudligining o'zi muhim.
Angular `[ ]` bilan **property**'ni boshqaradi.

<details>
<summary><strong>Under the Hood</strong></summary>

Har bir binding turi alohida Ivy instruction'iga compile bo'ladi:

| Template | Instruction |
|----------|-------------|
| `{{ x }}` (text) | `ɵɵtextInterpolate(x)` / `ɵɵtextInterpolate1(...)` |
| `[prop]="x"` | `ɵɵproperty('prop', x)` |
| `[attr.aria-label]="x"` | `ɵɵattribute('aria-label', x)` |
| `[class.active]="x"` | `ɵɵclassProp('active', x)` |
| `[style.color]="x"` | `ɵɵstyleProp('color', x)` |
| `(click)="f()"` | `ɵɵlistener('click', () => f())` |

`ɵɵadvance(n)` instruction'i muhim: u change detection paytida "kursor"ni
keyingi binding slot'iga suradi. Angular har bir binding qiymatini `LView`
massivining ma'lum indeksida saqlaydi va **oldingi qiymat bilan solishtiradi**
(`!==`). Agar o'zgargan bo'lsa — DOM'ni yangilaydi, bo'lmasa — tegmaydi. Bu
"dirty checking"ning aniq joyi.

Event binding'da `ɵɵlistener` haqiqiy DOM `addEventListener`'ni o'rab oladi.
Zone.js bo'lganda bu listener change detection'ni avtomatik trigger qilardi;
zoneless rejimda esa Angular handler ichida holat o'zgarganini boshqacha
(masalan signal yoki `markForCheck` orqali) biladi.

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```ts
@Component({
  selector: 'app-toggle',
  template: `
    <button [disabled]="busy()" (click)="run()">
      {{ label() }}
    </button>
  `,
})
export class Toggle {
  busy = signal(false);
  label = signal('Boshlash');
  run() { /* ... */ }
}
```

Template funksiyasi taxminan:

```js
template: function Toggle_Template(rf, ctx) {
  if (rf & 1) {
    ɵɵelementStart(0, 'button');
    ɵɵlistener('click', function () { return ctx.run(); });
    ɵɵtext(1);
    ɵɵelementEnd();
  }
  if (rf & 2) {
    ɵɵproperty('disabled', ctx.busy());        // property binding
    ɵɵadvance(1);
    ɵɵtextInterpolate1(' ', ctx.label(), ' '); // interpolation
  }
}
```

Diqqat: `ctx.busy()` va `ctx.label()` — bular **signal'larni o'qish**.
Template ichida signal o'qilganda Angular bu komponentni o'sha signal'ning
"consumer"i sifatida ro'yxatga oladi — signal o'zgarsa, komponent
yangilanishi kerakligini biladi (signal ↔ change detection bog'liqligi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// form-field.ts — barcha binding turlari bir joyda
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-form-field',
  template: `
    <!-- Interpolation -->
    <label>{{ labelText() }}</label>

    <!-- Property binding + event binding -->
    <input
      [value]="value()"
      [class.invalid]="!isValid()"
      (input)="onInput($event)"
    />

    <!-- Attribute binding (property emas!) -->
    <button [attr.aria-label]="labelText()" (click)="clear()">
      Tozalash
    </button>

    <p>Uzunligi: {{ value().length }}</p>
  `,
})
export class FormField {
  labelText = signal('Ismingiz');
  value = signal('');

  isValid(): boolean {
    return this.value().length >= 3;
  }

  onInput(event: Event): void {
    const input = event.target as HTMLInputElement;
    this.value.set(input.value);
  }

  clear(): void {
    this.value.set('');
  }
}
```

</details>

---

## Bootstrap jarayoni: ilova qanday ishga tushadi

### Nazariya

Angular ilovasi `main.ts` dan boshlanadi. Zamonaviy (standalone) yondashuvda
bu bitta funksiya chaqiruvi:

```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { App } from './app/app';
import { appConfig } from './app/app.config';

bootstrapApplication(App, appConfig)
  .catch((err) => console.error(err));
```

Bu yerda:

- `App` — **ildiz komponent** (root component). Uning selektori (`app-root`)
  `index.html`'da turadi.
- `appConfig` — `ApplicationConfig` tipidagi obyekt; ilova darajasidagi
  barcha `providers`'ni (router, HTTP, zoneless CD ...) yig'adi.

`app.config.ts` odatda shunday:

```ts
// app.config.ts
import {
  ApplicationConfig,
  provideZonelessChangeDetection,
  provideBrowserGlobalErrorListeners,
} from '@angular/core';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideZonelessChangeDetection(),
    provideRouter(routes),
  ],
};
```

> **Legacy callout:** Eski (NgModule) ilovalarda bootstrap shunday edi:
> `platformBrowserDynamic().bootstrapModule(AppModule)`, va providerlar
> `AppModule`'ning `@NgModule({ providers: [...] })` ichida turardi. Yangi
> kodda `bootstrapApplication` + `ApplicationConfig` ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`bootstrapApplication` ichida quyidagi bosqichlar bajariladi:

1. **Platforma yaratiladi.** `platformBrowser()` orqali `PlatformRef`
   olinadi (agar mavjud bo'lmasa). Bu bir sahifada bitta marta bo'ladi.
2. **EnvironmentInjector quriladi.** `appConfig.providers` + Angular'ning
   default providerlari (`provideZonelessChangeDetection` yo'q bo'lsa Zone.js
   asosidagi CD provideri) yig'ilib, **`R3Injector`** (EnvironmentInjector
   implementatsiyasi) yaratiladi. Bu — `providedIn: 'root'` servislar yashaydigan
   joy.
3. **`ApplicationRef` olinadi** shu injektordan.
4. **Ildiz komponent bootstrap qilinadi.** `ApplicationRef.bootstrap(App)`
   `App.ɵcmp` (ComponentDef)'ni topadi, `app-root` selektorini DOM'da qidiradi,
   uning uchun **ildiz `LView`** yaratadi va birinchi change detection'ni
   ishga tushiradi (birinchi render).
5. Ildiz view `ApplicationRef`'ning "attached views" ro'yxatiga qo'shiladi —
   keyingi `tick()`'larda u tekshiriladi.

`bootstrapApplication` `Promise<ApplicationRef>` qaytaradi — shuning uchun
`.catch(...)` bilan bootstrap xatolarini ushlash mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// app.ts — ildiz komponent
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet],
  template: `
    <header><h1>Mening ilovam</h1></header>
    <router-outlet />
  `,
})
export class App {}
```

```ts
// app.routes.ts
import { Routes } from '@angular/router';
import { Home } from './home/home';

export const routes: Routes = [
  { path: '', component: Home },
  {
    path: 'about',
    // lazy loading — alohida chunk sifatida yuklanadi
    loadComponent: () => import('./about/about').then((m) => m.About),
  },
];
```

```ts
// main.ts — yakuniy ishga tushirish
import { bootstrapApplication } from '@angular/platform-browser';
import { App } from './app/app';
import { appConfig } from './app/app.config';

bootstrapApplication(App, appConfig)
  .then(() => console.log('Ilova ishga tushdi'))
  .catch((err) => console.error('Bootstrap xatosi:', err));
```

</details>

---

## Standalone arxitektura

### Nazariya

**Standalone** — komponent, directive yoki pipe'ning o'zi-o'zicha, NgModule'siz
yashashi. Bog'liqliklarni `imports` massivida e'lon qiladi:

```ts
@Component({
  selector: 'app-dashboard',
  imports: [Counter, DatePipe, RouterLink], // hammasi shu yerda
  template: `...`,
})
export class Dashboard {}
```

Bu — Angular'ning eng katta arxitektura o'zgarishlaridan biri. Avvalroq har
bir narsa qaysidir `NgModule`'ga tegishli bo'lishi va u modulning
`declarations`/`exports`/`imports`'i orqali ulanishi shart edi. Bu ko'p
boilerplate va "qaysi modulda nima bor" muammosini keltirardi.

Standalone afzalliklari:

- **Kamroq boilerplate** — `app.module.ts`, `shared.module.ts` kabi fayllar
  yo'qoladi.
- **Aniq bog'liqlik** — komponent nimaga muhtojligini to'g'ridan-to'g'ri
  o'zida ko'rasan.
- **Yaxshiroq lazy loading** — `loadComponent` bilan bitta komponentni lazy
  yuklash mumkin, butun modulni emas.

> **Legacy callout:** NgModule (`@NgModule`) hali ham qo'llab-quvvatlanadi
> (ko'p eski loyihalar va kutubxonalar uni ishlatadi), lekin **yangi kod uchun
> tavsiya etilmaydi**. `CommonModule`'ni import qilish o'rniga endi `@if`/`@for`
> to'g'ridan-to'g'ri ishlaydi; `RouterModule` o'rniga `provideRouter` ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Standalone komponentning `ɵcmp` (ComponentDef)'ida `standalone: true` flagi
bo'ladi va u o'zining **`imports`**'ini metadata sifatida saqlaydi.
Kompilyator komponentni "compile" qilganda, `imports`'dagi har bir
komponent/directive/pipe'ning selektorlarini ushbu komponent template'ining
**scope**'iga qo'shadi. Ya'ni template ichida qaysi selektorlar (`<app-counter>`)
yoki pipe'lar (`| date`) "ko'rinishi" aynan shu `imports` bilan aniqlanadi —
global emas, **lokal** (locality prinsipi).

DI nuqtai nazaridan: standalone komponent o'zining `providers`'ini ham e'lon
qila oladi (`@Component({ providers: [...] })`), bu uning **NodeInjector**'iga
yoziladi. `bootstrapApplication`'ga uzatilgan providerlar esa
**EnvironmentInjector**'ga boradi. Bu ikki injector ierarxiyasi keyingi
"Dependency Injection" bo'limida ochiladi.

`importProvidersFrom()` — eski NgModule'lardan providerlarni standalone
dunyosiga "ko'prik" qilish uchun ishlatiladi (masalan, faqat NgModule sifatida
keladigan kutubxona bilan).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// Zamonaviy: standalone (tavsiya etiladi)
import { Component } from '@angular/core';
import { DatePipe } from '@angular/common';
import { Counter } from './counter';

@Component({
  selector: 'app-dashboard',
  imports: [DatePipe, Counter],
  template: `
    <p>Sana: {{ today | date }}</p>
    <app-counter />
  `,
})
export class Dashboard {
  today = new Date();
}
```

```ts
// LEGACY (faqat solishtirish uchun — eski kodda uchraydi)
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { CounterComponent } from './counter.component';
import { DashboardComponent } from './dashboard.component';

@NgModule({
  declarations: [DashboardComponent, CounterComponent], // declarations kerak
  imports: [CommonModule],                              // DatePipe shu yerdan
  exports: [DashboardComponent],
})
export class DashboardModule {}
```

</details>

---

## Dependency Injection asoslari

### Nazariya

**Dependency Injection (DI)** — Angular'ning markaziy patterni. Komponent yoki
servis o'ziga kerakli bog'liqlikni (masalan `HttpClient`) **o'zi yaratmaydi**,
balki Angular'dan "so'raydi". Angular esa kerakli instance'ni topib (yoki
yaratib) beradi. Bu — *Inversion of Control*.

Zamonaviy yo'l — `inject()` funksiyasi:

```ts
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({ /* ... */ })
export class UserList {
  private http = inject(HttpClient); // DI shu yerda ishlaydi
}
```

Servis yaratish:

```ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' }) // butun ilova uchun bitta instance (singleton)
export class UserService {
  getUsers() { /* ... */ }
}
```

`providedIn: 'root'` — eng ko'p ishlatiladigan variant: servis **tree-shakable**
(ishlatilmasa bundle'ga tushmaydi) va butun ilova uchun yagona (singleton).

> **Legacy callout:** Avval bog'liqliklar **constructor injection** orqali
> olinardi: `constructor(private http: HttpClient) {}`. Bu hali ishlaydi, lekin
> `inject()` yanada moslashuvchan (masalan, field initializer'da, funksiya
> ichida ishlatish mumkin) va yangi kodda afzal ko'riladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Angular'da **ikki xil injector ierarxiyasi** bor:

1. **EnvironmentInjector** (`R3Injector`) — ilova/modul darajasi.
   `providedIn: 'root'`, `bootstrapApplication`'ga uzatilgan providerlar va
   route-level providerlar shu yerda. Ular daraxt hosil qiladi: root →
   lazy-loaded route injectorlari.
2. **NodeInjector** (ElementInjector) — DOM element/komponent darajasi.
   `@Component({ providers: [...] })` shu yerda. Har bir komponent
   instance'ining o'z NodeInjector'i bor va u DOM daraxti bo'ylab yuqoriga
   ko'tariladi.

`inject(Token)` chaqirilganda **resolution algoritmi**:

1. Avval joriy **NodeInjector**'dan boshlanadi va element daraxti bo'ylab
   yuqoriga (parent komponentlar tomon) qidiradi.
2. NodeInjector'lar tugaganda, **EnvironmentInjector** ierarxiyasiga o'tadi
   (joriy → parent → root).
3. Topilsa — instance qaytariladi (kerak bo'lsa lazy yaratiladi va
   keshlanadi). Topilmasa — `NullInjectorError` tashlanadi.

`inject()` faqat **injection context** ichida ishlaydi: konstruktorda, field
initializer'da, `@Injectable`/`@Component` provider'lar yaratilayotganda yoki
`runInInjectionContext()` ichida. Aks holda xato beradi.

`providedIn: 'root'`ning tree-shakable bo'lishi shundan: servis o'zining
`ɵprov` (ProviderDef)'ida qaysi injector'ga tegishliligini saqlaydi —
"top-down" emas, "bottom-up" deklaratsiya. Hech kim `inject(UserService)`
qilmasa, bundler uni o'chirib tashlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// logger.ts — servis (singleton, root)
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class Logger {
  log(message: string): void {
    console.log(`[${new Date().toISOString()}] ${message}`);
  }
}
```

```ts
// user.service.ts — boshqa servisga bog'liq servis
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Logger } from './logger';

@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private logger = inject(Logger);

  getUsers() {
    this.logger.log('Foydalanuvchilar so\'ralmoqda');
    return this.http.get<{ id: number; name: string }[]>('/api/users');
  }
}
```

```ts
// user-list.ts — komponentda DI ishlatish
import { Component, inject } from '@angular/core';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-list',
  template: `<button (click)="load()">Yuklash</button>`,
})
export class UserList {
  private users = inject(UserService);

  load(): void {
    this.users.getUsers().subscribe((list) => console.log(list));
  }
}
```

</details>

---

## Ivy compiler: AOT va JIT

### Nazariya

**Ivy** — Angular 9+ dan beri default bo'lgan compiler va runtime arxitekturasi.
U `@Component`/`@Directive`/`@Pipe` dekoratorlarini va HTML template'larni
**ishga tushadigan JavaScript'ga** aylantiradi.

Ikki compilation rejimi bor:

- **AOT (Ahead-of-Time)** — **default va production uchun yagona to'g'ri yo'l**.
  Template'lar **build vaqtida** JS'ga o'giriladi. Natija:
  - Brauzerga `@angular/compiler` jo'natilmaydi → kichik bundle.
  - Tezroq birinchi render (compile qilish kerak emas).
  - **Template type-checking** — template'dagi xatolar build'da topiladi.
- **JIT (Just-in-Time)** — template'lar **brauzerda, runtime'da** compile
  qilinadi. Bundle'ga compiler kiradi, sekinroq. Bugun deyarli faqat ba'zi
  test stsenariylarida uchraydi.

> **Tarixiy izoh:** Ivy'dan oldin **View Engine** bor edi. Ivy uni
> "locality" (har komponent mustaqil compile bo'ladi), tree-shaking va kichik
> bundle bilan almashtirdi.

<details>
<summary><strong>Under the Hood</strong></summary>

Ivy compilation'ning yuragi — **`ngtsc`** (Angular TypeScript Compiler):
TypeScript'ning `tsc` compileriga ulanadigan **plugin** (transformer). U
quyidagini qiladi:

1. `.ts` fayllardagi `@Component` kabi dekoratorlarni topadi.
2. `templateUrl`/`template`'ni o'qib, HTML'ni **AST**'ga parse qiladi.
3. AST'ni Ivy **instruction'lariga** (`ɵɵelementStart`, `ɵɵtext`, ...) o'giradi.
4. Class'ga `ɵcmp` (ComponentDef), `ɵfac` (factory) static maydonlarni qo'shadi.
5. Template type-checking uchun maxsus "type check block" (TCB) generatsiya
   qiladi — TypeScript template ichidagi tip xatolarini ham tekshiradi.

**Partial compilation va Angular Linker:** Kutubxonalar (npm paketlar)
"partial" rejimda compile bo'ladi (`ɵɵngDeclareComponent`) — ya'ni Angular
versiyasiga bog'liq bo'lmagan oraliq formatda. Iste'molchi ilova build
qilinganda **Angular Linker** bu partial deklaratsiyalarni o'sha ilovaning
Angular versiyasiga mos to'liq Ivy instruction'lariga "linklaydi". Bu
kutubxonalar turli Angular versiyalari bilan ishlashini ta'minlaydi.

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```ts
// Manba
@Component({
  selector: 'app-badge',
  template: `<span class="badge">{{ text }}</span>`,
})
export class Badge {
  text = 'Yangi';
}
```

AOT chiqindisi (soddalashtirilgan):

```js
export class Badge {
  constructor() { this.text = 'Yangi'; }
}

Badge.ɵfac = function Badge_Factory(t) { return new (t || Badge)(); };

Badge.ɵcmp = ɵɵdefineComponent({
  type: Badge,
  selectors: [['app-badge']],
  standalone: true,
  decls: 2,
  vars: 1,
  consts: [['class', 'badge']],   // static atribut shu yerda
  template: function Badge_Template(rf, ctx) {
    if (rf & 1) {
      ɵɵelementStart(0, 'span', 0); // 0 — consts indeksidagi atribut
      ɵɵtext(1);
      ɵɵelementEnd();
    }
    if (rf & 2) {
      ɵɵadvance(1);
      ɵɵtextInterpolate(ctx.text);
    }
  },
});
```

Bu kod brauzerda hech qanday HTML parse qilmaydi — DOM to'g'ridan-to'g'ri
instruction'lar orqali quriladi. Ana shuning uchun AOT tez.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```jsonc
// angular.json (qism) — production build sozlamalari odatda AOT'ni majburlaydi
{
  "configurations": {
    "production": {
      "optimization": true,
      "outputHashing": "all"
      // Zamonaviy Angular CLI'da AOT har doim yoqilgan (default)
    }
  }
}
```

```bash
# AOT build (default) — compiler bundle'ga tushmaydi
ng build

# Template type-checking xatosini ko'rish uchun:
# Agar template'da mavjud bo'lmagan property ishlatsangiz, build to'xtaydi:
#   error NG2: Property 'titlee' does not exist on type 'Badge'.
```

</details>

---

## Change Detection: view qanday yangilanadi

### Nazariya

**Change Detection (CD)** — komponent holati o'zgarganda Angular DOM'ni qanday
yangilashini boshqaradigan mexanizm. Asosiy g'oya: Angular har bir binding
qiymatini eslab qoladi va keyingi tekshiruvda **eski qiymat bilan solishtiradi**
(`!==`); o'zgargan bo'lsa — o'sha DOM bo'lagini yangilaydi (**dirty checking**).

Asosiy savol — **CD qachon ishga tushadi?** Bu yerda ikki dunyo bor:

1. **Zone.js bilan (an'anaviy):** Zone.js brauzerning barcha async API'larini
   (`setTimeout`, `addEventListener`, `Promise`, XHR ...) "monkey-patch" qiladi.
   Har qanday async vazifa tugaganda Zone Angular'ga "balki nimadir o'zgardi"
   deb signal beradi va Angular butun komponent daraxtini tekshiradi.
2. **Zoneless (zamonaviy, Angular 21+ da default):** Zone.js yo'q. Angular CD'ni
   faqat **aniq signallar** asosida ishga tushiradi: signal o'zgargan va u
   template'da o'qilgan bo'lsa, `markForCheck()` chaqirilsa, `AsyncPipe` yangi
   qiymat olsa, yoki komponent input'i `setInput` orqali o'zgartirilsa.

Zoneless — kelajak: kamroq overhead, aniqroq, kichikroq bundle (Zone.js ~13KB).

> **Termin:** **`OnPush`** — change detection strategiyasi: komponent faqat
> input'lari o'zgarganda yoki signal'i yangilanganda tekshiriladi. Zoneless
> dunyoda signal'lar tabiiy ravishda shunga o'xshash ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

CD'ning kirish nuqtasi — **`ApplicationRef.tick()`**. U barcha "attached
root view"lar bo'ylab yuradi va har biri uchun ichki **`refreshView`**
funksiyasini chaqiradi. `refreshView` `LView` ichidagi template funksiyasini
**update rejimida** (`rf & 2`) ishga tushiradi — bindings'ni qayta hisoblaydi
va o'zgarganlarini DOM'ga yozadi.

Har bir `LView` o'zining **flags**'ini saqlaydi (`LViewFlags`):
- `Dirty` — bu view yangilanishi kerak.
- `CheckAlways` — har tick'da tekshiriladi (default strategiya).
- `Attached` — CD daraxtiga ulangan.

`OnPush` komponent `CheckAlways` o'rniga faqat `Dirty` belgilangani uchun
tekshiriladi. `markForCheck()` — komponentdan ildizgacha bo'lgan yo'ldagi
barcha view'larga `Dirty` flagini qo'yadi.

**Zoneless scheduler:** Zone.js o'rnida Angular **`ChangeDetectionScheduler`**
ishlatadi. Signal o'zgarganda yoki `markForCheck` chaqirilganda scheduler
"notify" oladi va keyingi microtask/`requestAnimationFrame`'da bitta `tick()`
rejalashtiradi (bir nechta o'zgarish bitta tick'ga "coalesce" bo'ladi).

**Signal ↔ CD:** Template ichida `count()` o'qilganda, o'sha komponentning
view'i o'sha signal'ning **consumer**'iga aylanadi. Signal `set`/`update`
qilinganda u barcha consumer view'larni `markForCheck` qiladi — natijada faqat
**kerakli** komponentlar yangilanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// zoneless-counter.ts — signal o'zgarishi avtomatik CD'ni trigger qiladi
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-zoneless-counter',
  template: `
    <p>Qiymat: {{ value() }}</p>
    <button (click)="add()">+1</button>
  `,
})
export class ZonelessCounter {
  value = signal(0);

  add(): void {
    // value() template'da o'qilgani uchun bu o'zgarish CD'ni ishga tushiradi
    this.value.update((v) => v + 1);
  }
}
```

```ts
// app.config.ts — zoneless'ni yoqish (Angular 21+ da default, lekin aniq yozsa ham bo'ladi)
import { ApplicationConfig, provideZonelessChangeDetection } from '@angular/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZonelessChangeDetection(),
  ],
};
```

```ts
// manual-tick.ts — DI orqali ApplicationRef olib, qo'lda tick chaqirish (kamdan-kam)
import { Component, inject, ApplicationRef } from '@angular/core';

@Component({
  selector: 'app-manual',
  template: `<button (click)="force()">Majburiy yangilash</button>`,
})
export class Manual {
  private appRef = inject(ApplicationRef);

  force(): void {
    // Odatda kerak emas — faqat tashqi (Angular bilmaydigan) o'zgarishlarda
    this.appRef.tick();
  }
}
```

</details>

---

## Signals: zamonaviy reaktivlik

### Nazariya

**Signals** (Angular 16+, 17+ da stable) — Angular'ning yangi reaktivlik
modeli. Signal — bu **qiymatni o'rab turuvchi konteyner**, uni o'qib (`count()`)
yoki o'zgartirib (`count.set(5)`, `count.update(...)`) bo'ladi, va u
o'zgarganda bog'liq joylar **avtomatik** xabardor bo'ladi.

Uch asosiy primitiv:

```ts
import { signal, computed, effect } from '@angular/core';

const count = signal(0);                       // yoziladigan signal
const doubled = computed(() => count() * 2);   // hosila (derived), memoized
effect(() => console.log('Qiymat:', count())); // side-effect, avtomatik qayta ishlaydi
```

- **`signal(value)`** — holat manbai. `()` bilan o'qiladi.
- **`computed(fn)`** — boshqa signal'lardan hosil bo'ladigan qiymat. **Lazy**
  (faqat o'qilganda hisoblanadi) va **memoized** (bog'liqliklar o'zgarmasa,
  qayta hisoblanmaydi).
- **`effect(fn)`** — signal'lar o'zgarganda ishlaydigan side-effect (logging,
  localStorage, DOM bilan ishlash).

Qo'shimcha: **`linkedSignal`** (yoziladigan, lekin manba o'zgarsa qayta
hisoblanadigan), **`resource`** (async ma'lumotni signal sifatida olish),
**`input()`/`output()`/`model()`** (signal-based komponent API'lari).

<details>
<summary><strong>Under the Hood</strong></summary>

Signal'lar **producer-consumer grafi** asosida ishlaydi. Har bir signal va
computed — bu **`ReactiveNode`**. Ikki tomonlama bog'liqlik saqlanadi:

- **Producer** (signal/computed) — kim undan o'qiyotganini (consumers) biladi.
- **Consumer** (computed/effect/template) — qaysi producer'lardan o'qiganini
  (producers) biladi.

Har bir node'da **version** raqami bor. Signal `set` qilinganda:
1. Uning `version`'i oshadi.
2. Barcha bog'liq consumer'larga "ifloslanding" (notification) yetkaziladi —
   ammo qiymat **darhol qayta hisoblanmaydi** (push faqat xabar beradi).

Consumer (masalan `computed`) o'qilganda (**pull**), u o'z producer'larining
version'larini eslab qolingan version bilan solishtiradi. Agar birortasi
o'zgargan bo'lsa — qayta hisoblaydi, bo'lmasa — keshlangan qiymatni qaytaradi
(**memoization**).

Bu **push-then-pull** modeli **glitch-free** (nosozliksiz) hisoblashni
ta'minlaydi: `a` ga bog'liq `b` va `c`, ikkalasiga bog'liq `d` bo'lsa, `a`
bir marta o'zgarganda `d` faqat **bir marta** va to'g'ri qiymat bilan
hisoblanadi (oraliq "yarim yangilangan" holatni hech kim ko'rmaydi).

Template ham consumer: Angular template funksiyasini ishlatganda joriy
"active consumer"ni o'rnatadi; template ichida o'qilgan har bir signal o'sha
komponent view'ining producer'iga ulanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// cart.ts — signal, computed va effect birga
import { Component, signal, computed, effect } from '@angular/core';

interface Item { name: string; price: number; qty: number; }

@Component({
  selector: 'app-cart',
  template: `
    <ul>
      @for (item of items(); track item.name) {
        <li>{{ item.name }} — {{ item.price }} x {{ item.qty }}</li>
      }
    </ul>
    <p>Jami: {{ total() }} so'm</p>
    <button (click)="addOne()">Mahsulot qo'shish</button>
  `,
})
export class Cart {
  items = signal<Item[]>([
    { name: 'Olma', price: 1000, qty: 3 },
    { name: 'Non', price: 2000, qty: 2 },
  ]);

  // computed — items o'zgarganda avtomatik qayta hisoblanadi (memoized)
  total = computed(() =>
    this.items().reduce((sum, i) => sum + i.price * i.qty, 0),
  );

  constructor() {
    // effect — total o'zgarganda log qiladi (injection context'da yaratiladi)
    effect(() => console.log('Yangi jami:', this.total()));
  }

  addOne(): void {
    this.items.update((list) => [
      ...list,
      { name: 'Suv', price: 1500, qty: 1 },
    ]);
  }
}
```

```ts
// signal-input.ts — signal-based input/output/model (zamonaviy komponent API)
import { Component, input, output, model } from '@angular/core';

@Component({
  selector: 'app-rating',
  template: `
    <p>Baho: {{ value() }} / {{ max() }}</p>
    <button (click)="up()">Oshirish</button>
  `,
})
export class Rating {
  max = input(5);                    // <app-rating [max]="10" /> — read-only signal
  value = model(0);                  // [(value)]="..." — two-way
  changed = output<number>();        // (changed)="..." — event

  up(): void {
    if (this.value() < this.max()) {
      this.value.update((v) => v + 1);
      this.changed.emit(this.value());
    }
  }
}
```

</details>

---

## Yangi control flow: @if, @for, @switch, @defer

### Nazariya

Angular 17+ template'ga **built-in control flow** kiritdi — `*ngIf`/`*ngFor`
o'rniga to'g'ridan-to'g'ri template sintaksisi:

```html
@if (user(); as u) {
  <p>Salom, {{ u.name }}</p>
} @else {
  <p>Tizimga kiring</p>
}

@for (item of items(); track item.id) {
  <li>{{ item.name }}</li>
} @empty {
  <li>Ro'yxat bo'sh</li>
}

@switch (status()) {
  @case ('loading') { <spinner /> }
  @case ('error')   { <p>Xato</p> }
  @default          { <p>Tayyor</p> }
}
```

Afzalliklar:

- **`CommonModule` import qilish shart emas** — control flow til darajasida.
- **`@for` da `track` majburiy** — bu performance uchun kritik (qaysi DOM
  element qaysi ma'lumotga tegishliligini aniqlaydi).
- **`@defer`** — kontentni **lazy** yuklash (viewport'ga kirganda, idle'da,
  hover'da ...) — bundle'ni bo'lib, birinchi yuklashni tezlashtiradi.

> **Legacy callout:** `*ngIf`, `*ngFor`, `[ngSwitch]` hali ishlaydi, lekin
> ular **structural directive** sifatida `CommonModule`'ni talab qiladi va
> sintaksisi og'irroq. Yangi kodda `@if`/`@for`/`@switch` ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Yangi control flow **directive emas** — u to'g'ridan-to'g'ri compiler darajasida
maxsus Ivy instruction'lariga compile bo'ladi:

- **`@if`/`@switch`** → **`ɵɵconditional`** instruction'i. U qaysi "embedded
  view"ni ko'rsatishni indeks bo'yicha tanlaydi (yoki hech qaysisini).
- **`@for`** → **`ɵɵrepeater`** + `ɵɵrepeaterCreate`. `track` ifodasi esa
  `ɵɵrepeaterTrackByIdentity` yoki `ɵɵrepeaterTrackByIndex` kabi funksiyaga
  o'giriladi. `track` Angular'ga "qaysi item o'zgarmadi"ni aytib, DOM node'larni
  qayta yaratish o'rniga **qayta ishlatishga** imkon beradi (`*ngFor`'dagi
  `trackBy`'ning til ichiga kirgan, majburiy varianti).
- **`@defer`** → **`ɵɵdefer`** va trigger'larga qarab `ɵɵdeferOnIdle`,
  `ɵɵdeferOnViewport`, `ɵɵdeferOnHover` va h.k. Defer bloki ichidagi
  komponentlar alohida bundle chunk'iga ajratiladi va trigger ishga tushganda
  dinamik `import()` orqali yuklanadi.

`@for`'ning `track`'siz ishlamasligi — compiler darajasidagi qoida: `track`
bo'lmasa, har CD'da Angular barcha DOM'ni qayta yaratishi mumkin edi, bu esa
performance falokati. Shuning uchun u **majburiy**.

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```ts
@Component({
  selector: 'app-list',
  template: `
    @if (loaded()) {
      <ul>
        @for (u of users(); track u.id) {
          <li>{{ u.name }}</li>
        }
      </ul>
    }
  `,
})
export class List {
  loaded = signal(true);
  users = signal([{ id: 1, name: 'Ali' }]);
}
```

Soddalashtirilgan chiqindi g'oyasi:

```js
template: function List_Template(rf, ctx) {
  if (rf & 1) {
    // @if uchun conditional slot ochiladi
    ɵɵtemplate(0, List_Conditional_0_Template, /* decls */ 2, /* vars */ 0);
    ɵɵconditionalCreate(0); // (versiyaga qarab nom farq qilishi mumkin)
  }
  if (rf & 2) {
    // loaded() true bo'lsa 0-template ko'rsatiladi, aks holda -1 (hech narsa)
    ɵɵconditional(ctx.loaded() ? 0 : -1);
  }
}
// @for esa ichki template'da ɵɵrepeater(...) + track funksiyasi bilan ishlaydi
```

> **Eslatma:** `@defer`/`@for`/`@if` uchun aniq instruction nomlari Angular
> versiyasiga qarab o'zgarishi mumkin — yuqoridagi g'oyani ko'rsatish uchun.
> Aniqligini bilish uchun `ng build` chiqindisini yoki Angular manba kodini
> ko'rish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// data-view.ts — @if + @for + @switch + @empty birga
import { Component, signal } from '@angular/core';

type Status = 'loading' | 'ready' | 'error';

@Component({
  selector: 'app-data-view',
  template: `
    @switch (status()) {
      @case ('loading') { <p>Yuklanmoqda…</p> }
      @case ('error')   { <p>Xatolik yuz berdi</p> }
      @default {
        @if (items().length > 0) {
          <ul>
            @for (item of items(); track item.id) {
              <li>{{ $index + 1 }}. {{ item.title }}</li>
            }
          </ul>
        } @else {
          <p>Ma'lumot yo'q</p>
        }
      }
    }
  `,
})
export class DataView {
  status = signal<Status>('ready');
  items = signal([
    { id: 1, title: 'Birinchi' },
    { id: 2, title: 'Ikkinchi' },
  ]);
}
```

```html
<!-- defer.html — @defer bilan lazy loading (chart faqat viewport'ga kirganda yuklanadi) -->
@defer (on viewport) {
  <app-heavy-chart />
} @placeholder {
  <p>Diagramma joyi…</p>
} @loading (minimum 500ms) {
  <p>Diagramma yuklanmoqda…</p>
} @error {
  <p>Diagramma yuklanmadi</p>
}
```

</details>

---

## Rendering pipeline va ilova hayot tsikli

### Nazariya

Bootstrap'dan keyin Angular ilovasining "hayoti" quyidagicha kechadi:

1. **Birinchi render (create).** Ildiz komponent template funksiyasi *create*
   rejimida (`rf & 1`) ishlaydi — DOM elementlari yaratiladi, child
   komponentlar rekursiv quriladi.
2. **Birinchi update.** O'sha tick ichida *update* rejimi (`rf & 2`) ishlaydi
   — binding'lar hisoblanadi va DOM'ga yoziladi.
3. **Keyingi yangilanishlar.** Signal o'zgarganda (yoki Zone.js dunyosida async
   vazifa tugaganda) `ApplicationRef.tick()` ishlaydi va faqat update rejimi
   qayta ishlaydi — yangi DOM yaratilmaydi, mavjudi yangilanadi.
4. **Yo'q qilish (destroy).** Komponent DOM'dan olib tashlanganda Angular uning
   `LView`'ini "destroy" qiladi — subscription'lar tozalanadi, `ngOnDestroy`
   /`DestroyRef` callback'lari ishlaydi.

Angular DOM bilan to'g'ridan-to'g'ri emas, **Renderer** (abstraktsiya) orqali
ishlaydi — shuning uchun bir xil kod brauzerda ham, serverda (SSR) ham ishlay
oladi.

**`afterRender` / `afterNextRender`** — DOM yangilangach (render'dan keyin)
ishlaydigan hook'lar; o'lcham olish (measure) yoki tashqi DOM kutubxonalari
bilan ishlash uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

Render'ning markaziy ma'lumot strukturalari:

- **`TView` (Template View)** — template'ning **statik qolipi**. Bir xil
  komponentning barcha instance'lari **bitta** `TView`'ni baham ko'radi:
  unda TNode'lar (static node ma'lumoti), binding metadata, query'lar va h.k.
  Bu — "qolip bir marta, instance ko'p" optimizatsiyasi.
- **`LView` (Logical View)** — har bir instance uchun alohida **massiv**. Unda
  haqiqiy DOM node havolalari, binding qiymatlari (oldingi qiymatlar dirty
  checking uchun), komponent instance'i (`LView[CONTEXT]`), parent/child
  havolalar saqlanadi.

CD daraxti — bu `LView`'lar daraxti. `refreshView(tView, lView, ...)`
rekursiv yuradi: avval host bindings, keyin template (update), keyin child
komponentlar.

**SSR va Hydration:** Serverda Angular DOM o'rniga **HTML matn** generatsiya
qiladi (`@angular/ssr`) va elementlarga **`ngh`** atributlarini qo'shadi —
bular hydration uchun "xarita". Brauzerda ilova qayta ishga tushganda
**hydration** server bergan DOM'ni qaytadan yaratmaydi, balki mavjud DOM'ga
"yopishadi" (event listener'larni ulaydi, `LView`'ni mavjud node'larga
bog'laydi). **Incremental hydration** (Angular 19+) esa `@defer` bloklarini
faqat kerak bo'lganda hydrate qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```ts
// lifecycle.ts — yaratilish va yo'q qilinish hayot tsikli
import { Component, signal, afterNextRender, inject, DestroyRef } from '@angular/core';

@Component({
  selector: 'app-lifecycle',
  template: `<div #box>O'lchanadigan blok: {{ width() }}px</div>`,
})
export class Lifecycle {
  width = signal(0);
  private destroyRef = inject(DestroyRef);

  constructor() {
    // DOM birinchi marta render bo'lgach bir marta ishlaydi (o'lcham olish uchun xavfsiz)
    afterNextRender(() => {
      const el = document.querySelector('#box') as HTMLElement;
      this.width.set(el.offsetWidth);
    });

    // Komponent yo'q qilinganda tozalash
    this.destroyRef.onDestroy(() => console.log('Lifecycle komponenti yo\'q qilindi'));
  }
}
```

```ts
// main.server.ts g'oyasi — SSR uchun bootstrap (server tarafi)
// (real loyihada @angular/ssr va server entry CLI tomonidan generatsiya qilinadi)
import { bootstrapApplication } from '@angular/platform-browser';
import { provideClientHydration } from '@angular/platform-browser';
import { App } from './app/app';

export default function bootstrap() {
  return bootstrapApplication(App, {
    providers: [
      provideClientHydration(), // brauzerda DOM'ni hydrate qilish
    ],
  });
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: `inject()` faqat injection context ichida ishlaydi

`inject()` ni xohlagan joyda chaqirib bo'lmaydi — u faqat **injection context**
(konstruktor, field initializer, `@Injectable` factory, yoki
`runInInjectionContext`) ichida ishlaydi.

```ts
export class Bad {
  load() {
    const http = inject(HttpClient); // ❌ Xato: metod ichida injection context yo'q
  }
}
```

```ts
export class Good {
  private http = inject(HttpClient); // ✅ field initializer — injection context bor
  load() { this.http.get('/api'); }
}
```

**Nega:** Angular `inject()` chaqirilganda "joriy injector" qaysiligini
ichki (`setCurrentInjector`) mexanizm orqali biladi. Bu kontekst faqat
komponent/servis yaratilayotgan paytda o'rnatiladi; oddiy metod chaqiruvida
yo'q.

### Gotcha 2: `@for` da `track` noto'g'ri tanlansa, DOM butunlay qayta yaratiladi

```html
<!-- ❌ track $index — element o'rin almashsa, DOM noto'g'ri qayta ishlatiladi -->
@for (user of users(); track $index) { <user-card [user]="user" /> }

<!-- ✅ track barqaror, noyob ID bilan -->
@for (user of users(); track user.id) { <user-card [user]="user" /> }
```

**Nega:** `track` Angular'ga "bu element o'sha element"ligini aytadi. `$index`
ishlatilsa va ro'yxat boshiga element qo'shilsa, barcha indekslar suriladi —
Angular har bir item "o'zgargan" deb o'ylaydi va `<user-card>`'larni qayta
yaratadi (state yo'qoladi, performance tushadi). Barqaror ID — to'g'ri yo'l.

### Gotcha 3: Zoneless rejimda Angular bilmagan o'zgarish DOM'ni yangilamaydi

```ts
// ❌ Zoneless rejimda: oddiy property — Angular bundan xabardor emas
export class Clock {
  time = new Date().toLocaleTimeString();
  constructor() {
    setInterval(() => {
      this.time = new Date().toLocaleTimeString(); // template yangilanmaydi!
    }, 1000);
  }
}
```

```ts
// ✅ signal ishlating — o'zgarish CD'ni trigger qiladi
export class Clock {
  time = signal(new Date().toLocaleTimeString());
  constructor() {
    setInterval(() => this.time.set(new Date().toLocaleTimeString()), 1000);
  }
}
```

**Nega:** Zoneless rejimda Zone.js `setInterval`'ni patch qilmaydi. Oddiy
property o'zgarishi haqida Angular xabar olmaydi — shuning uchun tick
rejalashtirilmaydi. Signal esa o'zgarganda scheduler'ni "notify" qiladi.

### Gotcha 4: `effect()` ichida signal `set` qilish — cheksiz tsikl xavfi

```ts
constructor() {
  effect(() => {
    // ❌ effect count'ni o'qiydi VA o'zgartiradi — qayta-qayta ishlaydi
    this.count.set(this.count() + 1);
  });
}
```

**Nega:** `effect` o'qigan signal'lariga bog'lanadi. Agar effect o'zi
o'qiyotgan signal'ni o'zgartirsa, u qayta ishlashga rejalashtiriladi —
potentsial cheksiz tsikl. Bunday holatlar uchun `computed` yoki `linkedSignal`
ishlating. (Angular bunday yozuvni odatda runtime'da bloklaydi/ogohlantiradi.)

### Gotcha 5: `index.html` selektori komponent selektoriga mos kelmasa, hech narsa render bo'lmaydi

```html
<!-- index.html -->
<app-root></app-root>
```

```ts
@Component({ selector: 'app-roott' /* ❌ typo */, template: `...` })
export class App {}
```

**Nega:** `bootstrapApplication(App)` `App.ɵcmp.selectors` bilan DOM'da
mos elementni qidiradi. Topa olmasa, ilova "ishladi", lekin ekranda hech narsa
yo'q (xato ham bermasligi mumkin). Selektorlar **aynan** mos bo'lishi shart.

---

## Common Mistakes

### ❌ Xato 1: NgModule bilan boshlash (eski tutorial bo'yicha)

```ts
// ❌ Noto'g'ri (yangi loyihada keraksiz boilerplate)
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  bootstrap: [AppComponent],
})
export class AppModule {}
// + platformBrowserDynamic().bootstrapModule(AppModule)
```

```ts
// ✅ To'g'ri (standalone)
@Component({ selector: 'app-root', template: `...` })
export class App {}
// main.ts
bootstrapApplication(App, appConfig);
```

**Nega:** Angular 19+ da standalone — default. NgModule qo'shimcha fayllar,
`declarations`/`imports` boshqaruvi va kognitiv yukni keltiradi. Yangi kod
uchun u keraksiz; faqat eski loyihalar yoki ba'zi kutubxonalar bilan
integratsiyada uchraydi.

### ❌ Xato 2: Template'da signal'ni `()` siz o'qish

```html
<!-- ❌ count — bu funksiya obyekti, qiymat emas -->
<p>{{ count }}</p>
```

```html
<!-- ✅ count() — signal'ni chaqirib qiymatini olamiz -->
<p>{{ count() }}</p>
```

**Nega:** Signal — bu **getter funksiya**. `count` yozsangiz, ekranda
qiymat emas, funksiyaning string ko'rinishi (`function ...`) chiqadi yoki
type-checking xatosi bo'ladi. Qiymatni olish uchun `()` shart. Bu signal'ning
oddiy property'dan eng ko'p chalkashtiriladigan farqi.

### ❌ Xato 3: Bog'liqlikni `new` bilan qo'lda yaratish

```ts
// ❌ DI'ni chetlab o'tish — testlash qiyin, singleton buziladi
export class UserList {
  private service = new UserService(); // har joyda yangi instance!
}
```

```ts
// ✅ DI orqali — Angular singleton'ni boshqaradi
export class UserList {
  private service = inject(UserService);
}
```

**Nega:** `new` bilan yaratsangiz, DI ierarxiyasini chetlab o'tasiz:
`UserService`'ning o'z bog'liqliklari (`HttpClient` ...) inject qilinmaydi va
har bir komponent o'z nusxasini yaratadi (singleton emas). Bundan tashqari,
testda mock bilan almashtirib bo'lmaydi. Har doim `inject()` yoki constructor
injection ishlating.

### ❌ Xato 4: `@for` ni `track` siz yozish

```html
<!-- ❌ track yo'q — Angular 17+ da compile xatosi -->
@for (item of items()) { <li>{{ item.name }}</li> }
```

```html
<!-- ✅ track majburiy -->
@for (item of items(); track item.id) { <li>{{ item.name }}</li> }
```

**Nega:** `track` — `@for` sintaksisining **majburiy** qismi. U bo'lmasa
build to'xtaydi. Sabab — performance: `track` Angular'ga DOM node'larni qaysi
ma'lumotga bog'lashni aytadi. Barqaror identifikator (ID) tanlang; iloji
bo'lmasagina `track $index`.

### ❌ Xato 5: Production'da JIT'ga tayanish yoki `@angular/compiler`'ni bundle'ga qo'shish

```ts
// ❌ Runtime'da template compile qilishga urinish (sekin, katta bundle)
import '@angular/compiler';
// ... JIT rejimida ishga tushirish
```

```bash
# ✅ Standart AOT build — compiler bundle'ga tushmaydi
ng build
```

**Nega:** AOT default va deyarli har doim to'g'ri tanlov: kichikroq bundle
(compiler yo'q), tezroq birinchi render, va build vaqtidagi template
type-checking. JIT'ni qo'lda yoqish bundle'ni ~kattalashtiradi va xatolarni
runtime'ga suradi.

---

## Amaliy Mashqlar

### Mashq 1: Birinchi standalone komponent (Oson)

**Savol:** `app-greeting` selektorli standalone komponent yozing. Unda
`name` signal'i bo'lsin (boshlang'ich qiymat `'Dunyo'`), template'da
`Salom, <name>!` ko'rsatsin va tugma bosilganda ismni `'Angular'`ga
o'zgartirsin. So'ng uni `main.ts`'da bootstrap qiling.

<details>
<summary>Javob</summary>

```ts
// greeting.ts
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-greeting',
  template: `
    <h1>Salom, {{ name() }}!</h1>
    <button (click)="change()">O'zgartirish</button>
  `,
})
export class Greeting {
  name = signal('Dunyo');

  change(): void {
    this.name.set('Angular');
  }
}
```

```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { Greeting } from './greeting';

bootstrapApplication(Greeting).catch((err) => console.error(err));
```

```html
<!-- index.html -->
<app-greeting></app-greeting>
```

**Tushuntirish:** `signal('Dunyo')` o'zgaruvchan holat yaratadi. Template'da
`name()` (`()` bilan!) uni o'qiydi va shu komponentni o'sha signal'ning
consumer'i qiladi. `set('Angular')` chaqirilganda Angular (zoneless'da ham)
CD'ni trigger qiladi va `<h1>` yangilanadi.

</details>

### Mashq 2: Servis va DI (O'rta)

**Savol:** `CounterStore` nomli `providedIn: 'root'` servis yozing. Unda
`count` signal'i va `increment()` metodi bo'lsin. So'ng **ikkita** alohida
komponent (`app-display` va `app-controls`) yozing: birinchisi `count()`ni
ko'rsatsin, ikkinchisida tugma `increment()`ni chaqirsin. Ikkala komponent
ham bir xil holatni baham ko'rishini ko'rsating.

<details>
<summary>Javob</summary>

```ts
// counter.store.ts
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class CounterStore {
  readonly count = signal(0);

  increment(): void {
    this.count.update((c) => c + 1);
  }
}
```

```ts
// display.ts
import { Component, inject } from '@angular/core';
import { CounterStore } from './counter.store';

@Component({
  selector: 'app-display',
  template: `<p>Joriy qiymat: {{ store.count() }}</p>`,
})
export class Display {
  protected store = inject(CounterStore);
}
```

```ts
// controls.ts
import { Component, inject } from '@angular/core';
import { CounterStore } from './counter.store';

@Component({
  selector: 'app-controls',
  template: `<button (click)="store.increment()">+1</button>`,
})
export class Controls {
  protected store = inject(CounterStore);
}
```

```ts
// app.ts — ikkalasini birga ishlatish
import { Component } from '@angular/core';
import { Display } from './display';
import { Controls } from './controls';

@Component({
  selector: 'app-root',
  imports: [Display, Controls],
  template: `
    <app-display />
    <app-controls />
  `,
})
export class App {}
```

**Tushuntirish:** `providedIn: 'root'` tufayli `CounterStore` butun ilova uchun
**bitta** instance (singleton). Ikkala komponent ham `inject(CounterStore)` orqali
**aynan o'sha** obyektni oladi. `Controls` `count`'ni oshirganda, `Display`'da
o'qilgan `count()` signal'i o'zgaradi va u avtomatik yangilanadi — komponentlar
to'g'ridan-to'g'ri bir-birini bilmaydi, lekin holatni baham ko'radi.

</details>

### Mashq 3: Compiled output'ni "o'qish" (Qiyin)

**Savol:** Quyidagi komponent uchun Ivy AOT taxminan qanday template funksiya
generatsiya qiladi? `create` (`rf & 1`) va `update` (`rf & 2`) bloklarida qaysi
instruction'lar bo'lishini yozing va `decls`/`vars` qiymatlarini taxmin qiling.

```ts
@Component({
  selector: 'app-tag',
  template: `<span [class.on]="active()">{{ text() }}</span>`,
})
export class Tag {
  active = signal(false);
  text = signal('label');
}
```

<details>
<summary>Javob</summary>

```js
// Taxminiy AOT chiqindisi (soddalashtirilgan)
Tag.ɵcmp = ɵɵdefineComponent({
  type: Tag,
  selectors: [['app-tag']],
  standalone: true,
  decls: 2,  // <span> elementi (0) + ichidagi text node (1)
  vars: 2,   // [class.on] uchun 1 + {{ text() }} uchun 1
  template: function Tag_Template(rf, ctx) {
    if (rf & 1) {                       // CREATE — bir marta
      ɵɵelementStart(0, 'span');
      ɵɵtext(1);
      ɵɵelementEnd();
    }
    if (rf & 2) {                       // UPDATE — har CD'da
      ɵɵclassProp('on', ctx.active());  // [class.on] binding
      ɵɵadvance(1);                     // text node'ga (slot 1) o't
      ɵɵtextInterpolate(ctx.text());    // {{ text() }}
    }
  },
});
```

**Tushuntirish:**

- **`decls: 2`** — template'da ikkita "deklaratsiya" bor: `<span>` elementi
  (slot 0) va uning ichidagi text node (slot 1).
- **`vars: 2`** — ikkita binding bor: `[class.on]` va `{{ text() }}`. Har biri
  oldingi qiymatini saqlash uchun bitta slot oladi (dirty checking uchun).
- **CREATE blok** DOM strukturasini bir marta quradi: `ɵɵelementStart`/`ɵɵtext`/
  `ɵɵelementEnd`.
- **UPDATE blok** har change detection'da binding'larni qayta hisoblaydi.
  `ɵɵclassProp` `<span>`'ga tegishli, shuning uchun `ɵɵadvance(1)`'dan **oldin**
  (kursor hali 0-slot/`<span>`'da). `ɵɵadvance(1)` kursorni text node'ga suradi,
  keyin `ɵɵtextInterpolate` interpolation'ni yangilaydi.
- `ctx.active()` va `ctx.text()` — signal o'qilishi; bu komponentni shu
  signal'larning consumer'iga ulaydi, shuning uchun ular o'zgarganda aynan shu
  view yangilanadi.

</details>

---

## Xulosa

- **Angular nima** — opinionated, batteries-included framework (component,
  DI, signals, router, forms, compiler bir butun); React kabi library emas,
  framework — u sening kodingni chaqiradi.
- **Paket tuzilishi** — `@angular/core`, `common`, `compiler`,
  `platform-browser`, `router` ...; compiler build vaqtida ishlaydi, AOT tufayli
  production bundle'ga tushmaydi.
- **Component** — Angular atomi: class + template + styles, `@Component`
  dekoratori orqali bog'lanadi; kaput ostida `ɵcmp` (ComponentDef) bo'lib
  compile bo'ladi; Angular 19+ da standalone default.
- **Template va binding** — interpolation/property/event/two-way; har biri
  Ivy instruction'iga (`ɵɵtextInterpolate`, `ɵɵproperty`, `ɵɵlistener`)
  compile bo'ladi va create/update rejimlarida ishlaydi.
- **Bootstrap** — `bootstrapApplication(App, appConfig)`; ichida platforma,
  EnvironmentInjector, ApplicationRef quriladi va ildiz komponent render qilinadi.
- **Standalone arxitektura** — NgModule'siz; bog'liqliklar `imports`'da; kamroq
  boilerplate, aniq va lokal scope.
- **Dependency Injection** — `inject()` + `providedIn: 'root'`; ikki injector
  ierarxiyasi (Environment + Node); tree-shakable singleton servislar.
- **Ivy compiler** — AOT (default) template'ni JS instruction'larga aylantiradi;
  kichik bundle, tez render, template type-checking; JIT — kamdan-kam.
- **Change Detection** — dirty checking + `ApplicationRef.tick()`; Zone.js
  (an'anaviy) yoki **zoneless** (Angular 21+ default), signal asosida.
- **Signals** — `signal`/`computed`/`effect`; producer-consumer grafi,
  versioning, glitch-free, memoization; CD bilan chambarchas bog'liq.
- **Yangi control flow** — `@if`/`@for`/`@switch`/`@defer`; `CommonModule`siz,
  `@for`'da `track` majburiy, `@defer` bilan lazy loading.
- **Rendering va hayot tsikli** — `TView` (statik qolip) + `LView` (instance);
  create → update → destroy; Renderer abstraksiyasi SSR/hydration'ni mumkin qiladi.

---

**Keyingi bo'lim:** [02-bootstrap-jarayoni.md](02-bootstrap-jarayoni.md) — Bootstrap jarayoni chuqur:
`main.ts` dan birinchi render'gacha to'liq yo'l, `platformBrowser`/`PlatformRef`,
`bootstrapApplication` ichki bosqichlari, `ApplicationConfig` va providerlar
yig'ilishi, `EnvironmentInjector` (`R3Injector`) qurilishi, `ApplicationRef`
va ildiz `LView` yaratilishi, `APP_INITIALIZER`/`provideAppInitializer` va
bootstrap lifecycle'i.

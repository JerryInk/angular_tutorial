## Неделя 11. Маршрутизация (Routing) и Навигация
------------------------------
## ЛЕКЦИЯ 11: Построение многостраничной архитектуры SPA в Angular 22
Продолжительность: 2 академических часа (90 минут)

## 1. Концепция маршрутизации на стороне клиента (Client-Side Routing) (20 минут)
В классических многостраничных сайтах (MPA) при переходе по ссылке браузер отправляет новый запрос на сервер и полностью перезагружает страницу. В Single Page Applications на Angular 22 перезагрузка происходит всего один раз — при старте.
Дальнейшую навигацию берет на себя Angular Router:

* Он перехватывает изменение URL в адресной строке браузера.
* Сопоставляет текущий путь со списком зарегистрированных маршрутов (Routes).
* Динамически уничтожает старый компонент и монтирует новый в специальную область разметки.
* Сохраняет историю переходов с помощью HTML5 History API (методы ```pushState``` и ```replaceState```), благодаря чему кнопки «Назад» и «Вперед» в браузере работают корректно.

## 2. Конфигурация роутера через provideRouter (20 минут)
В Angular 22 конфигурация маршрутизации полностью переведена на функциональные рельсы. Старый ```RouterModule.forRoot()``` удален. Настройка выполняется в app.config.ts с помощью функции ```provideRouter(routes)```.
Массив маршрутов app.routes.ts представляет собой набор объектов со строгой типизацией:

```typescript
  export const routes: Routes = [
    { path: '', redirectTo: 'dashboard', pathMatch: 'full' }, // Перенаправление по умолчанию
    { path: 'dashboard', component: DashboardComponent },       // Статический роут
    { path: 'courses/:id', component: CourseDetailComponent },  // Динамический роут с параметром
    { path: '**', component: NotFoundComponent }                 // Обработка несуществующих страниц (Wildcard)
  ];
```

------------------------------

Параметр ```pathMatch``` — это критически важная настройка маршрутизатора Angular, которая определяет алгоритм сопоставления URL-адреса в строке браузера с путями, описанными в массиве Routes.
Ошибка в нем — главная причина бесконечных циклов перенаправлений или «белых экранов» у новичков.
Параметр ```pathMatch``` может принимать два значения: 'prefix' и 'full'.

## 1). Режим pathMatch: 'prefix' (По умолчанию)
В этом режиме Angular Router сканирует строку URL слева направо и считает маршрут подошедшим, если указанный path совпадает с началом (префиксом) текущего адреса.

* Как это работает: Если у вас описан маршрут ```{ path: 'courses', component: CoursesComponent }```, он сработает и для URL '/courses', и для '/courses/12', и для '/courses/details/modules', так как все они начинаются с префикса 'courses'.

## 2). Режим pathMatch: 'full' (Точное совпадение)
В этом режиме Angular Router требует, чтобы указанный ```path``` абсолютно полностью и до последнего символа совпадал с тем, что написано в строке URL (за вычетом доменного имени и query-параметров).

## Почему для пустого пути (path: '') строго обязателен full?
Рассмотрим, что произойдет, если совершить классическую ошибку и не указать ```pathMatch: 'full'``` при перенаправлении с пустого пути:
```typescript
  // ❌ КРИТИЧЕСКАЯ ОШИБКА, КОТОРАЯ СЛОМАЕТ ПРИЛОЖЕНИЕ:
  { path: '', redirectTo: 'dashboard' } 
```
   1. По умолчанию Angular применит режим ```pathMatch: 'prefix'```.
   2. Пустая строка '' является префиксом абсолютно любого URL-адреса (любая строка начинается с «ничего»).
   3. Когда пользователь заходит на сайт, роутер видит пустой путь, сопоставляет его по префиксу и перенаправляет на '/dashboard'.
   4. На адресе '/dashboard' роутер снова начинает проверку массива маршрутов сверху вниз. Он доходит до первой строчки, видит path: '' и думает: «Так, адрес "/dashboard" содержит в самом начале пустую строку? Да, содержит!».
   5. Роутер опять выполняет инструкцию ```redirectTo: 'dashboard'```.
   6. Приложение зацикливается, браузер выбрасывает ошибку ```Error: Infinite redirect loop``` и страница падает.

## Правильное решение в Angular 22:
```typescript
  //  ИДЕАЛЬНО И БЕЗОПАСНО:
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' }
```
Инструкция говорит роутеру: «Выполняй перенаправление на страницу 'dashboard' строго в том случае, если в адресной строке вообще ничего не указано после домена (то есть пользователь открыл корень сайта)». Когда роутер перейдет на '/dashboard', точное совпадение с пустой строкой уже не сработает, и цикл не возникнет.

------------------------------

## 3. Навигация в шаблонах и TypeScript коде (25 минут)
Для организации переходов между страницами используются два механизма:

   1. Декларативный (в HTML шаблоне): Директива ```routerLink``` вместо стандартного атрибута ```href```. Если использовать ```href```, страница перезагрузится, обнулив все Angular Сигналы в памяти. Директива ```routerLinkActive``` позволяет автоматически вешать CSS-класс (например, '.active') на активную ссылку меню.
```html
  <a routerLink="/dashboard" routerLinkActive="active">Главная</a>
```
   2. Программный (в TypeScript классе): Использование сервиса Router через функцию ```inject()```.
```typescript
  private router = inject(Router);
  navigateToCourse(id: number) {
    this.router.navigate(['/courses', id]);
  }
```
   
## 4. Продвинутая фича: ```withComponentInputBinding()``` (25 минут)
Исторически, чтобы прочитать динамический параметр из URL (например, ID курса из пути '/courses/42'), разработчикам приходилось внедрять тяжелый сервис ```ActivatedRoute```, подписываться на RxJS поток ```paramMap``` или конвертировать его в Сигнал.
В Angular 22 этот процесс радикально упрощен на уровне конфигурации роутера с помощью утилиты ```withComponentInputBinding()```.

* Как это работает: Маршрутизатор автоматически связывает параметры URL (path params, query params) с сигнальными инпутами (```input()```) компонента. Если в пути указано ```:id```, компоненту достаточно просто объявить инпут с именем ```id```, и роутер сам запишет туда нужное значение.

------------------------------
## ПРАКТИЧЕСКОЕ ЗАНЯТИЕ 11: Создание многостраничной навигации и детальной страницы курса
Продолжительность: 2 академических часа (90 минут)
Цель работы: Сконфигурировать карту маршрутов приложения в Angular 22, реализовать сквозное меню навигации без перезагрузки страниц и настроить автоматическое чтение динамических параметров URL через новые сигнальные инпуты.

## Шаг 1: Генерация страниц-компонентов (15 минут)
Создаем два новых изолированных экрана для реализации перехода:
```typescript
  ng g c pages/home-dashboard --standalone
  ng g c pages/course-detail --standalone
```

## Шаг 2: Настройка карты маршрутов и связывания параметров (25 минут)

   1. Открыть файл src/app/app.routes.ts и описать конфигурацию путей:
```typescript
  import { Routes } from '@angular/router';
  import { HomeDashboardComponent } from './pages/home-dashboard/home-dashboard.component';
  import { CourseDetailComponent } from './pages/course-detail/course-detail.component';
  
  export const routes: Routes = [
    // При пустом пути перенаправляем на главную панель
    { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
    
    { path: 'dashboard', component: HomeDashboardComponent },
    
    // Динамический маршрут. Имя параметра ":courseId" важно для инпута!
    { path: 'course/:courseId', component: CourseDetailComponent }
  ];
```

   1. Открыть src/app/app.config.ts и активировать автоматическое связывание параметров с инпутами через функцию ```withComponentInputBinding()```:
```typescript
  import { ApplicationConfig, provideZonelessChangeDetection } from '@angular/core';
  import { provideRouter, withComponentInputBinding } from '@angular/router'; // Импортируем утилиту опций
  import { routes } from './app.routes';
  import { provideHttpClient } from '@angular/common/http';
  
  export const appConfig: ApplicationConfig = {
    providers: [
      provideZonelessChangeDetection(),
      // Включаем роутер с поддержкой маппинга параметров URL в сигнальные инпуты
      provideRouter(routes, withComponentInputBinding()), 
      provideHttpClient()
    ]
  };
```

## Шаг 3: Реализация детальной страницы с чтением параметра (25 минут)
Открыть course-detail.component.ts. Вместо импорта ```ActivatedRoute``` мы объявляем сигнальный инпут, чье имя строго совпадает с параметром ```:courseId``` из конфигурации роута. Angular 22 сам пробросит туда значение из URL.
```typescript
  import { Component, ChangeDetectionStrategy, input, computed, inject, OnInit } from '@angular/core';
  import { RouterLink } from '@angular/router';
  
  @Component({
    selector: 'app-course-detail',
    standalone: true,
    imports: [RouterLink], // Импортируем RouterLink для кнопки "Назад"
    templateUrl: './course-detail.component.html',
    styleUrl: './course-detail.component.scss',
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class CourseDetailComponent implements OnInit {
    // Название инпута строго совпадает с ":courseId" в файле app.routes.ts
    courseId = input.required<string>();
  
    // Вычисляемый сигнал на базе параметра URL
    pageTitle = computed(() => `Детальный обзор учебного курса №${this.courseId()}`);
  
    ngOnInit(): void {
      // В OnPush режиме данные из роутера гарантированно доступны уже в ngOnInit
      console.log(`Компонент успешно инициализирован для ID курса: ${this.courseId()}`);
      // Здесь будем вызывать метод: this.apiService.getCourseById(this.courseId())
    }
  }
```
Открыть шаблон course-detail.component.html:
```html
  <div class="detail-container">
    <h2>{{ pageTitle() }}</h2>
    <div class="mock-data-card">
      <p><strong>Параметр из URL успешно прочитан:</strong> {{ courseId() }}</p>
      <p class="desc">На этой странице отображается подробная техническая программа и список лабораторных работ для выбранного ИТ-направления.</p>
    </div>
    
    <!-- Возврат на главную через RouterLink -->
    <button routerLink="/dashboard" class="btn-back">← Вернуться в каталог</button>
  </div>
```

## Шаг 4: Верстка глобального меню и разводки роутера (15 минут)
Чтобы компоненты страниц рендерились, нам нужно указать Angular место на экране.

   1. Открыть src/app/app.component.ts. Убедиться, что в массиве ```imports``` подключены ```RouterOutlet```, ```RouterLink``` и ```RouterLinkActive```.
   2. Открыть src/app/app.component.html и спроектировать сквозной каркас приложения с шапкой навигации:
```html
  <header class="main-header">
    <h1>Учебный портал кафедры ИТ (Angular 22)</h1>
    <nav class="navigation-menu">
      <!-- Декларативные ссылки без перезагрузки страницы -->
      <a routerLink="/dashboard" routerLinkActive="active-route">Каталог курсов</a>
      <!-- Ссылка на тестовый ID курса для проверки динамического роутинга -->
      <a [routerLink]="['/course', '105']" routerLinkActive="active-route">Тестовый курс #105</a>
    </nav>
  </header>
  
  <main class="page-content-wrapper">
    <!-- Сюда Angular Router будет динамически монтировать компоненты страниц -->
    <router-outlet />
  </main>
```

## Шаг 5: Проверка динамики переходов (10 минут)

   1. Запустить dev-сервер: ```ng serve```.
   2. Покликать по вкладкам меню. Убедиться, что URL в адресной строке меняется, компоненты подменяют друг друга, но вкладка браузера не показывает индикатор перезагрузки (приложение работает как истинное SPA).
   3. Перейти по ссылке «Тестовый курс #105». Убедиться, что компонент ```CourseDetailComponent``` успешно поймал значение '105' из URL прямо в сигнальный инпут без использования потоков RxJS, а ```computed``` сигнал обновил заголовок страницы.

------------------------------
## Домашнее задание к Неделе 12

   1. Реализовать на странице ```HomeDashboardComponent``` (или в вашем каталоге из Недели 9) кнопку «Подробнее» у каждой карточки курса, которая программно перенаправляет пользователя на детальную страницу соответствующего ID курса с помощью вызова ```router.navigate```.
   2. Изучить концепцию оптимизации Lazy Loading (Ленивая загрузка). Узнать, как заставить браузер скачивать файлы страниц только в тот момент, когда пользователь на них кликает.

------------------------------

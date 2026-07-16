## Модуль: Современный UI-слой (Tailwind CSS & Taiga UI v5)
Данный модуль является сквозным расширением основного курса. Материалы интегрируются в практическую и лекционную часть семестра, начиная с Недели 3 (после того как студенты развернули Zoneless-архитектуру и изучили базовую файловую структуру).
Ниже представлена детальная стенограмма и методические материалы для преподавателя по каждой специализированной неделе.

------------------------------
## НЕДЕЛЯ А: Интеграция Tailwind CSS и атомарный дизайн (Utility-First)
## 1. Критика классического CSS/SCSS в крупных SPA-приложениях
Когда мы пишем классический CSS или SCSS (даже используя методологию BEM), мы сталкиваемся с тремя инженерными проблемами в SPA:

   1. Проблема раздувания стилей (Style Bloat): Мы постоянно пишем похожие классы: ```.btn-primary```, ```.task-btn```, ```.save-button```. В каждом из них дублируются свойства ```display: flex, border-radius: 4px, padding: 8px 16px```. Бандл стилей растет линейно с ростом количества компонентов.
   2. Ментальный оверхед (Naming Things): Разработчик тратит до 20% времени на выдумывание имен классов: ```.card-wrapper-inner-avatar-container```.
   3. Мертвый CSS (Dead Code): При удалении компонента разработчики часто забывают удалить связанные с ним стили из глобальных файлов.

## 2. Философия Utility-First (Атомарный CSS)
Tailwind CSS предлагает противоположную парадигму: вместо создания классов под конкретные сущности, мы используем атомарные утилитарные классы. Каждый класс выполняет строго одно CSS-свойство.

* Нам больше не нужно придумывать имена классов.
* Мы пишем стили прямо в HTML-шаблоне.
* Главный плюс для Angular 22: Компилятор Tailwind сканирует шаблоны проекта на этапе сборки и формирует CSS-файл, содержащий только реально использованные утилиты. Если вы использовали класс ```flex``` 1000 раз, в итоговом CSS он запишется всего один раз. Размер CSS-файла продакшена перестает расти с увеличением размера приложения и стабилизируется на уровне ~15-40 Кб.

## 3. Как это работает в современной сборке Angular 22 (Vite + Esbuild)
В Angular 22 старый Webpack полностью удален. Новый application-билдер компилирует проект с помощью сверхбыстрого инструмента Esbuild.
Интеграция Tailwind CSS выполняется через современную связку PostCSS и официальный плагин @tailwindcss/postcss. На этапе компиляции билдер Angular перехватывает файлы стилей, пропускает их через PostCSS, компилятор Tailwind ищет текстовые маркеры в TypeScript и HTML файлах, генерирует атомарные правила и на лету встраивает их в итоговый бандл. Нам больше не нужны тяжелые конфигурационные JavaScript-файлы tailwind.config.js — вся настройка тем в современной версии Tailwind выполняется прямо через директивы в главном CSS/SCSS файле с помощью стандартного синтаксиса CSS-переменных.

------------------------------
## 💻 ПРАКТИЧЕСКОЕ ЗАНЯТИЕ
Цель: Полностью перевести UI-слой проекта на утилитарные классы Tailwind CSS, настроить сборку PostCSS и реализовать адаптивный, OnPush-совместимый компонент карточки курса.
## Шаг 1: Инсталляция зависимостей
Установить пакеты Tailwind, PostCSS и официальный современный плагин интеграции:
```bash
  npm install tailwindcss @tailwindcss/postcss postcss --save-dev
```

## Шаг 2: Создание файла конфигурации PostCSS
Создать в корневой папке проекта файл конфигурации .postcssrc.json, чтобы Esbuild-билдер Angular знал, как обрабатывать стили:
```json
  {
    "plugins": {
      "@tailwindcss/postcss": {}
    }
  }
```

## Шаг 3: Активация Tailwind в глобальных стилях
Открыть файл src/styles.scss (или styles.css), удалить оттуда все старые стили и добавить современный импорт дизайн-системы:
```scss
  @import "tailwindcss";
```
Эта директива автоматически разворачивает базовые стили (Preflight), утилиты и адаптивные сетки Tailwind.

## Шаг 4: Разработка адаптивного компонента карточки
Открыть HTML-шаблон карточки курса (например, course-card.component.html) и написать разметку, комбинирующую сетку, тени, отступы и состояние наведения (hover), без написания ни одной строчки в .scss файле:
```html
  <div class="p-6 max-w-sm mx-auto bg-white rounded-xl shadow-md space-y-4 border-l-4 border-indigo-500 transition-all duration-300 hover:scale-102 hover:shadow-lg">
    
    <div class="flex items-center justify-between">
      <span class="text-xs font-semibold inline-block py-1 px-2 uppercase rounded-full text-indigo-600 bg-indigo-200">
        ИТ-Направление
      </span>
      <span class="text-sm text-slate-500 font-mono">{{ course().hours }} акад. ч.</span>
    </div>
  
    <div>
      <h4 class="text-xl font-bold text-slate-900 leading-tight">
        {{ course().title }}
      </h4>
      <p class="text-slate-500 text-sm mt-1">
        Изучение актуальных стандартов фреймворка в Zoneless-режиме на Сигналах.
      </p>
    </div>
  
    <div class="flex justify-end pt-2">
      <button (click)="triggerDelete()" class="px-4 py-1.5 text-sm text-red-600 font-semibold rounded-md border border-red-200 hover:text-white hover:bg-red-500 hover:border-transparent transition-colors duration-200 focus:outline-none">
        Удалить из плана
      </button>
    </div>
    
  </div>
```
## Шаг 5: Проверка компиляции и HMR
Запустите ```ng serve```. Откройте DevTools. Посмотрите, как плагин Vite HMR мгновенно подменяет классы в браузере при их изменении в коде, что файлы стилей самого компонента course-card.component.scss теперь абсолютно пустые, но компонент выглядит как готовое enterprise-решение.

------------------------------
## НЕДЕЛЯ Б: Внедрение Taiga UI v5 и построение продвинутых форм
## 1. Почему именно Taiga UI v5 в 2026 году?
На ИТ-рынке существует много UI-библиотек: Angular Material, NG-ZORRO, PrimeNG. Почему мы берем именно Taiga UI v5?
Потому что в пятой версии произошла техническая революция, которая идеально мэтчится с нашей OnPush/Zoneless архитектурой:

   1. Полный отказ от @angular/animations: Исторически UI-киты требовали подключения тяжелого пакета анимаций Angular, который постоянно дергал Change Detection. В Taiga UI v5 все переходы, выпадающие списки и плавные появления написаны на нативном CSS. Библиотека работает молниеносно в приложениях без зон.
   2. Signals-First: Компоненты Taiga UI v5 полностью переписаны под использование Сигналов на входе (Signal Inputs), что гарантирует точечную перерисовку элементов интерфейса.

## 2. Концепция полиморфизма (ng-polymorpheus)
Уникальная черта Taiga UI — глубокое использование паттерна полиморфизма. Что это значит? Если у компонента (например, кнопки или заголовка) есть инпут для иконки или контента, туда можно передать:

* Обычную строку текста.
* Число.
* Чистую функцию, возвращающую контент.
* Специальный Angular-шаблон ```<ng-template>```.
Компонент сам поймет тип данных и отрендерит его корректно. Это избавляет от необходимости писать десятки условных директив ```@if``` внутри разметки библиотеки.

------------------------------
## 💻 ПРАКТИЧЕСКОЕ ЗАНЯТИЕ
Цель: Интегрировать Taiga UI v5 в Angular 22 проект через Angular Schematics, настроить корневой контекст приложения и разработать строго типизированную форму с использованием интеллектуальных полей ввода.

## Шаг 1: Автоматизированная установка через CLI
Запустить официальный скрипт установки:
```bash
  npm install less --save-dev
  ng add taiga-ui
```
Во время выполнения скрипта CLI задаст вопросы в терминале. Нужно выбрать установку пакетов ```core```, ```cdk``` и ```kit```, а также согласиться на автоматическую модификацию файлов. Скрипт сам пропишет зависимости в package.json.

## Шаг 2: Настройка глобального корневого тега приложения
Компоненты Taiga UI (особенно выпадающие списки и диалоги) требуют наличия единого корневого менеджера в DOM.

Открыть src/app/app.component.html и обернуть всю разметку в тег ```<tui-root>```:
```html
  <!-- Комбинируем Taiga UI менеджер с Tailwind разметкой -->
  <tui-root class="min-h-screen font-sans antialiased">

    <header class="shadow-sm border-b p-4 mb-6">
      <div class="container mx-auto flex justify-between items-center">
        <h1 class="text-xl font-bold text-white">ИТ-Кафедра | Angular 22</h1>
      </div>
    </header>

    <main class="container mx-auto px-4">
      <!-- Сюда роутер монтирует страницы -->
      <app-course-form />
      <router-outlet />
    </main>

  </tui-root>
```
В файле app.component.ts в массив ```imports``` необходимо добавить ```TuiRoot``` из @taiga-ui/core.

## Шаг 3: Модернизация Реактивной формы
Перепишем форму добавления курса, заменив нативные HTML инпуты на интеллектуальные компоненты Taiga UI с плавающими лейблами и встроенной валидацией.
Открыть course-form.component.ts, импортировать компоненты библиотеки:
```typescript
  import { Component, ChangeDetectionStrategy, inject } from '@angular/core';
  import { ReactiveFormsModule, FormBuilder, Validators } from '@angular/forms';
  import { TuiButton, TuiInput } from '@taiga-ui/core';
  
  @Component({
    selector: 'app-course-form',
    standalone: true,
    imports: [
      ReactiveFormsModule,
      TuiButton,
      TuiInput,
    ],
    templateUrl: './course-form.html',
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class CourseForm {
    private fb = inject(FormBuilder);
  
    courseForm = this.fb.group({
      title: this.fb.control<string>('', { nonNullable: true, validators: [Validators.required, Validators.minLength(5)] }),
      hours: this.fb.control<number>(16, { nonNullable: true, validators: [Validators.required, Validators.min(16)] })
    });
  
    onSubmit(): void {
      if (this.courseForm.invalid) return;
      console.log(this.courseForm.getRawValue());
      this.courseForm.reset();
    }
  }
```

## Шаг 4: Верстка шаблона формы
Открыть course-form.component.html. Обратите внимание на то, как лаконично выглядит разметка Taiga UI:
```html
  <div class="p-6 rounded-lg shadow-sm max-w-md mx-auto font-sans bg-neutral-800">
    <h3 class="text-lg font-bold text-white mb-4">Добавить новую дисциплину</h3>
  
    <form [formGroup]="courseForm" (ngSubmit)="onSubmit()" class="space-y-5">
  
      <tui-textfield>
        <input
          tuiInput
          type="text"
          formControlName="title"
          placeholder="Например, Архитектура Angular 22" />
        <label tuiLabel>Название дисциплины</label>
      </tui-textfield>
  
      <tui-textfield>
        <input
          tuiInput
          type="number"
          formControlName="hours" />
        <label tuiLabel>Объем (академические часы)</label>
      </tui-textfield>
  
      <button tuiButton type="submit" [disabled]="courseForm.invalid" class="w-full">
        Отправить в учебный план
      </button>
  
    </form>
  </div>
```

## Шаг 5: Анализ UX-валидации
Запустите проект. Кликните в поле «Название дисциплины» и сразу выйдете из него. Элемент tui-input автоматически окрасится в фирменный красный цвет ошибки. Taiga UI из коробки считывает состояние ```touched``` и ```invalid``` у связанного ```FormControl``` в OnPush режиме, избавляя нас от написания ручных блоков ```@if``` для базовой подсветки ошибок.

------------------------------
## НЕДЕЛЯ В: Диалоговые окна и уведомления (Taiga UI Services)
## 1. Концепция Порталов (Portals) и динамический DOM
Давайте разберем, как модальные окна физически рендерятся на экране. Если мы опишем разметку модального окна внутри обычного компонента страницы, мы столкнемся с проблемой CSS-изоляции. Если у родительского компонента прописано свойство ```overflow: hidden``` или выставлен определенный ```z-index```, наше модальное окно может обрезаться или отобразиться под другими элементами.
Чтобы этого избежать, в современных SPA используется механизм Порталов (Portals). Когда мы вызываем диалоговое окно программно, Angular Router и механизмы CDK берут шаблон окна и физически рендерят его в самом низу тега <body>, за пределами иерархии нашего компонента. Окно перекрывает весь интерфейс гарантированно и безопасно.

## 2. Инъекция сервисов управления UI
В Angular 22 для вызова всплывающих уведомлений (алертов) и диалогов используется строго декларативный сервис-ориентированный подход через DI (Dependency Injection). Мы внедряем сервисы TuiAlertService и TuiDialogService с помощью функции inject().
Методы этих сервисов возвращают RxJS-потоки (Observable). Это позволяет нам подписываться на события закрытия окна или нажатия кнопок согласия, организуя чистую цепочку асинхронных операций при работе с базовым бэкендом.

------------------------------
## 💻 ПРАКТИЧЕСКОЕ ЗАНЯТИЕ
Цель: Реализовать централизованное уведомление пользователя об успешных сетевых CRUD-операциях и настроить модальное окно подтверждения критических действий (удаления курса) с интеграцией в HTTP-слой.

## Шаг 1: Интеграция TuiNotificationService в Smart-компонент
Откроем умный компонент управления данными ```CourseManagerComponent```. Внедряем сервис алертов Taiga UI.
Открыть course-manager.component.ts:
```typescript
  import { Component, OnInit, ChangeDetectionStrategy, inject, signal } from '@angular/core';
  import { CourseApiService, Course } from '../../services/course-api.service';
  import { TuiNotificationService } from '@taiga-ui/core'; // Импортируем сервис алертов
  
  @Component({
    selector: 'app-course-manager',
    templateUrl: './course-manager.component.html',
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class CourseManagerComponent implements OnInit {
    private apiService = inject(CourseApiService);
    
    // Внедряем UI сервис через функциональный DI
    protected readonly notifications = inject(TuiNotificationService);
  
    courses = signal<Course[]>([]);
  
    ngOnInit(): void {
      this.loadData();
    }
  
    loadData(): void {
      this.apiService.getCourses().subscribe(data => this.courses.set(data));
    }
  }
```

## Шаг 2: Реализация красивого всплывающего алерта при POST запросе
Модифицируем метод добавления новой дисциплины. При успешном ответе от сервера бэкенда мы будем вызывать красивый push-алерт с автоматическим закрытием через 1 секунду.
```typescript
  // Внутри класса CourseManagerComponent
  
  handleCreateCourse(newCourseData: Course): void {
    this.apiService.createCourse(newCourseData).subscribe({
      next: (createdItem) => {
        // 1. Обновляем реактивный Сигнал состояния UI
        this.courses.update(current => [...current, createdItem]);
  
        // 2. Вызываем императивное уведомление Taiga UI
        this.notifications.open('Дисциплина успешно внесена в базу данных бэкенда.', {
          label: 'Успешная синхронизация',
          autoClose: 1000        // Закроется само через 1 секунду
        }).subscribe(); // Важно: подписка активирует рендеринг алерта в портале
      },
      error: (err) => {
        // В случае сбоя сети выводим красное error-уведомление
        this.alerts.open(err.message, {
          label: 'Ошибка сервера',
          appearance: 'warning',
          autoClose: 2000
        }).subscribe();
      }
    });
  }
```

## Шаг 3: Модальное окно подтверждения удаления через TuiDialogService
Добавим строгую логику: при клике на кнопку «Удалить» мы не стираем данные сразу, а открываем диалоговое окно. Если пользователь нажимает «Подтвердить», мы отправляем DELETE запрос на сервер.
Импортируем ```TuiDialogService``` в класс компонента:
```typescript
  import { TuiDialogService } from '@taiga-ui/core';// ...private readonly dialogs = inject(TuiDialogService);
```
Напишем метод открытия диалога, передавая туда кастомный шаблон или строку (используем полиморфизм Taiga UI v5):
```typescript
  confirmDelete(id: number | undefined): void {
    if (!id) return;
  
    // Открываем встроенный диалог подтверждения, возвращающий boolean поток
    this.dialogs.open<boolean>(
      'Вы уверены, что хотите безвозвратно удалить данную дисциплину из учебного плана семестра?',
      {
        label: 'Внимание! Критическое действие',
        size: 's',
        data: 'Удалить'
      }
    ).subscribe({
      next: () => {
        // Этот блок сработает, если пользователь нажал подтверждающую кнопку в диалоге
        this.apiService.deleteCourse(id).subscribe(() => {
          this.courses.update(current => current.filter(c => c.id !== id));
          this.alerts.open('Объект успешно удален из базы данных.', { appearance: 'info' }).subscribe();
        });
      }
    });
  }
```

## Шаг 4: Тестирование сквозного UI-сценария

   1. Проверяем интерфейс в браузере.
   2. При нажатии на кнопку удаления поверх экрана плавно всплывает оверлей модального окна (сгенерированный в корне ```<body>``` через портал CDK). Клик по фону вокруг окна автоматически закрывает его без выполнения каких-либо действий.
   3. При нажатии кнопки подтверждения окно исчезает, отправляется запрос к бэкенду, список OnPush перерисовывается, и в правом верхнем углу экрана всплывает тост-уведомление ```appearance: 'info'```.

------------------------------
## 📝 Руководство по проверке финальных работ (для преподавателя)
   1. Файловая гигиена стиля: В папках 'components/' и 'pages/' файлы с расширением .scss должны содержать 0 строк кода (или быть удалены). Если у студента обнаружены классические стили типа ```margin-top: 20px```, оценка снижается. Вся верстка должна быть выполнена на утилитах Tailwind CSS.
   2. Изоляция UI компонентов: Поля ввода данных, календари, селекторы, кнопки действий и индикаторы статусов должны быть импортированы из ```@taiga-ui/kit``` и ```@taiga-ui/core```. Наличие нативных серых HTML-кнопок ```<button>``` не допускается.
   3. Реактивный сбор состояния: Все алерты и вызовы диалогов должны активироваться строго внутри методов подписок на HTTP-сервисы (```CourseApiService```), демонстрируя синхронизацию асинхронного сетевого слоя с интерфейсным слоем Taiga UI в Zoneless/OnPush среде.

------------------------------

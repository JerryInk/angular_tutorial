## Неделя 8. Основы RxJS в Angular
------------------------------
## ЛЕКЦИЯ 8: Асинхронное реактивное программирование и потоки RxJS в Angular 22
Продолжительность: 2 академических часа (90 минут)
## 1. Концепция RxJS: Паттерн Наблюдатель (Observer) и Потоки данных (25 минут)
До появления Сигналов библиотека RxJS (Reactive Extensions for JavaScript) была единственным способом реализации реактивности в Angular. В Angular 22, несмотря на доминирование архитектуры Signals-First, RxJS остается критически важным инструментом.
Главное правило разделения обязанностей:

* Signals идеально подходят для управления состоянием (синхронные значения, UI-флаги, переменные шаблона).
* RxJS незаменим для обработки асинхронных событий (сетевые HTTP-запросы, веб-сокеты, сложные цепочки обработки пользовательского ввода во времени).

Базовые понятия:

* Observable (Поток / Наблюдаемый объект): Источник данных, который «излучает» события во времени. Это может быть бесконечный поток кликов мыши или конечный поток, возвращающий один HTTP-ответ.
* Observer (Наблюдатель): Потребитель данных, который подписывается на Observable и реагирует на три типа сигналов: next (пришло новое значение), error (произошла ошибка) и complete (поток успешно завершился).

## 2. Базовые операторы трансформации и фильтрации (20 минут)
Сила RxJS заключается в операторах — чистых функциях, которые позволяют трансформировать, комбинировать и фильтровать потоки данных с помощью метода .pipe():

* map: Преобразует каждое значение в потоке (аналог Array.prototype.map).
* filter: Пропускает через поток только те значения, которые удовлетворяют условию (аналог Array.prototype.filter).
* tap: Позволяет выполнять побочные эффекты (например, логирование console.log), не изменяя сами данные в потоке.

## 3. Безопасное управление подписками во времени (20 минут)
Самая опасная ошибка начинающего разработчика при работе с RxJS — забытая подписка. Если компонент подписался на бесконечный поток (например, таймер или глобальное событие окна), а затем пользователь перешел на другую страницу, подписка останется активной в памяти. Это приводит к утечкам памяти (memory leaks).
В Angular 22 для безопасного уничтожения подписок используются современные декларативные инструменты:

* Оператор takeUntilDestroyed(): Самый надежный способ. Он автоматически отписывает Observable от наблюдателя в момент, когда компонент или сервис уничтожается (DestroyRef).
```typescript
  import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
  
  myStream$.pipe(
    takeUntilDestroyed() // Автоматическая отписка!
  ).subscribe(val => console.log(val));
```
## 4. Мост между мирами: Функции toSignal() и toObservable() (25 минут)
Поскольку современный Angular 22 рендерит шаблоны на Сигналах, а асинхронные операции (например, HTTP-клиент) возвращают RxJS Observable, фреймворк предоставляет встроенные инструменты для их бесшовной конвертации из пакета @angular/core/rxjs-interop.
## toSignal(observable$)
Превращает поток RxJS в Read-Only Сигнал. Каждый раз, когда поток излучает новое значение, Сигнал автоматически обновляется, заставляя OnPush-компонент перерисоваться.
```typescript
  // Конвертация потока в сигнал
  dataSignal = toSignal(this.httpData$); 
```
Важно: Поскольку Сигнал всегда должен иметь начальное значение, а поток может вернуть данные позже, в toSignal часто передают опцию { initialValue: null } или { requireSync: true }.
## toObservable(mySignal)
Превращает Сигнал в поток RxJS, позволяя применять к нему мощные операторы фильтрации или задержки (например, debounceTime при вводе текста).

------------------------------
## ПРАКТИЧЕСКОЕ ЗАНЯТИЕ 8: Создание реактивного трекера асинхронных событий
Продолжительность: 2 академических часа (90 минут)
Цель работы: Освоить работу с потоками RxJS, научиться трансформировать данные через операторы map и filter, настроить безопасную автоматическую отписку через takeUntilDestroyed и интегрировать RxJS-поток в OnPush-шаблон с помощью toSignal().
## Шаг 1: Инициализация компонента мониторинга активности (15 минут)

   1. Сгенерировать компонент:
```bash
  ng g c components/activity-monitor
```
   2. Открыть activity-monitor.component.ts и перевести его в ChangeDetectionStrategy.OnPush:
```typescript
  import { Component, ChangeDetectionStrategy, signal } from '@angular/core';
  
  @Component({
    selector: 'app-activity-monitor',
    imports: [],
    templateUrl: './activity-monitor.component.html',
    styleUrl: './activity-monitor.component.scss',
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class ActivityMonitorComponent {
    // Логика работы с RxJS будет описана здесь
  }
```
   
## Шаг 2: Создание асинхронного потока и применение операторов (25 минут)
Реализуем генератор случайных логов событий (например, имитация фоновой отправки метрик или пингов бэкенда) с помощью встроенного конструктора потоков interval.
```typescript
  import { Component, ChangeDetectionStrategy, inject } from '@angular/core';
  import { interval, map, filter, tap } from 'rxjs';
  import { takeUntilDestroyed, toSignal } from '@angular/core/rxjs-interop';
  
  interface SystemLog {
    id: number;
    timestamp: string;
    type: 'INFO' | 'WARNING' | 'CRITICAL';
    message: string;
  }

  @Component({
    selector: 'app-activity-monitor',
    standalone: true,
    templateUrl: './activity-monitor.component.html',
    styleUrl: './activity-monitor.component.scss',
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class ActivityMonitorComponent {
  
    // 1. Создаем поток RxJS, который генерирует событие каждые 3 секунды
    #rawLogsStream$ = interval(3000).pipe(
      // Трансформируем число секунд в объект SystemLog
      map((tick): SystemLog => {
        const types: ('INFO' | 'WARNING' | 'CRITICAL')[] = ['INFO', 'WARNING', 'CRITICAL'];
        const randomType = types[Math.floor(Math.random() * types.length)];
        return {
          id: tick,
          timestamp: new Date().toLocaleTimeString(),
          type: randomType,
          message: `Лог системы #${tick}: Зафиксирована сетевая активность базового бэкенда.`
        };
      }),
      // Логируем событие в консоль для отладки
      tap(log => console.log('RxJS Излучение:', log)),
      // Отфильтруем поток, пропуская только INFO и WARNING (игнорируем CRITICAL для теста)
      filter(log => log.type !== 'CRITICAL'),
      // Защита от утечки памяти — подписка умрет вместе с компонентом
      takeUntilDestroyed()
    );
  
    // 2. Конвертируем RxJS поток в Angular Signal для отображения в OnPush-шаблоне
    latestLog = toSignal(this.#rawLogsStream$, { initialValue: null });
  }
```
## Шаг 3: Верстка шаблона под Сигнал из RxJS (20 минут)
Открыть activity-monitor.component.html. Обратите внимание, что поскольку мы использовали ```toSignal()```, в HTML мы работаем со стандартным вызовом сигнала latestLog(), полностью избегая старого тяжелого пайпа | async.
```html
  <div class="monitor-card">
    <h3>Мониторинг асинхронных логов (RxJS + Signals)</h3>
    <p class="sub">Данные генерируются асинхронным потоком `interval` и фильтруются на лету.</p>
  
    @if (latestLog(); as log) {
      <div class="log-display" [class.warn]="log.type === 'WARNING'">
        <div class="log-header">
          <span class="badge">{{ log.type }}</span>
          <span class="time">{{ log.timestamp }}</span>
        </div>
        <p class="msg">{{ log.message }}</p>
      </div>
    } @else {
      <div class="skeleton-loader">
        <p>Ожидание первого асинхронного события от потока...</p>
      </div>
    }
  </div>
```
## Шаг 4: Подключение и проверка HMR (15 минут)

   1. Импортировать ActivityMonitorComponent в массив imports корневого AppComponent.
   2. Разместить тег ```<app-activity-monitor />``` в app.component.html.
   3. Запустить dev-сервер: ```ng serve```.

## Шаг 5: Анализ результатов и дебаг (15 минут)
Должны увидеть следующее:

   1. Первые 3 секунды отображается блок ожидания.
   2. Каждые 3 секунды в консоли появляется лог от оператора tap.
   3. Если сгенерировался лог типа INFO или WARNING, интерфейс мгновенно обновляется точечно.
   4. Если сгенерировался лог CRITICAL, в консоли строчка появится (благодаря tap), но экран останется прежним, так как оператор filter заблокировал передачу этого значения дальше в toSignal. Это наглядно демонстрирует концепцию «конвейера» обработки данных в RxJS.

------------------------------
## Домашнее задание к Неделе 9

   1. Модифицировать компонент ActivityMonitorComponent: добавить кнопку «Остановить поток». Реализовать логику остановки (подсказка: познакомиться с оператором takeWhile или управлять флагом внутри потока).
   2. Повторить основы REST архитектуры (методы GET, POST, PUT, DELETE). Подготовиться к следующей неделе, на которой мы начнем делать реальные сетевые запросы к базовому приложению бэкенда через HttpClient.

------------------------------

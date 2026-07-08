## Неделя 7. Внедрение зависимостей (Dependency Injection) и глобальное состояние
------------------------------
## ЛЕКЦИЯ 7: Архитектура DI и паттерн State Management на Сигналах в Angular 22
Продолжительность: 2 академических часа (90 минут)
## 1. Шаблон проектирования Dependency Injection (DI) в Angular (20 минут)
Внедрение зависимостей (DI) — это базовый архитектурный паттерн Angular, реализующий принцип инверсии управления (IoC). Компоненты не должны самостоятельно создавать экземпляры сервисов через оператор new. Вместо этого компонент декларирует: «Мне нужен инструмент для работы с бэкендом», а встроенный движок Angular (Injector) сам находит, создает и поставляет нужный объект.
Преимущества встроенного DI:

* Слабая связанность (Loose Coupling): Логику отображения легко отделить от бизнес-логики и работы с сетью.
* Тестируемость: В Unit-тестах реальный сервис бэкенда можно за наносекунду подменить на заглушку (Mock-сервис).
* Управление жизненным циклом: Angular сам контролирует, когда создавать сервис и когда его уничтожать.

## 2. Революция функции inject() против инъекции через конструктор (20 минут)
Долгое время единственным способом получить зависимость в Angular проброс через параметры конструктора класса. В Angular 22 этот подход окончательно признан устаревшим legacy-синтаксисом. Современным стандартом стала функциональная инъекция через inject().
```typescript
  // Старый подход (Legacy ❌)
  constructor(private apiService: ApiService, private router: Router) {}
  
  // Современный подход (Angular 22)
  private apiService = inject(ApiService);
  private router = inject(Router);
```
## Почему inject() победил конструкторы?

   1. Чистота наследования класса: При наследовании TypeScript-классов больше нет необходимости прокидывать зависимости базового класса через ключевое слово super(apiService, router). Дочерний класс просто использует свои вызовы inject().
   2. Типизация и лаконичность: Переменные объявляются прямо в теле класса как обычные свойства. Код становится чище и читается сверху вниз.
   3. Динамический контекст: Функция inject() может использоваться внутри функциональных роутер-гвардов, интерцепторов и кастомных функций, где конструкторов классов физически не существует.

## 3. Иерархия инжекторов и область видимости сервисов (25 минут)
Каждый сервис снабжается декоратором ```@Injectable()```. Ключевой параметр в нем — ```providedIn```, который определяет область видимости и время жизни объекта:

* providedIn: 'root' (Глобальный Singleton): Сервис создается в единственном экземпляре на все приложение. Он живет всё время, пока открыта вкладка браузера. Идеально подходит для хранения сессии пользователя или глобальной корзины. Включает механизм Tree Shaking (если сервис нигде не вызван, компилятор вырежет его из сборки).
* Провайдинг на уровне Компонента (providers: [LocalService]): Экземпляр сервиса создается заново для каждого конкретного компонента. Когда компонент удаляется с экрана, этот экземпляр сервиса тоже уничтожается. Подходит для хранения временного состояния изолированной сложной формы или кастомного виджета.

## 4. Паттерн State Management (Управление состоянием) на Сигналах (25 минут)
В мире Zoneless-архитектуры Angular 22 тяжелые сторонние библиотеки управления состоянием (вроде NgRx или Akita) для большинства средних проектов стали избыточными. Комбинация Angular Services + Writable Signals позволяет написать предсказуемое, реактивное и легковесное хранилище данных (Store).
Правило хорошего тона (инкапсуляция состояния):

* Внутри сервиса Сигнал объявляется как приватный (#state = signal(...)), чтобы компоненты не могли хаотично перезаписывать его извне.
* Наружу сервис отдает этот Сигнал в режиме Read-Only (state = this.#state.asReadonly()).
* Изменение состояния происходит строго через публичные методы сервиса (например, addTask(), loadData()), инкапсулируя бизнес-логику в одном месте.

------------------------------
## ПРАКТИЧЕСКОЕ ЗАНЯТИЕ 7: Разработка глобального хранилища состояния (Signals Store)
Продолжительность: 2 академических часа (90 минут)
Цель работы: Избавиться от хранения данных внутри компонентов, создать глобальный синглтон-сервис для управления состоянием задач/курсов с использованием функции inject(), реализовать инкапсуляцию реактивного состояния через #private поля и метод .asReadonly().
## Шаг 1: Создание и проектирование архитектуры Сервиса (25 минут)
Создать глобальный сервис управления состоянием (например, для таск-менеджера или каталога обучения):
```bash
  ng g s services/task-store
```
Открыть task-store.service.ts и реализовать безопасное хранилище на Сигналах с использованием современных стандартов приватных полей JS (#):
```typescript
  import { Injectable, signal, computed } from '@angular/core';
  export interface Task {
    id: number;
    title: string;
    completed: boolean;
  }

  @Injectable({
    providedIn: 'root' // Глобальный синглтон, доступный везде
  })
  export class TaskStoreService {
    // 1. Приватное состояние (никто снаружи не может сделать .set() напрямую)
    #tasksState = signal<Task[]>([
      { id: 1, title: 'Изучить архитектуру DI в Angular 22', completed: true },
      { id: 2, title: 'Реализовать глобальный Signals Store', completed: false }
    ]);
  
    // 2. Публичный Сигнал только для чтения (компоненты вызывают его в шаблонах)
    tasks = this.#tasksState.asReadonly();
  
    // 3. Вычисляемые глобальные метрики через computed
    totalCount = computed(() => this.tasks().length);
    completedCount = computed(() => this.tasks().filter(t => t.completed).length);
  
    // 4. Публичные методы (Action) для контролируемого изменения состояния
    addTask(title: string): void {
      if (!title.trim()) return;
      const newTask: Task = { id: Date.now(), title, completed: false };
      
      // Иммутабельное обновление приватного сигнала
      this.#tasksState.update(current => [...current, newTask]);
    }
  
    toggleTask(id: number): void {
      this.#tasksState.update(current =>
        current.map(task => task.id === id ? { ...task, completed: !task.completed } : task)
      );
    }
  
    deleteTask(id: number): void {
      this.#tasksState.update(current => current.filter(task => task.id !== id));
    }
  }
```
## Шаг 2: Реализация Smart-компонента формы добавления (20 минут)
Компонент будет брать метод addTask напрямую из глобального сервиса, используя inject().
```bash
  ng g c components/task-form
```
Открыть task-form.component.ts:
```typescript
  import { Component, ChangeDetectionStrategy, inject, signal } from '@angular/core';
  import { TaskStoreService } from '../../services/task-store.service';
  
  @Component({
    selector: 'app-task-form',
    template: `
      <div class="form-box">
        <input type="text" [value]="inputVal()" (input)="onInput($event)" placeholder="Новая лабораторная...">
        <button (click)="submit()">Добавить в план</button>
      </div>
    `,
    styleUrl: './task-form.component.scss',
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class TaskFormComponent {
    // Внедряем глобальный стор через функциональный inject()
    private store = inject(TaskStoreService);
    
    inputVal = signal('');
  
    onInput(e: Event): void {
      this.inputVal.set((e.target as HTMLInputElement).value);
    }
  
    submit(): void {
      this.store.addTask(this.inputVal());
      this.inputVal.set(''); // Очищаем поле ввода
    }
  }
```
## Шаг 3: Реализация Smart-компонента списка (20 минут)
Этот компонент будет подписываться на Read-Only Сигнал задач из сервиса.
```bash
  ng g c components/task-dashboard
```
Открыть task-dashboard.component.ts:
```typescript
  import { Component, ChangeDetectionStrategy, inject } from '@angular/core';
  import { TaskStoreService } from '../../services/task-store.service';

  @Component({
    selector: 'app-task-dashboard',
    templateUrl: './task-dashboard.component.html',
    styleUrl: './task-dashboard.component.scss',
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class TaskDashboardComponent {
    // Внедряем хранилище. Компонент имеет доступ только к публичным методам и Read-Only сигналам
    protected store = inject(TaskStoreService);
  }

Открыть шаблон task-dashboard.component.html:
```html
  <div class="dashboard-box">
    <h3>Мониторинг прогресса студента</h3>
    <p>Выполнено задач: {{ store.completedCount() }} из {{ store.totalCount() }}</p>
  
    <ul class="task-list">
      <!-- Напрямую читаем Read-Only Сигнал из сервиса -->
      @for (item of store.tasks(); track item.id) {
        <li [class.done]="item.completed">
          <span (click)="store.toggleTask(item.id)">{{ item.title }}</span>
          <button (click)="store.deleteTask(item.id)">❌</button>
        </li>
      } @empty {
        <p>Все задачи выполнены! Отличная работа.</p>
      }
    </ul>
  </div>
```
## Шаг 4: Сборка экрана и проверка реактивности в OnPush (15 минут)

   1. Подключить оба компонента (TaskFormComponent и TaskDashboardComponent) в импорты корневого AppComponent.
   2. Добавить теги разметки в app.component.html:
   ```html
   <app-task-form />
   <app-task-dashboard />
   ```
   3. Запустить проект.

## Шаг 5: Анализ архитектуры (10 минут)
Компоненты TaskFormComponent и TaskDashboardComponent вообще никак не связаны друг с другом напрямую (у них нет общих input() или output()). Они находятся на одном уровне и полностью изолированы.
Однако за счет того, что они внедряют один и тот же Singleton-сервис TaskStoreService, при добавлении задачи через форму список во втором компоненте обновляется мгновенно. Архитектурная изоляция OnPush соблюдена идеально.

------------------------------
## Домашнее задание к Неделе 8

   1. Добавить в TaskStoreService новый computed сигнал progressPercentage, который рассчитывает процент выполнения задач от общего количества (и обработать деление на ноль, если массив пуст). Вывести эту метрику в интерфейс в виде красивого прогресс-бара.
   2. Ознакомиться с базовыми концепциями асинхронного реактивного программирования библиотеки RxJS (Observable, Подписка). Подготовиться к пониманию того, чем поток RxJS концептуально отличается от Angular Сигнала.

------------------------------

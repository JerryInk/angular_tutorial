## Неделя 9. Работа с HTTP. Интеграция с готовым API
------------------------------
## ЛЕКЦИЯ 9: Клиент-серверное взаимодействие и модуль HttpClient в Angular 22
Продолжительность: 2 академических часа (90 минут)
## 1. Архитектура сетевого слоя в Angular 22 (20 минут)
В Angular 22 для выполнения асинхронных сетевых запросов по протоколу HTTP используется встроенный сервис HttpClient. Архитектурно сетевой слой строится на базе паттерна Сервис-Поставщик (Data Service).
Компоненты никогда не должны делать запросы к серверу напрямую через fetch или axios. Логика работы с сетью полностью делегируется изолированным сервисам.
Схема взаимодействия:

[ Шаблон HTML ] ◄--- (Сигнал) --- [ Компонент TS ] ◄--- (inject) --- [ API Сервис ] ◄--- (HTTP) --- [ Бэкенд REST API ]

## 2. Конфигурация и функции HttpClient (20 минут)
В современном Angular полностью упразднен старый модуль HttpClientModule. Конфигурация сетевого клиента выполняется на глобальном уровне в файле конфигурации приложения app.config.ts с помощью функционального провайдера provideHttpClient().
Все методы HttpClient (get, post, put, delete) являются строго типизированными и по умолчанию возвращают холодные потоки RxJS Observable. Поток автоматически завершается (complete) сразу после того, как сервер присылает ответ, поэтому ручная отписка от одиночных HTTP-запросов обычно не требуется.
Типизированный запрос возвращает строго описанный интерфейс:
```typescript
  this.http.get<Course[]>('https://example.com')
```
## 3. Реализация CRUD операций (25 минут)
CRUD (Create, Read, Update, Delete) — это четыре базовые функции для работы с данными на сервере:

* READ (GET): Получение данных. Ответ от сервера мапится в интерфейс TypeScript.
* CREATE (POST): Отправка нового объекта на сервер в теле запроса (request body). В ответ бэкенд обычно возвращает созданный объект с присвоенным ID базы данных.
* UPDATE (PUT / PATCH): Изменение существующей записи по её идентификатору (например, /api/tasks/12).
* DELETE (DELETE): Удаление ресурса на сервере.

------------------------------
## Новое в HTTP: Метод QUERY (RFC 10008, Июнь 2026)

До недавнего времени при проектировании сложных поисковых фильтров или тяжелых аналитических отчетов разработчики сталкивались с дилеммой:

* Использовать GET: Семантически верно (мы просто читаем данные). Но GET не поддерживает отправку тела запроса (Request Body). Приходилось кодировать огромные структуры фильтрации в строку URL, которая имеет жесткие ограничения по длине в браузерах и прокси-серверах.
* Использовать POST: Позволяет передать сложный JSON-объект с фильтрами в теле запроса. Но POST семантически означает создание нового ресурса. Из-за этого такие запросы по умолчанию не кэшируются, а браузеры не могут безопасно повторять их при сбое сети.

Решение: Метод QUERY объединяет плюсы обоих подходов. Он позволяет передавать тело запроса (как POST), но сохраняет семантику безопасного и идемпотентного чтения (как GET). Ответы на запросы QUERY могут полноценно кэшироваться на уровне браузера и CDN.

------------------------------

## 4. Обработка сетевых ошибок (25 минут)
Запросы к бэкенду могут завершаться неудачно (ошибки валидации, отсутствие авторизации 401, падение сервера 500, отсутствие сети). В RxJS-потоках HttpClient для перехвата и изоляции таких сбоев используется специализированный оператор catchError.
```typescript
  import { catchError, throwError } from 'rxjs';

  this.http.get('/api/data').pipe(
    catchError(error => {
      console.error('Произошла сетевая ошибка:', error);
      // Возвращаем поток с ошибкой дальше для обработки в компоненте
      return throwError(() => new Error('Сервер временно недоступен.')); 
    })
  );
```
------------------------------
## ПРАКТИЧЕСКОЕ ЗАНЯТИЕ 9: Интеграция приложения с базовым REST API бэкенда
Продолжительность: 2 академических часа (90 минут)
Цель работы: Подключить сетевой клиент Angular 22, разработать специализированный сервис для выполнения асинхронных CRUD-запросов к готовому бэкенду и вывести полученные данные на экран в OnPush-компоненте с помощью Сигналов.
## Шаг 1: Глобальный провайдинг HttpClient (10 минут)

   1. Открыть конфигурационный файл проекта src/app/app.config.ts.
   2. Импортировать и добавить функцию provideHttpClient() в массив провайдеров:
```typescript
  import { ApplicationConfig, provideZonelessChangeDetection } from '@angular/core';
  import { provideRouter } from '@angular/router';
  import { provideHttpClient } from '@angular/common/http'; // Импорт сетевого клиента
  import { routes } from './app.routes';
  
  export const appConfig: ApplicationConfig = {
    providers: [
      provideZonelessChangeDetection(),
      provideRouter(routes),
      provideHttpClient() // Регистрируем HttpClient глобально
    ]
  };
```
## Шаг 2: Создание типизированного API Сервиса (30 минут)

   1. Сгенерировать новый сервис для работы с бэкенд-сущностью (например, Course или Task в зависимости от выбранного проекта):
```bash
  ng g s services/course-api
```
   1. Открыть course-api.service.ts и написать методы взаимодействия с сервером с использованием функционального inject():
```typescript
  import { Injectable, inject } from '@angular/core';
  import { HttpClient } from '@angular/common/http';
  import { Observable, catchError, throwError } from 'rxjs';

  export interface Course {
    id?: number; // ID опционален при создании, но обязателен от сервера
    title: string;
    hours: number;
  }

  @Injectable({
    providedIn: 'root'
  })
  export class CourseApiService {
    private http = inject(HttpClient); // Внедряем HttpClient
    
    // URL запущенного локально базового бэкенд-приложения студентов
    private apiUrl = 'http://localhost:3000/api/courses'; 
  
    // GET: Получить все курсы
    getCourses(): Observable<Course[]> {
      return this.http.get<Course[]>(this.apiUrl).pipe(
        catchError(this.handleError)
      );
    }
  
    // POST: Добавить новый курс
    createCourse(course: Course): Observable<Course> {
      return this.http.post<Course>(this.apiUrl, course).pipe(
        catchError(this.handleError)
      );
    }
  
    // DELETE: Удалить курс по ID
    deleteCourse(id: number): Observable<void> {
      return this.http.delete<void>(`${this.apiUrl}/${id}`).pipe(
        catchError(this.handleError)
      );
    }
  
    // Централизованный обработчик ошибок внутри сервиса
    private handleError(error: any) {
      console.error('Ошибка сетевого уровня:', error);
      return throwError(() => new Error('Ошибка взаимодействия с бэкендом. Повторите попытку позже.'));
    }
  }
```
## Шаг 3: Связывание API с компонентом через Сигналы (25 минут)
Компонент вызывает асинхронные методы сервиса в хуке ngOnInit, получает RxJS поток и конвертирует данные в Сигнал для OnPush отображения.
```bash
  ng g c components/course-manager
```
Открыть course-manager.component.ts:
```typescript
  import { Component, OnInit, ChangeDetectionStrategy, inject, signal } from '@angular/core';
  import { CourseApiService, Course } from '../../services/course-api.service';
  import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

  @Component({
    selector: 'app-course-manager',
    templateUrl: './course-manager.component.html',
    styleUrl: './course-manager.component.scss',
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class CourseManagerComponent implements OnInit {
    private apiService = inject(CourseApiService);
  
    // Реактивное состояние UI на Сигналах
    courses = signal<Course[]>([]);
    errorMessage = signal<string | null>(null);
  
    ngOnInit(): void {
      this.loadAllCourses();
    }
  
    loadAllCourses(): void {
      this.apiService.getCourses().pipe(
        // Защита от утечки, если компонент закроется до ответа сервера
        takeUntilDestroyed() 
      ).subscribe({
        next: (data) => {
          this.courses.set(data);
          this.errorMessage.set(null);
        },
        error: (err) => this.errorMessage.set(err.message)
      });
    }
  
    addNewCourse(): void {
      const mockCourse: Course = { title: 'Архитектура баз данных', hours: 48 };
      
      this.apiService.createCourse(mockCourse).subscribe({
        next: (createdItem) => {
          // Иммутабельно добавляем пришедший с сервера объект (с валидным ID) в Сигнал
          this.courses.update(current => [...current, createdItem]);
        },
        error: (err) => this.errorMessage.set(err.message)
      });
    }
  
    removeCourse(id: number | undefined): void {
      if (!id) return;
  
      this.apiService.deleteCourse(id).subscribe({
        next: () => {
          // Удаляем из UI только после успешного удаления в базе бэкенда
          this.courses.update(current => current.filter(c => c.id !== id));
        },
        error: (err) => this.errorMessage.set(err.message)
      });
    }
  }
```
## Шаг 4: Разметка шаблона (15 минут)
Открыть course-manager.component.html:
```html
  <div class="manager-box">
    <h2>Панель интеграции с REST API бэкенда</h2>
    
    <button (click)="addNewCourse()" class="btn-add">➕ Сгенерировать и отправить курс</button>
  
    @if (errorMessage()) {
      <div class="error-banner">❌ {{ errorMessage() }}</div>
    }
  
    <div class="api-list">
      @for (item of courses(); track item.id) {
        <div class="api-item">
          <span><strong>ID: {{ item.id }}</strong> — {{ item.title }} ({{ item.hours }} ч.)</span>
          <button (click)="removeCourse(item.id)" class="btn-del">Удалить</button>
        </div>
      } @empty {
        <p>Данные на сервере отсутствуют или бэкенд не запущен.</p>
      }
    </div>
  </div>
```
## Шаг 5: Запуск сквозного тестирования (10 минут)

   1. Подключить ```<app-course-manager />``` в app.component.ts.
   2. Запустить локальный Node.js/JSON-Server бэкенд (базовая часть).
   3. Проверить отправку запросов во вкладке Network в DevTools браузера. Должны увидеть реальные сетевые GET, POST и DELETE запросы и синхронизацию состояния базы данных с OnPush-интерфейсом Angular 22.

------------------------------
## Домашнее задание к Неделе 10

   1. Доработать компонент: реализовать отправку данных из реальных текстовых инпутов, а не мок-объекта mockCourse.
   2. Изучить концепцию HTTP Интерцепторов (Interceptors). Выяснить, как автоматически прикреплять заголовки ко всем исходящим запросам приложения.

------------------------------

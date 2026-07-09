## Неделя 10. Продвинутый HTTP. Интерцепторы
------------------------------
## ЛЕКЦИЯ 10: Функциональные HTTP-интерцепторы и централизованная обработка данных в Angular 22
Продолжительность: 2 академических часа (90 минут)

## 1. Эволюция перехватчиков: Переход к функциональным интерцепторам (25 минут)
До версии Angular 15 интерцепторы описывались как классы, реализующие интерфейс ```HttpInterceptor```. Это требовало написания большого количества шаблонного кода (boilerplate) и усложняло их динамическую сборку.
В Angular 22 функциональные интерцепторы (Functional Interceptors) стали единственным стандартом.

* Что это такое: Это чистая функция TypeScript, которая принимает два аргумента: исходящий запрос ```HttpRequest``` и функцию следующего шага в цепочке обработчиков ```HttpHandlerFn```.
* Плюсы: Они работают значительно быстрее, легче поддаются Tree Shaking компилятором, а для их внедрения используются преимущества функционального DI через ```inject()```.

## 2. Модификация запросов и иммутабельность HttpRequest (20 минут)
Исходящий объект ```HttpRequest``` является строго иммутабельным (изменяемым только через копирование). Вы не можете напрямую перезаписать заголовок, вызвав ```request.headers.set()```.
Чтобы добавить токен авторизации или кастомный заголовок, разработчик обязан использовать метод ```.clone()```. Этот метод создает глубокую копию запроса и накатывает на нее указанные изменения:
```typescript
  const secureReq = req.clone({
    setHeaders: {
      Authorization: `Bearer ${authToken}`
    }
  });
```

## 3. Архитектура сквозных задач (Cross-Cutting Concerns) (25 минут)
Интерцепторы идеально подходят для выноса сквозной логики, которая должна работать для всего приложения автоматически:

   1. Глобальный Лоадер (Индикатор загрузки): Перехватчик увеличивает счетчик активных запросов при старте сети и уменьшает его при завершении потока. Компоненты OnPush просто слушают Сигнал счетчика и показывают спиннер.
   2. Логирование и Метрики: Подсчет точного времени ответа сервера для каждого URL.
   3. Авторизация: Проброс JWT-токенов из хранилища во все запросы к базовому бэкенду.

## 4. Централизованный перехват серверных ошибок (20 минут)
Вместо того чтобы писать блоки ```catchError``` в каждом компоненте или API-сервисе, в функциональном интерцепторе можно организовать глобальный перехват сбоев.

* Если бэкенд возвращает код 401 Unauthorized — интерцептор автоматически вызывает метод разлогирования и перенаправляет пользователя на страницу входа.
* Если возвращается 500 Server Error — на экран выводится глобальное push-уведомление.

------------------------------
## ПРАКТИЧЕСКОЕ ЗАНЯТИЕ 10: Создание функциональных интерцепторов для авторизации и глобального спиннера
Продолжительность: 2 академических часа (90 минут)
Цель работы: Написать и зарегистрировать два функциональных интерцептора в Angular 22, реализовать логику автоматической подстановки токенов авторизации и создать реактивный индикатор загрузки сети на Сигналах.

## Шаг 1: Разработка Сервиса Глобального Лоадера (15 минут)
Создадим легковесный сервис, который будет хранить состояние сети в виде Сигнала.
```bash
  ng g s services/loading-status
```
Открыть loading-status.service.ts:
```typescript
  import { Injectable, signal, computed } from '@angular/core';

  @Injectable({
    providedIn: 'root'
  })
  export class LoadingStatusService {
    // Счетчик активных параллельных HTTP-запросов
    #activeRequests = signal<number>(0);
  
    // Read-Only флаг: показывать ли лоадер на экране
    isLoading = computed(() => this.#activeRequests() > 0);
  
    show(): void {
      this.#activeRequests.update(count => count + 1);
    }
  
    hide(): void {
      this.#activeRequests.update(count => Math.max(0, count - 1));
    }
  }
```

## Шаг 2: Создание функциональных интерцепторов (30 минут)
Создадим два файла интерцепторов вручную (или через CLI, если поддерживается генерация функций):
```bash
ng g interceptor interceptors/auth
ng g interceptor interceptors/loading
```
   1. Открыть auth.interceptor.ts и реализовать подстановку токена авторизации:
```typescript
  import { HttpInterceptorFn } from '@angular/common/http';

  export const authInterceptor: HttpInterceptorFn = (req, next) => {
    console.log('--- Auth Interceptor сработал ---');
    
    // Имитируем получение токена (в реальном проекте тут будет inject(AuthService))
    const token = 'STUDENT_TEST_JWT_TOKEN_v22';
  
    // Иммутабельно клонируем запрос, добавляя заголовок авторизации
    const authReq = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
  
    // Передаем измененный запрос дальше по цепочке
    return next(authReq);
  };
```

   2. Открыть loading.interceptor.ts и настроить управление лоадером через RxJS оператор ```finalize```:
```typescript
  import { HttpInterceptorFn } from '@angular/common/http';
  import { inject } from '@angular/core';
  import { LoadingStatusService } from '../services/loading-status.service';
  import { finalize } from 'rxjs';
  
  export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
    const loader = inject(LoadingStatusService); // Функциональный DI
  
    // Включаем спиннер перед стартом запроса
    loader.show();
  
    return next(req).pipe(
      // Оператор finalize срабатывает ВСЕГДА при завершении потока (успех или ошибка)
      finalize(() => {
        loader.hide(); // Гарантированно выключаем спиннер
      })
    );
  };
```

## Шаг 3: Глобальная регистрация в app.config.ts (15 минут)
Функциональные интерцепторы передаются как массив внутрь функции конфигурации ```withInterceptors()```.
Открыть src/app/app.config.ts:
```typescript
import { ApplicationConfig, provideZonelessChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http'; // Импортируем утилиту
import { routes } from './app.routes';
import { authInterceptor } from './interceptors/auth.interceptor';
import { loadingInterceptor } from './interceptors/loading.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZonelessChangeDetection(),
    provideRouter(routes),
    // Регистрируем HttpClient со списком перехватчиков в строгом порядке их вызова
    provideHttpClient(
      withInterceptors([authInterceptor, loadingInterceptor])
    )
  ]
};
```

## Шаг 4: Создание UI Компонента Спиннера (15 минут)
Создать визуальный индикатор, который автоматически скрывается и показывается в зависимости от состояния сети.
```bash
  ng g c components/global-spinner
```
Открыть global-spinner.component.ts:
```typescript
  import { Component, ChangeDetectionStrategy, inject } from '@angular/core';
  import { LoadingStatusService } from '../../services/loading-status.service';
  
  @Component({
    selector: 'app-global-spinner',
    template: `
      <!-- Работаем с Сигналом в OnPush режиме -->
      @if (statusService.isLoading()) {
        <div class="spinner-overlay">
          <div class="loader-ring"></div>
          <p>Связь с базовым бэкендом...</p>
        </div>
      }
    `,
    styleUrl: './global-spinner.component.scss',
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class GlobalSpinnerComponent {
    protected statusService = inject(LoadingStatusService);
  }
```
## Шаг 5: Проверка сквозной работы системы (15 минут)

   1. Разместить тег ```<app-global-spinner />``` в самом верху шаблона app.component.html.
   2. Запустить проект и выполнить любой HTTP-запрос к бэкенду (созданный на Неделе 9).
   3. Увидим: в момент клика на экране на долю секунды всплывает оверлей загрузки.
   4. Открыть вкладку Network в браузере, кликнуть по запросу и убедиться, что в блоке Request Headers у каждого исходящего запроса теперь автоматически присутствует строчка ```Authorization: Bearer STUDENT_TEST_JWT_TOKEN_v22```. Логика полностью инкапсулирована от компонентов.

------------------------------
## Домашнее задание к Неделе 11

   1. Добавить в ```authInterceptor``` логику исключения: если запрос отправляется на сторонний открытый эндпоинт (например, '://dicebear.com' для аватарок), токен авторизации прикреплять не нужно.
   2. Повторить концепцию маршрутизации в SPA. Узнать, как работает навигация без перезагрузки вкладки браузера.

------------------------------

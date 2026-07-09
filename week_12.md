## Неделя 12. Ленивая загрузка и Защита маршрутов (Guards)
------------------------------
## ЛЕКЦИЯ 12: Оптимизация бандла через Lazy Loading и безопасность путей в Angular 22
Продолжительность: 2 академических часа (90 минут)

## 1. Проблема больших SPA-приложений и концепция Lazy Loading (20 минут)
По умолчанию, если все компоненты импортированы в массив маршрутов напрямую через стандартный JavaScript-импорт (```import { Component } from '...'```), сборщик компилирует их в один массивный файл — главный чанк приложения (main.js).

* Проблема: Когда пользователь заходит на главную страницу, его браузер вынужден скачивать код абсолютно всего приложения, включая тяжелую панель администратора, графики и настройки, которые ему могут никогда не понадобиться. Это критически ухудшает метрику First Contentful Paint (FCP).
* Решение: Ленивая загрузка (Lazy Loading). Это подход, при котором код конкретной страницы компилируется в отдельный изолированный JavaScript-файл (чанк) и скачивается браузером по сети строго в тот момент, когда пользователь кликает по ссылке и переходит на этот маршрут.

## 2. Ленивая загрузка Standalone-компонентов через loadComponent (20 минут)
В Angular 22 благодаря Standalone-архитектуре ленивая загрузка страниц стала декларативной и нативной. Старый синтаксис ```loadChildren``` для модулей больше не используется для одиночных экранов. Вместо свойства ```component``` в объекте маршрута используется свойство ```loadComponent```, принимающее стрелочную функцию с динамическим импортом ```import()```:
```typescript
  // Классический импорт (Скачивается СРАЗУ):
  { path: 'profile', component: ProfileComponent }
  // Ленивый импорт (Скачивается ТОЛЬКО при переходе):
  { path: 'profile', loadComponent: () => import('./pages/profile/profile.component').then(m => m.ProfileComponent) }
```
Сборщик на базе Esbuild автоматически видит динамические импорты и на этапе сборки (```ng build```) нарезает приложение на микро-чанки.

## 3. Безопасность на клиенте: Функциональные Guards (25 минут)
Маршрутизатор позволяет не только загружать страницы, но и контролировать доступ к ним. Для этого используются Гварды (Guards). Они отвечают на вопрос: «Имеет ли право текущий пользователь открыть этот URL?».
В Angular 22 гварды-классы полностью удалены из фреймворка. Им на смену пришли Функциональные гварды (Functional Guards), представляющие собой чистые функции типа ```CanActivateFn```.
```typescript
  export const authGuard: CanActivateFn = (route, state) => {
    const authService = inject(AuthService); // Функциональный DI внутри функции!
    return authService.isLoggedIn() ? true : inject(Router).createUrlTree(['/login']);
  };
```
Гард может возвращать:

* ```true``` — доступ разрешен, роутер монтирует компонент.
* ```false``` — доступ заблокирован, навигация отменяется.
* ```UrlTree``` — объект перенаправления (например, принудительный редирект на страницу '/login'), создаваемый через ```router.createUrlTree()```.

## 4. Архитектурная ремарка: Безопасность SPA vs Безопасность Бэкенда (25 минут)
Важно зафиксировать фундаментальное правило безопасности веб-приложений: Любая защита маршрутов на клиенте (в Angular) делается исключительно ради User Experience (UX), а не ради истинной безопасности данных.
Поскольку весь JavaScript-код Angular-приложения выполняется в браузере пользователя, злоумышленник теоретически может подменить значение гварда в консоли или перехватить состояние Сигнала авторизации, чтобы заставить интерфейс отобразить скрытую админку.

* Истинная безопасность данных должна обеспечиваться строго на бэкенде (путем проверки JWT-токена в заголовках каждого REST API запроса).
* Гвард в Angular нужен лишь для того, чтобы легитимный пользователь случайно не забрел на закрытую страницу, и чтобы приложение красиво скрыло UI, не тратя трафик на загрузку ленивого чанка.

------------------------------
## ПРАКТИЧЕСКОЕ ЗАНЯТИЕ 12: Настройка Lazy Loading и защита Личного кабинета гвардом
Продолжительность: 2 академических часа (90 минут)
Цель работы: Перевести архитектуру страниц проекта на ленивую загрузку (Lazy Loading), разработать функциональный гард ```canActivate``` для проверки авторизации пользователя и реализовать безопасный редирект с помощью ```UrlTree```.

## Шаг 1: Создание страницы авторизации и сервиса сессии (20 минут)

   1. Сгенерировать страницу входа и ленивый личный кабинет студента:
```bash
  ng g c pages/login --standalone
  ng g c pages/account --standalone
```

   1. Создать легковесный сервис авторизации src/app/services/auth.service.ts:
```typescript
  import { Injectable, signal } from '@angular/core';
  
  @Injectable({
    providedIn: 'root'
  })
  export class AuthService {
    // Сигнал, хранящий статус авторизации пользователя (UX-состояние)
    isAuthorized = signal<boolean>(false);
  
    login(): void {
      this.isAuthorized.set(true);
    }
  
    logout(): void {
      this.isAuthorized.set(false);
    }
  }
```

## Шаг 2: Написание функционального гарда (25 минут)
Создадим файл защиты маршрутов вручную в папке src/app/guards/auth.guard.ts. Функция будет использовать функциональный ```inject()``` для получения данных из созданного ```AuthService```.
```typescript
  import { inject } from '@angular/core';
  import { CanActivateFn, Router } from '@angular/router';
  import { AuthService } from '../services/auth.service';

  export const authGuard: CanActivateFn = (route, state) => {
    const authService = inject(AuthService);
    const router = inject(Router);
  
    console.log(`--- Гломальный глоссарий: проверка доступа к пути: ${state.url} ---`);
  
    if (authService.isAuthorized()) {
      return true; // Сигнал равен true -> пускаем пользователя на ленивый компонент
    } else {
      console.warn('Доступ запрещен. Перенаправление на /login');
      // Возвращаем UrlTree для мгновенного и безопасного редиректа
      return router.createUrlTree(['/login']); 
    }
  };
```

## Шаг 3: Конфигурация путей с Lazy Loading и Guard (25 минут)
Открыть файл маршрутов src/app/app.routes.ts и переписать его, внедрив динамические импорты и защиту массива ```canActivate```:
```typescript
  import { Routes } from '@angular/router';
  import { authGuard } from './guards/auth.guard';
  
  export const routes: Routes = [
    { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
  
    // Старый импорт заменяем на ленивый loadComponent
    { 
      path: 'dashboard', 
      loadComponent: () => import('./pages/home-dashboard/home-dashboard.component').then(m => m.HomeDashboardComponent) 
    },
    
    { 
      path: 'login', 
      loadComponent: () => import('./pages/login/login.component').then(m => m.LoginComponent) 
    },
  
    // Защищенный маршрут Личного кабинета с ленивой загрузкой кода
    { 
      path: 'account', 
      loadComponent: () => import('./pages/account/account.component').then(m => m.AccountComponent),
      canActivate: [authGuard] // Применяем наш функциональный гард
    }
  ];
```

## Шаг 4: Интеграция кнопок входа/выхода в Login-компонент (10 минут)
Открыть login.component.ts, внедрить ```AuthService``` и сделать простую разметку для переключения статуса авторизации, чтобы студенты могли протестировать гард:
```typescript
  import { Component, inject } from '@angular/core';
  import { AuthService } from '../../services/auth.service';
  
  @Component({
    selector: 'app-login',
    standalone: true,
    template: `
      <div class="login-page">
        <h3>Страница авторизации учебного портала</h3>
        <p>Статус: <strong>{{ authService.isAuthorized() ? 'Авторизован' : 'Гость' }}</strong></p>
        
        @if (!authService.isAuthorized()) {
          <button (click)="authService.login()">Имитировать успешный Вход</button>
        } @else {
          <button (click)="authService.logout()">Выйти из системы</button>
        }
      </div>
    `
  })
  export class LoginComponent {
    protected authService = inject(AuthService);
  }
```
Добавить в app.component.html навигационную ссылку ```<a routerLink="/account">Личный кабинет</a>``` для тестирования перехода.

## Шаг 5: Инспекция сетевых чанков в браузере (10 минут)

   1. Запустить dev-сервер: ```ng serve```.
   2. Открыть вкладку Network в инструментах разработчика браузера, отфильтровать по типу JS и очистить лог.
   3. Кликнуть на вкладку «Каталог курсов» ('/dashboard'). Должны увидеть, как в Network в этот самый момент подгрузился отдельный файл вида home-dashboard-XXXXX.js. Это доказывает работу Lazy Loading.
   4. Кликнуть на «Личный кабинет» ('/account'). Поскольку ```isAuthorized``` равен ```false```, сработает гвард, отменит загрузку компонента аккаунта и мгновенно перекинет пользователя на '/login' — при этом ленивый файл account.component.js даже не скачается по сети, экономя трафик.
   5. Нажать кнопку «Имитировать успешный Вход», повторно кликнуть на «Личный кабинет» — гард вернет ```true```, чанк кабинета скачается, и страница успешно отобразится.

------------------------------
## Домашнее задание к Неделе 13

   1. Добавить в шапку сайта (app.component.html) отображение имени пользователя, если он авторизован, скрывая кнопку перехода в личный кабинет, если он гость (использовать Сигналы и блок ```@if```).
   2. Изучить разницу между шаблонными (Template-driven) и реактивными (Reactive) формами в Angular. Подготовиться к изучению архитектуры Reactive Forms.

------------------------------

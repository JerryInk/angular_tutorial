## Неделя 14. Валидация форм
------------------------------
## ЛЕКЦИЯ 14: Проверка корректности данных и кастомная валидация в Angular 22
Продолжительность: 2 академических часа (90 минут)

## 1. Концепция валидации в реактивных формах (20 минут)
Валидация в Reactive Forms реализуется как декларативный набор правил, который передается в конфигурацию ```FormControl``` на этапе инициализации в TypeScript. В отличие от HTML-валидации (например, атрибута ```required``` в теге), валидаторы Angular — это чистые функции, которые полностью контролируются кодом приложения. Они гарантируют, что форма не будет отправлена на бэкенд, пока все её элементы не перейдут в статус ```VALID```.
Каждая проверка выполняется синхронно или асинхронно при любом изменении значения инпута. Результат записывается в объект ошибок ```control.errors```.

## 2. Встроенные валидаторы класса ```Validators``` (20 минут)
Angular предоставляет из коробки мощный класс ```Validators```, содержащий стандартные инженерные проверки:

* ```Validators.required``` — поле не должно быть пустым.
* ```Validators.minLength(n)``` / ```Validators.maxLength(n)``` — контроль длины строки.
* ```Validators.min(n)``` / ```Validators.max(n)``` — числовые ограничения.
* ```Validators.email``` — проверка соответствия регулярному выражению электронной почты.
* ```Validators.pattern(regex)``` — валидация по кастомному регулярному выражению (например, для проверки телефона или пароля).

Применение нескольких валидаторов к одному полю оборачивается в массив:
```typescript
  title: this.fb.control('', [Validators.required, Validators.minLength(3)])
```

## 3. Управление выводом ошибок на основе состояния инпута (25 минут)
Показывать ошибку сразу, как только пользователь открыл страницу, — плохой UX. Форма изначально пустая и подсветится красным.
Идеальное правило для интерфейсов: выводить ошибку только тогда, когда поле некорректно (```invalid```) **И** пользователь уже повзаимодействовал с ним.
Для этого проверяются флаги состояния, которые в Angular 22 доступны как реактивные сигналы:
```html
  @if (control.invalid && (control.touched || control.dirty)) {
    <!-- Показываем ошибку -->
  }
```

## 4. Разработка кастомных синхронных валидаторов (25 минут)
Стандартных валидаторов часто не хватает для специфических ИТ-задач (например, запретить использовать слово "test", проверить совпадение паролей или ограничить ввод только четными числами).
Кастомный валидатор — это функция, которая принимает на вход ```AbstractControl``` и возвращает:

* ```ValidationErrors (объект вида { key: value })```, если проверка не пройдена.
* ```null```, если данные корректны и проверка успешно пройдена.

Пример функции:
```typescript
  export function forbiddenWordsValidator(control: AbstractControl): ValidationErrors | null {
    const isForbidden = control.value?.includes('spam');
    return isForbidden ? { forbiddenWord: { value: control.value } } : null;
  }
```

------------------------------
## ПРАКТИЧЕСКОЕ ЗАНЯТИЕ 14: Реализация комплексной валидации формы с выводом ошибок
Продолжительность: 2 академических часа (90 минут)
Цель работы: Настроить встроенную валидацию для формы создания курса/задачи, написать собственный кастомный валидатор для проверки ИТ-кодов и реализовать реактивную блокировку кнопки отправки в режиме OnPush с помощью Signal Forms.

## Шаг 1: Расширение модели формы валидаторами (20 минут)
Открываем компонент ```CourseFormComponent``` и добавляем правила валидации в ```FormBuilder```. Также напишем кастомную проверку, запрещающую вводить нецензурные или нерелевантные слова в название.
Открыть course-form.component.ts:
```typescript
  import { Component, ChangeDetectionStrategy, inject } from '@angular/core';
  import { ReactiveFormsModule, FormBuilder, Validators, AbstractControl, ValidationErrors } from '@angular/forms';
  
  // 1. Кастомный валидатор: название не должно содержать слово "черновик"
  export function draftValidator(control: AbstractControl): ValidationErrors | null {
    const hasDraftWord = control.value?.toLowerCase().includes('черновик');
    return hasDraftWord ? { isDraft: true } : null;
  }

  @Component({
    selector: 'app-course-form',
    standalone: true,
    imports: [ReactiveFormsModule],
    templateUrl: './course-form.component.html',
    styleUrl: './course-form.component.scss',
    changeDetection: ChangeDetectionStrategy.OnPush
  })
  export class CourseFormComponent {
    private fb = inject(FormBuilder);
  
    // Спроектируем форму с массивами встроенных и кастомных валидаторов
    courseForm = this.fb.group({
      title: this.fb.control<string>('', {
        nonNullable: true,
        validators: [Validators.required, Validators.minLength(5), draftValidator] // Кастомный + Встроенные
      }),
      hours: this.fb.control<number>(16, {
        nonNullable: true,
        validators: [Validators.required, Validators.min(16), Validators.max(120)]
      }),
      level: this.fb.control<string>('beginner', { nonNullable: true })
    });
  
    // Удобные геттеры для краткого обращения к полям в HTML-шаблоне
    get titleControl() { return this.courseForm.controls.title; }
    get hoursControl() { return this.courseForm.controls.hours; }
  
    onSubmit(): void {
      if (this.courseForm.invalid) {
        this.courseForm.markAllAsTouched(); // Если форма невалидна, принудительно подсвечиваем все ошибки
        return;
      }
      console.log('Данные успешно прошли валидацию и отправлены:', this.courseForm.getRawValue());
      this.courseForm.reset();
    }
  }
```

## Шаг 2: Верстка шаблона с условным выводом ошибок (30 минут)
Открыть course-form.component.html и модифицировать поля. Используем новый Control Flow ```@if``` для проверки ошибок конкретного инпута, когда он находится в состоянии ```touched``` (пользователь вышел из поля).
```html
  <div class="form-container">
    <h3>Добавление нового ИТ-курса с валидацией данных</h3>
  
    <form [formGroup]="courseForm" (ngSubmit)="onSubmit()">
      
      <!-- Поле: Название -->
      <div class="input-field" [class.has-error]="titleControl.invalid && titleControl.touched">
        <label for="title">Название дисциплины (минимум 5 симв.):</label>
        <input id="title" type="text" formControlName="title" placeholder="Введите название...">
        
        <!-- Проверка состояния через Signal-совместимые флаги формы -->
        @if (titleControl.invalid && titleControl.touched) {
          <div class="error-msg">
            @if (titleControl.errors?.['required']) { <span>Поле обязательно для заполнения.</span> }
            @if (titleControl.errors?.['minlength']) { <span>Название слишком короткое (нужно от 5 символов).</span> }
            @if (titleControl.errors?.['isDraft']) { <span class="draft-warning">Нельзя использовать слово "черновик" в официальном плане!</span> }
          </div>
        }
      </div>
  
      <!-- Поле: Часы -->
      <div class="input-field" [class.has-error]="hoursControl.invalid && hoursControl.touched">
        <label for="hours">Количество академических часов (от 16 до 120):</label>
        <input id="hours" type="number" formControlName="hours">
        
        @if (hoursControl.invalid && hoursControl.touched) {
          <div class="error-msg">
            @if (hoursControl.errors?.['min'] || hoursControl.errors?.['max']) {
              <span>Объем курса должен быть в диапазоне от 16 до 120 часов.</span>
            }
          </div>
        }
      </div>
  
      <!-- Кнопка отправки. Блокируется автоматически, если форма невалидна -->
      <button type="submit" [disabled]="courseForm.invalid" class="submit-btn">
        Сохранить на сервере
      </button>
    </form>
  </div>
```

## Шаг 3: Стилизация состояния ошибок (15 минут)
Добавить в course-form.component.scss стили для визуального выделения некорректных инпутов:
```css
.input-field {
  margin-bottom: 15px;
  display: flex;
  flex-direction: column;
  
  input {
    padding: 8px;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

  &.has-error {
    input { border-color: #dc3545; background-color: #fff8f8; }
    label { color: #dc3545; }
  }
}

.error-msg {
  color: #dc3545;
  font-size: 0.85rem;
  margin-top: 4px;
}

.submit-btn:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}
```

## Шаг 4: Тестирование граничных условий (15 минут)

   1. Запустить dev-сервер: ```ng serve```.
   2. Тест 1 (Обязательное поле): Кликнуть в текстовое поле названия, ничего не вводить, нажать клавишу Tab (уйти из фокуса). Поле должно мгновенно стать красным и вывести ошибку.
   3. Тест 2 (Валидация длины): Начать писать "Web". Ошибка ```required``` исчезнет, но появится ошибка ```minlength```. Дописать "Web-разработка" — поле станет валидным.
   4. Тест 3 (Кастомный валидатор): Написать "Черновик ИТ курса". Сработает кастомная бизнес-логика ```draftValidator```, кнопка отправки автоматически заблокируется.

## Шаг 5: Анализ производительности в OnPush (10 минут)
Кнопка «Сохранить на сервере» активируется и деактивируется точечно. Благодаря интеграции валидаторов со строгой типизацией и Сигналами в Angular 22, проверка происходит в Zoneless-режиме без единого лишнего пересчета соседних DOM-элементов.

------------------------------
## Домашнее задание к Неделе 15

   1. Добавить в форму чекбокс «Согласие с учебным планом». Сделать его обязательным для отправки формы с помощью ```Validators.requiredTrue```.
   2. Изучить, что такое Модульное тестирование (Unit Testing). Узнать, для чего нужна команда ```ng test``` и какие файлы с расширением .spec.ts генерирует Angular CLI.

------------------------------

## Содержание

- [Модуль 1 и 2](#модуль-1-и-2--er-диаграмма-и-база-данных)
- [Модуль 3](#модуль-3--sql-запрос-полная-стоимость-заказа)
- [Модуль 4](#модуль-4--laravel-приложение-вход-с-капчей)
- [Модуль 5](#модуль-5--проектная-документация-методы-и-параметры)
  - [Текст РЕГИСТРАЦИЯ](#текст-регистрация)
  - [Текст ВАЛИДАЦИЯ](#текст-валидация)
- [Модуль 6](#модуль-6--валидация-данных-от-эмулятора)

## Модуль 1 и 2 — ER-диаграмма и база данных

```sql
CREATE DATABASE moloko CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE moloko;

CREATE TABLE zakazchiki (
    id VARCHAR(20) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    inn VARCHAR(20),
    address VARCHAR(255),
    phone VARCHAR(20),
    salesman TINYINT(1) DEFAULT 0,
    buyer TINYINT(1) DEFAULT 0
);

CREATE TABLE produkty (
    id INT AUTO_INCREMENT PRIMARY KEY,
    kod VARCHAR(20) UNIQUE,
    nazvanie VARCHAR(255) NOT NULL,
    ed_izm VARCHAR(20),
    cena DECIMAL(10,2)
);

CREATE TABLE materialy (
    id INT AUTO_INCREMENT PRIMARY KEY,
    kod VARCHAR(20) UNIQUE,
    nazvanie VARCHAR(255) NOT NULL,
    ed_izm VARCHAR(20),
    cena DECIMAL(10,2)
);

CREATE TABLE spetsifikatsiya (
    id INT AUTO_INCREMENT PRIMARY KEY,
    produkt_id INT NOT NULL,
    material_id INT NOT NULL,
    kolichestvo DECIMAL(10,4) NOT NULL,
    FOREIGN KEY (produkt_id) REFERENCES produkty(id) ON DELETE CASCADE,
    FOREIGN KEY (material_id) REFERENCES materialy(id) ON DELETE CASCADE,
    UNIQUE KEY uniq_spec (produkt_id, material_id)
);

CREATE TABLE zakazy (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nomer VARCHAR(20),
    data DATE NOT NULL,
    zakazchik_id VARCHAR(20) NOT NULL,
    FOREIGN KEY (zakazchik_id) REFERENCES zakazchiki(id) ON DELETE CASCADE
);

CREATE TABLE sostav_zakaza (
    id INT AUTO_INCREMENT PRIMARY KEY,
    zakaz_id INT NOT NULL,
    produkt_id INT NOT NULL,
    kolichestvo INT NOT NULL,
    cena DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (zakaz_id) REFERENCES zakazy(id) ON DELETE CASCADE,
    FOREIGN KEY (produkt_id) REFERENCES produkty(id) ON DELETE CASCADE
);

CREATE TABLE proizvodstvo (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nomer VARCHAR(20),
    data DATE NOT NULL,
    produkt_id INT NOT NULL,
    kolichestvo INT NOT NULL,
    FOREIGN KEY (produkt_id) REFERENCES produkty(id) ON DELETE CASCADE
);

INSERT INTO zakazchiki (id, name, inn, address, phone, salesman, buyer) VALUES
('000000001', 'ООО "Поставка"', '', 'г.Пятигорск', '+79198634592', 1, 1),
('000000002', 'ООО "Кинотеатр Квант"', '26320045123', 'г. Железноводск, ул. Мира, 123', '+79884581555', 1, 0),
('000000008', 'ООО "Новый JDTO"', '26320045111', 'г. Железноводсу', '+79884581555', 1, 0),
('000000003', 'ООО "Ромашка"', '4140784214', 'г. Омск, ул. Строителей, 294', '+79882584546', 0, 1),
('000000009', 'ООО "Ипподром"', '5874045632', 'г. Уфа, ул. Набережная, 37', '+79627486389', 1, 1),
('000000010', 'ООО "Ассоль"', '2629011278', 'г. Калуга, ул. Пушкина, 94', '+79184572398', 0, 1);
```



## Модуль 3 — SQL-запрос (полная стоимость заказа)

```sql
SELECT 
    z.id AS zakaz_id,
    z.nomer AS nomer_zakaza,
    zk.name AS zakazchik,
    SUM(sz.kolichestvo * mat_cost.cena_materialov) AS polnaya_stoimost_zakaza
FROM zakazy z
JOIN zakazchiki zk ON zk.id = z.zakazchik_id
JOIN sostav_zakaza sz ON sz.zakaz_id = z.id
JOIN (
    SELECT 
        s.produkt_id,
        SUM(s.kolichestvo * m.cena) AS cena_materialov
    FROM spetsifikatsiya s
    JOIN materialy m ON m.id = s.material_id
    GROUP BY s.produkt_id
) AS mat_cost ON mat_cost.produkt_id = sz.produkt_id
WHERE z.id = 1   -- сюда вставляешь id нужного заказа
GROUP BY z.id, z.nomer, zk.name;
```

## Модуль 4 — Laravel-приложение: вход с капчей

### Шаг 1. Создать проект

```bash
composer create-project laravel/laravel demo
cd demo
composer require laravel/ui
php artisan ui bootstrap --auth
```

### Шаг 2. `resources/views/layouts/app.blade.php` — заменить всё

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ config('app.name', 'Laravel') }}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark mb-4">
        <div class="container">
            <a class="navbar-brand" href="/">{{ config('app.name', 'Laravel') }}</a>
            <div class="navbar-nav ms-auto">
                @guest
                    <a class="nav-link" href="{{ route('login') }}">Вход</a>
                    <a class="nav-link" href="{{ route('register') }}">Регистрация</a>
                @else
                    <a class="nav-link" href="{{ route('logout') }}"
                       onclick="event.preventDefault();
                       document.getElementById('logout-form').submit();">
                        Выйти
                    </a>
                    <form id="logout-form" action="{{ route('logout') }}" method="POST" class="d-none">
                        @csrf
                    </form>
                @endguest
            </div>
        </div>
    </nav>
    <main>
        @yield('content')
    </main>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

### Шаг 3. Скопировать фото капчи

Создай папку `public/images/` и скопируй туда 4 фото:

| Файл | Расположение кусочка |
|------|----------------------|
| `1.png` | верхний левый |
| `2.png` | верхний правый |
| `3.png` | нижний левый |
| `4.png` | нижний правый |

### Шаг 4. `resources/views/auth/login.blade.php` — заменить всё

```html
@extends('layouts.app')

@section('content')
<div class="container">
    <div class="row justify-content-center">
        <div class="col-md-8">
            <div class="card">
                <div class="card-header">Вход</div>
                <div class="card-body">
                    <form method="POST" action="{{ route('login') }}">
                        @csrf

                        <div class="row mb-3">
                            <label for="email" class="col-md-4 col-form-label text-md-end">Email</label>
                            <div class="col-md-6">
                                <input id="email" type="email"
                                    class="form-control @error('email') is-invalid @enderror"
                                    name="email" value="{{ old('email') }}" required autofocus>
                                @error('email')
                                    <span class="invalid-feedback"><strong>{{ $message }}</strong></span>
                                @enderror
                            </div>
                        </div>

                        <div class="row mb-3">
                            <label for="password" class="col-md-4 col-form-label text-md-end">Пароль</label>
                            <div class="col-md-6">
                                <input id="password" type="password"
                                    class="form-control @error('password') is-invalid @enderror"
                                    name="password" required>
                                @error('password')
                                    <span class="invalid-feedback"><strong>{{ $message }}</strong></span>
                                @enderror
                            </div>
                        </div>

                        <!-- КАПЧА -->
                        <div class="row mb-3">
                            <label class="col-md-4 col-form-label text-md-end fw-bold">Капча:</label>
                            <div class="col-md-6">
                                <p class="text-muted small mb-2">Собери картинку: перетащи кусочки</p>
                                <div id="captcha-container"
                                     style="display:grid;
                                            grid-template-columns: 120px 120px;
                                            grid-template-rows: 120px 120px;
                                            gap:2px; width:244px;
                                            border:2px dashed #ccc;
                                            padding:2px; border-radius:6px;">

                                    <!-- Кусочки перемешаны! data-index = правильная позиция -->
                                    <div class="captcha-piece" draggable="true" data-index="3"
                                         style="width:120px;height:120px;cursor:grab;
                                                border:2px solid #aaa;border-radius:4px;overflow:hidden;">
                                        <img src="{{ asset('images/3.png') }}"
                                             style="width:100%;height:100%;object-fit:cover;pointer-events:none;">
                                    </div>

                                    <div class="captcha-piece" draggable="true" data-index="1"
                                         style="width:120px;height:120px;cursor:grab;
                                                border:2px solid #aaa;border-radius:4px;overflow:hidden;">
                                        <img src="{{ asset('images/1.png') }}"
                                             style="width:100%;height:100%;object-fit:cover;pointer-events:none;">
                                    </div>

                                    <div class="captcha-piece" draggable="true" data-index="4"
                                         style="width:120px;height:120px;cursor:grab;
                                                border:2px solid #aaa;border-radius:4px;overflow:hidden;">
                                        <img src="{{ asset('images/4.png') }}"
                                             style="width:100%;height:100%;object-fit:cover;pointer-events:none;">
                                    </div>

                                    <div class="captcha-piece" draggable="true" data-index="2"
                                         style="width:120px;height:120px;cursor:grab;
                                                border:2px solid #aaa;border-radius:4px;overflow:hidden;">
                                        <img src="{{ asset('images/2.png') }}"
                                             style="width:100%;height:100%;object-fit:cover;pointer-events:none;">
                                    </div>

                                </div>
                                <div id="captcha-status"
                                     style="font-size:13px;min-height:20px;margin-top:6px;"></div>
                                <input type="hidden" name="captcha_solved" id="captcha_solved" value="0">
                            </div>
                        </div>

                        <div class="row mb-0">
                            <div class="col-md-8 offset-md-4">
                                <button type="submit" class="btn btn-primary" id="login-btn" disabled>
                                    Войти
                                </button>
                            </div>
                        </div>

                    </form>
                </div>
            </div>
        </div>
    </div>
</div>

<script>
let dragSrc = null;

function attachEvents() {
    document.querySelectorAll('.captcha-piece').forEach(el => {
        el.addEventListener('dragstart', function() {
            dragSrc = this;
            this.style.opacity = '0.4';
        });
        el.addEventListener('dragend', function() {
            this.style.opacity = '1';
        });
        el.addEventListener('dragover', function(e) {
            e.preventDefault();
            this.style.border = '2px solid #3498db';
        });
        el.addEventListener('dragleave', function() {
            this.style.border = '2px solid #aaa';
        });
        el.addEventListener('drop', function(e) {
            e.preventDefault();
            this.style.border = '2px solid #aaa';
            if (dragSrc !== this) {
                const srcClone = dragSrc.cloneNode(true);
                const tgtClone = this.cloneNode(true);
                dragSrc.parentNode.replaceChild(tgtClone, dragSrc);
                this.parentNode.replaceChild(srcClone, this);
                attachEvents();
            }
            checkOrder();
        });
    });
}

function checkOrder() {
    const container = document.getElementById('captcha-container');
    const pieces = [...container.children];
    const order = pieces.map(p => parseInt(p.dataset.index));
    const isCorrect = JSON.stringify(order) === JSON.stringify([1,2,3,4]);
    const status = document.getElementById('captcha-status');
    const input = document.getElementById('captcha_solved');
    const btn = document.getElementById('login-btn');
    if (isCorrect) {
        status.style.color = 'green';
        status.textContent = 'Капча пройдена!';
        input.value = '1';
        btn.disabled = false;
    } else {
        status.style.color = 'red';
        status.textContent = 'Неверный порядок!';
        input.value = '0';
        btn.disabled = true;
    }
}

attachEvents();
</script>
@endsection
```

### Шаг 5. `routes/web.php` — заменить всё

```php
<?php

use Illuminate\Support\Facades\Route;
use Illuminate\Support\Facades\Auth;
Route::get('/', function () {
    return view('layouts.app');
});

Auth::routes();

Route::get('/home', [App\Http\Controllers\HomeController::class, 'index'])->name('home');

```

### Шаг 6. Запустить проект

```bash
php artisan migrate
php artisan serve
```

## Модуль 5 — Проектная документация (методы и параметры)

**Методы `LoginController` и других контроллеров:**

- `showLoginForm()` — отображает форму входа с капчей, параметры отсутствуют
- `login(Request $request)` — авторизует пользователя, параметры: `email` (string), `password` (string), `captcha_solved` (int)
- `logout(Request $request)` — выход из системы, параметры отсутствуют
- `index()` — отображает главную страницу со списком статей, параметры отсутствуют
- `show(Request $request)` — отображает страницу отдельной статьи, параметры: `id` (int)
- `create()` — отображает форму создания новой статьи, параметры отсутствуют
- `store(Request $request)` — сохраняет новую статью в базу данных, параметры: `title` (string), `content` (string), `section_id` (int), `author_id` (int), `article_status_id` (int)
- `edit(Request $request)` — отображает форму редактирования статьи, параметры: `id` (int)
- `update(Request $request)` — обновляет статью в базе данных, параметры: `id` (int), `title` (string), `content` (string), `section_id` (int)
- `destroy(Request $request)` — удаляет статью из базы данных, параметры: `id` (int)

---


### Текст РЕГИСТРАЦИЯ
```
Функционал регистрации и авторизации пользователей с графической капчей

В разработанном приложении реализована система регистрации и авторизации пользователей. Основной задачей данного функционала является предоставление доступа к системе только зарегистрированным пользователям, а также защита формы входа от автоматизированных попыток авторизации.

Функция регистрации пользователей

Для регистрации нового пользователя используется стандартный механизм аутентификации Laravel. Пользователь переходит на страницу регистрации, где заполняет обязательные поля:

- имя пользователя;
- адрес электронной почты;
- пароль;
- подтверждение пароля.

После заполнения формы выполняется проверка корректности введённых данных. В случае успешного прохождения проверки создаётся новая запись в таблице пользователей базы данных. Пароль пользователя сохраняется в зашифрованном виде с использованием встроенного механизма хеширования Laravel.

После успешной регистрации пользователь может выполнить вход в систему с использованием созданных учётных данных.

Функция авторизации пользователей

Для входа в систему пользователь переходит на страницу авторизации. Форма входа содержит следующие элементы:

- поле ввода электронной почты;
- поле ввода пароля;
- интерактивную графическую капчу;
- кнопку «Войти».

При вводе данных система проверяет наличие пользователя в базе данных и корректность введённого пароля. В случае успешной проверки создаётся пользовательская сессия и предоставляется доступ к защищённым разделам приложения.

Если введённые данные не соответствуют информации, хранящейся в базе данных, пользователю выводится сообщение об ошибке авторизации.

Функция графической капчи

Для повышения безопасности формы входа реализована графическая капча в виде пазла.

Капча состоит из четырёх фрагментов изображения, которые отображаются в случайном порядке. Пользователю необходимо собрать исходное изображение путём перестановки элементов.

Для реализации механизма используется технология Drag and Drop, позволяющая перетаскивать фрагменты изображения внутри рабочей области.

После каждого перемещения выполняется автоматическая проверка текущего расположения элементов. Система сравнивает фактический порядок расположения фрагментов с эталонной последовательностью.

Если изображение собрано правильно:

- выводится сообщение «Капча пройдена»;
- скрытому полю captcha_solved присваивается значение 1;
- кнопка авторизации становится активной.

Если изображение собрано неправильно:

- выводится сообщение «Неверный порядок»;
- значение captcha_solved устанавливается в 0;
- кнопка авторизации остаётся заблокированной.

Таким образом, пользователь не может выполнить вход в систему до успешного прохождения проверки.

Особенности реализации

Проверка правильности сборки пазла осуществляется средствами JavaScript без перезагрузки страницы. Для хранения результата проверки используется скрытое поле формы. Состояние капчи обновляется автоматически после каждого действия пользователя.

Использование графической капчи позволяет предотвратить автоматические попытки входа в систему и обеспечивает дополнительный уровень защиты пользовательских учётных записей.

В результате реализации данного функционала была создана система регистрации и авторизации пользователей с дополнительной графической проверкой, основанной на сборке изображения из четырёх фрагментов.

```

### Текст ВАЛИДАЦИЯ

```
Функционал валидации пользовательских данных

В приложении реализован механизм валидации данных, предназначенный для проверки корректности информации, вводимой пользователем. Основной задачей валидации является предотвращение сохранения ошибочных, неполных или некорректных данных в базе данных.

Назначение валидации

Валидация позволяет контролировать правильность заполнения форм пользователем до момента обработки и сохранения информации. Благодаря этому обеспечивается целостность данных, повышается стабильность работы системы и снижается вероятность возникновения ошибок при дальнейшей обработке информации.

Принцип работы

После отправки формы данные передаются на сервер, где выполняется их проверка в соответствии с установленными правилами. Для каждого поля могут быть заданы индивидуальные ограничения и требования.

Если данные соответствуют всем требованиям, система продолжает выполнение операции и сохраняет информацию в базе данных.

Если хотя бы одно из правил нарушено, операция отменяется, а пользователю отображается сообщение с описанием обнаруженной ошибки.

Реализованные проверки

В системе могут использоваться следующие виды проверок:

- обязательность заполнения поля;
- проверка минимальной и максимальной длины значения;
- проверка типа данных;
- проверка корректности адреса электронной почты;
- проверка уникальности значения в базе данных;
- проверка диапазона числовых значений;
- проверка допустимого формата вводимых данных;
- проверка совпадения связанных полей.

Проверка обязательных полей

Для ключевых данных реализована проверка обязательности заполнения. Пользователь не может отправить форму, если одно или несколько обязательных полей остались пустыми.

При обнаружении ошибки выводится соответствующее уведомление с указанием поля, требующего заполнения.

Проверка уникальности данных

Для предотвращения появления дублирующихся записей используется проверка уникальности отдельных значений.

Перед сохранением новой записи система выполняет поиск аналогичных значений в базе данных. Если совпадение найдено, операция сохранения отменяется и пользователю отображается соответствующее сообщение.

Проверка форматов данных

Для отдельных полей выполняется проверка соответствия установленному формату.

Например:

- адрес электронной почты должен соответствовать общепринятому формату;
- числовые поля должны содержать только числовые значения;
- текстовые поля должны соответствовать установленным ограничениям по длине.

Отображение ошибок

Все ошибки валидации отображаются непосредственно пользователю. Сообщения содержат описание причины ошибки и позволяют быстро определить поле, требующее исправления.

Ошибочные поля дополнительно выделяются средствами пользовательского интерфейса, что повышает удобство работы с системой.

Преимущества реализации

Использование механизма валидации обеспечивает:

- повышение качества данных;
- предотвращение появления некорректных записей;
- снижение вероятности ошибок в работе приложения;
- повышение удобства взаимодействия пользователя с системой;
- обеспечение целостности базы данных.

Результат работы

В результате реализации механизма валидации все данные проходят обязательную проверку перед сохранением. Это позволяет гарантировать корректность информации, хранящейся в системе, а также обеспечивает устойчивую и безопасную работу приложения.

```
## Модуль 6 — Валидация данных от эмулятора

### Регулярные выражения — таблица

| Что проверяем | Регулярное выражение | Пример правильного |
|---|---|---|
| ФИО | `/^[А-Яа-яЁё]+ [А-Яа-яЁё]+ [А-Яа-яЁё]+$/u` | Иванов Иван Иванович |
| СНИЛС | `/^[0-9]{3}-[0-9]{3}-[0-9]{3} [0-9]{2}$/` | 123-456-789 01 |
| Телефон | `/^\+7 [0-9]{3} [0-9]{3}-[0-9]{2}-[0-9]{2}$/` | +7 999 123-45-67 |
| ИНН | `/^[0-9]+$/` | 1234567890 |
| Паспорт | `/^[0-9]{2} [0-9]{2} [0-9]{6}$/` | 12 34 567890 |
| Email | через Laravel: `'email' => 'required\|email'` | ivan@mail.ru |

### Шаг 1. Создать проект

```bash
composer create-project laravel/laravel --prefer-dist demo6
cd demo6
php artisan make:controller TestCaseController
```

### Шаг 2. `routes/web.php` — заменить всё

```php
<?php
use App\Http\Controllers\TestCaseController;
use Illuminate\Support\Facades\Route;

Route::get('/', [TestCaseController::class, 'show'])->name('test-case');
Route::get('/get', [TestCaseController::class, 'getData'])->name('test-case.get');
Route::post('/check', [TestCaseController::class, 'checkData'])->name('test-case.check');
```

### Шаг 3. `app/Http/Controllers/TestCaseController.php` — заменить всё

```php
<?php
namespace App\Http\Controllers;

use Illuminate\Http\Request;

class TestCaseController extends Controller
{
    // Показать страницу
    public function show(Request $request)
    {
        return view('index');
    }

    // Получить данные с эмулятора
    public function getData()
    {
        // ВСТАВЬ СЮДА ССЫЛКУ С ЭКЗАМЕНА + нужное поле после слеша
        $data = file_get_contents('http://localhost:4444/TransferSimulator/fullName');

        if ($data) {
            $data = json_decode($data)->value;
        }

        return redirect()->route('test-case')->with('value', $data);
    }

    // Проверить данные
    public function checkData(Request $request)
    {
        $value = $request->input('value');

        // ВЫБЕРИ НУЖНУЮ РЕГУЛЯРКУ ИЗ ТАБЛИЦЫ ВЫШЕ И ВСТАВЬ СЮДА:
        if (preg_match('/^[А-Яа-яЁё]+ [А-Яа-яЁё]+ [А-Яа-яЁё]+$/u', $value)) {
            return redirect()->route('test-case')
                ->with('value', $value)
                ->with('message', 'ФИО корректно');
        } else {
            return redirect()->route('test-case')
                ->with('value', $value)
                ->with('message', 'ФИО содержит некорректные символы');
        }
    }
}
```

### Шаг 4. Создать страницу

Удали `resources/views/welcome.blade.php`, создай `resources/views/index.blade.php`:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Валидация данных</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 600px; margin: 50px auto; padding: 20px; }
        .box { border: 1px solid #ccc; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
        button { padding: 8px 20px; background: #4ecca3; border: none; border-radius: 4px; cursor: pointer; font-size: 14px; }
        button:hover { background: #38b2ac; }
        .value { margin-left: 10px; font-weight: bold; color: #1a1a2e; }
        .message { margin-left: 10px; color: green; font-weight: bold; }
        .error { color: red; }
        h2 { color: #1a1a2e; }
    </style>
</head>
<body>
    <h2>Валидация данных</h2>

    <!-- Кнопка 1: Получить данные -->
    <div class="box">
        <form action="{{ route('test-case.get') }}" method="GET">
            <button type="submit">Получить данные</button>
            <span class="value">{{ session('value') }}</span>
        </form>
    </div>

    <!-- Кнопка 2: Отправить результат теста -->
    <div class="box">
        <form method="POST" action="{{ route('test-case.check') }}">
            @csrf
            <input type="hidden" name="value" value="{{ session('value') }}">
            <button type="submit">Отправить результат теста</button>
            <span class="message">{{ session('message') }}</span>
        </form>
    </div>

</body>
</html>
```

### Шаг 5. Запустить

```bash
php artisan serve
```

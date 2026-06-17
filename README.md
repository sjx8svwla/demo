## М1 М2 XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```
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




## М3 XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```
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





## М4 XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

### ШАГ 1: Создать проект Laravel
```
composer create-project laravel/laravel demo
cd demo
composer require laravel/ui
php artisan ui bootstrap --auth
```
### ШАГ 2: Настроить layouts/app.blade.php
Открой файл: resources/views/layouts/app.blade.php

Удали ВСЁ содержимое и вставь:
```
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
### ШАГ 3: Скопировать фото капчи
```
Создай папку: public/images/

Скопируй туда 4 фото и назови их: 1.png, 2.png, 3.png, 4.png

Расположение кусочков:
[1] = верхний левый
[2] = верхний правый
[3] = нижний левый
[4] = нижний правый
```
### ШАГ 4: Создать форму входа с капчей
Открой файл: resources/views/auth/login.blade.php

Удали ВСЁ содержимое и вставь:
```
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
### ШАГ 5: Запустить проект
В терминале запусти по очереди:
```
php artisan migrate
php artisan serve
Открой браузер: http://localhost:8000/login
```






## М5 XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```
showLoginForm() — отображает форму входа с капчей, параметры 
отсутствуют  
login(Request $request) — авторизует пользователя, параметры: email 
(string), password (string), captcha_solved (int)  
logout(Request $request) — выход из системы, параметры отсутствуют  
index() — отображает главную страницу со списком статей, 
параметры отсутствуют  
show(Request $request) — отображает страницу отдельной статьи, 
параметры: id (int)  
create() — отображает форму создания новой статьи, параметры 
отсутствуют  
store(Request $request) — сохраняет новую статью в базу данных, 
параметры: title (string), content (string), section_id (int), author_id (int), 
article_status_id (int)  
edit(Request $request) — отображает форму редактирования статьи, 
параметры: id (int)  
update(Request $request) — обновляет статью в базе данных, 
параметры: id (int), title (string), content (string), section_id (int)  
destroy(Request $request) — удаляет статью из базы данных, 
параметры: id (int)
```




## М6 XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

### ШАГ 1: Создать проект
Открой терминал и выполни:
```
composer create-project laravel/laravel --prefer-dist demo6
cd demo6
php artisan make:controller TestCaseController
```

### ШАГ 2: Настроить маршруты — routes/web.php
Открой routes/web.php и замени всё содержимое на:
```
<?php
use App\Http\Controllers\TestCaseController;
use Illuminate\Support\Facades\Route;

Route::get('/', [TestCaseController::class, 'show'])->name('test-case');
Route::get('/get', [TestCaseController::class, 'getData'])->name('test-case.get');
Route::post('/check', [TestCaseController::class, 'checkData'])->name('test-case.check');
```

### ШАГ 3: Написать контроллер — app/Http/Controllers/TestCaseController.php
Открой app/Http/Controllers/TestCaseController.php и замени всё на:
 ```
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
        $data = file_get_contents('http://localhost:8080/api/fullName');

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

### ШАГ 4: Создать страницу — resources/views/index.blade.php
Удали файл resources/views/welcome.blade.php

Создай новый файл resources/views/index.blade.php и вставь:
```
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

### ШАГ 5: Запустить проект
```
php artisan serve
```





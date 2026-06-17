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
```

### Шаг 2. Создать миграцию пользователей

Найди файл, который заканчивается на `_create_users_table.php`, и замени его содержимое полностью:

```bash
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->unsignedTinyInteger('wrong_counter')->default(0);
            $table->timestamps();
        });

        Schema::create('sessions', function (Blueprint $table) {
            $table->string('id')->primary();
            $table->foreignId('user_id')->nullable()->index();
            $table->string('ip_address', 45)->nullable();
            $table->text('user_agent')->nullable();
            $table->longText('payload');
            $table->integer('last_activity')->index();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
        Schema::dropIfExists('sessions');
    }
};
```

### Шаг 3. Создать модель User 

Открой файл `app/Models/User.php` и измени на:

```bash
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    protected $fillable = ['name', 'email', 'password', 'wrong_counter'];

    protected $hidden = ['password', 'remember_token'];

    protected function casts(): array
    {
        return [
            'password' => 'hashed',
        ];
    }
}
```

---

### Шаг 4. Создать папку для картинок капчи и сами файлы

```bash
mkdir -p public/images
```

### Шаг 5. Создать контроллеры командами artisan

```bash
php artisan make:controller ActionsController
php artisan make:controller ViewsController
```

### Шаг 6. Заполнить ViewsController.php
 Открой `app/Http/Controllers/ViewsController.php` и замени на:
```bash
<?php

namespace App\Http\Controllers;

use App\Models\User;

class ViewsController extends Controller
{
    public function register()
    {
        return view('register');
    }

    public function login()
    {
        return view('login');
    }

    public function index()
    {
        return view('index', [
            'users' => User::all()
        ]);
    }

    public function editUser(?User $user = null)
    {
        return view('user_form', [
            'user' => $user
        ]);
    }
}
```


### Шаг 7. Заполнить ActionsController.php
 
```bash
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class ActionsController extends Controller
{
    public function register(Request $request)
    {
        $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users,email',
            'password' => 'required|string|min:6',
            'captcha_solved' => 'required|in:1',
        ]);

        $user = new User();
        $user->name = $request->input('name');
        $user->email = $request->input('email');
        $user->password = $request->input('password');
        $user->save();

        return redirect()->route('login')->with('status', 'Регистрация успешна, войдите.');
    }

    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required|string|min:6',
            'captcha_solved' => 'required|in:1',
        ]);

        $user = User::firstWhere('email', $request->input('email'));

        if ($user && $user->wrong_counter >= 3) {
            return redirect()
                ->route('login')
                ->withErrors(['email' => 'Ваш аккаунт заблокирован. Обратитесь к администратору.']);
        }

        if (Auth::attempt($request->only(['email', 'password']))) {
            /** @var User $user */
            $user = Auth::user();
            $user->wrong_counter = 0;
            $user->save();

            return redirect()->route('dashboard');
        }

        if ($user) {
            $user->wrong_counter++;
            $user->save();

            if ($user->wrong_counter >= 3) {
                return redirect()
                    ->route('login')
                    ->withErrors(['email' => 'Аккаунт заблокирован после 3 неверных попыток.']);
            }
        }

        return redirect()
            ->route('login')
            ->withErrors(['email' => 'Неверный логин или пароль.'])
            ->withInput($request->only('email'));
    }

    public function editUser(User $user, Request $request)
    {
        $request->validate([
            'name' => 'required|string|max:255',
            'email' => "required|email|unique:users,email,{$user->id}",
            'password' => 'nullable|string|min:6',
        ]);

        $user->name = $request->input('name');
        $user->email = $request->input('email');
        if ($request->filled('password')) {
            $user->password = $request->input('password');
        }
        $user->save();

        return redirect()->route('dashboard')->with('status', 'Пользователь обновлён.');
    }

    public function createUser(Request $request)
    {
        $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users,email',
            'password' => 'required|string|min:6',
        ]);

        $user = new User();
        $user->name = $request->input('name');
        $user->email = $request->input('email');
        $user->password = $request->input('password');
        $user->save();

        return redirect()->route('dashboard')->with('status', 'Пользователь создан.');
    }

    public function deleteUser(User $user)
    {
        if ($user->id === Auth::id()) {
            return redirect()->route('dashboard')->withErrors(['user' => 'Нельзя удалить себя.']);
        }
        $user->delete();
        return redirect()->route('dashboard')->with('status', 'Пользователь удалён.');
    }

    public function unlockUser(User $user)
    {
        if ($user->id === Auth::id()) {
            return redirect()->route('dashboard')->withErrors(['user' => 'Нельзя блокировать/разблокировать себя.']);
        }

        if ($user->wrong_counter < 3) {
            $user->wrong_counter = 3;
            $user->save();
            return redirect()->route('dashboard')->with('status', 'Пользователь заблокирован.');
        }

        $user->wrong_counter = 0;
        $user->save();
        return redirect()->route('dashboard')->with('status', 'Пользователь разблокирован.');
    }

    public function logout()
    {
        Auth::logout();
        return redirect()->route('login');
    }
}
```

---

### Шаг 8. Заполнить routes/web.php

```bash
<?php

use App\Http\Controllers\ViewsController;
use App\Http\Controllers\ActionsController;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return Auth::check() ? redirect()->route('dashboard') : redirect()->route('register');
});

Route::middleware('guest')->group(function () {
    Route::get('/register', [ViewsController::class, 'register'])->name('register');
    Route::post('/register', [ActionsController::class, 'register'])->name('register.store');

    Route::get('/login', [ViewsController::class, 'login'])->name('login');
    Route::post('/login', [ActionsController::class, 'login'])->name('login.attempt');
});

Route::middleware('auth')->group(function () {
    Route::get('/dashboard', [ViewsController::class, 'index'])->name('dashboard');

    Route::get('/user/{user?}', [ViewsController::class, 'editUser'])->name('user');
    Route::post('/user/{user?}', [ActionsController::class, 'createUser'])->name('user.save');
    Route::put('/user/{user}', [ActionsController::class, 'editUser'])->name('user.update');
    Route::delete('/user/{user}', [ActionsController::class, 'deleteUser'])->name('user.delete');
    Route::patch('/user/{user}', [ActionsController::class, 'unlockUser'])->name('user.unlock');

    Route::post('/logout', [ActionsController::class, 'logout'])->name('logout');
});
```

---

### Шаг 9. Создать resources/views/register.blade.php

```bash
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Регистрация</title>
</head>
<body>
    <form method="POST" action="{{ route('register.store') }}">
        @csrf

        <p>
            Имя: <input type="text" name="name" value="{{ old('name') }}" required>
            @error('name') {{ $message }} @enderror
        </p>

        <p>
            Email: <input type="email" name="email" value="{{ old('email') }}" required>
            @error('email') {{ $message }} @enderror
        </p>

        <p>
            Пароль: <input type="password" name="password" required>
            @error('password') {{ $message }} @enderror
        </p>

        <p>Собери картинку (перетащи кусочки в порядок 1,2,3,4):</p>

        <div id="cap" style="display:grid;grid-template-columns:120px 120px;grid-template-rows:120px 120px;gap:2px;width:244px;">
            <div class="p" draggable="true" data-i="3" style="overflow:hidden;cursor:grab;">
                <img src="{{ asset('images/3.png') }}" style="width:120px;height:120px;object-fit:cover;pointer-events:none;">
            </div>
            <div class="p" draggable="true" data-i="1" style="overflow:hidden;cursor:grab;">
                <img src="{{ asset('images/1.png') }}" style="width:120px;height:120px;object-fit:cover;pointer-events:none;">
            </div>
            <div class="p" draggable="true" data-i="4" style="overflow:hidden;cursor:grab;">
                <img src="{{ asset('images/4.png') }}" style="width:120px;height:120px;object-fit:cover;pointer-events:none;">
            </div>
            <div class="p" draggable="true" data-i="2" style="overflow:hidden;cursor:grab;">
                <img src="{{ asset('images/2.png') }}" style="width:120px;height:120px;object-fit:cover;pointer-events:none;">
            </div>
        </div>

        <p id="status"></p>
        <input type="hidden" name="captcha_solved" id="cs" value="0">

        <p><button type="submit" id="btn" disabled>Зарегистрироваться</button></p>
        <p><a href="{{ route('login') }}">Уже есть аккаунт? Войти</a></p>
    </form>

    <script>
        let src = null;
        function bind() {
            document.querySelectorAll('.p').forEach(el => {
                el.ondragstart = () => { src = el; el.style.opacity = '0.4'; };
                el.ondragend = () => el.style.opacity = '1';
                el.ondragover = e => e.preventDefault();
                el.ondrop = e => {
                    e.preventDefault();
                    if (src !== el) {
                        let sc = src.cloneNode(true);
                        let tc = el.cloneNode(true);
                        src.parentNode.replaceChild(tc, src);
                        el.parentNode.replaceChild(sc, el);
                        bind();
                    }
                    check();
                };
            });
        }
        function check() {
            let order = [...document.getElementById('cap').children].map(p => +p.dataset.i);
            let ok = JSON.stringify(order) === JSON.stringify([1,2,3,4]);
            document.getElementById('status').textContent = ok ? 'Капча пройдена!' : 'Неверный порядок!';
            document.getElementById('cs').value = ok ? '1' : '0';
            document.getElementById('btn').disabled = !ok;
        }
        bind();
    </script>
</body>
</html>
```

---

### Шаг 10. Создать resources/views/login.blade.php

```bash
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Вход</title>
</head>
<body>
    @if(session('status'))
        <p>{{ session('status') }}</p>
    @endif

    <form method="POST" action="{{ route('login.attempt') }}">
        @csrf

        <p>
            Email: <input type="email" name="email" value="{{ old('email') }}" required>
            @error('email') {{ $message }} @enderror
        </p>

        <p>
            Пароль: <input type="password" name="password" required>
            @error('password') {{ $message }} @enderror
        </p>

        <p>Собери картинку (перетащи кусочки в порядок 1,2,3,4):</p>

        <div id="cap" style="display:grid;grid-template-columns:120px 120px;grid-template-rows:120px 120px;gap:2px;width:244px;">
            <div class="p" draggable="true" data-i="3" style="overflow:hidden;cursor:grab;">
                <img src="{{ asset('images/3.png') }}" style="width:120px;height:120px;object-fit:cover;pointer-events:none;">
            </div>
            <div class="p" draggable="true" data-i="1" style="overflow:hidden;cursor:grab;">
                <img src="{{ asset('images/1.png') }}" style="width:120px;height:120px;object-fit:cover;pointer-events:none;">
            </div>
            <div class="p" draggable="true" data-i="4" style="overflow:hidden;cursor:grab;">
                <img src="{{ asset('images/4.png') }}" style="width:120px;height:120px;object-fit:cover;pointer-events:none;">
            </div>
            <div class="p" draggable="true" data-i="2" style="overflow:hidden;cursor:grab;">
                <img src="{{ asset('images/2.png') }}" style="width:120px;height:120px;object-fit:cover;pointer-events:none;">
            </div>
        </div>

        <p id="status"></p>
        <input type="hidden" name="captcha_solved" id="cs" value="0">

        <p><button type="submit" id="btn" disabled>Войти</button></p>
        <p><a href="{{ route('register') }}">Нет аккаунта? Зарегистрироваться</a></p>
    </form>

    <script>
        let src = null;
        function bind() {
            document.querySelectorAll('.p').forEach(el => {
                el.ondragstart = () => { src = el; el.style.opacity = '0.4'; };
                el.ondragend = () => el.style.opacity = '1';
                el.ondragover = e => e.preventDefault();
                el.ondrop = e => {
                    e.preventDefault();
                    if (src !== el) {
                        let sc = src.cloneNode(true);
                        let tc = el.cloneNode(true);
                        src.parentNode.replaceChild(tc, src);
                        el.parentNode.replaceChild(sc, el);
                        bind();
                    }
                    check();
                };
            });
        }
        function check() {
            let order = [...document.getElementById('cap').children].map(p => +p.dataset.i);
            let ok = JSON.stringify(order) === JSON.stringify([1,2,3,4]);
            document.getElementById('status').textContent = ok ? 'Капча пройдена!' : 'Неверный порядок!';
            document.getElementById('cs').value = ok ? '1' : '0';
            document.getElementById('btn').disabled = !ok;
        }
        bind();
    </script>
</body>
</html>
```

---

### Шаг 11. Создать resources/views/index.blade.php

```bash
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Dashboard</title>
</head>
<body>
    <h1>Список пользователей</h1>
    <table border="1">
        <thead>
            <tr>
                <th>Имя</th>
                <th>Email</th>
                <th>Заблокирован</th>
                <th>Действия</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($users as $user)
                <tr>
                    <td>{{ $user->name }}</td>
                    <td>{{ $user->email }}</td>
                    <td>{{ $user->wrong_counter >= 3 ? 'Да' : 'Нет' }}</td>
                    <td>
                        <a href="{{ route('user', ['user' => $user->id]) }}">Edit</a>
                        <form action="{{ route('user.delete', ['user' => $user->id]) }}" method="POST" style="display:inline;">
                            @csrf
                            @method('DELETE')
                            <button type="submit">Delete</button>
                        </form>
                        <form action="{{ route('user.unlock', ['user' => $user->id]) }}" method="POST" style="display:inline;">
                            @csrf
                            @method('PATCH')
                            <button type="submit">{{ $user->wrong_counter >= 3 ? 'Unlock' : 'Block' }}</button>
                        </form>
                    </td>
                </tr>
            @endforeach
        </tbody>
        <tfoot>
            <tr>
                <td colspan="4">
                    <a href="{{ route('user') }}">Create New User</a>
                    <form action="{{ route('logout') }}" method="POST" style="display:inline;">
                        @csrf
                        <button type="submit">Logout</button>
                    </form>
                </td>
            </tr>
        </tfoot>
    </table>
    @if(session('status'))
        <div>{{ session('status') }}</div>
    @endif
    @if($errors->any())
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    @endif
</body>
</html>
```

---

### Шаг 12. Создать resources/views/user_form.blade.php

```bash
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>{{ $user ? 'Edit user' : 'Create user' }}</title>
</head>
<body>
    <h1>{{ $user ? 'Edit user' : 'Create user' }}</h1>
    <form action="{{ $user ? route('user.update', $user->id) : route('user.save') }}" method="POST">
        @csrf
        @if ($user)
            @method('PUT')
        @endif
        <div>
            <label for="name">Name:</label>
            <input type="text" id="name" name="name" value="{{ old('name', $user->name ?? '') }}" required>
        </div>
        <div>
            <label for="email">Email:</label>
            <input type="email" id="email" name="email" value="{{ old('email', $user->email ?? '') }}" required>
        </div>
        <div>
            <label for="password">Password:</label>
            <input type="password" id="password" name="password" {{ $user ? '' : 'required' }}>
        </div>
        <button type="submit">{{ $user ? 'Update user' : 'Create user' }}</button>
        <a href="{{ route('dashboard') }}">Cancel</a>
    </form>
    @if($errors->any())
        <ul>
            @foreach ($errors->all() as $error)
                <li>{{ $error }}</li>
            @endforeach
        </ul>
    @endif
</body>
</html>
```

---

### Шаг 13. Удалить файл-заглушку Laravel

```bash
rm resources/views/welcome.blade.php
```

(на Windows в PowerShell, если `rm` не сработает: `Remove-Item resources/views/welcome.blade.php`)

---

### Шаг 14. Выполнить миграцию и запустить сервер

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

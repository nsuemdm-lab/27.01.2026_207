### ВИРТУАЛЬНЫЙ МЕТОДИЧЕСКИЙ ОТДЕЛ
**Преподаватель:** Ивлев А.Л.
**Дисциплина:** МДК.09.01 Проектирование и разработка веб-приложений
**Группа:** 9-ИС207
**Дата:** Понедельник, 26 января 2026 г. (Расчетная дата следующего понедельника)

**ВНИМАНИЕ:** В связи с сокращением времени (2 пары вместо 3), темп работы интенсивный. Мы объединяем создание прав доступа и создание контента в первую пару, а вывод и защиту переносим на вторую.

---

### ПЕРВАЯ ПАРА (11:25 – 12:55)
**Тема:** Фундамент системы: Разделение прав (RBAC) + Создание сущностей (CRUD: Create)
**Цель:** К концу пары администратор должен уметь входить в закрытую зону и добавлять товары/услуги в базу данных.

#### 1. МОДИФИКАЦИЯ ВХОДА (RBAC)
**Задача:** Сервер должен запомнить не только ID, но и РОЛЬ пользователя («браслет» admin или client).

1.  Откройте `login.php`.
2.  Найдите проверку `password_verify`.
3.  Добавьте сохранение роли в сессию.

**Код для вставки в `login.php`:**
```php
<?php
session_start();
require 'db.php';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $email = $_POST['email'];
    $pass = $_POST['password'];

    // Ищем пользователя по email
    $stmt = $pdo->prepare("SELECT * FROM users WHERE email = ?");
    $stmt->execute([$email]);
    $user = $stmt->fetch();

    // Проверяем пароль
    if ($user && password_verify($pass, $user['password_hash'])) {
        
        // --- ВАЖНЫЕ ИЗМЕНЕНИЯ ---
        $_SESSION['user_id'] = $user['id'];
        $_SESSION['user_role'] = $user['role']; // Сохраняем "браслет" (роль)
        // ------------------------

        // Маршрутизация: Админа — в панель, остальных — на главную
        if ($user['role'] === 'admin') {
            header("Location: admin_panel.php");
        } else {
            header("Location: index.php");
        }
        exit;
    } else {
        echo "Неверный логин или пароль";
    }
}
?>
```

#### 2. СОЗДАНИЕ «ВЫШИБАЛЫ» (Middleware)
**Задача:** Создать файл-фильтр, который будем подключать ко всем админским страницам.

1.  Создайте файл `check_admin.php`.
2.  Вставьте код:
```php
<?php
// check_admin.php — Скрипт защиты
session_start();

// Проверяем: авторизован ли И является ли админом
if (!isset($_SESSION['user_id']) || $_SESSION['user_role'] !== 'admin') {
    die("ДОСТУП ЗАПРЕЩЕН. У вас нет прав администратора. <a href='login.php'>Войти</a>");
}
?>
```

#### 3. ЗАЩИТА РЕГИСТРАЦИИ (Fix)
**Угроза:** Хакер может подделать форму и отправить `role=admin`.
**Действие:** Откройте `register.php` и убедитесь, что роль 'client' прописана **жестко** в коде, а не берется из `$_POST`.

```php
// Правильный фрагмент register.php
$sql = "INSERT INTO users (email, password_hash, role) VALUES (:email, :hash, 'client')";
```

#### 4. ПАНЕЛЬ АДМИНИСТРАТОРА
1.  Создайте `admin_panel.php`.
2.  Вставьте код:
```php
<?php
require 'check_admin.php'; // Вызов охраны
?>
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Админка</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="p-5">
    <div class="alert alert-success">
        <h1>Панель Администратора</h1>
        <p>Добро пожаловать в систему управления.</p>
        <a href="add_item.php" class="btn btn-primary">Добавить запись</a>
        <a href="logout.php" class="btn btn-danger">Выйти</a>
    </div>
</body>
</html>
```

#### 5. ПРОЕКТИРОВАНИЕ БАЗЫ ДАННЫХ
**Задача:** Создать таблицу для вашей темы (Товары, Книги, Заявки).
**Действие:** Beget -> MySQL -> phpMyAdmin -> SQL. Выполните запрос (пример для Магазина):

```sql
CREATE TABLE `products` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `title` VARCHAR(255) NOT NULL COMMENT 'Название',
  `description` TEXT COMMENT 'Описание',
  `price` DECIMAL(10, 2) NOT NULL COMMENT 'Цена',
  `image_url` VARCHAR(255) DEFAULT NULL COMMENT 'Ссылка на фото',
  `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
*(Для заявок/блога меняйте поля `price` на `status` или `author` соответственно).*

#### 6. СТРАНИЦА ДОБАВЛЕНИЯ (Create)
**Задача:** Форма для наполнения базы.
1.  Создайте `add_item.php`.
2.  Вставьте код:
```php
<?php
require 'db.php';
require 'check_admin.php'; // Только для админа!

$message = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $title = trim($_POST['title']);
    $price = $_POST['price'];
    $desc  = trim($_POST['description']);
    $img   = trim($_POST['image_url']);

    if (empty($title)) {
        $message = '<div class="alert alert-danger">Заполните название!</div>';
    } else {
        $sql = "INSERT INTO products (title, description, price, image_url) VALUES (:t, :d, :p, :i)";
        $stmt = $pdo->prepare($sql);
        $stmt->execute([':t' => $title, ':d' => $desc, ':p' => $price, ':i' => $img]);
        $message = '<div class="alert alert-success">Успешно добавлено!</div>';
    }
}
?>

<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="p-4">
    <div class="container">
        <h1>Новый товар</h1>
        <a href="admin_panel.php" class="btn btn-secondary mb-3">← Назад</a>
        <?= $message ?>
        <form method="POST" class="card p-4">
            <input type="text" name="title" class="form-control mb-2" placeholder="Название" required>
            <input type="number" name="price" class="form-control mb-2" placeholder="Цена" step="0.01">
            <input type="text" name="image_url" class="form-control mb-2" placeholder="URL картинки">
            <textarea name="description" class="form-control mb-2" placeholder="Описание"></textarea>
            <button type="submit" class="btn btn-success">Сохранить</button>
        </form>
    </div>
</body>
</html>
```

---

### ВТОРАЯ ПАРА (13:20 – 14:50)
**Тема:** Публичная часть (Read) и Безопасность (XSS)
**Цель:** Вывести данные для пользователей и защитить сайт от внедрения вредоносного JS-кода.

#### 1. ВЫВОД СПИСКА НА ГЛАВНОЙ (Read)
**Задача:** Сформировать витрину.
1.  Откройте `index.php`.
2.  Вставьте код:
```php
<?php
session_start();
require 'db.php';

// Получаем данные (Сортировка: новые сверху)
$stmt = $pdo->query("SELECT * FROM products ORDER BY id DESC");
$products = $stmt->fetchAll();
?>

<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Главная</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
<nav class="navbar navbar-light bg-light px-4 mb-4">
    <span class="navbar-brand">Мой Проект</span>
    <div>
        <?php if (isset($_SESSION['user_id'])): ?>
            <?php if ($_SESSION['user_role'] === 'admin'): ?>
                <a href="admin_panel.php" class="btn btn-danger btn-sm">Админка</a>
            <?php endif; ?>
            <a href="logout.php" class="btn btn-dark btn-sm">Выйти</a>
        <?php else: ?>
            <a href="login.php" class="btn btn-primary btn-sm">Войти</a>
        <?php endif; ?>
    </div>
</nav>

<div class="container">
    <div class="row">
        <?php foreach ($products as $p): ?>
            <div class="col-md-4 mb-4">
                <div class="card h-100">
                    <img src="<?= htmlspecialchars($p['image_url'] ?: 'https://via.placeholder.com/300') ?>" class="card-img-top" style="height: 200px; object-fit: cover;">
                    <div class="card-body">
                        <!-- ВНИМАНИЕ: Здесь пока потенциальная уязвимость XSS -->
                        <h5 class="card-title"><?= $p['title'] ?></h5> 
                        <p class="card-text"><?= $p['description'] ?></p>
                        <p class="text-primary fw-bold"><?= $p['price'] ?> ₽</p>
                    </div>
                </div>
            </div>
        <?php endforeach; ?>
    </div>
</div>
</body>
</html>
```

#### 2. ЛАБОРАТОРНАЯ «САМ СЕБЕ ХАКЕР» (XSS)
1.  Зайдите под Админом.
2.  Добавьте товар с названием: `<h1>ВЗЛОМ</h1>`. Проверьте главную — верстка сломалась?
3.  Добавьте товар с описанием: `<script>alert(1)</script>`. Проверьте главную — вылезло окно?
**Вывод:** Ваш сайт уязвим.

#### 3. ЗАЩИТА (Fixing)
**Задача:** Экранировать вывод данных.
1.  В конец файла `db.php` добавьте функцию:
```php
function h($str) {
    return htmlspecialchars($str, ENT_QUOTES, 'UTF-8');
}
```
2.  В `index.php` оберните вывод переменных в функцию `h()`:
```php
<!-- БЫЛО (ОПАСНО) -->
<h5 class="card-title"><?= $p['title'] ?></h5>

<!-- СТАЛО (БЕЗОПАСНО) -->
<h5 class="card-title"><?= h($p['title']) ?></h5>
<p class="card-text"><?= h($p['description']) ?></p>
```
3.  Проверьте снова. Теги должны отображаться текстом, скрипты не должны выполняться.

---

### ИТОГОВЫЙ ЧЕК-ЛИСТ (Критерии оценки)
К концу 14:50 у вас должно работать:
1.  **RBAC:** Файл `check_admin.php` защищает `add_item.php` и `admin_panel.php`.
2.  **БД:** Таблица создана, данные сохраняются через форму.
3.  **Интерфейс:** Данные выводятся на `index.php` карточками.
4.  **Безопасность:** В `register.php` роль 'client' жестко задана. В `index.php` используется `h()` или `htmlspecialchars()`. Скрипты в названиях товаров не выполняются.

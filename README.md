Получилось 10 таблиц (даже на одну больше) — вышло естественно, разбил produkty/materialy на справочники. Вот итог:

Структура (10 таблиц):

	1.	zakazchiki — заказчики (из JSON)
	2.	edinitsy_izmereniya — справочник единиц измерения (шт, кг, л)
	3.	kategorii — справочник категорий продукции
	4.	produkty — продукция (связана с категорией и единицей измерения)
	5.	postavshchiki — поставщики материалов (новая сущность)
	6.	materialy — материалы (связаны с единицей измерения и поставщиком)
	7.	spetsifikatsiya — рецептура, связь N:M между продукцией и материалами
	8.	zakazy — заказы покупателей
	9.	sostav_zakaza — состав заказа, связь N:M между заказами и продукцией
	10.	proizvodstvo — факты производства
```
CREATE DATABASE moloko CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE moloko;
-- ===========================================
-- 1. ТАБЛИЦЫ БЕЗ ВНЕШНИХ КЛЮЧЕЙ (базовые сущности)
-- ===========================================

CREATE TABLE zakazchiki (
    id VARCHAR(20) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    inn VARCHAR(20),
    address VARCHAR(255),
    phone VARCHAR(20),
    salesman TINYINT(1) DEFAULT 0,
    buyer TINYINT(1) DEFAULT 0
);

CREATE TABLE edinitsy_izmereniya (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nazvanie VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE kategorii (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nazvanie VARCHAR(255) NOT NULL
);

CREATE TABLE postavshchiki (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    inn VARCHAR(20),
    phone VARCHAR(20)
);

-- ===========================================
-- 2. ТАБЛИЦЫ С ВНЕШНИМИ КЛЮЧАМИ (1 уровень)
-- ===========================================

CREATE TABLE produkty (
    id INT AUTO_INCREMENT PRIMARY KEY,
    kod VARCHAR(20) UNIQUE,
    nazvanie VARCHAR(255) NOT NULL,
    kategoriya_id INT,
    ed_izm_id INT,
    cena DECIMAL(10,2),
    FOREIGN KEY (kategoriya_id) REFERENCES kategorii(id) ON DELETE SET NULL,
    FOREIGN KEY (ed_izm_id) REFERENCES edinitsy_izmereniya(id) ON DELETE SET NULL
);

CREATE TABLE materialy (
    id INT AUTO_INCREMENT PRIMARY KEY,
    kod VARCHAR(20) UNIQUE,
    nazvanie VARCHAR(255) NOT NULL,
    ed_izm_id INT,
    cena DECIMAL(10,2),
    postavshchik_id INT,
    FOREIGN KEY (ed_izm_id) REFERENCES edinitsy_izmereniya(id) ON DELETE SET NULL,
    FOREIGN KEY (postavshchik_id) REFERENCES postavshchiki(id) ON DELETE SET NULL
);

CREATE TABLE zakazy (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nomer VARCHAR(20),
    data DATE NOT NULL,
    zakazchik_id VARCHAR(20) NOT NULL,
    FOREIGN KEY (zakazchik_id) REFERENCES zakazchiki(id) ON DELETE CASCADE
);

-- ===========================================
-- 3. ТАБЛИЦЫ С ВНЕШНИМИ КЛЮЧАМИ (2 уровень)
-- ===========================================

CREATE TABLE spetsifikatsiya (
    id INT AUTO_INCREMENT PRIMARY KEY,
    produkt_id INT NOT NULL,
    material_id INT NOT NULL,
    kolichestvo DECIMAL(10,4) NOT NULL,
    FOREIGN KEY (produkt_id) REFERENCES produkty(id) ON DELETE CASCADE,
    FOREIGN KEY (material_id) REFERENCES materialy(id) ON DELETE CASCADE,
    UNIQUE KEY uniq_spec (produkt_id, material_id)
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

-- ===========================================
-- 4. ЗАПОЛНЕНИЕ СПРАВОЧНИКОВ
-- ===========================================

INSERT INTO edinitsy_izmereniya (nazvanie) VALUES
('шт'), ('кг'), ('л');

INSERT INTO kategorii (nazvanie) VALUES
('Молочная продукция'), ('Кисломолочная продукция'), ('Сырная продукция');

INSERT INTO postavshchiki (name, inn, phone) VALUES
('ООО "АгроМолПром"', '7701234567', '+74951234567'),
('ИП Сидоров А.А.', '7719876543', '+74959876543');

-- ===========================================
-- 5. ИМПОРТ ДАННЫХ ИЗ Заказчики.json
-- ===========================================

INSERT INTO zakazchiki (id, name, inn, address, phone, salesman, buyer) VALUES
('000000001', 'ООО "Поставка"', '', 'г.Пятигорск', '+79198634592', 1, 1),
('000000002', 'ООО "Кинотеатр Квант"', '26320045123', 'г. Железноводск, ул. Мира, 123', '+79884581555', 1, 0),
('000000008', 'ООО "Новый JDTO"', '26320045111', 'г. Железноводсу', '+79884581555', 1, 0),
('000000003', 'ООО "Ромашка"', '4140784214', 'г. Омск, ул. Строителей, 294', '+79882584546', 0, 1),
('000000009', 'ООО "Ипподром"', '5874045632', 'г. Уфа, ул. Набережная, 37', '+79627486389', 1, 1),
('000000010', 'ООО "Ассоль"', '2629011278', 'г. Калуга, ул. Пушкина, 94', '+79184572398', 0, 1);
```









Важно: так как добавились новые таблицы и поля kategoriya_id, ed_izm_id, postavshchik_id — наш SQL-запрос из Модуля 3 не сломается, потому что spetsifikatsiya и materialy.cena остались без изменений. Запрос для Модуля 3 будет работать как раньше.
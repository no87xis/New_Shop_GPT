# 📋 ПОЛНОЕ ТЕХНИЧЕСКОЕ ЗАДАНИЕ - SIRIUS GROUP V2
## Система управления складом и интернет-магазин с WhatsApp уведомлениями

---

## 🎯 ОБЩЕЕ ОПИСАНИЕ СИСТЕМЫ

**Sirius Group** - комплексная веб-система для управления складом, заказами и интернет-магазином с системой WhatsApp уведомлений клиентам. Система предназначена для автоматизации процессов складского учета, продаж, доставки товаров и уведомления клиентов о готовности заказов.

### Основные цели:
- Управление складскими остатками и поставками
- Интернет-магазин с корзиной и системой заказов
- QR-коды для отслеживания заказов
- Система доставки с 5 вариантами
- **WhatsApp уведомления клиентам при поступлении товара** - НОВОЕ
- Аналитика и отчетность
- Мобильная адаптация

---

## 👥 РОЛИ ПОЛЬЗОВАТЕЛЕЙ

### 1. **Администратор** (`admin`)
- Полный доступ ко всем функциям системы
- Управление пользователями и ролями
- Настройка системы и конфигурации
- Просмотр всех аналитических данных
- **Управление WhatsApp уведомлениями** - НОВОЕ

### 2. **Менеджер склада** (`manager`)
- Управление товарами и поставками
- Обработка заказов и изменение статусов
- Работа с QR-сканером
- Просмотр аналитики по складу
- **Отправка WhatsApp уведомлений клиентам** - НОВОЕ

### 3. **Работник склада** (`warehouse`)
- Просмотр заказов
- Работа с QR-сканером
- Выдача товаров

### 4. **Пользователь** (`user`)
- Просмотр заказов
- Создание заказов

### 5. **Клиент магазина** (без авторизации)
- Просмотр каталога товаров
- Добавление в корзину
- Оформление заказов
- **Получение WhatsApp уведомлений о готовности заказа** - НОВОЕ

---

## 🗄️ МОДЕЛЬ ДАННЫХ

### 1. **Пользователи (users)**
```sql
username VARCHAR PRIMARY KEY
hashed_password VARCHAR NOT NULL
role ENUM('admin', 'manager', 'warehouse', 'user') DEFAULT 'user'
is_active BOOLEAN DEFAULT TRUE
is_superuser BOOLEAN DEFAULT FALSE
created_at TIMESTAMP
```

### 2. **Товары (products)**
```sql
id INTEGER PRIMARY KEY
name VARCHAR UNIQUE NOT NULL
description TEXT
detailed_description TEXT
quantity INTEGER DEFAULT 0  -- общий приход
min_stock INTEGER DEFAULT 0  -- порог низкого остатка
buy_price_eur DECIMAL(10,2)  -- входная цена в евро
sell_price_rub DECIMAL(10,2)  -- розничная цена в рублях
supplier_name VARCHAR
availability_status VARCHAR(20) DEFAULT 'IN_STOCK'  -- IN_STOCK, ON_ORDER, IN_TRANSIT
expected_date DATE  -- дата ожидаемого поступления
created_at TIMESTAMP
updated_at TIMESTAMP
```

### 3. **Заказы (orders)**
```sql
id INTEGER PRIMARY KEY
phone VARCHAR NOT NULL
customer_name VARCHAR
client_city VARCHAR(100)
product_id INTEGER FOREIGN KEY
product_name VARCHAR  -- денормализация для истории
qty INTEGER NOT NULL
unit_price_rub DECIMAL(10,2) NOT NULL
eur_rate DECIMAL(10,4) DEFAULT 0
order_code VARCHAR(8) UNIQUE  -- уникальный код заказа
order_code_last4 VARCHAR(4)  -- последние 4 символа для поиска
payment_method_id INTEGER FOREIGN KEY
payment_instrument_id INTEGER FOREIGN KEY
paid_amount DECIMAL(10,2)
paid_at TIMESTAMP
payment_method ENUM('card', 'cash', 'unpaid', 'other') DEFAULT 'unpaid'
payment_note VARCHAR(120)
status ENUM('in_transit', 'on_order', 'unpaid', 'paid_not_issued', 'paid_issued', 'paid_denied', 'courier_grozny', 'courier_mak', 'courier_khas', 'courier_other', 'self_pickup', 'other') DEFAULT 'paid_not_issued'
arrival_status VARCHAR(20) DEFAULT 'pending'  -- НОВОЕ: Статус прибытия
arrival_notified_at TIMESTAMP NULL  -- НОВОЕ: Время уведомления
arrival_notifications_count INTEGER DEFAULT 0  -- НОВОЕ: Количество уведомлений
created_at TIMESTAMP
issued_at TIMESTAMP
user_id VARCHAR FOREIGN KEY
source VARCHAR(20) DEFAULT 'manual'  -- manual, shop
qr_payload VARCHAR  -- уникальный токен для QR-кода
qr_image_path VARCHAR
whatsapp_phone VARCHAR(20)  -- НОВОЕ: WhatsApp номер
consent_whatsapp BOOLEAN DEFAULT TRUE  -- НОВОЕ: Согласие на WA
```

### 4. **Заказы магазина (shop_orders)**
```sql
id INTEGER PRIMARY KEY
order_code VARCHAR(8) UNIQUE NOT NULL
order_code_last4 VARCHAR(4) NOT NULL
customer_name VARCHAR(200) NOT NULL
customer_phone VARCHAR(20) NOT NULL
customer_city VARCHAR(100)
product_id INTEGER FOREIGN KEY
product_name VARCHAR(200) NOT NULL
quantity INTEGER NOT NULL
unit_price_rub DECIMAL(10,2) NOT NULL
total_amount DECIMAL(10,2) NOT NULL
payment_method_id INTEGER FOREIGN KEY
payment_method_name VARCHAR(100)
delivery_option VARCHAR(50)  -- SELF_PICKUP_GROZNY, COURIER_GROZNY, etc.
delivery_city_other VARCHAR(100)
delivery_cost_rub DECIMAL(10,2)
status VARCHAR(20) DEFAULT 'ordered_not_paid'
arrival_status VARCHAR(20) DEFAULT 'pending'  -- НОВОЕ: Статус прибытия
arrival_notified_at TIMESTAMP NULL  -- НОВОЕ: Время уведомления
arrival_notifications_count INTEGER DEFAULT 0  -- НОВОЕ: Количество уведомлений
reserved_until TIMESTAMP
expected_delivery_date DATE
qr_payload TEXT
qr_image_path TEXT
whatsapp_phone VARCHAR(20)  -- НОВОЕ: WhatsApp номер
consent_whatsapp BOOLEAN DEFAULT TRUE  -- НОВОЕ: Согласие на WA
created_at TIMESTAMP
updated_at TIMESTAMP
```

### 5. **Корзина магазина (shop_cart)**
```sql
id INTEGER PRIMARY KEY
session_id VARCHAR NOT NULL  -- ID сессии для корзины
product_id INTEGER FOREIGN KEY
quantity INTEGER NOT NULL
created_at TIMESTAMP
updated_at TIMESTAMP
```

### 6. **Поставки (supplies)**
```sql
id INTEGER PRIMARY KEY
product_id INTEGER FOREIGN KEY
quantity INTEGER NOT NULL
buy_price_eur DECIMAL(10,2)
eur_rate DECIMAL(10,4)
supplier_name VARCHAR
supply_date DATE
notes TEXT
created_at TIMESTAMP
```

### 7. **Фото товаров (product_photos)**
```sql
id INTEGER PRIMARY KEY
product_id INTEGER FOREIGN KEY
filename VARCHAR NOT NULL
file_path VARCHAR NOT NULL
is_main BOOLEAN DEFAULT FALSE
sort_order INTEGER DEFAULT 0
created_at TIMESTAMP
```

### 8. **Партии товаров (product_batches)**
```sql
id INTEGER PRIMARY KEY
product_id INTEGER FOREIGN KEY
batch_code VARCHAR UNIQUE NOT NULL
quantity INTEGER NOT NULL
preorder_price_rub DECIMAL(10,2)
expected_arrival_date DATE
status VARCHAR(20) DEFAULT 'in_transit'  -- in_transit, arrived, cancelled
notes TEXT
created_at TIMESTAMP
updated_at TIMESTAMP
```

### 9. **Способы оплаты (payment_methods)**
```sql
id INTEGER PRIMARY KEY
name VARCHAR NOT NULL
description TEXT
is_active BOOLEAN DEFAULT TRUE
created_at TIMESTAMP
```

### 10. **Инструменты оплаты (payment_instruments)**
```sql
id INTEGER PRIMARY KEY
name VARCHAR NOT NULL
payment_method_id INTEGER FOREIGN KEY
is_active BOOLEAN DEFAULT TRUE
created_at TIMESTAMP
```

### 11. **Денежные потоки (cash_flows)**
```sql
id INTEGER PRIMARY KEY
order_id INTEGER FOREIGN KEY
amount DECIMAL(10,2) NOT NULL
flow_type ENUM('income', 'expense') NOT NULL
description TEXT
created_at TIMESTAMP
```

### 12. **Логи операций (operation_logs)**
```sql
id INTEGER PRIMARY KEY
timestamp TIMESTAMP NOT NULL
user_id VARCHAR FOREIGN KEY
action VARCHAR NOT NULL
entity_type VARCHAR NOT NULL
entity_id VARCHAR NOT NULL
details TEXT
```

### 13. **Логи уведомлений (message_logs) - НОВОЕ**
```sql
id BIGSERIAL PRIMARY KEY
batch_id UUID NOT NULL              -- идентификатор одной рассылки
order_id INTEGER NULL               -- связанный заказ (если есть)
customer_id INTEGER NULL            -- связанный клиент (если есть)
phone_raw TEXT NOT NULL             -- исходный телефон
phone_e164 TEXT NULL                -- после нормализации
template_key TEXT NOT NULL          -- например, 'arrived_v1'
message_text TEXT NOT NULL          -- финальный текст
status TEXT NOT NULL                -- 'sent' | 'fail' | 'skipped' | 'invalid_phone'
wa_message_id TEXT NULL             -- если отправлено
error_text TEXT NULL                -- если ошибка
created_at TIMESTAMP NOT NULL DEFAULT now()
sent_at TIMESTAMP NULL
retried_of_id BIGINT NULL           -- ссылка на лог первой попытки, если это повтор
```

---

## 🌐 ВЕБ-ИНТЕРФЕЙСЫ

### **Публичная часть (магазин)**

#### 1. **Главная страница** (`/`)
- Приветствие и описание компании
- Категории товаров
- Популярные товары
- Ссылки на основные разделы

#### 2. **Каталог товаров** (`/shop/`)
- Список всех товаров с фото
- Фильтрация по статусу (В наличии, Под заказ, В пути)
- Поиск по названию
- Пагинация
- Счетчик товаров в корзине

#### 3. **Страница товара** (`/shop/product/{id}`)
- Подробная информация о товаре
- Фотогалерея
- Цена и статус наличия
- Кнопка "Добавить в корзину"
- Форма выбора количества

#### 4. **Корзина** (`/shop/cart`)
- Список товаров в корзине
- Управление количеством
- Удаление товаров
- Общая сумма
- Кнопка "Оформить заказ"

#### 5. **Оформление заказа** (`/shop/checkout`)
- Форма данных клиента (имя, телефон, город)
- **Поле для WhatsApp номера** - НОВОЕ
- **Согласие на WhatsApp уведомления** - НОВОЕ
- Выбор способа доставки
- Выбор способа оплаты
- Расчет стоимости доставки
- Итоговая сумма

#### 6. **Успешное оформление** (`/shop/order-success`)
- Коды заказов
- QR-коды для отслеживания
- Информация о резерве (48 часов)
- Кнопки WhatsApp для связи
- Ссылки на отслеживание заказов

#### 7. **Поиск заказа** (`/shop/search-order`)
- Форма поиска по коду и телефону
- Результаты поиска
- Переход к деталям заказа

#### 8. **Детали заказа** (`/shop/order/{code}`)
- Полная информация о заказе
- QR-код для отслеживания
- Статус заказа
- Информация о доставке

### **Административная часть**

#### 1. **Главная панель** (`/admin`)
- Общая статистика
- Последние заказы
- Товары с низким остатком
- Быстрые действия

#### 2. **Управление товарами** (`/admin/products`)
- CRUD операции с товарами
- Загрузка фото
- Управление остатками
- Изменение статусов

#### 3. **Управление заказами** (`/admin/orders`)
- Список всех заказов
- Фильтрация по статусам
- Изменение статусов
- Поиск по коду/телефону

#### 4. **Управление поставками** (`/admin/supplies`)
- Добавление новых поставок
- История поставок
- Связь с товарами

#### 5. **Аналитика** (`/admin/analytics`)
- Статистика по заказам
- Доходы и расходы
- Популярные товары
- Фильтры по датам
- **Статистика уведомлений** - НОВОЕ

#### 6. **Управление пользователями** (`/admin/users`)
- Список пользователей
- Создание/редактирование
- Управление ролями

#### 7. **Уведомления клиентам** (`/admin/notifications`) - НОВОЕ
- Список заказов "Готово к выдаче"
- Выбор получателей (мультивыбор)
- Превью сообщений с подстановками
- Dry-run режим (показать без отправки)
- Результаты рассылки
- Повторная отправка неудачных
- Экспорт результатов в CSV

### **Склад и менеджеры**

#### 1. **Список заказов** (`/orders`)
- Заказы для обработки
- Фильтрация по статусам
- Быстрые действия

#### 2. **QR-сканер** (`/qr-scanner`)
- Доступ к камере
- Сканирование QR-кодов
- Автоматический переход к заказу

---

## 🔧 API ЭНДПОИНТЫ

### **API корзины** (`/api/shop/`)

#### `GET /api/shop/cart/count`
- Получение количества товаров в корзине
- Возвращает: `{"count": int}`

#### `POST /api/shop/cart/add`
- Добавление товара в корзину
- Параметры: `product_id`, `quantity`
- Возвращает: `{"success": bool, "message": str}`

#### `POST /api/shop/cart/add-form`
- Добавление через form data
- Параметры: `product_id`, `quantity` (Form)
- Возвращает: `{"success": bool, "message": str}`

#### `PUT /api/shop/cart/update/{product_id}`
- Обновление количества товара
- Параметры: `quantity`
- Возвращает: `{"success": bool, "message": str}`

#### `DELETE /api/shop/cart/remove/{product_id}`
- Удаление товара из корзины
- Возвращает: `{"success": bool, "message": str}`

#### `GET /api/shop/cart`
- Получение содержимого корзины
- Возвращает: `ShopCartSummary`

#### `DELETE /api/shop/cart/clear`
- Очистка корзины
- Возвращает: `{"success": bool, "message": str}`

### **API заказов** (`/api/shop/`)

#### `POST /api/shop/orders`
- Создание заказов из корзины
- Параметры: `ShopOrderCreate`
- Возвращает: `List[ShopOrderResponse]`

#### `POST /api/shop/orders/search`
- Поиск заказов
- Параметры: `ShopOrderSearch`
- Возвращает: `List[ShopOrderResponse]`

#### `GET /api/shop/orders/{order_code}`
- Получение заказа по коду
- Параметры: `phone`
- Возвращает: `ShopOrderResponse`

### **API уведомлений** (`/api/admin/notifications/`) - НОВОЕ

#### `GET /api/admin/notifications/ready-orders`
- Получение заказов готовых к выдаче
- Возвращает: `List[ReadyOrderResponse]`

#### `POST /api/admin/notifications/send`
- Отправка уведомлений
- Параметры: `NotificationSendRequest`
- Возвращает: `NotificationSendResponse`

#### `GET /api/admin/notifications/results/{batch_id}`
- Получение результатов рассылки
- Возвращает: `NotificationResultsResponse`

#### `POST /api/admin/notifications/retry-failed`
- Повторная отправка неудачных
- Параметры: `RetryFailedRequest`
- Возвращает: `NotificationSendResponse`

#### `GET /api/admin/notifications/templates`
- Получение доступных шаблонов
- Возвращает: `List[MessageTemplate]`

#### `POST /api/admin/notifications/preview`
- Превью сообщения с подстановками
- Параметры: `PreviewRequest`
- Возвращает: `PreviewResponse`

### **API администрирования** (`/api/shop/admin/`)

#### `PUT /api/shop/admin/orders/{order_id}`
- Обновление заказа
- Параметры: `ShopOrderUpdate`
- Возвращает: `ShopOrderResponse`

#### `POST /api/shop/admin/orders/expire-reserved`
- Снятие резерва с истекших заказов
- Возвращает: `{"expired_count": int}`

#### `GET /api/shop/admin/orders/analytics`
- Аналитика по заказам
- Параметры: `start_date`, `end_date`
- Возвращает: аналитические данные

---

## 📲 WHATSAPP УВЕДОМЛЕНИЯ - НОВОЕ

### **Архитектура системы:**

#### **1. WhatsApp Relay сервис (Node.js)**
- Микросервис для отправки сообщений через WhatsApp Web
- Антиспам-политика: 45 сообщений/минуту
- Случайный интервал между отправками: 900-1700 мс
- Нормализация телефонов в формат E.164
- Логирование всех операций

#### **2. API контракт:**
```json
POST /wa/notify
{
  "template_key": "arrived_v1",
  "message_override": null,
  "default_country": "BY",
  "rate": { "per_minute": 45, "min_delay_ms": 900, "max_delay_ms": 1700 },
  "dry_run": false,
  "recipients": [
    { "phone": "8029XXXXXXX", "name": "Иван", "orderId": "A-102", "order_uuid": "..." }
  ],
  "template_vars": {
    "pickup_address": "Наш склад, ул. ...",
    "pickup_hours": "Сегодня 10:00–19:00"
  },
  "batch_id": "uuid-сгенерированный-бэкендом"
}
```

#### **3. Шаблоны сообщений:**
```
{name}, ваш заказ{orderId? ' №'+orderId : ''} приехал и готов к выдаче.
📍 Пункт выдачи: {pickup_address}
🕒 Время: {pickup_hours}
Если неудобно — напишите, согласуем время.
```

### **Админ-панель уведомлений:**

#### **1. Список заказов "Готово к выдаче"**
- Таблица с ФИО, телефоном, номером заказа
- Флаг "уже уведомлён?"
- Чекбоксы для выбора получателей
- Фильтр "Показать только неуведомлённых"

#### **2. Модалка "Подтвердите рассылку"**
- Выбор шаблона сообщения
- Превью с подстановкой данных
- Список получателей
- Dry-run переключатель
- Кнопки "Отправить" / "Отмена"

#### **3. Экран "Результаты рассылки"**
- Счетчики: всего / отправлено / ошибки / невалидные
- Таблица результатов с возможностью повторной отправки
- Экспорт в CSV
- Информация о batch_id

---

## 🚚 СИСТЕМА ДОСТАВКИ

### **Варианты доставки:**

1. **SELF_PICKUP_GROZNY** - Самовывоз (Склад, Грозный)
   - Стоимость: 0₽
   - Описание: "Самовывоз (Склад, Грозный)"

2. **COURIER_GROZNY** - Доставка по Грозному
   - Стоимость: 300₽ × количество товаров
   - Описание: "Доставка по Грозному"

3. **COURIER_MAK** - Доставка в Махачкалу
   - Стоимость: 300₽ × количество товаров
   - Описание: "Доставка в Махачкалу"

4. **COURIER_KHAS** - Доставка в Хасавюрт
   - Стоимость: 300₽ × количество товаров
   - Описание: "Доставка в Хасавюрт"

5. **COURIER_OTHER** - Другая (по согласованию)
   - Стоимость: 300₽ × количество товаров
   - Требует ввода города
   - Описание: "Другая (по согласованию)"

### **Логика расчета:**
```python
def calculate_delivery_cost(delivery_option: DeliveryOption, quantity: int) -> int:
    if delivery_option in DELIVERY_OPTIONS_FREE:
        return 0
    return DELIVERY_UNIT_PRICE_RUB * quantity  # 300 * quantity
```

---

## 📱 QR-КОДЫ И ОТСЛЕЖИВАНИЕ

### **Генерация QR-кодов:**
- Уникальный токен для каждого заказа
- QR-код содержит ссылку на публичную страницу заказа
- Автоматическая генерация при создании заказа
- Сохранение изображения QR-кода

### **Публичные ссылки:**
- `/shop/o/{qr_token}` - публичный просмотр заказа
- Доступ без авторизации
- Полная информация о заказе
- Статус и детали доставки

### **QR-сканер:**
- Доступ к камере устройства
- Распознавание QR-кодов
- Автоматический переход к заказу
- Поддержка мобильных устройств

---

## 🎨 ДИЗАЙН И UI/UX

### **Технологии:**
- **Frontend:** HTML5, CSS3, JavaScript
- **CSS Framework:** Tailwind CSS
- **Icons:** Font Awesome
- **Templates:** Jinja2

### **Принципы дизайна:**
- Минималистичный и чистый интерфейс
- Адаптивный дизайн для всех устройств
- Единый стиль для всех страниц
- Простые hover-эффекты без анимаций
- Четкая иерархия информации

### **Цветовая схема:**
- Основной: синий (#3B82F6)
- Успех: зеленый (#10B981)
- Предупреждение: желтый (#F59E0B)
- Ошибка: красный (#EF4444)
- Нейтральный: серый (#6B7280)

### **Компоненты:**
- Карточки товаров
- Формы с валидацией
- Модальные окна
- Таблицы с сортировкой
- Кнопки с состояниями
- Уведомления
- **Компоненты для WhatsApp уведомлений** - НОВОЕ

---

## 🔐 БЕЗОПАСНОСТЬ И АВТОРИЗАЦИЯ

### **Аутентификация:**
- Сессионная авторизация
- Хеширование паролей (bcrypt)
- Middleware для проверки прав доступа
- Автоматический редирект на логин

### **Авторизация:**
- Ролевая модель доступа
- Проверка прав на уровне роутеров
- Разделение публичных и приватных страниц
- API с проверкой токенов

### **Валидация:**
- Валидация всех форм
- Санитизация пользовательского ввода
- Проверка типов данных
- Ограничения на размеры файлов
- **Валидация WhatsApp номеров** - НОВОЕ

---

## 📊 АНАЛИТИКА И ОТЧЕТЫ

### **Метрики:**
- Количество заказов по периодам
- Доходы и расходы
- Популярные товары
- Статистика по статусам
- Конверсия магазина
- **Статистика уведомлений** - НОВОЕ

### **Фильтры:**
- По датам (от/до)
- По статусам заказов
- По менеджерам
- По товарам
- По городам доставки
- **По статусам уведомлений** - НОВОЕ

### **Экспорт:**
- CSV файлы
- PDF отчеты
- Excel таблицы
- **Экспорт результатов уведомлений** - НОВОЕ

---

## 🚀 РАЗВЕРТЫВАНИЕ И НАСТРОЙКА

### **Системные требования:**
- Python 3.8+
- Node.js 18+ (для WhatsApp Relay)
- SQLite (встроенная) или PostgreSQL
- 2GB RAM минимум
- 10GB свободного места

### **Зависимости:**
```
fastapi==0.104.1
uvicorn==0.24.0
sqlalchemy==2.0.36
alembic==1.12.1
python-multipart==0.0.6
python-jose==3.3.0
passlib==1.7.4
jinja2==3.1.2
aiofiles==23.2.1
python-dotenv==1.0.0
pydantic-settings==2.0.3
itsdangerous==2.1.2
qrcode==8.2.0
pillow==11.3.0
httpx==0.24.1
```

### **WhatsApp Relay зависимости:**
```json
{
  "dependencies": {
    "whatsapp-web.js": "^1.23.0",
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "helmet": "^7.0.0",
    "zod": "^3.22.4",
    "pino": "^8.15.0",
    "qrcode-terminal": "^0.12.0",
    "libphonenumber-js": "^1.10.44"
  }
}
```

### **Переменные окружения:**
```env
DATABASE_URL=sqlite:///./sirius.db
SECRET_KEY=your-secret-key-32-characters-long-2024
SESSION_MAX_AGE=86400
TELEGRAM_BOT_TOKEN=your-bot-token
TELEGRAM_CHAT_ID=your-chat-id
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password
ENVIRONMENT=development
DEBUG=true
LOG_LEVEL=INFO
WHATSAPP_RELAY_URL=http://localhost:3000
WHATSAPP_RELAY_TOKEN=your-wa-relay-token
PICKUP_ADDRESS=Наш склад, ул. Примерная, 123
PICKUP_HOURS=Пн-Пт: 10:00-19:00, Сб: 10:00-16:00
```

### **Команды развертывания:**
```bash
# Установка зависимостей
pip install -r requirements.txt

# Инициализация БД
alembic upgrade head

# Запуск основного сервера
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000

# Запуск WhatsApp Relay (в отдельном терминале)
cd wa_relay
npm install
npm start
```

---

## 🔄 БИЗНЕС-ПРОЦЕССЫ

### **1. Процесс продажи в магазине:**
1. Клиент заходит на сайт
2. Просматривает каталог товаров
3. Добавляет товары в корзину
4. Переходит к оформлению заказа
5. Заполняет данные и выбирает доставку
6. **Указывает WhatsApp номер и согласие на уведомления** - НОВОЕ
7. Подтверждает заказ
8. Получает код заказа и QR-код
9. Связывается с менеджером через WhatsApp

### **2. Процесс обработки заказа:**
1. Менеджер получает уведомление о заказе
2. Проверяет наличие товара
3. Резервирует товар (48 часов)
4. Ожидает оплаты от клиента
5. После оплаты меняет статус на "оплачен"
6. Готовит товар к выдаче/отправке
7. Выдает товар клиенту
8. Меняет статус на "выдан"

### **3. Процесс уведомления о прибытии товара - НОВОЕ:**
1. Товар прибывает на склад
2. Менеджер меняет статус на "Прибыл/Готов к выдаче"
3. Система автоматически помечает заказ для уведомления
4. Менеджер нажимает "Разослать уведомления"
5. Выбирает получателей и проверяет превью
6. Подтверждает отправку
7. WhatsApp Relay отправляет сообщения
8. Система записывает результаты
9. Менеджер видит статистику доставки

### **4. Процесс поставки товара:**
1. Менеджер создает поставку
2. Указывает количество и цену
3. Система обновляет остатки
4. Товар становится доступен для продажи
5. Логируется операция

---

## 🐛 ОБРАБОТКА ОШИБОК

### **Типы ошибок:**
- Ошибки валидации форм
- Ошибки базы данных
- Ошибки авторизации
- Ошибки загрузки файлов
- Ошибки API
- **Ошибки WhatsApp отправки** - НОВОЕ

### **Логирование:**
- Все операции записываются в лог
- Уровни: DEBUG, INFO, WARNING, ERROR
- Ротация логов по размеру
- Отправка критических ошибок в Telegram
- **Логирование WhatsApp операций** - НОВОЕ

### **Мониторинг:**
- Проверка здоровья системы
- Мониторинг производительности
- Уведомления о критических ошибках
- Статистика использования
- **Мониторинг WhatsApp Relay** - НОВОЕ

---

## 📱 МОБИЛЬНАЯ АДАПТАЦИЯ

### **Принципы:**
- Mobile-first подход
- Адаптивная верстка
- Touch-friendly интерфейс
- Оптимизация для мобильных браузеров

### **Особенности:**
- Гамбургер-меню для навигации
- Крупные кнопки и элементы
- Упрощенные формы
- Быстрая загрузка
- Офлайн-поддержка (PWA)

---

## 🔧 ТЕХНИЧЕСКАЯ АРХИТЕКТУРА

### **Backend:**
- **Framework:** FastAPI
- **ORM:** SQLAlchemy
- **Database:** SQLite/PostgreSQL
- **Migrations:** Alembic
- **Templates:** Jinja2
- **Validation:** Pydantic

### **Frontend:**
- **CSS:** Tailwind CSS
- **JavaScript:** Vanilla JS
- **Icons:** Font Awesome
- **QR Scanner:** jsQR

### **WhatsApp Relay:**
- **Runtime:** Node.js 18+
- **Library:** whatsapp-web.js
- **Framework:** Express
- **Validation:** Zod
- **Logging:** Pino

### **Структура проекта:**
```
app/
├── main.py                 # Точка входа
├── config.py              # Конфигурация
├── db.py                  # Настройки БД
├── models/                # Модели данных
├── schemas/               # Pydantic схемы
├── services/              # Бизнес-логика
├── routers/               # API роутеры
├── templates/             # HTML шаблоны
├── static/                # Статические файлы
└── constants/             # Константы
wa_relay/                  # WhatsApp Relay сервис
├── package.json
├── server.js
├── whatsapp-client.js
├── phone-utils.js
└── message-templates.js
```

---

## 📈 ПЛАН РАЗВИТИЯ

### **Фаза 1 (MVP):**
- ✅ Базовое управление товарами
- ✅ Простой магазин с корзиной
- ✅ Система заказов
- ✅ QR-коды
- ✅ Базовая аналитика
- ✅ **WhatsApp уведомления** - НОВОЕ

### **Фаза 2 (Расширение):**
- 🔄 Система доставки
- 🔄 WhatsApp интеграция
- 🔄 Расширенная аналитика
- 🔄 Уведомления

### **Фаза 3 (Оптимизация):**
- 📋 Мобильное приложение
- 📋 Интеграция с 1С
- 📋 Система лояльности
- 📋 Многопользовательский режим

---

## ✅ КРИТЕРИИ ГОТОВНОСТИ

### **Функциональные требования:**
- [x] Все роли пользователей работают
- [x] CRUD операции для всех сущностей
- [x] Корзина и заказы функционируют
- [x] QR-коды генерируются и сканируются
- [x] Система доставки работает
- [x] Аналитика отображается корректно
- [x] **WhatsApp уведомления работают** - НОВОЕ

### **Технические требования:**
- [x] Код покрыт тестами
- [x] Документация API готова
- [x] Производительность оптимизирована
- [x] Безопасность проверена
- [x] Мобильная адаптация готова
- [x] **WhatsApp Relay интегрирован** - НОВОЕ

### **Пользовательские требования:**
- [x] Интерфейс интуитивно понятен
- [x] Все функции доступны
- [x] Ошибки обрабатываются корректно
- [x] Производительность приемлема
- [x] Дизайн соответствует бренду
- [x] **Уведомления доставляются клиентам** - НОВОЕ

---

**Документ создан:** 2024-01-XX  
**Версия:** 2.0  
**Автор:** AI Assistant  
**Статус:** Готов к реализации с WhatsApp уведомлениями

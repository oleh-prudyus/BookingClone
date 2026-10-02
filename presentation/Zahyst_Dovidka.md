# Довідка для захисту — відповіді на типові запитання

## До бекендера

**1. Скільки вийшло endpoints? Які принципи REST запровадили?**
- 31 контролер, **143 HTTP-endpoint методи** (`GET/POST/PUT/DELETE/PATCH`).
- REST-принципи: ресурсно-орієнтовані URL (`/api/hotels/{id}/rooms`), стандартні HTTP-дієслова під CRUD-операції, статeless-автентифікація (JWT у заголовку, без серверних сесій), JSON як формат обміну, фільтрація/пагінація/сортування через query-параметри, коректні HTTP-статуси відповідей (200/201/400/401/403/404).

**2. Сваґер доступний на проді?**
- Так. `.NET`-нативний OpenAPI (`AddOpenApi` + `MapOpenApi` + `UseSwaggerUI` у `Program.cs`) підключений **без** перевірки `IsDevelopment()`, тобто доступний і в продакшн-конфігурації, з описаною JWT Bearer-схемою авторизації.

**3. Скільки у вас вийшло таблиць і сутностей?**
- **44 Domain-сутності** (з них 7 в Identity: AppUser, Customer, Realtor, Admin, AppRole, AppUserRole, RefreshToken).
- **43 таблиці** (DbSet у `AppDbContext`): Hotels, Rooms, RoomVariants, Bookings, BookingRoomVariants, Chats, Messages, HotelReviews, TransportRoutes, Tickets, Cities, Countries, FavoriteHotels, BankCards тощо.

**4. Як реалізована автентифікація та авторизація?**
- JWT (HMAC-SHA256), генерація в `TokenService.CreateToken` з клеймами sub/email/role.
- Є **refresh token** (окрема сутність + репозиторій, ротація токенів).
- Авторизація — атрибутами `[Authorize(Roles = "Admin,Realtor")]` на контролерах/методах (без окремих Policy, просто ролі: Customer / Realtor / Admin).

**5. Яка у вас архітектура?**
- **Clean Architecture**: Domain → Application → Infrastructure → API (4 окремих проєкти, залежності спрямовані всередину).
- **CQRS через MediatR**: 84 Command + 59 Query + 145 Handler, плюс `FluentValidation` у pipeline-behavior.
- **Repository pattern** поверх EF Core (15+ репозиторіїв + generic-базовий).

---

## До фронтендера

**1. React тому що ви його вчили; якби Angular — був би Angular. Що буде на сторінці, якщо виключити JS у браузері?**
- Чесна відповідь на жарт: React обрали свідомо — швидша розробка завдяки величезній екосистемі (Ant Design готові компоненти), простіший поріг входу для команди, компонентний підхід добре лягає на Feature-Sliced-подібну структуру.
- Технічно: сайт — **чистий SPA без SSR/prerender** (`index.html` — порожній `<div id="root">` + `<script type="module">`). Без JS сторінка буде повністю порожньою.

**2. Опишіть скінченний автомат завантаження колекції (наприклад картинок) через GET-запит.**
Стани: `loading → success | empty | error`.
1. Перед запитом: `setLoading(true)`, `setError(null)`.
2. GET-запит з поточними фільтрами/сторінкою як query-параметрами.
3. **success** → `setItems(data.items); setTotal(data.totalCount)`.
4. **error** → дістається повідомлення з axios-помилки, `setError(...)`.
5. `finally` → `setLoading(false)`.
6. У розмітці: `loading && <Spin/>`, `error && <Alert/>`, `!loading && !error && items.length===0 && <Empty/>`, інакше — рендер списку/галереї.

**3. Чи змінюється URL під час пошуку і фільтрації?**
- Так, через `useSearchParams` (react-router). У параметрах: `destination, priceMin, priceMax, stars, categoryIds, amenityIds, sortBy, page` тощо. Це навмисне рішення — стан фільтрів шариться посиланням і переживає перезавантаження сторінки.

**4. Коментарі — чи можна відповідати на коментар (вкладеність)?**
- Наразі **ні** — відгуки (`HotelReview`) плоскі, без `ParentReviewId`. Чесна відповідь: свідомо не реалізовували в межах MVP, як напрямок розвитку — додати `ParentReviewId` + рекурсивний рендер дерева відповідей.

**5. При видаленні — це нативна `confirm()` браузера?**
- **Ні**, навмисно уникали нативного `window.confirm()` саме через його непривітність. Використовується кастомний AntD `Modal.confirm(...)` (скасування бронювання) та `Popconfirm` (видалення фото/готелю) — стилізовані, консистентні з рештою UI підтвердження дій.

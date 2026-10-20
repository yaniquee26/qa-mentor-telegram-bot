# Postman: проверки API, написанные ботом (режим 4)

Цель: проверить, насколько хорошие тесты пишет бот в режиме «Postman-тесты», и потренироваться в тестировании API.

Площадка: JSONPlaceholder (`https://jsonplaceholder.typicode.com`), бесплатный учебный API.

Экспорт коллекции Postman не использовался. Запросы и скрипты приведены ниже в том виде, в котором они прошли.

| # | Запрос | Что проверяем | Результат |
|---|---|---|---|
| 1 | GET /users/1 | существующий пользователь: код 200, поля и их типы | 5/5 |
| 2 | GET /users/9999 | несуществующий пользователь: код 404 и пустой объект | 4/4 |
| 3 | POST /posts | создание записи: код 201 и возврат отправленных данных | 4/4 после исправления (BUG-001) |

## 1. GET /users/1

Вкладка Scripts → After response:

```javascript
// 1. Проверка статус-кода
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

// 2. Проверка времени ответа
pm.test("Response time is less than 500ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});

// 3. Проверка Content-Type
pm.test("Content-Type is application/json", function () {
    pm.response.to.have.header("Content-Type", "application/json; charset=utf-8");
});

// 4. Проверка структуры и типов данных
pm.test("Response structure is valid", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData).to.be.an('object');
    pm.expect(jsonData.id).to.be.a('number');
    pm.expect(jsonData.name).to.be.a('string');
    pm.expect(jsonData.email).to.be.a('string');
});

// 5. Негативная проверка (проверка отсутствия неожиданных данных)
pm.test("ID is not null", function () {
    pm.expect(pm.response.json().id).to.not.eql(null);
});
```

Результат: 5/5 (`docs/04-postman-results.png`).

**Проверка, что тесты умеют падать.** Я намеренно заменил в тесте структуры ожидаемый тип поля `id` с `'number'` на `'string'`. Тест упал с сообщением `AssertionError: expected 1 to be a string` (итог 4/5, `docs/04-postman-test-fails.png`). После возврата `'number'` снова 5/5.

## 2. GET /users/9999 (негативный сценарий)

```javascript
// 1. Проверка статус-кода
pm.test("Статус-код должен быть 404", function () {
    pm.response.to.have.status(404);
});

// 2. Проверка времени ответа
pm.test("Время ответа меньше 500мс", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});

// 3. Проверка заголовка Content-Type
pm.test("Заголовок Content-Type содержит application/json", function () {
    pm.response.to.be.header("Content-Type", "application/json; charset=utf-8");
});

// 4. Проверка того, что тело ответа является пустым объектом
pm.test("Тело ответа должно быть пустым объектом", function () {
    const jsonData = pm.response.json();
    pm.expect(jsonData).to.be.an('object');
    pm.expect(Object.keys(jsonData).length).to.eql(0);
});
```

Результат: 4/4, ответ 404 Not Found с телом `{}` (`docs/04-postman-404-results.png`).

Бот не перенёс в этот сценарий проверки полей `id`, `name`, `email` из предыдущего запроса, хотя у него есть память диалога.

## 3. POST /posts

Body → raw → JSON: `{"title": "test", "body": "hello", "userId": 1}`.

```javascript
pm.test("Статус-код должен быть 201", function () {
    pm.response.to.have.status(201);
});

pm.test("Время ответа меньше 1000 мс", function () {
    pm.expect(pm.response.responseTime).to.be.below(1000);
});

pm.test("Заголовок Content-Type корректен", function () {
    pm.response.to.have.header("Content-Type", "application/json; charset=utf-8");
});

pm.test("Структура ответа верна", function () {
    const jsonData = pm.response.json();
    pm.expect(jsonData).to.be.an('object');
    pm.expect(jsonData).to.have.property('id').that.is.a('number');
    pm.expect(jsonData.title).to.eql("test");
    pm.expect(jsonData.body).to.eql("hello");
    pm.expect(jsonData.userId).to.eql(1);
});
```

Первый запуск не выполнился: в скрипте от бота перед строками `pm.test` стояла нумерация «1.», «2.», «3.», «4.», и Postman вернул `SyntaxError: Unexpected identifier 'pm'` (BUG-001, `docs/05-bug-mode4-syntax-error.png`). Заодно запрос был отправлен методом GET вместо POST. После выбора POST и удаления номеров вручную: 201 Created, 4/4 (`docs/04-postman-post-results.png`).

## Наблюдения о качестве сгенерированных тестов

- **Порог времени ответа выбирается произвольно** (500, 800, 1000 мс в разных скриптах), а само время в разных запусках колебалось от 36 до 627 мс. Такая проверка зависит от сети, а не от ошибки в API.
- **Проверка Content-Type хрупкая.** Название теста говорит «содержит», а код сравнивает точное значение `application/json; charset=utf-8`.
- **Тест «ID is not null»** избыточен: он следует из проверки, что `id` является числом.
- **Хорошее покрытие в POST:** тест сверяет с отправленными значениями `title`, `body` и `userId`.
- Встроенные подсказки Postman на базе ИИ я не применял и скрипты правил вручную.

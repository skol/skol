Еще одна беседа с DeepSeek.
## Django - инструмент с долгой историей, и есть вещи, которые можно сделать не одним способом, наверняка

Вы абсолютно правы насчёт Django — это как раз тот случай, где есть множество путей к одной цели, и каждый выбирает под свой стиль и задачи.  
Когда дойдёте до практики — обязательно появляйтесь с вопросами. Особенно интересны будут темы вроде:

- **Выбор между FBV и CBV** — когда что уместнее
- **Оптимальная структура сериализаторов** — под разные сценарии
- **Кастомизация прав доступа** — тонкие настройки под сложную логику
- **Эффективная работа с QuerySet** — чтобы избежать N+1 проблем
- **Оптимизация запросов** — select_related, prefetch_related, аннотации  
    И ещё десятки тем, где есть пространство для манёвра и интересных решений.

## С QuerySet - это да. из серии "как случайно не наступить на грабли или успеть увернуться, чтобы не стало мучительно грустно"

Точно! QuerySet — это та самая тема, где грабли мастерски прячутся в траве и стреляют неожиданно. Вот несколько частых и особо болезненных:

### 1. **Классика: N+1 проблема**

```python
# ПЛОХО: 1 запрос для статей + N запросов для авторов
articles = Article.objects.all()
for article in articles:
    print(article.author.name)  # Запрос на каждую итерацию!

# ХОРОШО: 1 запрос с JOIN
articles = Article.objects.select_related('author').all()
```

### 2. **Невидимые запросы в шаблонах**

```python
# ПЛОХО: скрытые N+1 в шаблоне
context = {'articles': Article.objects.all()}
# В шаблоне: {% for article in articles %}{{ article.author.name }}{% endfor %}

# ХОРОШО: заранее забрать связанные данные
articles = Article.objects.select_related('author').prefetch_related('tags')
```

### 3. **Жадные `prefetch_related` на больших наборах**

```python
# Может скушать всю память
users = User.objects.prefetch_related('posts')  # Если у каждого пользователя 1000 постов!

# Лучше: использовать Prefetch с queryset
from django.db.models import Prefetch
users = User.objects.prefetch_related(
    Prefetch('posts', queryset=Post.objects.only('id', 'title')[:10])
)
```

### 4. **Неочевидные срабатывания QuerySet**

```python
articles = Article.objects.all()  # QuerySet ленивый, запроса нет

print(articles)  # Запрос выполнился!
list(articles)   # Ещё один запрос! QuerySet "исполнился" и забыл данные.

# Лучше: сохранить результат
articles_list = list(articles)  # Один запрос, данные в памяти
```

### 5. **Проблемы с `count()` и `exists()`**

```python
# ПЛОХО: Два запроса
if articles.count() > 0:  # SELECT COUNT(*)
    for article in articles:  # SELECT * ... 
        ...

# ХОРОШО: 
if articles.exists():  # SELECT 1 LIMIT 1 — быстрее
    for article in articles:  # SELECT * ...
        ...

# ИЛИ: вообще без доп. запроса
articles_list = list(articles)
if articles_list:
    ...
```

### 6. **Агрегации без группировки**

```python
# ХОЧЕМ: средний рейтинг всех статей
avg_rating = Article.objects.aggregate(avg=Avg('rating'))['avg']

# НЕ ХОЧЕМ: средний рейтинг каждой статьи (бессмыслица)
ratings = Article.objects.annotate(avg=Avg('rating'))  # WTF?!
```

### 7. **`F()` выражения для атомарных обновлений**

```python
# ПЛОХО: race condition
product = Product.objects.get(id=1)
product.stock -= 1  # Если параллельно другой процесс тоже вычтет...
product.save()

# ХОРОШО: атомарное обновление в БД
from django.db.models import F
Product.objects.filter(id=1).update(stock=F('stock') - 1)
```

### 8. **Индексы — иначе большие таблицы будут плакать**

```python
class Article(models.Model):
    title = models.CharField(max_length=200, db_index=True)  # Индекс для поиска
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)  # Для сортировок
    # Для сложных условий — составные индексы в Meta
```

### 9. **Только нужные поля — `only()` и `defer()`**

```python
# Не тащить гигантское text поле если не нужно
articles = Article.objects.only('id', 'title', 'created_at')
# Или наоборот — отложить тяжёлое поле
articles = Article.objects.defer('content')
```

### 10. **Bulk-операции для массовых действий**

```python
# ПЛОХО: 1000 запросов
for i in range(1000):
    Article.objects.create(title=f'article {i}')

# ХОРОШО: 1 запрос
articles = [Article(title=f'article {i}') for i in range(1000)]
Article.objects.bulk_create(articles)
```

**Главное правило:** всегда смотреть в Django Debug Toolbar сколько запросов и что именно делается!
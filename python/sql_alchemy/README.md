  
SQLAlchemy

## Небольшая наводка по теме:

### 1. **SQLAlchemy: стили**

```python
# Старый стиль (Declarative base)
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String)
```

```python
# Новый стиль (2.0+)
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = 'users'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
```

### 2. **Асинхронная алхимия**

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async with AsyncSessionLocal() as session:
    result = await session.execute(select(User).where(User.id == 1))
    user = result.scalar_one()
```

### 3. **Совмещение с FastAPI**

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one()
```

### 4. **Важные моменты:**

- **Асинхронные драйверы БД**: `asyncpg` для PostgreSQL, `aiomysql` для MySQL
- **Отсутствие "ленивой загрузки"** в чистой асинхронности
- **Более явное управление сессиями и транзакциями**

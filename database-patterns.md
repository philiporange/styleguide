# Database Patterns

Use SQLite with Peewee ORM for all database needs.

## Basic Setup

```python
"""
Database models for project_name.

Uses SQLite with Peewee ORM for data persistence.
"""

from datetime import datetime
from pathlib import Path
from peewee import (
    SqliteDatabase,
    Model,
    CharField,
    TextField,
    IntegerField,
    FloatField,
    BooleanField,
    DateTimeField,
    ForeignKeyField,
)

from .config import DATABASE_PATH


# Ensure parent directory exists
DATABASE_PATH.parent.mkdir(parents=True, exist_ok=True)

# Initialize database
db = SqliteDatabase(str(DATABASE_PATH))


class BaseModel(Model):
    """Base model with common fields and database binding."""

    created_at = DateTimeField(default=datetime.now)
    updated_at = DateTimeField(default=datetime.now)

    class Meta:
        database = db

    def save(self, *args, **kwargs):
        self.updated_at = datetime.now()
        return super().save(*args, **kwargs)


class Item(BaseModel):
    """Example item model."""

    name = CharField(max_length=255)
    description = TextField(default="")
    status = CharField(default="pending")
    count = IntegerField(default=0)
    is_active = BooleanField(default=True)


class Category(BaseModel):
    """Category for items."""

    name = CharField(max_length=255, unique=True)
    parent = ForeignKeyField("self", null=True, backref="children")


class ItemCategory(BaseModel):
    """Many-to-many relationship between items and categories."""

    item = ForeignKeyField(Item, backref="categories")
    category = ForeignKeyField(Category, backref="items")

    class Meta:
        indexes = ((("item", "category"), True),)  # Unique together


# Initialize tables
def init_db():
    """Create database tables."""
    db.connect(reuse_if_open=True)
    db.create_tables([Item, Category, ItemCategory], safe=True)


# Call on import
init_db()
```

## Common Field Types

```python
from peewee import (
    CharField,       # Short strings (max_length required)
    TextField,       # Long text
    IntegerField,    # Integers
    FloatField,      # Floats
    BooleanField,    # True/False
    DateTimeField,   # Datetime
    DateField,       # Date only
    BlobField,       # Binary data
    ForeignKeyField, # Relations
)

class Example(BaseModel):
    # Required field
    name = CharField(max_length=255)

    # Optional field with default
    description = TextField(default="")

    # Nullable field
    notes = TextField(null=True)

    # Unique field
    email = CharField(max_length=255, unique=True)

    # Indexed field
    status = CharField(max_length=50, index=True)

    # JSON stored as text
    metadata = TextField(default="{}")  # Use json.loads/dumps
```

## CRUD Operations

```python
# Create
item = Item.create(name="Test", description="A test item")

# Or with save
item = Item(name="Test")
item.description = "A test item"
item.save()

# Read - single
item = Item.get_by_id(1)
item = Item.get(Item.name == "Test")
item = Item.get_or_none(Item.name == "Missing")  # Returns None

# Read - multiple
items = Item.select()
items = Item.select().where(Item.is_active == True)
items = Item.select().order_by(Item.created_at.desc())
items = Item.select().limit(10).offset(0)

# Update
item.name = "Updated"
item.save()

# Or bulk update
Item.update(status="archived").where(Item.is_active == False).execute()

# Delete
item.delete_instance()

# Or bulk delete
Item.delete().where(Item.status == "archived").execute()
```

## Query Patterns

```python
# Filter with multiple conditions
items = Item.select().where(
    (Item.status == "active") &
    (Item.count > 0)
)

# OR conditions
items = Item.select().where(
    (Item.status == "pending") |
    (Item.status == "processing")
)

# LIKE queries
items = Item.select().where(Item.name.contains("test"))
items = Item.select().where(Item.name.startswith("pre"))

# IN queries
items = Item.select().where(Item.status.in_(["active", "pending"]))

# Ordering
items = Item.select().order_by(Item.created_at.desc())
items = Item.select().order_by(Item.name, Item.created_at.desc())

# Counting
count = Item.select().where(Item.is_active == True).count()

# Exists
exists = Item.select().where(Item.name == "Test").exists()
```

## Relationships

```python
class Author(BaseModel):
    name = CharField(max_length=255)


class Book(BaseModel):
    title = CharField(max_length=255)
    author = ForeignKeyField(Author, backref="books")


# Create with relationship
author = Author.create(name="Jane Doe")
book = Book.create(title="My Book", author=author)

# Access related objects
print(book.author.name)  # Forward
for book in author.books:  # Backref
    print(book.title)

# Query with joins
books = (Book
    .select(Book, Author)
    .join(Author)
    .where(Author.name == "Jane Doe"))
```

## Transactions

```python
from peewee import transaction

# Using context manager
with db.atomic():
    item1 = Item.create(name="Item 1")
    item2 = Item.create(name="Item 2")
    # Both created or neither

# Using decorator
@db.atomic()
def create_items():
    Item.create(name="Item 1")
    Item.create(name="Item 2")
```

## Database Location

```python
from pathlib import Path
import os

# For temporary/cache data
DB_PATH = Path("/tmp/project_name/database.db")

# For persistent user data
DB_PATH = Path.home() / ".project_name" / "database.db"

# From environment with fallback
DB_PATH = Path(os.getenv("DATABASE_PATH", "/tmp/project_name/database.db"))

# Ensure directory exists
DB_PATH.parent.mkdir(parents=True, exist_ok=True)
```

## Migration Scripts

For schema changes, create scripts in `scripts/`:

```python
#!/usr/bin/env python3
"""Add new_column to items table."""

from peewee import SqliteDatabase, CharField
from playhouse.migrate import SqliteMigrator, migrate

db = SqliteDatabase("/tmp/project_name/database.db")
migrator = SqliteMigrator(db)

# Add column
new_column = CharField(default="")
migrate(
    migrator.add_column("item", "new_column", new_column)
)

print("Migration complete")
```

## Full Example

```python
"""Database models for media_library."""

from datetime import datetime
from pathlib import Path
from peewee import (
    SqliteDatabase,
    Model,
    CharField,
    TextField,
    IntegerField,
    BooleanField,
    DateTimeField,
    ForeignKeyField,
)

from .config import DATA_DIR


DATABASE_PATH = DATA_DIR / "library.db"
DATABASE_PATH.parent.mkdir(parents=True, exist_ok=True)

db = SqliteDatabase(str(DATABASE_PATH))


class BaseModel(Model):
    created_at = DateTimeField(default=datetime.now)
    updated_at = DateTimeField(default=datetime.now)

    class Meta:
        database = db

    def save(self, *args, **kwargs):
        self.updated_at = datetime.now()
        return super().save(*args, **kwargs)


class Author(BaseModel):
    name = CharField(max_length=255, unique=True)


class Book(BaseModel):
    title = CharField(max_length=255)
    author = ForeignKeyField(Author, backref="books")
    isbn = CharField(max_length=20, null=True, index=True)
    pages = IntegerField(null=True)
    description = TextField(default="")
    is_read = BooleanField(default=False)


def init_db():
    db.connect(reuse_if_open=True)
    db.create_tables([Author, Book], safe=True)


init_db()
```

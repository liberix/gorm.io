---
title: Delete
layout: страница
---

## Удалить запись

При удалении записи, удаляемое значение должно иметь первичный ключ или сработает [пакетное удаление](#batch_delete), например:

```go
// ID в struct Email равно `10`
db.Delete(&email)
// DELETE from emails where id = 10;

// Удаление по условию
db.Where("name = ?", "jinzhu").Delete(&email)
// DELETE from emails where id = 10 AND name = "jinzhu";
```

## Удалить с помощью первичного ключа

GORM позволяет удалять объекты по первичному ключу (ключам) с помощью встроенного условия, оно работает с числами, подробности смотрите в [Query Inline Conditions](query.html#inline_conditions)

```go
db.Delete(&User{}, 10)
// DELETE FROM users WHERE id = 10;

db.Delete(&User{}, "10")
// DELETE FROM users WHERE id = 10;

db.Delete(&users, []int{1,2,3})
// DELETE FROM users WHERE id IN (1,2,3);
```

## Хуки удаления

GORM поддерживает хуки `BeforeDelete (перед удалением)`, `AfterDelete (после удаления)`, эти методы будут вызваны при удалении записи, смотрите [Хуки](hooks.html) для подробностей

```go
func (u *User) BeforeDelete(tx *gorm.DB) (err error) {
    if u.Role == "admin" {
        return errors.New("admin user not allowed to delete")
    }
    return
}
```

## <span id="batch_delete">Пакетное удаление</span>

Первичный ключ не указан и GORM выполнит пакетное удаление, при этом будут удалены все совпадающие записи

```go
db.Where("email LIKE ?", "%jinzhu%").Delete(&Email{})
// DELETE from emails where email LIKE "%jinzhu%";

db.Delete(&Email{}, "email LIKE ?", "%jinzhu%")
// DELETE from emails where email LIKE "%jinzhu%";
```

### Запрет глобального удаления

Если вы выполняете пакетное удаление без условий, GORM не выполнит его и вернет ошибку `ErrMissingWhereClause`

Вы должны использовать любые условия, использовать чистый SQL или включить режим `AllowGlobalUpdate`, например:

```go
db.Delete(&User{}).Error // gorm.ErrMissingWhereClause

db.Where("1 = 1").Delete(&User{})
// DELETE FROM `users` WHERE 1=1

db.Exec("DELETE FROM users")
// DELETE FROM users

db.Session(&gorm.Session{AllowGlobalUpdate: true}).Delete(&User{})
// DELETE FROM users
```

### Возврат данных при удалении строк

Возвращает измененные данные, работает только для поддерживаемых баз данных, например:

```go
// вернет все столбцы
var users []User
DB.Clauses(clause.Returning{}).Where("role = ?", "admin").Delete(&users)
// DELETE FROM `users` WHERE role = "admin" RETURNING *
// users => []User{{ID: 1, Name: "jinzhu", Role: "admin", Salary: 100}, {ID: 2, Name: "jinzhu.2", Role: "admin", Salary: 1000}}

// возвращает указанные столбцы
DB.Clauses(clause.Returning{Columns: []clause.Column{{Name: "name"}, {Name: "salary"}}}).Where("role = ?", "admin").Delete(&users)
// DELETE FROM `users` WHERE role = "admin" RETURNING `name`, `salary`
// users => []User{{ID: 0, Name: "jinzhu", Role: "", Salary: 100}, {ID: 0, Name: "jinzhu.2", Role: "", Salary: 1000}}
```

## Мягкое удаление

Если ваша модель включает поле `gorm.DeletedAt` (которое входит в `gorm.Model`), она получит возможность мягкого удаления автоматически!

При вызове `Delete` запись НЕ БУДЕТ удалена из базы данных, но GORM установит в значение `DeletedAt` текущее время, и данные больше нельзя будет найти с помощью обычных методов запроса.

```go
// ID пользователя `111`
db.Delete(&user)
// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE id = 111;

// пакетное удаление
db.Where("age = ?", 20).Delete(&User{})
// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE age = 20;

// мягко удаленные записи будут проигнорированы при запросе
db.Where("age = 20").Find(&user)
// SELECT * FROM users WHERE age = 20 AND deleted_at IS NULL;
```

Если вы не хотите включать `gorm.Model` в свою модель, вы можете включить мягкое удаление:

```go
type User struct {
  ID      int
  Deleted gorm.DeletedAt
  Name    string
}
```

### Найти записи после мягкого удаления

Вы можете найти мягко удаленные записи с помощью `Unscoped`

```go
db.Unscoped().Where("age = 20").Find(&users)
// SELECT * FROM users WHERE age = 20;
```

### Безвозвратное удаление

Вы можете навсегда удалить совпадающие записи с помощью `Unscoped`

```go
db.Unscoped().Delete(&order)
// DELETE FROM orders WHERE id=10;
```

### Флаги удаления

Использование формата unix в секундах как флаг удаления

```go
import "gorm.io/plugin/soft_delete"

type User struct {
  ID        uint
  Name      string
  DeletedAt soft_delete.DeletedAt
}

// Запрос
SELECT * FROM users WHERE deleted_at = 0;

// Удаление
UPDATE users SET deleted_at = /* текущее время в unix секундах */ WHERE ID = 1;
```

{% note warn %}
**ИНФОРМАЦИЯ** при использовании уникального поля с мягким удалением, вы сначала должны создать составной индекс с помощью поля, использующего unix-время в секундах, на основе `DeletedAt`, например:

```go
import "gorm.io/plugin/soft_delete"

type User struct {
  ID        uint
  Name      string                `gorm:"uniqueIndex:udx_name"`
  DeletedAt soft_delete.DeletedAt `gorm:"uniqueIndex:udx_name"`
}
```
{% endnote %}

Использование `1` / `0` в качестве флага удаления

```go
import "gorm.io/plugin/soft_delete"

type User struct {
  ID    uint
  Name  string
  IsDel soft_delete.DeletedAt `gorm:"softDelete:flag"`
}

// Запрос
SELECT * FROM users WHERE is_del = 0;

// Удаление
UPDATE users SET is_del = 1 WHERE ID = 1;
```

# GORM/GEN

The code generator base on [GORM](https://github.com/go-gorm/gorm), aims to be developer friendly.

## Overview

- CRUD or DIY query method code generation
- Auto migration from database to code
- Transactions, Nested Transactions, Save Point, RollbackTo to Saved Point
- Competely compatible with GORM
- Developer Friendly

## Installation

To install Gen package, you need to install Go and set your Go workspace first.

1. The first need Go installed(version 1.14+ is required), then you can use the below Go command to install Gen.

```bash
go get -u gorm.io/gen
```

2. Import it in your code:

```go
import "gorm.io/gen"
```

## Quick start

```bash
# assume the following code in generate.go file
$ cat generate.go
```

```go
package main

import "gorm.io/gen"

// generate code
func main() {
    // specify the output directory (default: "./query")
    g := gen.NewGenerator(gen.Config{OutPath: "../dal/query"})
  
    // reuse the database connection in Project or create a connection here
    // db, _ := gorm.Open(mysql.Open("root:@(127.0.0.1:3306)/demo?charset=utf8mb4&parseTime=True&loc=Local"))
    g.UseDB(db)
  
    // apply basic crud api on structs or table models which is specified by table name with function
    // GenerateModel/GenerateModelAs. And generator will generate table models' code when calling Excute.
    g.ApplyBasic(model.User{}, g.GenerateModel("company"), g.GenerateModelAs("people", "Person"),)
    
    // apply diy interfaces on structs or table models
    g.ApplyInterface(func(method model.Method) {}, model.User{}, g.GenerateModel("company"))

    // execute the action of code generation
    g.Execute()
}
```

### Project Directory

Here is a template for best practices:

```bash
demo
├── cmd
│   └── generate
│       └── generate.go # execute it will generate codes
├── dal
│   ├── dal.go # create connections with database server here
│   └── model
│   │   ├── method.go # DIY method interfaces
│   │   └── model.go  # store struct which corresponding to the database table
│   └── query  # generated code's directory
│       └── gorm_generated.go # generated code
├── biz
│   └── query.go # call function in dal/gorm_generated.go and query databases
├── config
│   └── config.go # DSN for database server
├── generate.sh # a shell to execute cmd/generate
├── go.mod
├── go.sum
└── main.go
```

## API Examples

### Field Expression 

#### Create Field

Actually, you're not supposed to create a new field variable, cause it will be accomplished in generated code.

| Field Type | Detail Type           | Crerate Function               | Supported Query Method                                       |
| ---------- | --------------------- | ------------------------------ | ------------------------------------------------------------ |
| generic    | field                 | NewField                       | IsNull/IsNotNull/Count                                       |
| int        | int/int8/.../int64    | NewInt/NewInt8/.../NewInt64    | Eq/Neq/Gt/Gte/Lt/Lte/In/NotIn/Between/NotBetween/Like/NotLike/Add/Sub/Mul/Div/Mod/FloorDiv/RightShift/LeftShift/BitXor/BitAnd/BitOr/BitFlip |
| uint       | uint/uint8/.../uint64 | NewUint/NewUint8/.../NewUint64 | same with int                                                |
| float      | float32/float64       | NewFloat32/NewFloat64          | Eq/Neq/Gt/Gte/Lt/Lte/In/NotIn/Between/NotBetween/Like/NotLike/Add/Sub/Mul/Div/FloorDiv |
| string     | string/[]byte         | NewString/NewBytes             | Eq/Neq/Gt/Gte/Lt/Lte/Between/NotBetween/In(val/NotIn(val/Like/NotLike/Regexp/NotRegxp |
| bool       | bool                  | NewBool                        | Not/Is/And/Or/Xor/BitXor/BitAnd/BitOr                        |
| time       | time.Time             | NewTime                        | Eq/Neq/Gt/Gte/Lt/Lte/Between/NotBetween/In/NotIn/Add/Sub     |

Create field examples:

```go
import "gorm.io/gen/field"

// create a new generic field map to `generic_a`
a := field.NewField("table_name", "generic_a")

// create a field map to `id`
i := field.NewInt("user", "id")

// create a field map to `address`
s := field.NewString("user", "address")

// create a field map to `create_time`
t := field.NewTime("user", "create_time")
```

### CRUD API

Here is a basic struct `user` and struct `DB`. 

```go
// generated code
// generated code
// generated code
package query

import "gorm.io/gen"

// struct map to table `users` 
type user struct {
    gen.DO
    ID       field.Uint
    Name     field.String
    Age      field.Int
    Address  field.Field
    Birthday field.Time
}

// struct collection
type DB struct {
    db       *gorm.DB
    User     *user
}
```

#### Create

##### Create Record

```go
// u refer to query.user
user := model.User{Name: "Modi", Age: 18, Birthday: time.Now()}

u := query.Query.User
err := u.Create(&user) // pass pointer of data to Create

err // returns error
```

#####  Create Record With Selected Fields

Create a record and assgin a value to the fields specified.

```go
u := query.Query.User
u.Select(u.Name, u.Age).Create(&user)
// INSERT INTO `users` (`name`,`age`) VALUES ("modi", 18)
```

Create a record and ignore the values for fields passed to omit

```go
u := query.Query.User
u.Omit(u.Name, u.Age).Create(&user)
// INSERT INTO `users` (`Address`, `Birthday`) VALUES ("2021-08-17 20:54:12.000", 18)
```

##### Batch Insert

To efficiently insert large number of records, pass a slice to the `Create` method. GORM will generate a single SQL statement to insert all the data and backfill primary key values.

```go
var users = []model.User{{Name: "modi"}, {Name: "zhangqiang"}, {Name: "songyuan"}}
query.Query.User.Create(&users)

for _, user := range users {
    user.ID // 1,2,3
}
```

You can specify batch size when creating with `CreateInBatches`, e.g:

```go
var users = []User{{Name: "modi_1"}, ...., {Name: "modi_10000"}}

// batch size 100
query.Query.User.CreateInBatches(users, 100)
```

It will works if you set `CreateBatchSize` in `gorm.Config` / `gorm.Session`

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
    CreateBatchSize: 1000,
})
// OR
db = db.Session(&gorm.Session{CreateBatchSize: 1000})

u := query.NewUser(db)

var users = []User{{Name: "modi_1"}, ...., {Name: "modi_5000"}}

u.Create(&users)
// INSERT INTO users xxx (5 batches)
```

#### Query

##### Retriving a single object

Generated code provides `First`, `Take`, `Last` methods to retrieve a single object from the database, it adds `LIMIT 1` condition when querying the database, and it will return the error `ErrRecordNotFound` if no record is found.

```go
u := query.Query.User

// Get the first record ordered by primary key
user, err := u.First()
// SELECT * FROM users ORDER BY id LIMIT 1;

// Get one record, no specified order
user, err := u.Take()
// SELECT * FROM users LIMIT 1;

// Get last record, ordered by primary key desc
user, err := db.Last()
// SELECT * FROM users ORDER BY id DESC LIMIT 1;

// check error ErrRecordNotFound
errors.Is(err, gorm.ErrRecordNotFound)
```

##### Retrieving objects with primary key

```go
u := query.Query.User

user, err := u.First(u.ID.Eq(10))
// SELECT * FROM users WHERE id = 10;

users, err := u.Find(u.ID.In(1,2,3))
// SELECT * FROM users WHERE id IN (1,2,3);
```

If the primary key is a string (for example, like a uuid), the query will be written as follows:

```go
user, err := db.First(u.ID.Eq("1b74413f-f3b8-409f-ac47-e8c062e3472a"))
// SELECT * FROM users WHERE id = "1b74413f-f3b8-409f-ac47-e8c062e3472a";
```

##### Retrieving all objects

```go
u := query.Query.User

// Get all records
users, err := u.Find()
// SELECT * FROM users;
```

##### Conditions

###### String Conditions

```go
u := query.Query.User

// Get first matched record
user, err := u.Where(u.Name.Eq("modi")).First()
// SELECT * FROM users WHERE name = 'modi' ORDER BY id LIMIT 1;

// Get all matched records
users, err := u.Where(u.Name.Neq("modi")).Find()
// SELECT * FROM users WHERE name <> 'modi';

// IN
users, err := u.Where(u.Name.In("modi", "zhangqiang")).Find()
// SELECT * FROM users WHERE name IN ('modi','zhangqiang');

// LIKE
users, err := u.Where(u.Name.Like("%modi%")).Find()
// SELECT * FROM users WHERE name LIKE '%modi%';

// AND
users, err := u.Where(u.Name.Eq("modi"), u.Age.Gte(17)).Find()
// SELECT * FROM users WHERE name = 'modi' AND age >= 17;

// Time
users, err := u.Where(u.Birthday.Gt(birthTime).Find()
// SELECT * FROM users WHERE birthday > '2000-01-01 00:00:00';

// BETWEEN
users, err := u.Where(u.Birthday.Between(lastWeek, today)).Find()
// SELECT * FROM users WHERE birthday BETWEEN '2000-01-01 00:00:00' AND '2000-01-08 00:00:00';
```

###### Inline Condition

```go
u := query.Query.User

// Get by primary key if it were a non-integer type
user, err := u.First(u.ID.Eq("string_primary_key"))
// SELECT * FROM users WHERE id = 'string_primary_key';

// Plain SQL
users, err := u.Find(u.Name.Eq("modi"))
// SELECT * FROM users WHERE name = "modi";

users, err := u.Find(u.Name.Neq("modi"), u.Age.Gt(17))
// SELECT * FROM users WHERE name <> "modi" AND age > 17;
```

###### Not Conditions

Build NOT conditions, works similar to `Where`

```go
u := query.Query.User

user, err := u.Not(u.Name.Eq("modi")).First()
// SELECT * FROM users WHERE NOT name = "modi" ORDER BY id LIMIT 1;

// Not In
users, err := u.Not(u.Name.In("modi", "zhangqiang")).Find()
// SELECT * FROM users WHERE name NOT IN ("modi", "zhangqiang");

// Not In slice of primary keys
user, err := u.Not(u.ID.In(1,2,3)).First()
// SELECT * FROM users WHERE id NOT IN (1,2,3) ORDER BY id LIMIT 1;
```

###### Or Conditions

```go
u := query.Query.User

users, err := u.Where(u.Role.Eq("admin")).Or(u.Role.Eq("super_admin")).Find()
// SELECT * FROM users WHERE role = 'admin' OR role = 'super_admin';
```

###### Group Conditions

Easier to write complicated SQL query with Group Conditions

```go
p := query.Query.Pizza

pizzas, err := p.Where(
    p.Where(p.Pizza.Eq("pepperoni")).Where(p.Where(p.Size.Eq("small")).Or(p.Size.Eq("medium"))),
).Or(
    p.Where(p.Pizza.Eq("hawaiian")).Where(p.Size.Eq("xlarge")),
).Find()

// SELECT * FROM `pizzas` WHERE (pizza = "pepperoni" AND (size = "small" OR size = "medium")) OR (pizza = "hawaiian" AND size = "xlarge")
```

###### Selecting Specific Fields

`Select` allows you to specify the fields that you want to retrieve from database. Otherwise, GORM will select all fields by default.

```go
u := query.Query.User

users, err := u.Select(u.Name, u.Age).Find()
// SELECT name, age FROM users;

u.Select(u.Age.Avg()).Rows()
// SELECT Avg(age) FROM users;
```

###### Order

Specify order when retrieving records from the database

```go
u := query.Query.User

users, err := u.Order(u.Age.Desc(), u.Name).Find()
// SELECT * FROM users ORDER BY age DESC, name;

// Multiple orders
users, err := u.Order(u.Age.Desc()).Order(u.Name).Find()
// SELECT * FROM users ORDER BY age DESC, name;
```

###### Limit & Offset

`Limit` specify the max number of records to retrieve
`Offset` specify the number of records to skip before starting to return the records

```go
u := query.Query.User

urers, err := u.Limit(3).Find()
// SELECT * FROM users LIMIT 3;

// Cancel limit condition with -1
users, err := u.Limit(10).Limit(-1).Find()
// SELECT * FROM users;

users, err := u.Offset(3).Find()
// SELECT * FROM users OFFSET 3;

users, err := u.Limit(10).Offset(5).Find()
// SELECT * FROM users OFFSET 5 LIMIT 10;

// Cancel offset condition with -1
users, err := u.Offset(10).Offset(-1).Find()
// SELECT * FROM users;
```

###### Group By & Having

```go
u := query.Query.User

type Result struct {
    Date  time.Time
    Total int
}

var result Result

err := u.Select(u.Name, u.Age.Sum().As("total")).Where(u.Name.Like("%modi%")).Group(u.Name).Scan(&result)
// SELECT name, sum(age) as total FROM `users` WHERE name LIKE "group%" GROUP BY `name`

err := u.Select(u.Name, u.Age.Sum().As("total")).Group(u.Name).Having(u.Name.Eq("group")).Scan(&result)
// SELECT name, sum(age) as total FROM `users` GROUP BY `name` HAVING name = "group"

rows, err := u.Select(u.Birthday.As("date"), u.Age.Sum().As("total")).Group(u.Birthday).Rows()
for rows.Next() {
  ...
}

o := query.Query.Order

rows, err := o.Select(o.CreateAt.Date().As("date"), o.Amount.Sum().As("total")).Group(o.CreateAt.Date()).Having(u.Amount.Sum().Gt(100)).Rows()
for rows.Next() {
  ...
}

var results []Result

o.Select(o.CreateAt.Date().As("date"), o.Amount.Sum().As("total")).Group(o.CreateAt.Date()).Having(u.Amount.Sum().Gt(100)).Scan(&results)
```

###### Distinct

Selecting distinct values from the model

```go
u := query.Query.User

users, err := u.Distinct(u.Name, u.Age).Order(u.Name, u.Age.Desc()).Find()
```

`Distinct` works with `Pluck` and `Count` too

###### Joins

Specify Joins conditions

```go
u := query.Query.User
e := query.Query.Email
c := query.Query.CreditCard

type Result struct {
    Name  string
    Email string
}

var result Result

err := u.Select(u.Name, e.Email).LeftJoin(e, e.UserId.EqCol(u.ID)).Scan(&result)
// SELECT users.name, emails.email FROM `users` left join emails on emails.user_id = users.id

rows, err := u.Select(u.Name, e.Email).LeftJoin(e, e.UserId.EqCol(u.ID)).Rows()
for rows.Next() {
  ...
}

var results []Result

err := u.Select(u.Name, e.Email).LeftJoin(e, e.UserId.EqCol(u.ID)).Scan(&results)

// multiple joins with parameter
users := u.Join(e, e.UserId.EqCol(u.id), e.Email.Eq("modi@example.org")).Join(c, c.UserId.EqCol(u.ID)).Where(c.Number.Eq("411111111111")).Find()
```

##### SubQuery

A subquery can be nested within a query, GEN can generate subquery when using a `Dao` object as param

```go
o := query.Query.Order
u := query.Query.User

orders, err := o.Where(gen.Gt(o.Amount, o.Select(u.Amount.Avg())).Find()
// SELECT * FROM "orders" WHERE amount > (SELECT AVG(amount) FROM "orders");

subQuery := u.Select(u.Age.Avg()).Where(u.Name.Like("name%"))
users, err := u.Select(u.Age.Avg().As("avgage")).Group(u.Name).Having(gen.Gt(u.Age.Avg(), subQuery).Find()
// SELECT AVG(age) as avgage FROM `users` GROUP BY `name` HAVING AVG(age) > (SELECT AVG(age) FROM `users` WHERE name LIKE "name%")
```

###### From SubQuery

GORM allows you using subquery in FROM clause with method `Table`, for example:

```go
u := query.Query.User
p := query.Query.Pet

users, err := gen.Table(u.Select(u.Name, u.Age).As("u")).Where(u.Age.Eq(18)).Find()
// SELECT * FROM (SELECT `name`,`age` FROM `users`) as u WHERE `age` = 18

subQuery1 := u.Select(u.Name)
subQuery2 := p.Select(p.Name)
users, err := gen.Table(subQuery1.As("u"), subQuery2.As("p")).Find()
db.Table("(?) as u, (?) as p", subQuery1, subQuery2).Find(&User{})
// SELECT * FROM (SELECT `name` FROM `users`) as u, (SELECT `name` FROM `pets`) as p
```

###### Update from SubQuery

Update a table by using SubQuery

```go
u := query.Query.User
c := query.Query.Company

u.Update(u.CompanyName, c.Select(c.Name).Where(c.ID.EqCol(u.CompanyId)))
// UPDATE "users" SET "company_name" = (SELECT name FROM companies WHERE companies.id = users.company_id);

u.Where(u.Name.Eq("modi")).Update(u.CompanyName, c.Select(c.Name).Where(c.ID.EqCol(u.CompanyId)))
```

##### Advanced Query

###### Iteration

GEN supports iterating through Rows

```go
rows, err := query.Query.User.Where("name = ?", "modi").Rows()
defer rows.Close()

for rows.Next() {
    var user User
    // ScanRows is a method of `gorm.DB`, it can be used to scan a row into a struct
    db.ScanRows(rows, &user)

    // do something
}
```

###### FindInBatches

Query and process records in batch

```go
u := query.Query.User

// batch size 100
err := u.Where(u.ID.Gt(9)).FindInBatches(&results, 100, func(tx gen.Dao, batch int) error {
    for _, result := range results {
      // batch processing found records
    }
  
    // build a new `u` to use it's api
    // queryUsery := query.NewUser(tx.UnderlyingDB())

    tx.Save(&results)

    batch // Batch 1, 2, 3

    // returns error will stop future batches
    return nil
})
```

###### Pluck

Query single column from database and scan into a slice, if you want to query multiple columns, use `Select` with `Scan` instead

```go
u := query.Query.User

var ages []int64
u.Pluck(u.Age, &ages)

var names []string
u.Pluck(u.Name, &names)

// Distinct Pluck
u.Distinct().Pluck(u.Name, &names)
// SELECT DISTINCT `name` FROM `users`

// Requesting more than one column, use `Scan` or `Find` like this:
db.Select(u.Name, u.Age).Scan(&users)
users, err := db.Select(u.Name, u.Age).Find()
```

###### Scopes

`Scopes` allows you to specify commonly-used queries which can be referenced as method calls

```go
o := query.Query.Order

func AmountGreaterThan1000(tx gen.Dao) gen.Dao {
    return tx.Where(o.Amount.Gt(1000))
}

func PaidWithCreditCard(tx gen.Dao) gen.Dao {
    return tx.Where(o.PayModeSign.Eq("C"))
}

func PaidWithCod(tx gen.Dao) gen.Dao {
    return tx.Where(o.PayModeSign.Eq("C"))
}

func OrderStatus(status []string) func (tx gen.Dao) gen.Dao {
    return func (tx gen.Dao) gen.Dao {
      return tx.Where(o.Status.In(status...))
    }
}

orders, err := o.Scopes(AmountGreaterThan1000, PaidWithCreditCard).Find()
// Find all credit card orders and amount greater than 1000

orders, err := o.Scopes(AmountGreaterThan1000, PaidWithCod).Find()
// Find all COD orders and amount greater than 1000

orders, err := o.Scopes(AmountGreaterThan1000, OrderStatus([]string{"paid", "shipped"})).Find()
// Find all paid, shipped orders that amount greater than 1000
```

###### Count

Get matched records count

```go
u := query.Query.User

count, err := u.Where(u.Name.Eq("modi")).Or(u.Name.Eq("zhangqiang")).Count()
// SELECT count(1) FROM users WHERE name = 'modi' OR name = 'zhangqiang'

count, err := u.Where(u.Name.Eq("modi")).Count()
// SELECT count(1) FROM users WHERE name = 'modi'; (count)

// Count with Distinct
u.Distinct(u.Name).Count()
// SELECT COUNT(DISTINCT(`name`)) FROM `users`
```

#### Update

##### Update single column

When updating a single column with `Update`, it needs to have any conditions or it will raise error `ErrMissingWhereClause`, for example:

```go
u := query.Query.User

// Update with conditions
u.Where(u.Activate.Is(true)).Update(u.Name, "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE active=true;

// Update with conditions
u.Where(u.Activate.Is(true)).Update(u.Age, u.Age.Add(1))
// UPDATE users SET age=age+1, updated_at='2013-11-17 21:34:10' WHERE active=true;
```

##### Updates multiple columns

`Updates` supports update with `struct` or `map[string]interface{}`, when updating with `struct` it will only update non-zero fields by default

```go
u := query.Query.User

// Update attributes with `struct`, will only update non-zero fields
u.Updates(User{Name: "hello", Age: 18, Active: false})
// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;

// Update attributes with `map`
u).Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello', age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
```

> **NOTE** When update with struct, GEN will only update non-zero fields, you might want to use `map` to update attributes or use `Select` to specify fields to update

##### Update Selected Fields

If you want to update selected fields or ignore some fields when updating, you can use `Select`, `Omit`

```go
// Select with Map
// User's ID is `111`:
db.Model(&user).Select("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello' WHERE id=111;

db.Model(&user).Omit("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;

// Select with Struct (select zero value fields)
db.Model(&user).Select("Name", "Age").Updates(User{Name: "new_name", Age: 0})
// UPDATE users SET name='new_name', age=0 WHERE id=111;

// Select all fields (select all fields include zero value fields)
db.Model(&user).Select("*").Update(User{Name: "jinzhu", Role: "admin", Age: 0})

// Select all fields but omit Role (select all fields include zero value fields)
db.Model(&user).Select("*").Omit("Role").Update(User{Name: "jinzhu", Role: "admin", Age: 0})
```

#### Delete

##### Delete a Record

```go
e := query.Query.Email

// Email's ID is `10`
e.Delete(e.ID.Eq(10))
// DELETE from emails where id = 10;

// Delete with additional conditions
e.Where(e.Name.Eq("modi")).Delete(e.ID.Eq(10))
// DELETE from emails where id = 10 AND name = "modi";
```

##### Delete with primary key

GEN allows to delete objects using primary key(s) with inline condition, it works with numbers.

```go
u.Delete(u.ID.In(1,2,3))
// DELETE FROM users WHERE id IN (1,2,3);
```

##### Batch Delete

The specified value has no primary value, GEN will perform a batch delete, it will delete all matched records

```go
e := query.Query.Email

err := e.Where(e.Name.Like("%modi%")).Delete()
// DELETE from emails where email LIKE "%modi%";

err := e.Delete(e.Name.Like("%modi%"))
// DELETE from emails where email LIKE "%jinzhu%";
```

##### Soft Delete

If your model includes a `gorm.DeletedAt` field (which is included in `gorm.Model`), it will get soft delete ability automatically!

When calling `Delete`, the record WON’T be removed from the database, but GORM will set the `DeletedAt`‘s value to the current time, and the data is not findable with normal Query methods anymore.

```go
// Batch Delete
u.Where(u.Age.Eq(20)).Delete()
// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE age = 20;

// Soft deleted records will be ignored when querying
u.Where(u.Age.Eq(20)).Delete()
// SELECT * FROM users WHERE age = 20 AND deleted_at IS NULL;
```

If you don’t want to include `gorm.Model`, you can enable the soft delete feature like:

```go
type User struct {
    ID      int
    Deleted gorm.DeletedAt
    Name    string
}
```

##### Find soft deleted records

You can find soft deleted records with `Unscoped`

```go
users, err := db.Unscoped().Where(u.Age.Eq(20)).Find()
// SELECT * FROM users WHERE age = 20;
```

##### Delete permanently

You can delete matched records permanently with `Unscoped`

```go
o.Unscoped().Delete(o.ID.Eq(10))
// DELETE FROM orders WHERE id=10;
```

### DIY method

#### Method interface

Method interface is a abstraction of query methods, all functions it contains are query methods and above comments describe the specific query conditions or logic.

```go
type Model interface {
    // Where("name=@name and age=@age")
    FindByNameAndAge(name string, age int) (gen.T, error)
    
    // sql(select * from user where name=@name)
    FindByName(name string) ([]gen.T, error)
   
    // select * from user where name=@id
    FindByAge(age int) ([]gen.T, error)
  
    // select * from user limit 1
    FindOne() (gen.M, error)
  
    // select * from user
    FindAll() ([]gen.M, error)
    
    // select * from @@table 
    //   {{where}}
    //        {{if cond}}id=@id {{end}}
    //        {{if name != ""}}@@key=@value{{end}}
    //    {{end}}
    FindByIDOrKey(cond bool, id int, key, value string) (gen.T, error)
}
```

Return values must contains less than 1 `gen.T`/`gen.M` and less than 1 error.

##### Syntax of template

###### placeholder

- `gen.T` represents specified `struct` or `table`
- `gen.M` represents `map[string]interface`
- `@@table` represents table's name
- `@name` represents variable

###### Where

```sql
{{where}}{{end}}
```

###### Raw

```go
// select * from user
methond() error
```

###### Raw template

```sql
select * from @@table 
{{where}}
    {{if cond}}id=@id {{end}}
    {{if name != ""}}@@key=@value{{end}}
{{end}}
```

#### Smart Select Fields

GEN allows select specific fields with `Select`, if you often use this in your application, maybe you want to define a smaller struct for API usage which can select specific fields automatically, for example:

```go
type User struct {
  ID     uint
  Name   string
  Age    int
  Gender string
  // hundreds of fields
}

type APIUser struct {
  ID   uint
  Name string
}

type Method interface{
    // select * from user
    FindSome() ([]APIUser, error)
}

apiusers, err := u.Limit(10).FindSome()
// SELECT `id`, `name` FROM `users` LIMIT 10
```

### Advanced Topics

#### Hints

Optimizer hints allow to control the query optimizer to choose a certain query execution plan, GORM supports it with `gorm.io/hints`, e.g:

```go
import "gorm.io/hints"

u := query.Query.User

users, err := u.Hints(hints.New("MAX_EXECUTION_TIME(10000)")).Find()
// SELECT * /*+ MAX_EXECUTION_TIME(10000) */ FROM `users`
```

Index hints allow passing index hints to the database in case the query planner gets confused.

```go
import "gorm.io/hints"

u := query.Query.User

users, err := u.Hints(hints.UseIndex("idx_user_name")).Find()
// SELECT * FROM `users` USE INDEX (`idx_user_name`)

users, err := u.Hints(hints.ForceIndex("idx_user_name", "idx_user_id").ForJoin()).Find()
// SELECT * FROM `users` FORCE INDEX FOR JOIN (`idx_user_name`,`idx_user_id`)"
```



## Contributing

You can help to deliver a better GORM/GEN



## License

Released under the [MIT License](https://github.com/go-gorm/gen/blob/master/License)


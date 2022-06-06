# 踩坑日记 GORM Updates
在Go语言里，一个比较知名的ORM框架就是GORM。GORM可以帮助我们处理一些简单的CURD，我们可以通过Updates方法来做到一些隐式的更新操作。比如对于一张Book表：
```sql
CREATE TABLE `book`
(
    id   int(10) primary key auto_increment,
    name varchar(190) not null default '',
    category varchar(190) not null default ''
) engine = innodb;

```
当我们需要更新表数据时，通常我们需要写大量的函数，比如：
```sql
update book set name = '计算机组成原理' where id = 1;
update book set category = '计算机科学' where id = 1;
```
对应的我们需要大量的更新函数如
```golang
func (d *db) UpdateBookName(name string) error
func (d *db) UpdateBookCategory(cate string) error
```
使用Updates可以避免这个问题：
```golang
type Book struct {
  Id        int
  Name      string
  Category  string
}

d.Updates(&Book{
  Name: "计算机组成原理",
  Category: "计算机科学",
  })
```
但是当这种写法对于我们需要将值更新为零值时不能很好地支持，虽然可以使用map来避免，但是会比较麻烦。当然也可以使用指针来处理，但是对于指针类型来说，虽然很好的解决了更新的默认值问题，但是在
查询时需要特别注意每次的使用，在没查询到数据时容易产生空指针引用，所以这里我们可以使用GORM的hook来规避这个问题，具体的，在初始化数据库连接时使用如下代码注册hook函数：
```golang
// 注册hook函数（只针对Query生效）
if err = db.Callback().Query().After("*").Register("handleNilPointerElems", handleNilPointerElems); err != nil {
			panic(err)
}

// 查询处理 
// handleNilPointerElems solve the problem when we find nothing but use the *pointer elem, it prevent the panic due to it.
func handleNilPointerElems(db *gorm.DB) {
  // 只在没查询到数据时处理
	if db.RowsAffected == 0 { 
		reflectValue := reflect.ValueOf(db.Statement.Dest)
		for reflectValue.Kind() == reflect.Ptr {
			reflectValue = reflectValue.Elem()
		}
		switch reflectValue.Kind() {
		case reflect.Struct:
			for i := 0; i < reflectValue.NumField(); i++ {
				elemi := reflectValue.Field(i)
				if elemi.Kind() == reflect.Ptr && elemi.CanSet() {
					switch elemi.Type().String() {
          // 枚举了基础的两种类型，如果为空，默认会生成零值的引用
					case "*int":
						elemi.Set(reflect.New(reflect.TypeOf(0)))
					case "*string":
						elemi.Set(reflect.New(reflect.TypeOf("")))
					}
				}
			}
		}
	}
}
```

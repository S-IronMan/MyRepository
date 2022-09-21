`Oracle`至`PostgreSQL`的数据库数据的迁移由组内的其他人完成，因此在此只记录代码层面的兼容操作。

# 1、`orafce`安装

`orafce`是`PostgreSQL`的一个extension，其对`Oracle`内的大部分函数和特性都进行了兼容，安装`orafce`可以省去迁移过程中的许多操作。

+ `orafce`的`github`地址：https://github.com/orafce/orafce

- `orafce`的官方文档：https://github.com/orafce/orafce/tree/master/doc/orafce_documentation

本次迁移目标的`PostgreSQL`是部署在Linux操作系统上的，因此需下载对应的rpm包。

- `PostgreSQL`官方yum仓库地址：https://yum.postgresql.org

根据安装的`PostgreSQL`版本、Linux操作系统版本在该仓库中寻找对应的`orafce`包。

解压rpm包：

```
rpm -hvi orafce_13-3.15.0-1.rhel7.aarch64.rpm
```

将解压得到的文件复制到`PostgreSQL`对应的目录下：

```
cd /data/postgresql-13.3/extension
cp -a /usr/pgsql-13/share/extension/orafce* .
cd /usr/local/postgresql-13.3/lib
cp -a /usr/pgsql-13/lib/orafce.so .
cp -a /usr/pgsql-13/lib/bitcode .
```

给so文件设置运行权限：

```
chmod +x /usr/local/postgresql-13.3/lib/orafce.so
```

连接`PostgreSQL`数据库：

```
psql -h [数据库IP] -p [端口号] -U [用户名] -d [数据库名]
```

输入指令给数据库创建extension：

```
create extension orafce;
```

在search_path内添加oracle模式名（数据库的连接`ur`l里不能加 `currentSchema=sde` 的配置，这会让search_path的设置不生效，如需指定项目所用的默认模式，使用用户名为该模式名的用户，或在search_path内添加该模式）：

```sql
方式1：数据库配置文件postgresql.conf内修改
search_path = '"$user", public, oracle, pg_catalog'
方式2：用户级别的设置
ALTER USER user1 SET search_path = "$user",public,oracle,pg_catalog;
方式3：数据库级别的设置
ALTER DATABASE db1 SET search_path = "$user",public,oracle,pg_catalog;
```

完成安装。

# 2、新增自动类型转换

`Oracle`自带所有数据类型的转换，不论是在赋值时还是在表达式中。而`PostgreSQL`虽内置了部分数据类型间的自动转换，但还是有数据类型间无法自动转换，产生数据类型冲突时（比如新增或编辑时不同数据类型间的赋值，或是多表关联时以两个数据类型不同的列作为连接条件）需手动进行类型转换。

手动进行类型转换有三种方式（本次迁移一般使用第二种）：

```sql
type colName   例如：integer '123'、varchar user_name
```

```sql
colName::type   例如：'123'::integer、user_name::varchar
```

```
cast(colName as type)   例如：cast('123' as integer)、cast(user_name as integer)
```

可以执行下列sql语句查看数据库中已有的类型转换（例如查看所有源类型为varchar的转换）：

```sql
select castsource::regtype, casttarget::regtype, castcontext, castfunc from pg_cast where castsource='varchar'::regtype;
```

除了`PostgreSQL`内置的类型转换，其余的转换因存在精度缺失或是报错的风险而没有内置。`integer`和`varchar`间的类型转换就没有内置，但本次迁移过程中发现了大量多表关联时用`integer`类型的列和`varchar`类型的列作为连接条件的情况，综合考虑下来还是决定加上这两个类型间的自动转换（`integer`类型转到`varchar`类型不存在精度缺失和报错的情况；而`varchar`类型转到`integer`类型如果会发生精度缺失或报错，那么原先连`Oracle`数据库时便会发现并解决，因此认定不存在精度缺失或报错的情况）。

新增自动类型转换的sql如下：

```sql
create cast (integer as varchar) with inout AS IMPLICIT;
create cast (varchar as integer) with inout AS IMPLICIT;
```

- 新增类型转换的官方文档：https://www.postgresql.org/docs/10/sql-createcast.html
- 删除类型转换的官方文档：https://www.postgresql.org/docs/10/sql-dropcast.html

# 3、自定义兼容函数

### 1、自定义`decode`函数的重载

`Oracle`内置的`decode`函数可以有任意数量的入参，`orafce`虽然兼容了`decode`函数，但也只兼容到了最多3种判断条件+1个默认值，即：

```sql
decode(colName, 条件1, value1, 条件2, value2， 条件3, value3, 默认值)
```

因此对于判断条件多于3种的情况，只能用`case-when`语句进行转换：

```sql
case colName when 条件1 then value1 when 条件2 then value2 when 条件3 then value3 when 条件4 then value4 else 默认值 end
```

但在迁移过程中发现不少判断条件和对应value均为字符串类型的情况，因此决定自定义一个入参全为字符串类型且数量不限的`deocde`函数重载：

```sql
create or replace function decode(variadic p_decode_list text[])
returns text
 as
$$
declare
 -- 获取数组长度(即入参个数)
 v_len integer := array_length(p_decode_list, 1);
begin
 /*
 * 功能说明:模拟Oracle中的DECODE功能(字符串处理, 其它格式可以自行转换返回值)
 * 参数说明:格式同Oracle相同,至少三个参数
 * 实现原理: 1、VARIADIC 允许变参; 2、Oracle中的DECODE是拿第一个数依次和之后的偶数位值进行比较,相同则取偶数位+1的数值,否则取最后一位值(最后一位为偶数位,否则为null)
 */
 
    -- 同Oracle相同当参数不足三个抛出异常
    if v_len >= 3 then
        -- Oracle中的DECODE是拿第一个数依次和之后的偶数位值进行比较,相同则取偶数位+1的数值
        for i in 2..(v_len - 1) loop
            if mod(i, 2) = 0 then
                -- 同Oracle可以匹配null值
                if p_decode_list[1] is null and p_decode_list[i] is null then
                    return p_decode_list[i + 1];
                -- 若两值相等，则返回后一个值
                elsif p_decode_list[1] = p_decode_list[i] then
                    return p_decode_list[i + 1];
                -- 若两值不等，但该值是最后一个匹配项且传参有默认值，则返回默认值
                -- 同Oracle兼容第一个值为null的情况
                elsif v_len = i + 2 and v_len > 3 then
                    return p_decode_list[v_len];
                end if;
            end if;
        end loop;
    else
        raise exception 'UPG-00938: not enough args for function.';
    end if;
    -- 同Oracle没有匹配项则返回null
    return null;
end;
$$
 language plpgsql;
```

此时对于判断条件和对应value均为字符串类型的情况，最多只需手动转换一下入参类型，而不必费心转换为`case-when`语句了。

### 2、自定义`sys_guid`函数

`sys_guid`函数为`Oracle`内置的uuid生成函数，`orafce`似乎没有对其进行兼容，因此自定义一个：

```sql
create or replace function sys_guid()
returns text
  as
$$
  -- gen_random_uuid()函数为PostgreSQL自带的uuid生成函数，为兼容原sys_guid()函数，需去掉‘-’分隔符并全部转为大写
  select upper(replace(gen_random_uuid()::text, '-', ''));
$$
  language sql strict;
```

### 3、自定义`to_char`函数的重载

`to_char`函数本为`PostgreSQL`自带的函数，但其不存在入参为`varchar`类型的重载，因此自定义一个：

```sql
create or replace function to_char(val character varying)
returns character varying
  as
$$
  begin
    -- 不作处理原样返回入参
    return val;
  end
$$
  language plpgsql;
```

# 4、其余的兼容操作

### 1、函数的入参

对于入参类型不匹配的函数，需手动转换入参的类型。

例如`orafce`兼容后的`nvl`函数，其默认模式下没有`(numeric, integer)`类型的入参：

```sql
nvl(longitude, 0)  -- longitude列为numeric类型
```

$\Longrightarrow$

```sql
nvl(longitude, 0::numeric)
```

（在`oracle`模式中，`orafce`定义了`(numeric, integer)`入参类型的`nvl`函数，因此上式也能改成下面所示：）

```sql
oracle.nvl(longitude, 0)
```

### 2、系统时间

```sql
sysdate
```

$\Longrightarrow$

```sql
current_timestamp
```

### 3、字符串转时间

```sql
to_date('2021-09-01 15:40:20', 'yyyy-mm-dd hh24:mi:ss')
```

$\Longrightarrow$

```sql
to_timestamp('2021-09-01 15:40:20', 'yyyy-mm-dd hh24:mi:ss')
或
oracle.to_date('2021-09-01 15:40:20', 'yyyy-mm-dd hh24:mi:ss')  -- 此为orafce兼容的函数
```

`PostgreSQL`虽然也有`to_date`函数，但其转换结果不带时分秒。

### 4、时间加减

时间 -1秒：

```sql
sysdate - (1/6/1440)
```

$\Longrightarrow$

```sql
current_timestamp + '-1 sec'
```

---

时间 +1天：

```sql
sysdate + 1
```

$\Longrightarrow$

```sql
current_timestamp + '1 day'
```

---

时间 +1个月：

```sql
add_months(sysdate, 1)
```

$\Longrightarrow$

```sql
current_timestamp + '1 month'
或
oracle.add_months(current_timestamp, 1)  -- 此为orafce兼容的函数
```

---

时间动态 +n个月：

```sql
add_months(sysdate, n)
```

$\Longrightarrow$

```sql
current_timestamp + (n || ' month')::interval
或
oracle.add_months(current_timestamp, n)  -- 此为orafce兼容的函数
```

---

时间相减：

```sql
begin_time - end_time  -- 返回相差的天数，带小数
```

$\Longrightarrow$

```sql
extract(epoch from begin_time - end_time)  -- 返回相差的秒数，带小数
```

---

获取相差的月份：

```sql
months_between(begin_time, end_time)  -- 结果带小数
```

$\Longrightarrow$

```sql
oracle.months_between(begin_time, end_time)  -- 此为orafce兼容的函数，结果带小数
```

---

附上`PostgreSQL`内全部种类时间的相加规则：

```sql
select current_timestamp + '1 year'  -- 当前时间+1年
select current_timestamp + '1 month'  -- 当前时间+1个月
select current_timestamp + '1 day'  -- 当前时间+1天
select current_timestamp + '1 hour'  -- 当前时间+1小时
select current_timestamp + '1 min'  -- 当前时间+1分钟
select current_timestamp + '1 sec'  -- 当前时间+1秒
select current_timestamp + '1 year 1 month 1 day 1 hour 1 min 1 sec'  -- 当前时间+1年1月1天1时1分1秒
```

### 5、分页查询

```sql
select * from (select t.*, rownum rownum_ from emp t) 
where rownum_ > 0 and rownum_ <= 5
```

$\Longrightarrow$

```sql
select * from emp limit (5 - 0) offset 0
```

---

查询中使用序号列：

```sql
select rownum rn, t.* from emp t
```

$\Longrightarrow$

```sql
select row_number() over(), t.* from emp t
```

### 6、递归查询

```sql
select org_id, level from org_table
where [过滤条件]
start with [过滤条件2]
connect by prior org_id = up_org_id
```

$\Longrightarrow$

```sql
with recursive rs as (
    select org1.org_id, 1 as level from org_table org1
    where [过滤条件2]
    union all
    select org2.org_id, rs1.level + 1 from org_table org2
    inner join rs rs1 on rs1.org_id = org2.up_org_id
)
select org_id, level from rs where [过滤条件]
```

### 7、`decode`函数

除了上面提到的情况外，`orafce`兼容的`decode`函数也无法识别`null`值，因此需用`case-when`语句代替：

```sql
decode(colName, null, value1, 条件2, value2, 条件3, value3, 默认值)
```

$\Longrightarrow$

```sql
case when colName is null then value1 else decode(colName, 条件2, value2, 条件3, value3, 默认值) end
```

注意，`PostgreSQL`的简单`case-when`语句也无法识别`null`值，下面的语句是无效的：

```sql
case colName when null then value1 else decode(colName, 条件2, value2, 条件3, value3, 默认值) end  -- 无法识别null值
```

### 8、聚合函数

```sql
listagg(colName1, ',') within group (order by colName2)
```

$\Longrightarrow$

```sql
string_agg(colName1, ',' order by colName2)
```

---

```sql
wm_concat(colName)
```

$\Longrightarrow$

```sql
string_agg(colName, ',')
```

### 9、使用序列

```sql
seqName.nextval
```

$\Longrightarrow$

```sql
nextval('seqName')
```

### 10、字符串操作函数

截取：

```sql
substr()
```

$\Longrightarrow$

```sql
oracle.substr()  -- 此为orafce兼容的函数
```

---

查找：

```sql
instr()
```

$\Longrightarrow$

```sql
instr()  -- 此为orafce兼容的函数
```

### 11、外连接符号

左外连接（左连接）：

```sql
select * from table_A a, table_B b where a.id = b.id(+)
```

$\Longrightarrow$

```sql
select * from table_A a left join table_B b on a.id = b.id
```

---

右外连接（右连接）：

```sql
select * from table_A a, table_B b where a.id(+) = b.id
```

$\Longrightarrow$

```sql
select * from table_A a right join table_B b on a.id = b.id
```

### 12、dual表

```sql
select 1 from dual
```

$\Longrightarrow$

```sql
select 1
或
select 1 from dual  -- 此dual为orafce创建的名为dual的视图
```

### 13、生成n个连续数字

```sql
select rownum from dual connect by rownum <= n
```

$\Longrightarrow$

```sql
select generate_series(1, n)
```

---

在函数中使用生成的连续数字：

```sql
select to_char(rownum + 1) from dual connect by rownum <= n
```

$\Longrightarrow$

```sql
select to_char(rn + 1) from generate_series(1, n) rn
```

### 14、`insert`和`update`注意点

`Oracle`内是不区分`null`值和`''`的，即对一个字段用`''`赋值，会以`null`值进行存储。但`PostgreSQL`内对此是进行区分的，即对一个字段用`''`赋值，就以`''`进行存储，用`null`值赋值，就以`null`值进行存储。而且如果对非字符串类型的字段（如`integer`类型的字段）用`''`进行赋值，`PostgreSQL`还会报语法错误。而对某些字符串类型的字段用`''`赋值，本该以`null`值存储的地方却用`''`进行了存储，这可能会在其他代码里产生难以预见的bug（例如前端调用新增接口时对存放图片url的字段用`''`赋值，`PostgreSQL`对该字段便以`''`进行存储，而后续取出这条数据时，由于前端只对`null`值而未对`''`进行判断处理，导致页面上出现了“无中生有”的加载失败的图片）。因此在调用sql前需对所有的`''`进行判断处理，本次迁移的项目采用了`Mybatis`框架，以此为例：

`insert`时：

```xml
<if test="XXX != null">...</if>
```

$\Longrightarrow$

```xml
<if test="XXX != null and XXX != ''">...</if>
```

`update`时：

```xml
<if test="XXX != null">...</if>
```

$\Longrightarrow$

```xml
<if test="XXX != null">
    <if test="XXX == ''">colName = null</if>
    <if test="XXX != ''">...</if>
</if>
```

或者定义`java`方法，在调用sql前对传入的参数对象或参数`Map`进行处理，将其中的`''`全部转为`null`值。

---

同理，在`update`的sql中，将所有用`''`赋值的地方改用`null`值赋值：

```sql
update tableName set colName = '' where [判断条件]
```

$\Longrightarrow$

```sql
update tableName set colName = null where [判断条件]
```

---

`PostgreSQL`内的`update`语句中，`set`的字段不能有别名：

```sql
update tableName t set t.colName = 'XXX' where [判断条件]
```

$\Longrightarrow$

```sql
update tableName set colName = 'XXX' where [判断条件]
```

---

将`''`转为`null`的`java`方法：

```java
/**
     * 将对象内值为空字符的String类型属性赋值为null
     * 用于未对空字符串作判断的sql新增和修改
     * oracle内对字段用空字符串赋值会以null值存储，而postgresql内对空字符串和null值是区别存储
     * 因此需将对象内值为空字符串的属性赋值为null以作兼容
     * 否则前端有些仅对null进行判断的地方会因忽略了空字符串而出现意想不到的bug
     * （比如图片url，本应为null值进行一些操作，却因空字符串显示为无法加载的图片）
     *
     * @param target 待转换的对象
     * @param <T> 待转换对象的类型
     * @return 转换好的对象
     */
    public static <T> T convertEmptyStringToNull(T target) {
        Class<?> clazz = target.getClass();
        Field[] fields = clazz.getDeclaredFields();
        final String stringType = "class java.lang.String";
        final String nullValue = null;
        for (Field field : fields) {
            // 判断该字段是否是String类型
            if (stringType.equals(field.getGenericType().toString())) {
                String fieldName = field.getName();
                // 将字段名首字母改为大写
                fieldName = fieldName.substring(0, 1).toUpperCase() + fieldName.substring(1);
                try {
                    // 该字段的get方法
                    Method getMethod = clazz.getMethod("get" + fieldName);
                    String value = (String) getMethod.invoke(target);
                    // 判断该字段的值是否为空字符串
                    if ("".equals(value)) {
                        // 该字段的set方法
                        Method setMethod = clazz.getMethod("set" + fieldName, String.class);
                        // 设置字段的值为null
                        setMethod.invoke(target, nullValue);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        return target;
    }

/**
     * convertEmptyStringToNull方法对map的实现，功能与上述方法相同
     *
     * @param target 待转换的map
     * @return 转换好的map
     */
    public static Map<String, Object> convertEmptyStringToNull(Map<String, Object> target) {
        if (target == null || target.isEmpty()) {
            return target;
        }
        for (String key : target.keySet()) {
            Object value = target.get(key);
            // 判断该值是否为空字符串
            if ("".equals(value)) {
                // 将空字符串改为null值
                target.put(key, null);
            }
        }
        return target;
    }

/**
     * 批量将空字符串转为null值
     *
     * @param target 待转换的list
     * @return 转换好的list
     */
    public static <T> List<T> convertEmptyStringToNullByGroup(List<T> target) {
        if (target == null || target.isEmpty()) {
            return target;
        }
        for (T data : target) {
            if (data instanceof Map) {
                convertEmptyStringToNull((Map<String, Object>) data);
            } else {
                convertEmptyStringToNull(data);
            }
        }
        return target;
    }
```

### 15、`select`注意点

`Oracle`内查询结果的列名默认是大写的，而`PostgreSQL`内查询结果的列名默认是小写的。在`Mybatis`中，如果查询的返回结果类型是对象或者自定义的结果集映射，那么该查询结果的字段名是恒定不变的；而如果返回结果类型是`Map`，那么其`key`值就会受到数据库种类的影响，即连接`Oracle`查询时`key`值全部大写，连接`PostgreSQL`查询时`key`值全部小写。因此需在sql内对查询的每一个列附加一个用`""`括起的大写别名（如果sql内原本就有`""`括起的别名则无需处理）：

```sql
select id, name from tableName
```

$\Longrightarrow$

```sql
select id "ID", name "NAME" from tableName
```

---

`Oracle`内子查询可以省略别名，而`PostgreSQL`内子查询必须要有别名：

```sql
select * from (select ...)
```

$\Longrightarrow$

```sql
select * from (select ...) result
```

---

`PostgreSQL`内部分特殊词作为列的别名时会无效报错，需用`""`将其括起：

```sql
select colName name from tableName
```

$\Longrightarrow$

```sql
select colName "name" from tableName
```

或者用`as`关键字修饰别名：

```sql
select colName as name from tableName
```

---

将`Map`的`key`值全部转为大写的`java`方法：

```java
/**
     * 将map内所有的key转为大写
     * 用于兼容返回类型为map且查询结果没加带引号的别名的sql查询
     * oracle内查询结果的列名默认大写，postgresql内查询结果的列名默认小写
     * 因此需将结果集map的key转为大写以作兼容
     * 否则前端会出现字段取不到的情况
     *
     * @param target 待转换的map
     * @return 转换好的map
     */
    public static Map<String, Object> convertKeyToUppercase(Map<String, Object> target) {
        if (target == null || target.isEmpty()) {
            return target;
        }
        Map<String, Object> result = new HashMap<>(target.size());
        for (String key : target.keySet()) {
            // 将key转为大写
            String newKey = key.toUpperCase();
            result.put(newKey, target.get(key));
        }
        return result;
    }

/**
     * 批量将key转换为大写
     *
     * @param target 待转换的list
     * @return 转换好的list
     */
    public static List convertKeyToUppercaseByGroup(List target) {
        if (target == null || target.isEmpty()) {
            return target;
        }
        List result = new ArrayList<>(target.size());
        for (Object map : target) {
            result.add(convertKeyToUppercase((Map<String, Object>) map));
        }
        return result;
    }
```

### 16、`delete`注意点

`Oracle`内`delete`语句可以省略`from`，而`PostgreSQL`内`delete`语句不能省略`from`：

```sql
delete tableName where [判断条件]
```

$\Longrightarrow$

```sql
delete from tableName where [判断条件]
```

### 17、空格

`Oracle`内`!=`、`<>`、`>=`等操作符中间可以有空格，而`PostgreSQL`内不能：

```sql
select * from tableName where colName > = 1
```

$\Longrightarrow$

```sql
select * from tableName where colName >= 1
```

---

`PostgreSQL`内操作符与负数的负号间必须用空格隔开：

```sql
select * from tableName where colName !=-1
```

$\Longrightarrow$

```sql
select * from tableName where colName != -1
```

### 18、`distinct`/`unique`

`Oracle`支持用`unique`关键字去重，而`PostgreSQL`只能用`distinct`关键字去重：

```sql
select unique colName from tableName
```

$\Longrightarrow$

```sql
select distinct colName from tableName
```

### 19、日期截取

```sql
trunc(sysdate)
```

$\Longrightarrow$

```sql
trunc(current_timestamp)  -- 此为orafce兼容的函数
```

### 20、正则表达式

```sql
select regexp_substr('17,20,23', '[^,]+', 1, 1, 'i') from dual
```

$\Longrightarrow$

```sql
select (regexp_matches('17,20,23', '[^,]+'))[1]
```

### 21、`merge into`

```sql
merge into targetTable A
using sourceTable B  -- 或 using (子查询sql) B
on [判断条件]
when matched then update [更新语句sql]
when not matched then insert [新增语句sql]
```

$\Longrightarrow$

```sql
with upsert as (
    update targetTable A
    set colName1 = B.colName1,
    	colName2 = B.colName2  -- 设上面的[更新语句sql]为 set A.colName1 = B.colName1, A.colName2 = B.colName2
    from sourceTable B
    where A.ID = B.ID  -- 设上面的[判断条件]为 A.ID = B.ID
    returning A.*
)  -- 设上面的[新增语句sql]为 (colName1, colName2) values (B.colName1, B.colName2)
insert into targetTable (colName1, colName2)
select B1.colName1, B1.colName2
from sourceTable B1
where not exists (
    select 1 from upsert u
    where B1.ID = u.ID
)
```

`sourceTable`为子查询sql时：

```sql
with temp as (
    [子查询sql]
),
upsert as (
    update targetTable A
    set colName1 = B.colName1,
    	colName2 = B.colName2
    from temp B
    where A.ID = B.ID
    returning A.*
)
insert into targetTable (colName1, colName2)
select B1.colName1, B1.colName2
from temp B1
where not exists (
    select 1 from upsert u
    where u.ID = B1.ID
)
```

### 22、`arcgis` 相关函数

见官方文档：https://desktop.arcgis.com/zh-cn/arcmap/10.7/manage-data/using-sql-with-gdbs/st-geometry.htm

### 23、字符串连接符`||`

在`Oracle`中，`null`与其他字符串用符号`||`连接时会被忽略：

```sql
select 'abc' || null || '123' from dual  -- 输出'abc123'
```

但在`PostgreSQL`中，`null`与其他字符串用符号`||`连接时结果为`null`：

```sql
select 'abc' || null || '123'  -- 输出null
```

因此，对于多个列用符号`||`连接的情况：

```sql
colName1 || colName2 || colName3
```

$\Longrightarrow$

```sql
nvl(colName1::varchar, '') || nvl(colName2::varchar, '') || nvl(colName3::varchar, '')
-- nvl函数为orafce兼容的函数，亦可用 (case when colName is null then '' else colName end) 代替
```

对于`Mybatis`传参的情况：

```sql
colName like '%' || #{param} || '%'
```

$\Longrightarrow$

```sql
<if test="param != null">
	colName like '%' || #{param} || '%'
</if>
```

### 24、JSON操作

从JSON字符串中取某一字段的值：

```sql
select json_value('{"test1": "111", "test2": "222"}', '$.test1') from dual
```

$\Longrightarrow$

```sql
-- 第一个参数为json类型，返回text类型
select json_extract_path_text('{"test1": "111", "test2": "222"}', 'test1')
```

从JSON字符串中取嵌套对象的值：

```sql
-- 嵌套格式为“'$.path1.path2.path3....'”
select json_value('{"test1": {"test3": "333"}, "test2": "222"}', '$.test1.test3') from dual
```

$\Longrightarrow$

```sql
-- 嵌套格式为“'path1', 'path2', 'path3',...”
select json_extract_path_text('{"test1": {"test3": "333"}, "test2": "222"}', 'test1', 'test3')
-- json_extract_path函数返回的是json类型，而json_extract_path_text函数将json类型进一步转为了text类型进行返回，因此上面的语句也能写为下面的形式
select json_extract_path_text(json_extract_path('{"test1": {"test3": "333"}, "test2": "222"}', 'test1'), 'test3')
```

### 25、随机数

```sql
select dbms_random.value() from dual  -- 0-1之间的随机数，精度38位
```

$\Longrightarrow$

```sql
select random()  -- 0-1之间的随机数，精度17位
```

### 26、除数为0

在`Oracle`中，可以用`decode`函数规避除数为0的情况；但在`PostgreSQL`中，这样依旧会报除数为0的异常（即便真正执行时除数不会为0），必须通过`case when`语句定义分支：

```sql
select decode(num, 0, 0, 10 / num) as result from (
    select 0 as num from dual
) temp
```

$\Longrightarrow$

```sql
select (case when num = 0 then 0 else 10 / num end) as result from (
    select 0 as num
) temp
```

### 27、数据透视

```sql
select col3, "0", "1", "2"
from tablename
pivot(sum(col1) for col2 in (0, 1, 2))
-- sum(col1)可替换为其他聚合函数，col2为要从行转为列的字段，后面的(0, 1, 2)为将col2字段的哪些值转为列，col3为其他需要展示的字段
-- 转换为列的值也可以有别名
select col3, colname0, colname1, colname2
from tablename
pivot(sum(col1) for col2 in (0 colname0, 1 colname1, 2 colname2))
```

$\Longrightarrow$

```sql
-- case when语句形式
select col3,
sum(case when col2 = 0 then col1 else 0 end) as "0",
sum(case when col2 = 1 then col1 else 0 end) as "1",
sum(case when col2 = 2 then col1 else 0 end) as "2"
from tablename
group by col3
-- filter子句形式
select col3,
sum(col2) filter(where col1 = 0) as "0",
sum(col2) filter(where col1 = 1) as "1",
sum(col2) filter(where col1 = 2) as "2"
from tablename
group by col3
```


[WebSite](https://rbatis.github.io/rbatis.io/#/en/) | [简体中文](https://rbatis.github.io/rbatis.io/)

#### A highly Performant,Safe,Dynamic SQL ORM framework written in Rust, inspired by Mybatis and MybatisPlus.

[![Build Status](https://travis-ci.org/zhuxiujia/rbatis.svg?branch=master)](https://travis-ci.org/zhuxiujia/rbatis)

![Image text](logo.png)

##### Why not diesel or not sqlx ?

| Framework    | Async/.await | Learning curve | Dynamic SQL/py/Wrapper/built-in CRUD | Logical delete plugin| Pagination plugin
| ------ | ------ |------ |------ |------ |------ |
| rbatis | √     | easy   |   √     |    √     |   √     |  
| sqlx   | √     | hard (depends on macros and env. variables) |   x     |   x     |   x     |  
| diesel | x     | hard (use FFI, unsafe) |   x     |  x     |  x     |  

##### Performance comparison with Golang (in a docker environment)

| Framework     | Mysql（docker） | SQL statement（10k） | ns/operation（lower is better） | Qps(higher is better) |Memory usage(lower is better） |
|  ------ | ------ |------ |------ |------ |------ |
| Rust-rbatis/tokio  |  1 CPU, 1G memory    | select count(1) from table;    | 965649 ns/op   |  1035 Qps/s  |  2.1MB   |      
| Go-GoMybatis/http   |  1 CPU, 1G memory   | select count(1) from table;   | 1184503 ns/op  |  844  Qps/s   |  28.4MB  |     

* used json with serde_json for passing parameters and communication
* high performance, single threaded benchmark can easily achieve 200,000 QPS - data returned from database directly (
  zero lookup time) on a Windows 10 6 core i7 with 16 GB memory machine. Performace will be better using multiple
  threads, and it outperforms Go's GoMyBatis.
* supports logical deletes, pagination, py-like SQL and basic Mybatis functionalities.
* supports future,(in theory, if all io operations are replaced with async_std/tokio, it could achieve higher
  concurrency than Go-lang)
* supports logging, customizable logging based on `log` crate
* used 100% safe Rust with `#![forbid(unsafe_code)]` enabled
* [rbatis/example (import into Clion!)](https://github.com/rbatis/rbatis/tree/master/example/src)
* [website back end example(import into Clion!)](https://github.com/rbatis/abs_admin)

##### Example Rust backend service https://github.com/rbatis/abs_admin

##### Example Cargo.toml

``` rust
# add this library,and cargo install

# json (required)
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Date time (required)
chrono = { version = "0.4", features = ["serde"] }

# logging lib(required)
log = "0.4"
fast_log="1.3"

# BigDecimal lib(optional)
bigdecimal = "0.2"

# rbatis lib(required)
rbatis =  { version = "1.8" } 
```

##### Quick example: QueryWrapper and common usages (see example/crud_test.rs for details)

```rust
#[macro_use]
extern crate rbatis;

/// may also write `CRUDEnable` as `impl CRUDEnable for BizActivity{}`
/// #[crud_enable( table_name:biz_activity)]
/// #[crud_enable(id_name:"id"|id_type:"String"|table_name:"biz_activity"|table_columns:"id,name,version,delete_flag"|formats_pg:"id:{}::uuid")]
#[crud_enable]
#[derive(Clone, Debug)]
pub struct BizActivity {
    pub id: Option<String>,
    pub name: Option<String>,
    pub pc_link: Option<String>,
    pub h5_link: Option<String>,
    pub pc_banner_img: Option<String>,
    pub h5_banner_img: Option<String>,
    pub sort: Option<String>,
    pub status: Option<i32>,
    pub remark: Option<String>,
    pub create_time: Option<NaiveDateTime>,
    pub version: Option<i32>,
    pub delete_flag: Option<i32>,
}

// (optional) manually implement instead of using `derive(CRUDEnable)`. This allows manually rewriting `table_name()` function and supports  code completion in IDE.
//impl CRUDEnable for BizActivity {
//    type IdType = String;    
//    fn table_name()->String{
//        "biz_activity".to_string()
//    }
//    fn table_columns()->String{
//        "id,name,delete_flag".to_string()
//    }
//}

#[actix_rt::main]
async fn main() {
    /// initialize rbatis. May use `lazy_static` to define rbatis as a global variable because rbatis is thread safe
    let rb = Rbatis::new();
    /// connect to database  
    rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();
    /// customize connection pool parameters (optional)
// let mut opt =PoolOptions::new();
// opt.max_size=100;
// rb.link_opt("mysql://root:123456@localhost:3306/test",&opt).await.unwrap();
    /// newly constructed wrapper sql logic
    let wrapper = rb.new_wrapper()
        .eq("id", 1)                    //sql:  id = 1
        .and()                          //sql:  and 
        .ne("id", 1)                    //sql:  id <> 1
        .in_array("id", &[1, 2, 3])     //sql:  id in (1,2,3)
        .not_in("id", &[1, 2, 3])       //sql:  id not in (1,2,3)
        .like("name", 1)                //sql:  name like 1
        .or()                           //sql:  or
        .not_like("name", "asdf")       //sql:  name not like 'asdf'
        .between("create_time", "2020-01-01 00:00:00", "2020-12-12 00:00:00")//sql:  create_time between '2020-01-01 00:00:00' and '2020-01-01 00:00:00'
        .group_by(&["id"])              //sql:  group by id
        .order_by(true, &["id", "name"])//sql:  group by id,name
        ;

    let activity = BizActivity {
        id: Some("12312".to_string()),
        name: None,
        remark: None,
        create_time: Some(NaiveDateTime::now()),
        version: Some(1),
        delete_flag: Some(1),
    };
    /// saving
    rb.save("", &activity).await;
//Exec ==> INSERT INTO biz_activity (create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version) VALUES ( ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? )

    /// batch saving
    rb.save_batch("", &vec![activity]).await;
//Exec ==> INSERT INTO biz_activity (create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version) VALUES ( ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? ),( ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? )

    /// The query, Option wrapper, is None if the data is not found
    let result: Option<BizActivity> = rb.fetch_by_id("", &"1".to_string()).await.unwrap();
//Query ==> SELECT create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version  FROM biz_activity WHERE delete_flag = 1  AND id =  ? 

    /// query all
    let result: Vec<BizActivity> = rb.list("").await.unwrap();
//Query ==> SELECT create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version  FROM biz_activity WHERE delete_flag = 1

    ///query by id vec
    let result: Vec<BizActivity> = rb.list_by_ids("", &["1".to_string()]).await.unwrap();
//Query ==> SELECT create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version  FROM biz_activity WHERE delete_flag = 1  AND id IN  (?) 

    ///query by wrapper
    let w = rb.new_wrapper().eq("id", "1");
    let r: Result<Option<BizActivity>, Error> = rb.fetch_by_wrapper("", &w).await;
//Query ==> SELECT  create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version  FROM biz_activity WHERE delete_flag = 1  AND id =  ? 

    ///delete
    rb.remove_by_id::<BizActivity>("", &"1".to_string()).await;
//Exec ==> UPDATE biz_activity SET delete_flag = 0 WHERE id = 1

    ///delete batch
    rb.remove_batch_by_id::<BizActivity>("", &["1".to_string(), "2".to_string()]).await;
//Exec ==> UPDATE biz_activity SET delete_flag = 0 WHERE id IN (  ?  ,  ?  ) 

    ///update
    let w = rb.new_wrapper().eq("id", "12312");
    rb.update_by_wrapper("", &activity, &w).await;
//Exec ==> UPDATE biz_activity SET  create_time =  ? , delete_flag =  ? , status =  ? , version =  ?  WHERE id =  ? 
}

///...more usage,see crud.rs
```

#### macros (new addition)

```rust
    lazy_static! {
     static ref RB:Rbatis=Rbatis::new();
   }

/// Macro generates execution logic based on method definition, similar to @select dynamic SQL of Java/Mybatis
/// RB is the name referenced locally by Rbatis, for example DAO ::RB, com:: XXX ::RB... Can be
/// The second parameter is the standard driver SQL. Note that the corresponding database parameter mysql is? , pg is $1...
/// macro auto edit method to  'pub async fn select(name: &str) -> rbatis::core::Result<BizActivity> {}'
///
#[sql(RB, "select * from biz_activity where id = ?")]
async fn select(name: &str) -> BizActivity {}
//or： pub async fn select(name: &str) -> rbatis::core::Result<BizActivity> {}

#[async_std::test]
pub async fn test_macro() {
    fast_log::init_log("requests.log", 1000, log::Level::Info, None, true);
    RB.link("mysql://root:123456@localhost:3306/test").await.unwrap();
    let a = select("1").await.unwrap();
    println!("{:?}", a);
}
```

```rust
    lazy_static! {
     static ref RB:Rbatis=Rbatis::new();
   }

#[py_sql(RB, "select * from biz_activity where id = #{name}
                  if name != '':
                    and name=#{name}")]
async fn py_select(name: &str) -> Option<BizActivity> {}
//or： pub async fn select(name: &str) -> rbatis::core::Result<BizActivity> {}

#[async_std::test]
pub async fn test_macro_py_select() {
    fast_log::init_log("requests.log", 1000, log::Level::Info, None, true);
    RB.link("mysql://root:123456@localhost:3306/test").await.unwrap();
    let a = py_select("1").await.unwrap();
    println!("{:?}", a);
}
```

##### How to use logical deletes plugin (works for fetching or removing functions provided by rbatis，e.g. list**(),remove**()，fetch**())

```rust
   let mut rb = init_rbatis().await;
//rb.logic_plugin = Some(Box::new(RbatisLogicDeletePlugin::new_opt("delete_flag",1,0)));//Customize deleted/undeleted writing
rb.logic_plugin = Some(Box::new(RbatisLogicDeletePlugin::new("delete_flag")));
rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();
let r = rb.remove_batch_by_id::<BizActivity>("", & ["1".to_string(), "2".to_string()]).await;
if r.is_err() {
println ! ("{}", r.err().unwrap().to_string());
}
```

##### How to use pagination plugin

```rust
        let mut rb = Rbatis::new();
rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();
//框架默认RbatisReplacePagePlugin，如果需要自定义的话需要结构体 必须实现impl PagePlugin for Plugin***{}，例如：
//rb.page_plugin = Box::new(RbatisPagePlugin::new());

let req = PageRequest::new(1, 20);
let wraper= rb.new_wrapper()
.eq("delete_flag", 1);
let data: Page<BizActivity> = rb.fetch_page_by_wrapper("", & wraper, & req).await.unwrap();
println!("{}", serde_json::to_string(&data).unwrap());

//2020-07-10T21:28:40.036506700+08:00 INFO rbatis::rbatis - [rbatis] Query ==> SELECT count(1) FROM biz_activity  WHERE delete_flag =  ? LIMIT 0,20
//2020-07-10T21:28:40.040505200+08:00 INFO rbatis::rbatis - [rbatis] Args  ==> [1]
//2020-07-10T21:28:40.073506+08:00 INFO rbatis::rbatis - [rbatis] Total <== 1
//2020-07-10T21:28:40.073506+08:00 INFO rbatis::rbatis - [rbatis] Query ==> SELECT  create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version  FROM biz_activity  WHERE delete_flag =  ? LIMIT 0,20
//2020-07-10T21:28:40.073506+08:00 INFO rbatis::rbatis - [rbatis] Args  ==> [1]
//2020-07-10T21:28:40.076506500+08:00 INFO rbatis::rbatis - [rbatis] Total <== 5
```

```json
{
  "records": [
    {
      "id": "12312",
      "name": "null",
      "pc_link": "null",
      "h5_link": "null",
      "pc_banner_img": "null",
      "h5_banner_img": "null",
      "sort": "null",
      "status": 1,
      "remark": "null",
      "create_time": "2020-02-09T00:00:00+00:00",
      "version": 1,
      "delete_flag": 1
    }
  ],
  "total": 5,
  "size": 20,
  "current": 1,
  "serch_count": true
}
```

##### py-like sql example

``` python
//Execute to remote mysql and get the result. Supports any serializable type of SERde_JSON
        let rb = Rbatis::new();
        rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();
            let py = r#"
        SELECT * FROM biz_activity
        WHERE delete_flag = #{delete_flag}
        if name != null:
          AND name like #{name+'%'}
        if ids != null:
          AND id in (
          trim ',':
             for item in ids:
               #{item},
          )"#;
            let data: serde_json::Value = rb.py_fetch("", py, &json!({   "delete_flag": 1 })).await.unwrap();
            println!("{}", data);
```

#### logging system with fast_log here as an example

``` rust
 use log::{error, info, warn};
 fn  main(){
      fast_log::init_log("requests.log", 1000, log::Level::Info, None, true);
      info!("print data");
 }
```

#### Customize connection pool's size, timeout, active number of connections, and etc.

```rust
use rbatis::core::db::PoolOptions;

pub async fn init_rbatis() -> Rbatis {
    let rb = Rbatis::new();
    let mut opt = PoolOptions::new();
    opt.max_size = 20;
    rb.link_opt("mysql://root:123456@localhost:3306/test", &opt).await.unwrap();
}
```

#### `Async/.await` task support

``` rust
   async_std::task::block_on(async {
        let rb = Rbatis::new();
        rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();
        let context_id = "tx:1";//事务id号
        rb.begin(context_id).await.unwrap();
        let v: serde_json::Value = rb.fetch(context_id, "SELECT count(1) FROM biz_activity;").await.unwrap();
        println!("{}", v.clone());
        rb.commit(context_id).await.unwrap();
    });
```

### How to use rbatis with Rust web frameworks (actix-web is used here as an example, but all web frameworks based on tokio or async_std are supported)

``` rust
lazy_static! {
   static ref RB:Rbatis=Rbatis::new();
}

async fn index() -> impl Responder {
    let v:Result<i32,rbatis::core::Error> = RB.fetch("", "SELECT count(1) FROM biz_activity;").await;
    HttpResponse::Ok().body(format!("count(1)={}",v.unwrap_or(0)))
}

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    //日志
    fast_log::init_log("requests.log", 1000, log::Level::Info, None, true);
    //链接数据库
    RB.link("mysql://root:123456@localhost:3306/test").await.unwrap();
    //http路由
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(index))
    })
        .bind("127.0.0.1:8000")?
        .run()
        .await
}
```

### Supported data structures

| data structure    | is supported |
| ------ | ------ |
| Option                   | √     | 
| Vec                      | √     |  
| HashMap                      | √     |  
| Slice                    | √     |   
| i32,i64,f32,f64,bool,String...more rust type   | √     |  
| NativeDateTime           | √     |  
| BigDecimal               | √     |
| serde_json::Value...more serde type         | √     |

### Supported database √supported .WIP

| database    | is supported |
| ------ | ------ |
| Mysql            | √     |   
| Postgres         | √     |  
| Sqlite           | √     |  
| Mssql/Sqlserver            | √     |  
| MariaDB(Mysql)             | √     |
| TiDB(Mysql)             | √     |
| CockroachDB(Postgres)      | √     |

### Supported OS/Platforms

| platform   | is supported |
| ------ | ------ |
| Linux                   | √     | 
| Apple/MacOS             | √     |  
| Windows               | √     |

### Progress - in sequential order

| function    | is supported |
| ------ | ------ |
| CRUD, with built-in CRUD template (built-in CRUD supports logical deletes)                  | √     |
| LogSystem (logging component)                                          | √     | 
| Tx(task/Nested transactions)                                | √     |   
| Py(using py-like  statement in SQL)                         | √     | 
| async/await support                                             | √     | 
| PagePlugin(Pagincation)                                         | √     |
| LogicDelPlugin                                 | √    |
| DataBase Table ConvertPage(Web UI,Coming soon)                          | x     | 

* Conlusion: Assuming zero time consumed on IO, single threaded benchmark achieves 200K QPS or QPS, which is a few times
  more performant than GC languages like Go or Java.

### FAQ

* Postgres Types Define Please see Doc<br/>
  (中文)https://rbatis.github.io/rbatis.io/#/?id=%e6%95%b0%e6%8d%ae%e5%ba%93%e5%88%97%e6%a0%bc%e5%bc%8f%e5%8c%96%e5%ae%8f
  (En)https://rbatis.github.io/rbatis.io/#/en/?id=database-column-formatting-macro
* Support for DateTime and BigDecimal? <br/>
  Currently supports chrono::NaiveDateTime和bigdecimal::BigDecimal
* Supports for `async/.await` <br/>
  Currently supports both `async_std` and `tokio`
* Stmt in postgres uses $1, $2 instead of ? in Mysql, does this require some special treatment? No, because rbatis uses
  #{} to describe parametric variabls, you only need to write the correct parameter names and do not need to match it
  with the symbols used by the database.
* Supports for Oracle database driver? <br/>
  No, moving away from IOE is recommended.
* Which crate should be depended on if only the driver is needed? <br/>
  rbatis-core， Cargo.toml add rbatis-core = "*"
* How to select `async/.await` runtime? <br/>
  see https://rbatis.github.io/rbatis.io/
* column "id" is of type uuid but expression is of type text'? <br/>
  see https://rbatis.github.io/rbatis.io/#/en/?id=database-column-formatting-macro
* How to use '::uuid','::timestamp' on PostgreSQL? <br/>
  see https://rbatis.github.io/rbatis.io/#/en/?id=database-column-formatting-macro

# changelog

[changelog](https://github.com/rbatis/rbatis/releases/)

### Related Projects

* Logging: https://github.com/rbatis/fast_log

## In order to achieve the satisfaction of this ORM framework, your support is always our motivation, we are eager to welcome WeChat to donate to support us ~ or ~ star at the top right corner

## 为了称心如意的ORM框架，您的支持永远是我们的动力，迫切欢迎微信捐赠支持我们 ~或者~右上角点下star

![Image text](https://zhuxiujia.github.io/gomybatis.io/assets/wx_account.jpg)

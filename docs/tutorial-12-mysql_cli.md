# 异步MySQL客户端：mysql_cli
# 示例代码

[tutorial-12-mysql_cli.cc](../tutorial/tutorial-12-mysql_cli.cc)

# 关于mysql_cli

教程中的mysql_cli使用方式与官方客户端相似，是一个命令行交互式的异步MySQL客户端。

程序运行方式：./mysql_cli \<URL\>

启动之后可以直接在终端输入mysql命令与db进行交互，输入quit或Ctrl-C退出。

# MySQL URL的格式

mysql://username:password@host:port/dbname?character_set=charset

- username和password按需填写；

- port默认为3306；

- dbname为要用的数据库名，一般如果SQL语句只操作一个db的话建议填写；

- 如果用户在这一层有upstream选取需求，可以参考[upstream文档](../docs/about-upstream.md)；

- charset为字符集，默认utf8，具体可以参考MySQL官方文档[character-set.html](https://dev.mysql.com/doc/internals/en/character-set.html)。

MySQL URL示例：

mysql://root:password@127.0.0.1

mysql://@test.mysql.com:3306/db1?character_set=utf8

# 创建并启动MySQL任务

用户可以使用WFTaskFactory创建MySQL任务，创建接口与回调函数的用法都与workflow其他任务类似:
~~~cpp
using mysql_callback_t = std::function<void (WFMySQLTask *)>;

WFMySQLTask *create_mysql_task(const std::string& url, int retry_max, mysql_callback_t callback);

void set_query(const std::string& query);
~~~
用户创建完WFMySQLTask之后，可以对req调用 **set_query()** 写入SQL语句。

基于MySQL协议，如果建立完连接而发一个空包，server会等待而不是回包，因此用户会得到超时，因此我们对那些没有调用 ``set_query()`` 的task进行了特判并且会立刻返回**WFT_ERR_MYSQL_QUERY_NOT_SET**。

其他包括callback、series、user_data等与workflow其他task用法类似。

大致使用示例如下：
~~~cpp
int main(int argc, char *argv[])
{
    ...
    WFMySQLTask *task = WFTaskFactory::create_mysql_task(url, RETRY_MAX, mysql_callback);
    task->get_req()->set_query("SHOW TABLES;");
    ...
    task->start();
    ...
}
~~~

# 支持的命令

目前支持的命令为**COM_QUERY**，已经能涵盖用户基本的增删改查、建库删库、建表删表、预处理、使用存储过程和使用事务的需求。

因为我们的交互命令中不支持选库（**USE**命令），所以，如果SQL语句中有涉及到**跨库**的操作，则可以通过**db_name.table_name**的方式指定具体哪个库的哪张表。

**多条命令**可以拼接到一起通过 ``set_query()`` 传给WFMySQLTask，一般来说多条语句是可以一次把结果全部拿回来的，但由于MySQL协议中回包的方式与我们一问一答的通信有某些特例下不能兼容，因此 ``set_query()`` 中的SQL语句有以下注意事项：

- 可以多条单结果集语句的拼接（一般的INSERT/UPDATE/SELECT/PREPARE）

- 也可以是一条多结果集语句（比如CALL存储过程）

- 其他情况建议把SQL语句分开多次请求

举个例子：
~~~cpp
// 单结果集的多条语句拼接，可以正确拿到返回结果
req->set_query("SELECT * FROM table1; SELECT * FROM table2; INSERT INTO table3 (id) VALUES (1);");

// 多结果集的单条语句，也可以正确拿到返回结果
req->set_query("CALL procedure1();");

// 多结果集与其他拼接都不能完全拿到所有返回结果
req->set_query("CALL procedure1(); SELECT * FROM table1;");
~~~

# 结果解析

与workflow其他任务类似，可以用task->get_resp()拿到**MySQLResponse**，我们可以通过**MySQLResultCursor**遍历结果集及其中的每个列的信息**MySQLField**、每行和每个**MySQLCell**。具体接口可以查看：[MySQLResult.h](../src/protocol/MySQLResult.h)

具体使用从外到内的步骤应该是：

1. 判断任务状态（代表通信层面状态）：用户通过判断 **task->get_state()** 等于WFT_STATE_SUCCESS来查看任务执行是否成功；

2. 判断回复包类型（代表返回包解析状态）：调用 **resp->get_packet_type()** 查看MySQL返回包类型，常见的几个类型为：
  - MYSQL_PACKET_OK：返回非结果集的请求: 解析成功；
  - MYSQL_PACKET_EOF：返回结果集的请求: 解析成功；
  - MYSQL_PACKET_ERROR：请求:失败；

3. 判断结果集状态（代表结果集读取状态）：用户可以使用MySQLResultCursor读取结果集中的内容，因为MySQL server返回的数据是多结果集的，因此一开始cursor会**自动指向第一个结果集**的读取位置。通过 **cursor->get_cursor_status()** 可以拿到的几种状态：
  - MYSQL_STATUS_GET_RESULT：有数据可读；
  - MYSQL_STATUS_END：当前结果集已读完最后一行；
  - MYSQL_STATUS_EOF：所有结果集已取完；
  - MYSQL_STATUS_OK：此回复包为非结果集包，无需通过结果集接口读数据；
  - MYSQL_STATUS_ERROR：解析错误；

4. 读取columns中每个field：
  - ``int get_field_count() const;``
  - ``const MySQLField *fetch_field();``
    - ``const MySQLField *const *fetch_fields() const;``

5. 读取每一行：按行读取可以使用 **cursor->fetch_row()** 直到返回值为false。其中会移动cursor内部对当前结果集的指向每行的offset：
  - ``int get_rows_count() const;``
  - ``bool fetch_row(std::vector<MySQLCell>& row_arr);``
  - ``bool fetch_row(std::map<std::string, MySQLCell>& row_map);``
  - ``bool fetch_row(std::unordered_map<std::string, MySQLCell>& row_map);``
  - ``bool fetch_row_nocopy(const void **data, size_t *len, int *data_type);``

6. 直接把当前结果集的所有行拿出：所有行的读取可以使用 **cursor->fetch_all()** ，内部用来记录行的cursor会直接移动到最后；cursor状态会变成MYSQL_STATUS_END：
  - ``bool fetch_all(std::vector<std::vector<MySQLCell>>& rows);``

7. 返回当前结果集的头部：如果有必要重读这个结果集，可以使用 **cursor->rewind()** 回到当前结果集头部，再通过第5步或第6步进行读取；

8. 拿到下一个结果集：因为MySQL server返回的数据包可能是包含多结果集的（比如每个select语句为一个结果集；或者call procedure返回的多结果集数据），因此用户可以通过 **cursor->next_result_set()** 跳到下一个结果集，返回值为false表示所有结果集已取完。

9. 返回第一个结果集：**cursor->first_result_set()** 可以让我们返回到所有结果集的头部，然后可以从第3步开始重新拿数据；

10. 每列具体数据MySQLCell：第5步中读取到的一行，由多列组成，每列结果为MySQLCell，基本使用接口有：
  - ``int get_data_type();`` 返回MYSQL_TYPE_LONG、MYSQL_TYPE_STRING...具体参考[mysql_types.h](../src/protocol/mysql_types.h)
  - ``bool is_TYPE() const;`` TYPE为int、string、ulonglong，判断是否是某种类型
  - ``TYPE as_TYPE() const;`` 同上，以某种类型读出MySQLCell的数据
  - ``void get_cell_nocopy(const void **data, size_t *len, int *data_type) const;`` nocopy接口

整体示例如下：
~~~cpp
void task_callback(WFMySQLTask *task)
{
    // step-1. 判断任务状态 
    if (task->get_state() != WFT_STATE_SUCCESS)
    {
        fprintf(stderr, "task error = %d\n", task->get_error());
        return;
    }

    MySQLResultCursor cursor(task->get_resp());
    bool test_first_result_set_flag = false;
    bool test_rewind_flag = false;

begin:
    // step-2. 判断回复包状态
    switch (resp->get_packet_type())
    {
    case MYSQL_PACKET_OK:
        fprintf(stderr, "OK. %llu rows affected. %d warnings. insert_id=%llu.\n",
                task->get_resp()->get_affected_rows(),
                task->get_resp()->get_warnings(),
                task->get_resp()->get_last_insert_id());
        break;

    case MYSQL_PACKET_EOF:
        do {
            fprintf(stderr, "cursor_status=%d field_count=%u rows_count=%u ",
                    cursor.get_cursor_status(), cursor.get_field_count(), cursor.get_rows_count());
            // step-3. 判断结果集状态
            if (cursor.get_cursor_status() != MYSQL_STATUS_GET_RESULT)
                break;

            // step-4. 读取每个fields。这是个nocopy api
            const MySQLField *const *fields = cursor.fetch_fields();
            for (int i = 0; i < cursor.get_field_count(); i++)
            {
                fprintf(stderr, "db=%s table=%s name[%s] type[%s]\n",
                        fields[i]->get_db().c_str(), fields[i]->get_table().c_str(),
                        fields[i]->get_name().c_str(), datatype2str(fields[i]->get_data_type()));
            }

            // step-6. 把所有行读出，也可以while (cursor.fetch_row(map/vector)) 按step-5拿每一行
            std::vector<std::vector<MySQLCell>> rows;

            cursor.fetch_all(rows);
            for (unsigned int j = 0; j < rows.size(); j++)
            {
                // step-10. 具体每个cell的读取
                for (unsigned int i = 0; i < rows[j].size(); i++)
                {
                    fprintf(stderr, "[%s][%s]", fields[i]->get_name().c_str(),
                            datatype2str(rows[j][i].get_data_type()));
                    // step-10. 判断具体类型is_string()和转换具体类型as_string()
                    if (rows[j][i].is_string())
                    {
                        std::string res = rows[j][i].as_string();
                        fprintf(stderr, "[%s]\n", res.c_str());
                    } else if (rows[j][i].is_int()) {
                        fprintf(stderr, "[%d]\n", rows[j][i].as_int());
                    } // else if ...
                }
            }
        // step-8. 拿下一个结果集
        } while (cursor.next_result_set());

        if (test_first_result_set_flag == false)
        {
            test_first_result_set_flag = true;
            // step-9. 返回第一个结果集
            cursor.first_result_set();
            goto begin;
        }

        if (test_rewind_flag == false)
        {
            test_rewind_flag = true;
            // step-7. 返回当前结果集头部
            cursor.rewind();
            goto begin;
        }
        break;

    default:
        fprintf(stderr, "Abnormal packet_type=%d\n", resp->get_packet_type());
        break;
    }
    return;
}
~~~

# WFMySQLConnection

由于我们是高并发异步客户端，这意味着我们对一个server的连接可能会不止一个。而MySQL的事务和预处理都是带状态的，为了保证一次事务或预处理独占一个连接，用户可以使用我们封装的二级工厂WFMySQLConnection来创建任务，每个WFMySQLConnection保证独占一个连接，具体参考[WFMySQLConnection.h](../src/client/WFMySQLConnection.h)。

### 1. WFMySQLConnection的创建与初始化

创建一个WFMySQLConnection的时候需要传入一个**id**，必须全局唯一，之后的调用内部都会由这个id去唯一找到对应的那个连接。

初始化需要传入**url**，之后在这个connection上创建的任务就不需要再设置url了。

~~~cpp
class WFMySQLConnection
{
public:
    WFMySQLConnection(int id);
    int init(const std::string& url);
    ...
};
~~~

### 2. 创建任务与关闭连接

通过 **create_query_task()** ，写入SQL请求和回调函数即可创建任务，该任务一定从这一个connection发出。

有时候我们需要手动关闭这个连接。因为当我们不再使用它的时候，这个连接会一直保持到MySQL server超时。期间如果使用同一个id和url去创建WFMySQLConnection的话就可以复用到这个连接。

因此我们建议如果不准备复用连接，应使用 **create_disconnect_task()** 创建一个任务，手动关闭这个连接。
~~~cpp
class WFMySQLConnection
{
public:
    ...
    WFMySQLTask *create_query_task(const std::string& query,
                                   mysql_callback_t callback);
    WFMySQLTask *create_disconnect_task(mysql_callback_t callback);
}
~~~
WFMySQLConnection相当于一个二级工厂，我们约定任何工厂对象的生命周期无需保持到任务结束，以下代码完全合法：
~~~cpp
    WFMySQLConnection *conn = new WFMySQLConnection(1234);
    conn->init(url);
    auto *task = conn->create_query_task("SELECT * from table", my_callback);
    conn->deinit();
    delete conn;
    task->start();
~~~

### 3. 注意事项

如果在使用事务期间已经开始BEGIN但还没有COMMIT或ROLLBACK，且期间连接发生过中断，则连接会被框架内部自动重连，用户会在下一个task请求中拿到**ECONNRESET**错误。此时还没COMMIT的事务语句已经失效，需要重新再发一遍。

### 4. 预处理

用户也可以通过WFMySQLConnection来做预处理**PREPARE**，因此用户可以很方便地用作**防SQL注入**。如果连接发生了重连，也会得到一个**ECONNRESET**错误。

### 5. 完整示例

~~~cpp
WFMySQLConnection conn(1);
conn.init("mysql://root@127.0.0.1/test");

// test transaction
const char *query = "BEGIN;";
WFMySQLTask *t1 = conn.create_query_task(query, task_callback);
query = "SELECT * FROM check_tiny FOR UPDATE;";
WFMySQLTask *t2 = conn.create_query_task(query, task_callback);
query = "INSERT INTO check_tiny VALUES (8);";
WFMySQLTask *t3 = conn.create_query_task(query, task_callback);
query = "COMMIT;";
WFMySQLTask *t4 = conn.create_query_task(query, task_callback);
WFMySQLTask *t5 = conn.create_disconnect_task(task_callback);
((*t1) > t2 > t3 > t4 > t5).start();
~~~


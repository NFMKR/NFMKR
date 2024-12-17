---
title: MySQL的配置
date: 2024-12-17 13:52:46
tags:
categories:
  - 文章
---

### MySQL 数据库配置笔记 📝

#### 1. 基本配置结构
```go
// 数据库配置结构体
type DBConfig struct {
    Username string  // 数据库用户名
    Password string  // 数据库密码
    Host     string  // 数据库主机地址
    Port     string  // 数据库端口
    DBName   string  // 数据库名称
}
```

这就像填表一样，把数据库连接需要的信息都写好。

#### 2. 从环境变量获取配置
```go
// 创建新的数据库配置
func NewDBConfig() DBConfig {
    return DBConfig{
        Username: os.Getenv("DB_USERNAME"),  // 从环境变量读取用户名
        Password: os.Getenv("DB_PASSWORD"),  // 从环境变量读取密码
        Host:     os.Getenv("DB_HOST"),      // 从环境变量读取主机地址
        Port:     os.Getenv("DB_PORT"),      // 从环境变量读取端口
        DBName:   os.Getenv("DB_NAME"),      // 从环境变量读取数据库名
    }
}
```

就像从配置文件读取信息一样，把所有配置都放在环境变量里，更安全也更灵活。

#### 3. 连接数据库的步骤

```go
func ConnectToDatabase(config DBConfig) (*sql.DB, error) {
    // 第一步：先连接 MySQL 服务器（不指定数据库）
    dsn := fmt.Sprintf("%s:%s@tcp(%s:%s)/", 
        config.Username, 
        config.Password, 
        config.Host, 
        config.Port,
    )
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, fmt.Errorf("连接 MySQL 失败: %v", err)
    }
    defer db.Close()

    // 第二步：检查数据库是否存在
    query := fmt.Sprintf("SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA WHERE SCHEMA_NAME = '%s'", 
        config.DBName)
    var dbName string
    err = db.QueryRow(query).Scan(&dbName)

    // 第三步：如果数据库不存在，就创建它
    if dbName == "" {
        createDBQuery := fmt.Sprintf("CREATE DATABASE %s", config.DBName)
        _, err := db.Exec(createDBQuery)
        if err != nil {
            return nil, fmt.Errorf("创建数据库失败: %v", err)
        }
        fmt.Printf("数据库 %s 创建成功！\n", config.DBName)
    }

    // 第四步：连接到指定的数据库
    dsnWithDB := fmt.Sprintf("%s:%s@tcp(%s:%s)/%s", 
        config.Username, 
        config.Password, 
        config.Host, 
        config.Port, 
        config.DBName,
    )
    dbWithDB, err := sql.Open("mysql", dsnWithDB)
    if err != nil {
        return nil, fmt.Errorf("连接数据库失败: %v", err)
    }

    // 第五步：测试连接
    if err := dbWithDB.Ping(); err != nil {
        return nil, fmt.Errorf("ping 数据库失败: %v", err)
    }

    return dbWithDB, nil
}
```

#### 4. 使用示例

```go
func main() {
    // 1. 创建配置
    dbConfig := NewDBConfig()

    // 2. 连接数据库
    db, err := ConnectToDatabase(dbConfig)
    if err != nil {
        log.Fatalf("数据库连接失败: %v", err)
    }
    defer db.Close()

    fmt.Println("成功连接到数据库！")
}
```

#### 5. 环境变量配置示例 (.env 文件)
```env
DB_USERNAME=root
DB_PASSWORD=123456
DB_HOST=localhost
DB_PORT=3306
DB_NAME=my_database
```

#### 6. 重要知识点总结

1. **DSN (Data Source Name) 格式**：
   ```
   username:password@tcp(host:port)/dbname
   ```

2. **连接步骤**：
   - 先连接 MySQL 服务器
   - 检查数据库是否存在
   - 不存在则创建数据库
   - 连接到具体数据库
   - 测试连接是否成功

3. **错误处理**：
   - 每个步骤都要检查错误
   - 使用 defer 关闭数据库连接
   - 返回具体的错误信息

4. **最佳实践**：
   - 使用环境变量存储配置
   - 分离配置和连接逻辑
   - 使用结构体组织配置信息
   - 添加连接测试（Ping）

#### 7. 注意事项

- 记得导入必要的包：
  ```go
  import (
      "database/sql"
      "fmt"
      "log"
      "os"
      _ "github.com/go-sql-driver/mysql"
  )
  ```
- 保护好数据库凭证
- 合理设置连接池参数
- 记得处理连接关闭
- 使用事务处理重要操作

---
# Sequelize详解

### Sequelize — 关系型数据库 ORM

#### 1. **概述**
- **Sequelize** 是一个基于 Node.js 的 ORM（Object-Relational Mapping）库，用于与关系型数据库（如 MySQL、PostgreSQL、SQLite、MSSQL）进行交互。
- 通过 Sequelize，开发者可以使用 JavaScript 来操作数据库，而不需要直接编写 SQL 语句。
- Sequelize 提供了模型定义、关联关系、事务管理、查询构建等功能。

#### 2. **安装 Sequelize**
```bash
npm install sequelize mysql2
```

#### 3. **配置连接**
使用 Sequelize 创建数据库连接，支持 MySQL、PostgreSQL、SQLite、MSSQL 等数据库：
```javascript
const { Sequelize } = require('sequelize');

// 创建 Sequelize 实例，连接到数据库
const sequelize = new Sequelize('database', 'username', 'password', {
  host: 'localhost',
  dialect: 'mysql',  // 或 'postgres', 'sqlite', 'mssql'
});
```

#### 4. **测试数据库连接**
```javascript
sequelize.authenticate()
  .then(() => {
    console.log('Connection established successfully.');
  })
  .catch(err => {
    console.error('Unable to connect to the database:', err);
  });
```

#### 5. **定义模型**
使用 `sequelize.define()` 定义数据表模型：
```javascript
const { DataTypes } = require('sequelize');

// 定义模型（对应数据库中的表）
const User = sequelize.define('User', {
  name: {
    type: DataTypes.STRING,
    allowNull: false,  // 不允许为空
  },
  email: {
    type: DataTypes.STRING,
    unique: true,  // 唯一值
    allowNull: false,
  },
});
```

#### 6. **同步模型到数据库**
```javascript
sequelize.sync()
  .then(() => {
    console.log("Database synced.");
  })
  .catch(err => {
    console.error("Unable to sync database:", err);
  });
```

#### 7. **基本操作**
- **创建记录**:
  ```javascript
  User.create({
    name: 'John Doe',
    email: 'john.doe@example.com',
  }).then(user => {
    console.log(user.toJSON());
  });
  ```

- **查询记录**:
  ```javascript
  User.findAll().then(users => {
    console.log(users);
  });
  ```

- **更新记录**:
  ```javascript
  User.update({ name: 'Jane Doe' }, {
    where: { id: 1 },
  }).then(() => {
    console.log('User updated');
  });
  ```

- **删除记录**:
  ```javascript
  User.destroy({
    where: { id: 1 },
  }).then(() => {
    console.log('User deleted');
  });
  ```

#### 8. **模型关联**
Sequelize 支持多种关联关系，如一对一、一对多、多对多等：
- **一对多关联**:
  ```javascript
  User.hasMany(Post); // 一个人有多篇文章
  Post.belongsTo(User); // 每篇文章属于一个用户
  ```

- **多对多关联**:
  ```javascript
  User.belongsToMany(Role, { through: 'UserRoles' });
  Role.belongsToMany(User, { through: 'UserRoles' });
  ```

#### 9. **事务管理**
Sequelize 支持事务操作，确保多个数据库操作的原子性：
```javascript
const t = await sequelize.transaction();

try {
  await User.create({ name: 'John' }, { transaction: t });
  await Post.create({ title: 'New Post' }, { transaction: t });
  await t.commit();
} catch (error) {
  await t.rollback();
}
```
### 总结
- **Sequelize** 是用于关系型数据库（如 MySQL、PostgreSQL）的 ORM 框架，通过定义模型来管理数据库表，并简化数据库的增删改查操作。
---
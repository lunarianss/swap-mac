# MySQL 表结构设计规范


**适用场景**：并发量大、数据量大的互联网业务

## 一、通用字段规范

### 1.1 主键设计

```sql
`id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID'
-- 或
`id` bigint(20) NOT NULL COMMENT 'Primary Key ID'
```

**规范：**

- 使用 `bigint` 或 `bigint unsigned` 类型
- 自增主键使用 `AUTO_INCREMENT`
- 必须添加 `COMMENT` 注释

### 1.2 时间戳字段（必备）

```sql
`created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
`updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
`deleted_at` bigint NOT NULL DEFAULT '0' COMMENT '删除时间'
```

**规范：**

- `created_at`: 创建时间，自动设置为当前时间
- `updated_at`: 更新时间，自动更新
- `deleted_at`: 软删除标记，使用 bigint 存储时间戳（0 表示未删除）

**时间类型选择：**

- 优先使用 `timestamp` 类型（自动时区转换）
- 软删除使用 `bigint` 存储 Unix 时间戳

### 1.3 用户追踪字段

```sql
`created_by` varchar(128) NOT NULL DEFAULT '' COMMENT '创建人',
`updated_by` varchar(128) NOT NULL DEFAULT '' COMMENT '修改人'
```

**规范：**

- 记录创建人和修改人
- 使用 `varchar(128)` 类型
- 默认值为空字符串

### 1.4 空间隔离字段

```sql
`space_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '空间 ID'
```

**规范：**

- 多租户系统必备字段
- 用于数据隔离和分片
- 几乎所有业务表都需要此字段

## 二、字段类型规范

### 2.1 数值类型

```sql
-- ID 类型
`id` bigint unsigned NOT NULL
`user_id` bigint(20) unsigned NOT NULL DEFAULT 0

-- 计数器
`version_num` int NOT NULL DEFAULT '0'
`next_version_num` bigint unsigned NOT NULL DEFAULT '1'

-- 枚举/状态码
`status` int unsigned NOT NULL DEFAULT '0'
`space_type` tinyint(4) NOT NULL DEFAULT '0'
```

**规范：**

- ID 使用 `bigint unsigned`
- 计数器使用 `int` 或 `bigint unsigned`
- 状态码使用 `int unsigned` 或 `tinyint`
- 默认值根据业务需求设置（通常为 0）

### 2.2 字符串类型

```sql
-- 短字符串（名称、标识）
`name` varchar(255) NOT NULL DEFAULT '' COMMENT '名称',
`unique_name` varchar(128) NOT NULL DEFAULT '' COMMENT '唯一名称',
`status` varchar(128) NOT NULL DEFAULT '' COMMENT '状态',

-- 中等长度（描述）
`description` varchar(1024) NOT NULL DEFAULT '' COMMENT '描述',
`description` varchar(2048) NOT NULL DEFAULT '' COMMENT '数据集描述',

-- 长文本
`err_msg` blob COMMENT '错误信息',
`status_message` blob COMMENT '状态提示信息'
```

**规范：**

- 名称、标识：`varchar(128)` 或 `varchar(255)`
- 描述：`varchar(1024)` 或 `varchar(2048)`
- 长文本/错误信息：使用 `blob` 或 `text`
- 默认值为空字符串 `''`（除非是 blob/text）
- 必须指定字符集：`CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci`

### 2.3 JSON 类型

```sql
`spec` json DEFAULT NULL COMMENT '规格配置',
`features` json DEFAULT NULL COMMENT '功能开关',
`change_log` json NOT NULL COMMENT '变更日志'
```

**规范：**

- 用于存储结构化配置、动态字段
- 可以为 NULL 或 NOT NULL（根据业务需求）
- 适合存储不固定的扩展字段

### 2.4 时间类型

```sql
-- 自动时间戳
`created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
`updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

-- 可为空的时间
`deleted_at` timestamp NULL DEFAULT NULL,
`expired_at` timestamp NULL DEFAULT NULL,
`start_at` timestamp NULL DEFAULT NULL,

-- 软删除时间戳
`deleted_at` bigint NOT NULL DEFAULT '0'
```

**规范：**

- 自动时间戳使用 `timestamp` + `DEFAULT CURRENT_TIMESTAMP`
- 可选时间字段使用 `NULL DEFAULT NULL`
- 软删除推荐使用 `bigint` 存储 Unix 时间戳

## 三、索引设计规范

### 3.1 主键索引

```sql
PRIMARY KEY (`id`)
```

### 3.2 唯一索引

```sql
-- 单字段唯一
UNIQUE KEY `idx_unique_name` (`unique_name`),
UNIQUE KEY `idx_email` (`email`),

-- 多字段联合唯一（包含软删除）
UNIQUE KEY `uk_space_id_category_name` (`space_id`, `category`, `name`, `deleted_at`),
UNIQUE KEY `uk_space_id_tag_key_id_version_num_tag_type` (`space_id`, `tag_key_id`, `version_num`, `tag_type`)
```

**规范：**

- 唯一索引命名：`uk_` 或 `idx_` 前缀
- 软删除场景必须包含 `deleted_at` 字段
- 多租户场景必须包含 `space_id` 字段

### 3.3 普通索引

```sql
-- 单字段索引
KEY `idx_owner_id` (`owner_id`),
KEY `idx_session_key` (`session_key`),

-- 联合索引（查询优化）
KEY `idx_space_id_category_updated_at_id` (`space_id`, `category`, `updated_at`, `id`),
KEY `idx_space_deleted_created_by` (`space_id`, `created_by`, `deleted_at`),
KEY `idx_space_id_tag_type_status_updated_at` (`space_id`, `tag_type`, `status`, `updated_at`)
```

**规范：**

- 索引命名：`idx_` 前缀 + 字段名组合
- 联合索引遵循最左前缀原则
- 高频查询字段组合建立联合索引
- 包含 `space_id` 的查询必须建立相应索引

### 3.4 索引设计原则

1. **多租户隔离**：所有查询索引第一个字段必须是 `space_id`
2. **软删除支持**：涉及软删除的查询索引必须包含 `deleted_at`
3. **覆盖索引**：高频查询考虑建立覆盖索引
4. **索引顺序**：`space_id` > 等值查询 > 范围查询 > 排序字段

## 四、表结构设计模式

### 4.1 基础业务表模板

```sql
CREATE TABLE IF NOT EXISTS `table_name`
(
    -- 主键
    `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID',

    -- 多租户隔离
    `space_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '空间 ID',

    -- 业务字段
    `name` varchar(255) NOT NULL DEFAULT '' COMMENT '名称',
    `description` varchar(1024) NOT NULL DEFAULT '' COMMENT '描述',
    `status` varchar(128) NOT NULL DEFAULT '' COMMENT '状态',

    -- JSON 扩展字段
    `spec` json DEFAULT NULL COMMENT '规格配置',

    -- 审计字段
    `created_by` varchar(128) NOT NULL DEFAULT '' COMMENT '创建人',
    `updated_by` varchar(128) NOT NULL DEFAULT '' COMMENT '修改人',
    `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    `deleted_at` bigint NOT NULL DEFAULT '0' COMMENT '删除时间',

    -- 索引
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_space_id_name` (`space_id`, `name`, `deleted_at`),
    KEY `idx_space_id_status` (`space_id`, `status`, `deleted_at`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_general_ci COMMENT = '表注释';
```

### 4.2 关联表模板

```sql
CREATE TABLE IF NOT EXISTS `ref_table`
(
    `id` bigint unsigned NOT NULL DEFAULT '0' COMMENT 'id',
    `space_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '空间 id',
    `parent_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '父对象 id',
    `child_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '子对象 id',
    `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',

    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_parent_child` (`space_id`, `parent_id`, `child_id`),
    KEY `idx_parent_id` (`space_id`, `parent_id`),
    KEY `idx_child_id` (`space_id`, `child_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_general_ci COMMENT = '关联表';
```

### 4.3 版本管理表模板

```sql
CREATE TABLE IF NOT EXISTS `versioned_table`
(
    `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID',
    `space_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '空间 ID',
    `entity_id` bigint unsigned NOT NULL DEFAULT '0' COMMENT '实体 ID',
    `version` varchar(64) NOT NULL DEFAULT '' COMMENT '版本号',
    `version_num` int NOT NULL DEFAULT '0' COMMENT '数字版本号',
    `content` blob COMMENT '版本内容',
    `change_log` json NOT NULL COMMENT '变更日志',
    `created_by` varchar(128) NOT NULL DEFAULT '' COMMENT '创建人',
    `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',

    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_entity_version` (`space_id`, `entity_id`, `version_num`),
    KEY `idx_entity_id` (`space_id`, `entity_id`, `created_at`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_general_ci COMMENT = '版本表';
```

## 五、命名规范

### 5.1 表命名

- 使用小写字母和下划线
- 使用单数形式：`user` 而非 `users`
- 关联表：`parent_child_ref` 或 `expt_evaluator_ref`
- 结果表：`expt_item_result`
- 版本表：`dataset_version`

### 5.2 字段命名

- 使用小写字母和下划线
- ID 字段：`id`, `user_id`, `space_id`
- 时间字段：`created_at`, `updated_at`, `deleted_at`
- 状态字段：`status`, `status_message`
- 布尔字段：`is_active`, `user_verified`

### 5.3 索引命名

- 主键：`PRIMARY KEY`
- 唯一索引：`uk_` + 字段组合
- 普通索引：`idx_` + 字段组合
- 示例：`uk_space_id_name`, `idx_space_id_status`

## 六、字符集和排序规则

```sql
ENGINE = InnoDB
DEFAULT CHARSET = utf8mb4
COLLATE = utf8mb4_general_ci
```

**规范：**

- 统一使用 `utf8mb4` 字符集（支持 emoji）
- 排序规则使用 `utf8mb4_general_ci`（不区分大小写）
- 特殊需求可使用 `utf8mb4_bin`（区分大小写）
- 引擎统一使用 `InnoDB`

## 七、58到家数据库30条军规

**适用场景**：并发量大、数据量大的互联网业务

### 7.1 基础规范

**（1）必须使用 InnoDB 存储引擎**

解读：支持事务、行级锁、并发性能更好、CPU 及内存缓存页优化使得资源利用率更高

**（2）必须使用 UTF8 字符集**

解读：万国码，无需转码，无乱码风险，节省空间

**（3）数据表、数据字段必须加入中文注释**

解读：N 年后谁知道这个 r1, r2, r3 字段是干嘛的

**（4）禁止使用存储过程、视图、触发器、Event**

解读：高并发大数据的互联网业务，架构设计思路是"解放数据库 CPU，将计算转移到服务层"，并发量大的情况下，这些功能很可能将数据库拖死，业务逻辑放到服务层具备更好的扩展性，能够轻易实现"增机器就加性能"。数据库擅长存储与索引，CPU 计算还是上移吧

**（5）禁止存储大文件或者大照片**

解读：为何要让数据库做它不擅长的事情？大文件和照片存储在文件系统，数据库里存 URI 多好

### 7.2 命名规范

**（6）只允许使用内网域名，而不是 IP 连接数据库**

**（7）线上环境、开发环境、测试环境数据库内网域名遵循命名规范**

- 业务名称：xxx
- 线上环境：dj.xxx.db
- 开发环境：dj.xxx.rdb
- 测试环境：dj.xxx.tdb
- 从库在名称后加 -s 标识，备库在名称后加 -ss 标识
- 线上从库：dj.xxx-s.db
- 线上备库：dj.xxx-sss.db

**（8）库名、表名、字段名：小写，下划线风格，不超过 32 个字符，必须见名知意，禁止拼音英文混用**

**（9）表名 t_xxx，非唯一索引名 idx_xxx，唯一索引名 uniq_xxx**

### 7.3 表设计规范

**（10）单实例表数目必须小于 500**

**（11）单表列数目必须小于 30**

**（12）表必须有主键，例如自增主键**

解读：

- a）主键递增，数据行写入可以提高插入性能，可以避免 page 分裂，减少表碎片提升空间和内存的使用
- b）主键要选择较短的数据类型，InnoDB 引擎普通索引都会保存主键的值，较短的数据类型可以有效的减少索引的磁盘空间，提高索引的缓存效率
- c）无主键的表删除，在 row 模式的主从架构，会导致备库夯住

**（13）禁止使用外键，如果有外键完整性约束，需要应用程序控制**

解读：外键会导致表与表之间耦合，update 与 delete 操作都会涉及相关联的表，十分影响 SQL 的性能，甚至会造成死锁。高并发情况下容易造成数据库性能问题，大数据高并发业务场景数据库使用以性能优先

### 7.4 字段设计规范

**（14）必须把字段定义为 NOT NULL 并且提供默认值**

解读：

- a）null 的列使索引/索引统计/值比较都更加复杂，对 MySQL 来说更难优化
- b）null 这种类型 MySQL 内部需要进行特殊处理，增加数据库处理记录的复杂性；同等条件下，表中有较多空字段的时候，数据库的处理性能会降低很多
- c）null 值需要更多的存储空间，无论是表还是索引中每行中的 null 的列都需要额外的空间来标识
- d）对 null 的处理时候，只能采用 is null 或 is not null，而不能采用 =、in、<、<>、!=、not in 这些操作符号。如：where name!='shenjian'，如果存在 name 为 null 值的记录，查询结果就不会包含 name 为 null 值的记录

**（15）禁止使用 TEXT、BLOB 类型**

解读：会浪费更多的磁盘和内存空间，非必要的大量的大字段查询会淘汰掉热数据，导致内存命中率急剧降低，影响数据库性能

**（16）禁止使用小数存储货币**

解读：使用整数吧，小数容易导致钱对不上

**（17）必须使用 varchar(20) 存储手机号**

解读：

- a）涉及到区号或者国家代号，可能出现 +-()
- b）手机号会去做数学运算么？
- c）varchar 可以支持模糊查询，例如：like "138%"

**（18）禁止使用 ENUM，可使用 TINYINT 代替**

解读：

- a）增加新的 ENUM 值要做 DDL 操作
- b）ENUM 的内部实际存储就是整数，你以为自己定义的是字符串？

### 7.5 索引设计规范

**（19）单表索引建议控制在 5 个以内**

**（20）单索引字段数不允许超过 5 个**

解读：字段超过 5 个时，实际已经起不到有效过滤数据的作用了

**（21）禁止在更新十分频繁、区分度不高的属性上建立索引**

解读：

- a）更新会变更 B+ 树，更新频繁的字段建立索引会大大降低数据库性能
- b）"性别"这种区分度不大的属性，建立索引是没有什么意义的，不能有效过滤数据，性能与全表扫描类似

**（22）建立组合索引，必须把区分度高的字段放在前面**

解读：能够更加有效的过滤数据

### 7.6 SQL 使用规范

**（23）禁止使用 SELECT *，只获取必要的字段，需要显示说明列属性**

解读：

- a）读取不需要的列会增加 CPU、IO、NET 消耗
- b）不能有效的利用覆盖索引
- c）使用 SELECT * 容易在增加或者删除字段后出现程序 BUG

**（24）禁止使用 INSERT INTO t_xxx VALUES(xxx)，必须显示指定插入的列属性**

解读：容易在增加或者删除字段后出现程序 BUG

**（25）禁止使用属性隐式转换**

解读：`SELECT uid FROM t_user WHERE phone=13812345678` 会导致全表扫描，而不能命中 phone 索引

**（26）禁止在 WHERE 条件的属性上使用函数或者表达式**

解读：

- 错误：`SELECT uid FROM t_user WHERE from_unixtime(day)>='2017-02-15'` 会导致全表扫描
- 正确：`SELECT uid FROM t_user WHERE day>= unix_timestamp('2017-02-15 00:00:00')`

**（27）禁止负向查询，以及 % 开头的模糊查询**

解读：

- a）负向查询条件：NOT、!=、<>、!<、!>、NOT IN、NOT LIKE 等，会导致全表扫描
- b）% 开头的模糊查询，会导致全表扫描

**（28）禁止大表使用 JOIN 查询，禁止大表使用子查询**

解读：会产生临时表，消耗较多内存与 CPU，极大影响数据库性能

**（29）禁止使用 OR 条件，必须改为 IN 查询**

解读：旧版本 MySQL 的 OR 查询是不能命中索引的，即使能命中索引，为何要让数据库耗费更多的 CPU 帮助实施查询优化呢？

**（30）应用程序必须捕获 SQL 异常，并有相应处理**

### 7.7 总结

大数据量高并发的互联网业务，极大影响数据库性能的都不让用。

## 八、最佳实践

### 8.1 必须遵守

1. 所有表必须包含：`id`, `created_at`, `updated_at`, `deleted_at`
2. 业务表必须包含：`space_id`（多租户隔离）
3. 需要审计的表必须包含：`created_by`, `updated_by`
4. 所有字段必须添加 `COMMENT` 注释
5. 唯一约束必须包含 `deleted_at`（软删除场景）
6. 字段必须定义为 NOT NULL 并提供默认值
7. 禁止使用外键约束（应用层控制）
8. 禁止使用存储过程、视图、触发器

### 8.2 推荐实践

1. 使用 JSON 字段存储动态配置和扩展字段
2. 状态字段使用 `varchar` 而非枚举（便于扩展）
3. 避免使用 `blob`/`text` 类型（影响性能）
4. 时间戳使用 `timestamp` 类型
5. 软删除使用 `bigint` 存储 Unix 时间戳
6. 手机号使用 `varchar(20)` 存储
7. 货币金额使用整数存储（避免精度问题）

### 8.3 性能优化

1. 高频查询字段建立索引
2. 联合索引遵循最左前缀原则
3. 避免过多索引（影响写入性能，单表控制在 5 个以内）
4. 单索引字段数不超过 5 个
5. 禁止在更新频繁、区分度低的字段上建索引
6. 组合索引把区分度高的字段放前面
7. 定期分析和优化慢查询

### 8.4 SQL 编写规范

1. 禁止使用 `SELECT *`，显式指定需要的字段
2. INSERT 语句必须显式指定列名
3. 避免属性隐式转换（如字符串字段用数字查询）
4. 禁止在 WHERE 条件中使用函数
5. 避免负向查询（NOT、!=、<>）和 % 开头的模糊查询
6. 大表禁止使用 JOIN 和子查询
7. 使用 IN 代替 OR 条件
8. 应用程序必须捕获 SQL 异常

## 九、示例：完整表设计

```sql
CREATE TABLE IF NOT EXISTS `dataset`
(
    `id`               bigint unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID',
    `app_id`           int unsigned    NOT NULL DEFAULT '0' COMMENT '应用 ID',
    `space_id`         bigint unsigned NOT NULL DEFAULT '0' COMMENT '空间 ID',
    `schema_id`        bigint unsigned NOT NULL DEFAULT '0' COMMENT 'Schema ID',
    `name`             varchar(255)    NOT NULL DEFAULT '' COMMENT '数据集名称',
    `description`      varchar(2048)   NOT NULL DEFAULT '' COMMENT '数据集描述',
    `category`         varchar(64)     NOT NULL DEFAULT '' COMMENT '业务场景分类',
    `status`           varchar(128)    NOT NULL DEFAULT '' COMMENT '状态',
    `spec`             json                     DEFAULT NULL COMMENT '规格配置',
    `features`         json                     DEFAULT NULL COMMENT '功能开关',
    `latest_version`   varchar(64)     NOT NULL DEFAULT '' COMMENT '最新版本号',
    `next_version_num` bigint unsigned NOT NULL DEFAULT '1' COMMENT '下一个版本的数字版本号',
    `created_by`       varchar(128)    NOT NULL DEFAULT '' COMMENT '创建人',
    `created_at`       timestamp       NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_by`       varchar(128)    NOT NULL DEFAULT '' COMMENT '修改人',
    `updated_at`       timestamp       NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    `deleted_at`       bigint          NOT NULL DEFAULT '0' COMMENT '删除时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_space_id_category_name` (`space_id`, `category`, `name`, `deleted_at`),
    KEY `idx_space_id_category_updated_at_id` (`space_id`, `category`, `updated_at`, `id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_general_ci COMMENT = '数据集表';
```

---

**文档版本**: 2.0
**最后更新**: 2026-03-02
**适用项目**: Node.js + MySQL（高并发大数据量互联网业务）
**参考来源**: 项目现有 SQL 设计 + 58到家数据库30条军规

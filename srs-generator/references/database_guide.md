# 数据库选型指南

本文档为 SRS 文档生成时提供 MySQL、Oracle 11g 和达梦 DM8 三种数据库的详细选型参考和差异对比。

---

## 一、数据库概览对比

| 对比维度 | MySQL 8.0 | Oracle 11g R2 | 达梦 DM8 |
|----------|-----------|---------------|----------|
| 类型 | 开源关系型 | 商业关系型 | 国产信创关系型 |
| 许可证 | GPL / 商业 | 商业授权 | 商业授权 |
| 开发厂商 | Oracle Corp. | Oracle Corp. | 武汉达梦数据库股份有限公司 |
| 信创合规 | 否 | 否 | 是 |
| 等保支持 | 社区版有限 | 原生支持 | 原生支持 |
| 最大库容量 | 无限制 | 无限制 | 无限制 |
| 单表最大 | 64TB | 4M TB | 32TB |
| 编程语言 | SQL/PSM | PL/SQL | DM SQL（兼容 Oracle PL/SQL） |
| 典型用户 | 互联网、中小企业 | 金融、政务、大型企业 | 信创政务、国企、军队 |

---

## 二、字段类型映射表

生成数据模型时，根据所选数据库使用对应的字段类型：

| 业务含义 | MySQL 8.0 | Oracle 11g | 达梦 DM8 |
|----------|-----------|------------|----------|
| 自增主键 | `BIGINT AUTO_INCREMENT` | `NUMBER(19)` + SEQUENCE | `BIGINT IDENTITY` |
| 整数 | `INT` | `NUMBER(10)` | `INT` |
| 长整数 | `BIGINT` | `NUMBER(19)` | `BIGINT` |
| 变长字符串 | `VARCHAR(n)` | `VARCHAR2(n)` | `VARCHAR(n)` 或 `VARCHAR2(n)` |
| 长文本 | `TEXT` / `LONGTEXT` | `CLOB` | `TEXT` / `CLOB` |
| 日期时间 | `DATETIME` | `DATE` (含时间) | `DATETIME` / `TIMESTAMP` |
| 纯日期 | `DATE` | `DATE` (仅日期时用 TRUNC) | `DATE` |
| 布尔值 | `TINYINT(1)` | `NUMBER(1)` | `TINYINT` / `BIT` |
| 浮点数 | `DECIMAL(p,s)` | `NUMBER(p,s)` | `DECIMAL(p,s)` / `NUMBER(p,s)` |
| 二进制 | `BLOB` | `BLOB` | `BLOB` |
| JSON | `JSON` | `CLOB` (存储 JSON 字符串) | `TEXT` (存储 JSON 字符串) |

### 字段类型生成规则

1. 主键 ID 统一使用 `BIGINT`（MySQL 加 `AUTO_INCREMENT`，Oracle 配合 SEQUENCE + TRIGGER，达梦用 `IDENTITY`）
2. 创建时间和更新时间统一使用 `DATETIME`（Oracle 用 `DATE`）
3. 状态/类型等枚举字段统一使用 `VARCHAR(32)`（Oracle 用 `VARCHAR2(32)`）
4. 描述/内容等长文本统一使用 `TEXT`（Oracle 用 `CLOB`）

---

## 三、数据模型设计差异

### 3.1 自增主键策略

**MySQL:**
```sql
CREATE TABLE task_main (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    ...
);
```

**Oracle 11g:**
```sql
-- 创建序列
CREATE SEQUENCE seq_task_main START WITH 1 INCREMENT BY 1;

-- 建表（无 AUTO_INCREMENT）
CREATE TABLE task_main (
    id NUMBER(19) PRIMARY KEY,
    ...
);

-- 触发器自动赋值
CREATE OR REPLACE TRIGGER trg_task_main_id
BEFORE INSERT ON task_main FOR EACH ROW
BEGIN
    SELECT seq_task_main.NEXTVAL INTO :NEW.id FROM DUAL;
END;
```

**达梦 DM8:**
```sql
CREATE TABLE task_main (
    id BIGINT IDENTITY PRIMARY KEY,
    ...
);
```

### 3.2 分页查询差异

| 数据库 | 分页写法 |
|--------|----------|
| MySQL | `SELECT ... LIMIT offset, size` 或 `LIMIT size OFFSET offset` |
| Oracle 11g | `SELECT * FROM (SELECT ROWNUM rn, t.* FROM (...) t WHERE ROWNUM <= end) WHERE rn > start` |
| 达梦 DM8 | 兼容三种：`LIMIT`、Oracle ROWNUM、`OFFSET ... FETCH` |

> 使用 MyBatis-Plus 时，分页插件可自动适配不同数据库方言，无需手写差异 SQL。

### 3.3 索引差异

| 特性 | MySQL | Oracle 11g | 达梦 DM8 |
|------|-------|------------|----------|
| B-Tree 索引 | ✓ | ✓ | ✓ |
| 全文索引 | ✓ (InnoDB 5.6+) | ✓ (Oracle Text) | ✓ |
| 位图索引 | — | ✓ | ✓ |
| 函数索引 | ✓ (8.0.13+) | ✓ | ✓ |
| 索引命名规范 | `idx_表名_字段名` | `IDX_表名_字段名` | `IDX_表名_字段名` |

> SRS 附录中索引建议应使用统一的大写或小写规范，根据所选数据库调整。

---

## 四、连接配置

### MySQL
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mtcp?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
    driver-class-name: com.mysql.cj.jdbc.Driver
```

### Oracle 11g
```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@localhost:1521:orcl
    driver-class-name: oracle.jdbc.OracleDriver
    type: com.alibaba.druid.pool.DruidDataSource
```

> Oracle 11g 使用 `ojdbc6` (JDK 6-8) 或 `ojdbc8` (JDK 8+)，Maven 中央仓库无 Oracle 驱动，需手动安装到本地仓库或使用私服。

### 达梦 DM8
```yaml
spring:
  datasource:
    url: jdbc:dm://localhost:5236/MTCP
    driver-class-name: dm.jdbc.driver.DmDriver
    type: com.alibaba.druid.pool.DruidDataSource
```

> 达梦驱动需从达梦官网下载，Maven 需配置达梦私服或手动安装 JAR。

---

## 五、ORM 方言配置

### MyBatis-Plus 方言配置

```yaml
# MySQL
mybatis-plus:
  global-config:
    db-config:
      id-type: auto

# Oracle 11g
mybatis-plus:
  global-config:
    db-config:
      id-type: input        # Oracle 用序列手动赋值
  configuration:
    database-id: oracle

# 达梦 DM8
mybatis-plus:
  global-config:
    db-config:
      id-type: auto         # 达梦 IDENTITY 支持自动
  configuration:
    database-id: dm
```

### 分页插件通用配置

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    // 分页插件自动识别数据库方言
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
    return interceptor;
}
```

---

## 六、选型决策指南

在 SRS 中帮用户选择合适的数据库时，按以下决策逻辑：

```
用户有指定数据库？
├── 是 → 直接使用指定数据库
└── 否 → 按以下优先级判断：
    ├── 项目有信创合规要求？
    │   └── 是 → 推荐 达梦 DM8
    ├── 项目面向金融/政务/大型国企？
    │   └── 是 → 推荐 Oracle 11g（已有许可证）或 达梦 DM8（信创场景）
    ├── 项目规模大、数据一致性要求极高？
    │   └── 是 → 推荐 Oracle 11g
    └── 通用中小型项目？
        └── 是 → 推荐 MySQL 8.0（默认）
```

### 信创合规说明

当项目涉及信创要求时，应在 SRS 中明确标注：

- 在 **2.3 设计约束** 中添加：「信创合规 — 数据库须选用信创目录内产品」
- 在 **5.3 技术选型** 中说明选型理由：「达梦 DM8 — 满足信创合规要求，兼容 Oracle 语法，降低迁移成本」
- 部分行业（政务、军工、国企）的等保测评明确要求数据库国产化，必须在 SRS 中体现

---

## 七、SRS 中数据库相关章节检查清单

在完成 SRS 文档后，确认以下数据库相关内容已覆盖：

- [ ] **2.2 运行环境** — 数据库软件名称和版本号明确
- [ ] **2.3 设计约束** — 如涉及信创，已添加信创约束
- [ ] **5.3 技术选型表** — 数据库选型行完整（含版本、引擎/模式、说明）
- [ ] **6.1 核心数据表** — 字段类型与所选数据库一致
- [ ] **6.2 实体关系** — 如有自增主键，Oracle 场景已说明 SEQUENCE 方案
- [ ] **附录 — 数据库索引建议** — 索引命名规范与数据库一致
- [ ] **附录 — 定时任务** — 如有存储过程/函数，Oracle 使用 PL/SQL、达梦使用 DM SQL

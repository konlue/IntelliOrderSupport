## intelli-order-support

> 基于 Spring Boot 的外卖点餐平台，涵盖用户端小程序与商家管理后台，完整实现从点餐到配送的全流程业务。

## 📖 项目简介

苍穹外卖是一个前后端分离的外卖点餐系统，分为**用户端**（微信小程序）和**管理端**（Vue3 后台），后端采用 Spring Boot 架构，实现了外卖业务的核心全链路功能。

## 🏗️ 系统架构

```
┌──────────────┐    ┌──────────────┐
│  微信小程序    │    │  管理后台(Vue3)│
│  (用户端)     │    │  (商家端)     │
└──────┬───────┘    └──────┬───────┘
       │                   │
       └─────────┬─────────┘
                 │
          ┌──────▼───────┐
          │   Nginx      │
          │  反向代理/静态 │
          └──────┬───────┘
                 │
          ┌──────▼───────┐
          │ Spring Boot  │
          │   后端服务     │
          └──┬───┬───┬───┘
             │   │   │
        ┌────┘   │   └────┐
        ▼        ▼        ▼
   ┌────────┐ ┌─────┐ ┌────────┐
   │ MySQL  │ │Redis│ │阿里云OSS│
   │ 数据存储 │ │缓存 │ │文件存储  │
   └────────┘ └─────┘ └────────┘
```

## ✨ 核心功能

### 👤 用户端（微信小程序）

| 模块         | 功能                                     |
| ------------ | ---------------------------------------- |
| **登录注册** | 微信授权登录、手机号绑定                 |
| **地址管理** | 收货地址 CRUD、设置默认地址              |
| **浏览点餐** | 分类浏览、菜品搜索、规格选择、加入购物车 |
| **下单支付** | 购物车结算、订单确认、微信支付           |
| **订单管理** | 订单列表、订单详情、订单状态追踪         |
| **历史订单** | 再来一单、订单评价                       |

### 🖥️ 管理端（Vue3 后台）

| 模块         | 功能                                     |
| ------------ | ---------------------------------------- |
| **员工管理** | 员工 CRUD、登录/退出、权限控制（RBAC）   |
| **分类管理** | 菜品/套餐分类的增删改查                  |
| **菜品管理** | 菜品 CRUD、图片上传（OSS）、起售/停售    |
| **套餐管理** | 套餐 CRUD、关联菜品、起售/停售           |
| **订单管理** | 订单列表、订单详情、订单状态变更、取消   |
| **数据统计** | 营业额统计、用户数据、订单数据、销量排行 |
| **来单提醒** | WebSocket 实时推送新订单通知             |

## 🗄️ 数据库设计

```sql
-- 员工表
CREATE TABLE employee (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(32) NOT NULL,
    username    VARCHAR(32) UNIQUE NOT NULL,
    password    VARCHAR(128) NOT NULL,
    phone       VARCHAR(11),
    sex         VARCHAR(2),
    id_number   VARCHAR(18),
    status      INT DEFAULT 1,         -- 1启用 0禁用
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    create_user BIGINT,
    update_user BIGINT
);

-- 分类表（菜品分类 + 套餐分类，通过 type 区分）
CREATE TABLE category (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    type        INT NOT NULL,           -- 1菜品分类 2套餐分类
    name        VARCHAR(32) UNIQUE NOT NULL,
    sort        INT DEFAULT 0,
    status      INT DEFAULT 1,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    create_user BIGINT,
    update_user BIGINT
);

-- 菜品表
CREATE TABLE dish (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(32) UNIQUE NOT NULL,
    category_id BIGINT NOT NULL,
    price       DECIMAL(10,2) NOT NULL,
    image       VARCHAR(255),
    description VARCHAR(255),
    status      INT DEFAULT 1,          -- 1起售 0停售
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    create_user BIGINT,
    update_user BIGINT,
    INDEX idx_category (category_id)
);

-- 菜品口味关系表
CREATE TABLE dish_flavor (
    id      BIGINT PRIMARY KEY AUTO_INCREMENT,
    dish_id BIGINT NOT NULL,
    name    VARCHAR(32),               -- 口味名（如：辣度、甜度）
    value   VARCHAR(255),              -- 口味值（如：["不辣","微辣","中辣"]）
    INDEX idx_dish_id (dish_id)
);

-- 套餐表
CREATE TABLE setmeal (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    category_id BIGINT NOT NULL,
    name        VARCHAR(32) UNIQUE NOT NULL,
    price       DECIMAL(10,2) NOT NULL,
    image       VARCHAR(255),
    description VARCHAR(255),
    status      INT DEFAULT 1,
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    create_user BIGINT,
    update_user BIGINT,
    INDEX idx_category (category_id)
);

-- 套餐菜品关系表
CREATE TABLE setmeal_dish (
    id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    setmeal_id BIGINT NOT NULL,
    dish_id    BIGINT NOT NULL,
    copies     INT DEFAULT 1,           -- 份数
    INDEX idx_setmeal (setmeal_id)
);

-- 用户表
CREATE TABLE user (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    openid      VARCHAR(45) UNIQUE,     -- 微信openid
    name        VARCHAR(32),
    phone       VARCHAR(11),
    sex         VARCHAR(2),
    id_number   VARCHAR(18),
    avatar      VARCHAR(255),
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 地址簿表
CREATE TABLE address_book (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id     BIGINT NOT NULL,
    consignee   VARCHAR(50) NOT NULL,   -- 收货人
    sex         VARCHAR(2),
    phone       VARCHAR(11) NOT NULL,
    province    VARCHAR(32),
    city        VARCHAR(32),
    district    VARCHAR(32),
    detail      VARCHAR(255),           -- 详细地址
    label       VARCHAR(32),
    is_default  TINYINT DEFAULT 0,      -- 1是 0否
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_user (user_id)
);

-- 购物车表
CREATE TABLE shopping_cart (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id     BIGINT NOT NULL,
    dish_id     BIGINT,
    setmeal_id  BIGINT,
    dish_flavor VARCHAR(255),
    number      INT DEFAULT 1,
    amount      DECIMAL(10,2) NOT NULL,
    image       VARCHAR(255),
    name        VARCHAR(50),
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user (user_id)
);

-- 订单表
CREATE TABLE orders (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    number              VARCHAR(50) UNIQUE,     -- 订单号
    status              INT DEFAULT 1,          -- 1待付款 2待接单 3已接单 4派送中 5已完成 6已取消
    user_id             BIGINT NOT NULL,
    address_book_id     BIGINT NOT NULL,
    order_time          DATETIME NOT NULL,
    checkout_time       DATETIME,
    pay_method          INT,                    -- 1微信 2支付宝
    pay_status          TINYINT DEFAULT 0,      -- 0未支付 1已支付
    amount              DECIMAL(10,2) NOT NULL,
    remark              VARCHAR(255),
    phone               VARCHAR(11),
    address             VARCHAR(255),
    consignee           VARCHAR(50),
    cancel_reason       VARCHAR(255),
    rejection_reason    VARCHAR(255),
    delivery_time       DATETIME,
    estimated_delivery_time DATETIME,
    pack_amount         DECIMAL(10,2),          -- 打包费
    tableware_number    INT,
    tableware_status    INT DEFAULT 1,          -- 1按餐量提供 2选择份数
    create_time         DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user (user_id),
    INDEX idx_status (status)
);

-- 订单明细表
CREATE TABLE order_detail (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id        BIGINT NOT NULL,
    dish_id         BIGINT,
    setmeal_id      BIGINT,
    dish_flavor     VARCHAR(255),
    name            VARCHAR(50) NOT NULL,
    image           VARCHAR(255),
    number          INT NOT NULL DEFAULT 1,
    amount          DECIMAL(10,2) NOT NULL,
    INDEX idx_order (order_id)
);
```

## 🔌 接口设计

```yaml
# ===== 用户端 =====
POST   /api/user/login              # 微信登录
GET    /api/user/status              # 用户状态

GET    /api/category/list            # 分类列表
GET    /api/dish/list                # 菜品列表（按分类）
GET    /api/dish/{id}/flavors        # 菜品口味
GET    /api/setmeal/list             # 套餐列表
GET    /api/setmeal/{id}/dish        # 套餐内菜品

POST   /api/shoppingCart             # 添加购物车
GET    /api/shoppingCart/list        # 购物车列表
DELETE /api/shoppingCart/clean       # 清空购物车

POST   /api/addressBook              # 新增地址
GET    /api/addressBook/list        # 地址列表
PUT    /api/addressBook             # 修改地址
DELETE /api/addressBook/{id}        # 删除地址
PUT    /api/addressBook/default/{id} # 设为默认

POST   /api/order/submit             # 提交订单
GET    /api/order/history            # 历史订单
GET    /api/order/{id}               # 订单详情
GET    /api/order/{id}/detail        # 订单明细
POST   /api/order/repetition/{id}    # 再来一单

# ===== 管理端 =====
POST   /api/admin/employee/login     # 员工登录
POST   /api/admin/employee/logout    # 员工退出
GET    /api/admin/employee/page      # 员工分页
POST   /api/admin/employee           # 新增员工
PUT    /api/admin/employee           # 修改员工

GET    /api/admin/category/page      # 分类分页
POST   /api/admin/category           # 新增分类
PUT    /api/admin/category           # 修改分类
DELETE /api/admin/category?id={id}   # 删除分类

POST   /api/admin/dish               # 新增菜品（含口味）
GET    /api/admin/dish/page          # 菜品分页
GET    /api/admin/dish/{id}          # 菜品详情（含口味）
PUT    /api/admin/dish               # 修改菜品
POST   /api/admin/dish/status/{status}  # 起售/停售

POST   /api/admin/setmeal            # 新增套餐
GET    /api/admin/setmeal/page       # 套餐分页
GET    /api/admin/setmeal/{id}       # 套餐详情
PUT    /api/admin/setmeal            # 修改套餐
POST   /api/admin/setmeal/status/{status}  # 起售/停售

GET    /api/admin/order/page         # 订单分页
GET    /api/admin/order/{id}         # 订单详情
PUT    /api/admin/order/confirm      # 接单
PUT    /api/admin/order/reject       # 拒单
PUT    /api/admin/order/delivery     # 派送
PUT    /api/admin/order/complete     # 完成
PUT    /api/admin/order/cancel       # 取消

GET    /api/admin/report/overview    # 营业额统计
GET    /api/admin/report/top10       # 销量 Top10
GET    /api/admin/report/user        # 用户数据
```

## 🛠️ 技术栈

| 层次     | 技术                                |
| -------- | ----------------------------------- |
| 后端框架 | Spring Boot 2.x / 3.x               |
| 持久层   | MyBatis / MyBatis-Plus              |
| 数据库   | MySQL 8.0                           |
| 缓存     | Redis（登录态、菜品缓存、地址缓存） |
| 对象存储 | 阿里云 OSS（菜品/套餐图片）         |
| 微信支付 | 微信支付 API v3                     |
| 实时通信 | WebSocket（来单提醒）               |
| 定时任务 | Spring Task（订单超时自动取消）     |
| 接口文档 | Knife4j (Swagger)                   |
| 前端框架 | Vue 3 + Element Plus                |
| 小程序   | 微信原生小程序                      |

## 📁 项目结构

```
sky-take-out/
├── sky-server/                    # 后端服务
│   ├── controller/
│   │   ├── admin/                 # 管理端接口
│   │   └── user/                  # 用户端接口
│   ├── service/
│   │   ├── impl/                  # 业务实现
│   ├── mapper/                    # MyBatis Mapper
│   ├── entity/                    # 实体类
│   ├── dto/                       # 数据传输对象
│   ├── vo/                        # 视图对象
│   ├── config/                    # 配置类
│   ├── interceptor/               # 拦截器（登录校验）
│   ├── aspect/                    # AOP（自动填充公共字段）
│   ├── websocket/                 # WebSocket 服务
│   └── task/                      # 定时任务
├── sky-common/                    # 公共模块
│   ├── constant/                  # 常量
│   ├── context/                   # ThreadLocal 上下文
│   ├── enumeration/               # 枚举
│   ├── exception/                 # 自定义异常
│   ├── json/                      # Jackson 配置
│   ├── properties/                # 配置属性类
│   └── result/                    # 统一返回结果
├── sky-pojo/                      # 实体模块
│   ├── entity/                    # 数据库实体
│   ├── dto/                       # DTO
│   └── vo/                        # VO
└── sky-admin/                     # 前端管理台
```

## 🚀 快速启动

```bash
# 1. 克隆项目
git clone https://github.com/konlue/intelli-order-support.git

# 2. 初始化数据库
mysql -u root -p < sql/sky.sql

# 3. 修改配置
# application.yml 中配置 MySQL、Redis、OSS 等连接信息

# 4. 启动后端
mvn spring-boot:run -pl sky-server

# 5. 启动前端
cd sky-admin
npm install && npm run dev
```

## 📌 项目亮点

- **AOP 自动填充**：通过自定义注解 + 切面，自动填充 `create_time`、`update_time`、`create_user`、`update_user`，减少重复代码
- **Redis 缓存优化**：菜品数据缓存到 Redis，分类变更时精确清理缓存，提升查询性能
- **WebSocket 来单提醒**：管理端实时接收新订单推送，无需轮询
- **订单超时处理**：Spring Task 定时扫描超时未支付订单，自动取消并释放库存
- **微信支付集成**：完整对接微信支付 API，支持支付通知回调与退款
- **公共字段自动填充**：ThreadLocal 存储当前登录用户 ID，拦截器统一校验登录态

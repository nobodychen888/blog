---
title: 利用 Cloudflare + Telegram 搭建永久免费图床：零成本 Pages/D1/KV 全攻略
tags:
  - Cloudflare
  - 图床
  - Telegram
  - 免费工具
  - 教程
categories:
  - 技术分享
  - 实用工具
keywords: 'Cloudflare图床, 免费图床, Telegram图床, CloudFlare-ImgBed, 搭建教程'
description: 详细教你如何利用 Cloudflare Pages、D1 数据库和 KV 存储，配合 Telegram 机器人搭建一个完全免费、不限容量的个人图床系统。
abbrlink: 5023
date: 2026-05-06 01:57:28
---

今天给大家带来一个非常实用的 GitHub 开源项目，教大家如何利用 Cloudflare 的生态系统搭建一个**完全免费**的个人图床。

该方案的巧妙之处在于：**使用 Cloudflare Pages 部署前端，KV 存储元数据，D1 存储数据库，而真正的图片文件则存储在 Telegram 上**（利用 TG 的无限容量作为免费硬盘）。

## 1. 项目准备

首先，进入本项目地址并 Fork 到你的 GitHub 账号下：
- **项目地址**：[https://github.com/MarSeventh/CloudFlare-ImgBed](https://github.com/MarSeventh/CloudFlare-ImgBed)

## 2. 配置 Cloudflare Pages

1. 登录 Cloudflare 控制台，点击 **Workers 和 Pages** -> **创建应用程序** -> **Pages** -> **连接到 Git**。
2. 选择你刚刚 Fork 的项目 `CloudFlare-ImgBed`。
3. 在“构建设置”中进行如下配置：
   - **框架预设**：`没有 (None)`
   - **构建命令**：`npm install`
   - **构建输出路径**：`frontend-dist`
4. 点击“保存并部署”。

## 3. 配置 KV 数据库

1. 回到 Cloudflare 控制台，选择 **Workers 和 Pages** -> **KV**。
2. 点击“创建命名空间”，名称输入：`img_url`。

## 4. 绑定 KV 到项目

1. 进入你刚刚创建的 Pages 项目设置。
2. 选择 **设置** -> **函数** -> **KV 命名空间绑定**。
3. 点击“添加绑定”：
   - **变量名称**：`img_url` （**必须是这个名称，否则程序无法读取**）
   - **KV 命名空间**：选择刚才创建的 `img_url`。

> **说明**：在这个项目中，KV 仅用来存储图片的元数据（如短链接索引），它并不存储真实的图片文件。我们利用 Telegram 作为后端“硬盘”来存储实际图片。

## 5. 创建 D1 数据库并初始化

1. 在控制台选择 **Workers 和 Pages** -> **D1**。
2. 点击“创建数据库”，名称建议使用：`img_d1`。
3. 数据库位置建议选第一个。
4. 进入该数据库，点击 **控制台**，复制并执行以下 SQL 脚本来创建表结构：

```sql
CREATE TABLE IF NOT EXISTS files (
    id TEXT PRIMARY KEY,
    value TEXT,
    metadata TEXT NOT NULL,
    file_name TEXT,
    file_type TEXT,
    file_size TEXT,
    upload_ip TEXT,
    upload_address TEXT,
    list_type TEXT,
    timestamp INTEGER,
    label TEXT,
    directory TEXT,
    channel TEXT,
    channel_name TEXT,
    tg_file_id TEXT,
    tg_chat_id TEXT,
    tg_bot_token TEXT,
    is_chunked BOOLEAN DEFAULT FALSE,
    tags TEXT, 
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    category TEXT,
    description TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS index_operations (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL,
    timestamp INTEGER NOT NULL,
    data TEXT NOT NULL,
    processed BOOLEAN DEFAULT FALSE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS index_metadata (
    key TEXT PRIMARY KEY,
    last_updated INTEGER,
    total_count INTEGER DEFAULT 0,
    last_operation_id TEXT,
    chunk_count INTEGER DEFAULT 0,
    chunk_size INTEGER DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS other_data (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    type TEXT,
    description TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_files_timestamp ON files(timestamp DESC);
CREATE INDEX IF NOT EXISTS idx_files_directory ON files(directory);
CREATE INDEX IF NOT EXISTS idx_files_channel ON files(channel);
CREATE INDEX IF NOT EXISTS idx_files_file_type ON files(file_type);
CREATE INDEX IF NOT EXISTS idx_files_upload_ip ON files(upload_ip);
CREATE INDEX IF NOT EXISTS idx_files_created_at ON files(created_at DESC);
CREATE INDEX IF NOT EXISTS idx_files_tags ON files(tags);

CREATE INDEX IF NOT EXISTS idx_settings_category ON settings(category);

CREATE INDEX IF NOT EXISTS idx_index_operations_timestamp ON index_operations(timestamp);
CREATE INDEX IF NOT EXISTS idx_index_operations_processed ON index_operations(processed);
CREATE INDEX IF NOT EXISTS idx_index_operations_type ON index_operations(type);

CREATE INDEX IF NOT EXISTS idx_other_data_type ON other_data(type);

CREATE TRIGGER IF NOT EXISTS update_files_updated_at 
    AFTER UPDATE ON files
    BEGIN
        UPDATE files SET updated_at = CURRENT_TIMESTAMP WHERE id = NEW.id;
    END;

CREATE TRIGGER IF NOT EXISTS update_settings_updated_at 
    AFTER UPDATE ON settings
    BEGIN
        UPDATE settings SET updated_at = CURRENT_TIMESTAMP WHERE key = NEW.key;
    END;

CREATE TRIGGER IF NOT EXISTS update_index_metadata_updated_at 
    AFTER UPDATE ON index_metadata
    BEGIN
        UPDATE index_metadata SET updated_at = CURRENT_TIMESTAMP WHERE key = NEW.key;
    END;

CREATE TRIGGER IF NOT EXISTS update_other_data_updated_at 
    AFTER UPDATE ON other_data
    BEGIN
        UPDATE other_data SET updated_at = CURRENT_TIMESTAMP WHERE key = NEW.key;
    END;
	```

## 6. 绑定 D1 数据库

1. 回到 Pages 项目的 **设置** -> **函数**。
2. 找到 **D1 数据库绑定**，点击“添加绑定”：
   - **变量名称**：`img_d1`
   - **D1 数据库**：选择刚才创建的 `img_d1`

## 7. 重新部署并生效

1. 在 Pages 项目中点击 **部署** 选项卡。
2. 在最近的一条部署记录右侧点击三个点 `...`，选择 **重新部署**。
3. 等待部署完成，访问你的 Pages 域名。
4. **安全提醒**：首次登录后，请务必第一时间进入后台修改管理员账号、密码以及上传密码。

## 8. 获取 Telegram 配置

1. 获取 Token：关注机器人 `@BotFather`，根据提示创建一个新机器人，获取 API Token。
2. 获取 Chat ID：关注机器人 `@VersaToolsBot`，向其发送一条消息即可获得你的个人 Chat ID。


> ### 🚀 [DJKK.me - 极致性价比之选](https://djkk.me)
>
> * **超级价格**：低至 **3元/月**
> * **流量充足**：每月 **128GB** 纯净流量。
> * **最佳搭档**：完美支持 ChatGPT、Gemini。
>
> **👉 [立即点击直达（手慢无！）](https://djkk.me)**
- [官方频道](https://t.me/Trumpchina888)
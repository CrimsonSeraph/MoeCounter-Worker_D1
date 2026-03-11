# MoeCounter-Worker_D1

一个基于 Cloudflare Workers、D1 数据库和 KV 存储的极简风格计数器。支持 SVG 图片输出（类似经典的网络计数器）和纯文本 API，可自定义主题、数字长度等。

 > 详细教程请看 [MoeCounter-Worker_D1 是如何使用的](https://personal-web.crimsonseraph.top/post/how-to-use-moe-counter/)

## 特性

- 🖼️ 动态生成 SVG 图片，显示计数数字
- 🔢 支持纯文本 API，方便集成
- 🎨 多主题切换（内置 gelbooru 等）
- 📏 可指定数字长度（自动或固定位数）
- ⚡ 使用 D1 持久化存储计数，KV 缓存生成的 SVG
- 🚀 基于 Hono 框架，轻量快速

## 快速开始

### 前置要求

- [Node.js](https://nodejs.org/) (v16 或更高)
- [pnpm](https://pnpm.io/) 包管理器
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) (已集成在 devDependencies 中)
- 一个 [Cloudflare 账户](https://dash.cloudflare.com/)

### 克隆仓库

```bash
git clone https://github.com/yourusername/MoeCounter-Worker_D1.git
cd MoeCounter-Worker_D1
```

### 安装依赖

```bash
pnpm install
```

## 配置

### 1. 创建 wrangler.toml

复制示例文件并重命名：

```bash
cp wrangler.example.toml wrangler.toml
```

编辑 `wrangler.toml`，至少需要修改以下部分：

```toml
name = "moe-counter"                # 你的 Worker 名称
compatibility_date = "2023-01-01"
main = "dist/index.js"

[build]
command = "pnpm run build"

[[d1_databases]]
binding = "DB"                       # 与环境变量绑定名称，对应代码中的 env.DB
database_name = "moe-counter-db"      # 你的 D1 数据库名称
database_id = "your-database-id"      # 稍后创建的数据库 ID

[[kv_namespaces]]
binding = "KV"                        # 对应代码中的 env.KV
id = "your-kv-namespace-id"           # 稍后创建的 KV 命名空间 ID
```

### 2. 创建 D1 数据库

通过 Wrangler 命令行创建：

```bash
wrangler d1 create moe-counter-db
```

输出类似：

```
✅ Successfully created DB 'moe-counter-db' in region WEUR
Created database 'moe-counter-db' with id: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

将输出的 `database_id` 填入 `wrangler.toml`。

### 3. 创建 KV 命名空间

```bash
wrangler kv:namespace create "KV"
```

输出：

```
✅ Successfully created KV namespace with id: "yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy"
```

将输出的 `id` 填入 `wrangler.toml` 中的 `kv_namespaces`。

### 4. 初始化数据库表

使用 Wrangler 执行 SQL 初始化：

```bash
wrangler d1 execute moe-counter-db --file=./schema.sql
```

如果尚未登录，Wrangler 会引导你完成登录。

### 5. 本地开发与测试

运行开发服务器：

```bash
pnpm run dev
```

访问 `http://localhost:8787` 应看到 "Hello Hono!"。

测试计数器图片：`http://localhost:8787/test?theme=gelbooru&length=7&add=1`

测试 API：`http://localhost:8787/api/test`

## 部署

### 通过 Wrangler 部署

```bash
pnpm run deploy
```

部署成功后，你会获得一个 `https://moe-counter.your-subdomain.workers.dev` 类似的地址。

### 在 Cloudflare Dashboard 上管理

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 进入 **Workers & Pages** 页面，可以看到你刚部署的 Worker。
3. 点击 Worker 名称，可以查看日志、环境变量、触发器等。
4. 进入 **D1** 页面，可以查看数据库、执行 SQL 查询。
5. 进入 **KV** 页面，可以查看存储的键值对（缓存的 SVG 图片）。

## 使用指南

### 图片计数器 URL 格式

```
https://你的worker域名/:name?theme=主题&length=位数&add=0/1&pixelated=pixelated
```

| 参数       | 说明                                                                 | 默认值    | 示例                |
|------------|----------------------------------------------------------------------|-----------|---------------------|
| `:name`    | 计数器名称（路径参数），不同名称独立计数                             | 必填      | `visitors`          |
| `theme`    | 图片主题，目前内置 `gelbooru` 等（见下文）                           | `gelbooru`| `theme=gelbooru`    |
| `length`   | 数字显示位数：`auto` 表示自动（不补零），数字如 `7` 表示固定7位补零  | `7`       | `length=5`          |
| `add`      | 是否增加计数：`0` 表示只读，`1` 表示访问一次加1                      | `1`       | `add=0`             |
| `pixelated`| 如果指定此参数（值任意），SVG 会添加 `style="image-rendering: pixelated"` 使图片更清晰（适合像素风格） | 无 | `pixelated=1`       |

**示例：**

- 基础访问（每次加1）：`https://moe-counter.workers.dev/visitors`
- 只查看当前计数，不加1：`https://moe-counter.workers.dev/visitors?add=0`
- 使用像素风格，5位数字：`https://moe-counter.workers.dev/visitors?theme=gelbooru&length=5&pixelated=1`

### 纯文本 API

```
https://你的worker域名/api/:name?add=0/1
```

返回纯数字文本，适合在代码中调用。

| 参数       | 说明                                         | 默认值 |
|------------|----------------------------------------------|--------|
| `:name`    | 计数器名称                                   | 必填   |
| `add`      | 是否增加计数：`0` 只读，`1` 访问一次加1      | `1`    |

**示例：**

- 获取当前值并加1：`https://moe-counter.workers.dev/api/visitors`
- 仅获取当前值：`https://moe-counter.workers.dev/api/visitors?add=0`

### 主题说明

主题定义在 `src/themes/index.ts` 中（你需要自行创建或查看已有主题）。每个主题包含：

- `width`: 单个数字图片的宽度
- `height`: 高度
- `images`: 一个长度为10的数组，索引0~9分别对应数字0~9的 Data URI（base64 图片）

内置主题示例（以 gelbooru 风格为例），你可以在 `themes` 目录下扩展更多主题。

## 自定义主题

1. 在 `src/themes/` 下新建一个 ts 文件，例如 `mytheme.ts`，导出符合 `Theme` 接口的对象。
2. 在 `src/themes/index.ts` 中导入并添加到 `themes` 对象中。
3. 重新构建部署。

## 工作原理

1. 用户访问图片 URL，Worker 从 URL 参数中解析计数器名称、主题、长度等。
2. 从 D1 数据库查询该计数器的当前值。
3. 如果 `add=1`，异步更新数据库中的计数器值（不阻塞响应）。
4. 生成用于缓存的 KV key，包含版本、主题、长度、像素标志和当前数值。
5. 尝试从 KV 获取已缓存的 SVG：
   - 若命中，直接返回。
   - 若未命中，调用 `generateImage` 生成 SVG，并异步存入 KV（TTL 1小时）。
6. 返回 SVG 响应，并设置适当的 HTTP 缓存头（强制每次验证，实际由 KV 缓存控制）。

## 注意事项

- D1 数据库按读取行数和写入次数计费，KV 按读取、写入和存储空间计费。请参考 Cloudflare 官方定价。
- 默认生成的 SVG 会被 KV 缓存 1 小时（`cacheTtl: 3600`），减少重复生成开销。
- 如果计数器数值变化频繁，KV 缓存可能导致短暂延迟（最多1小时图片不更新）。可通过调整 `cacheTtl` 或移除 KV 缓存解决。
- 由于 D1 最终一致性，刚更新后的计数器可能在极短时间内读取到旧值（但通常无影响）。

## 开发命令

| 命令               | 说明                           |
|--------------------|--------------------------------|
| `pnpm run dev`     | 启动本地开发服务器 (wrangler dev) |
| `pnpm run build`   | 使用 esbuild 构建 Worker 代码    |
| `pnpm run deploy`  | 部署到 Cloudflare Workers        |

## 许可证

MIT

---

如果遇到任何问题，欢迎提交 Issue 或 Pull Request。
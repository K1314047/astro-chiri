---
title: 'Telegram 频道展示 项目完整部署指南'
pubDate: '2025-11-11'
---

## **技术栈**

- **后端**: Node.js + Express
- **前端**: 原生 HTML/CSS/JavaScript
- **代理**: Nginx 反向代理
- **API**: Telegram Bot API
- **部署**: PM2 进程管理

------

## **项目结构**

text

```
telegram-blog/
├── package.json          # 项目依赖配置
├── server.js             # 主服务器文件
├── .env                  # 环境变量配置
└── public/               # 前端静态文件
    ├── index.html        # 主页面
    ├── style.css         # 样式文件
    └── app.js            # 前端逻辑
```

------

## **文件配置详解**

### **1. package.json**

json

```
{
  "name": "telegram-blog",
  "version": "1.0.0",
  "description": "Telegram频道内容展示网站",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "node-cron": "^3.0.2",
    "axios": "^1.5.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

### **2. .env 环境变量配置**

env

```
# 服务器配置
PORT=3000
NODE_ENV=production

# Telegram Bot 配置
BOT_TOKEN=你的Bot_Token
CHANNEL_ID=你的频道ID

# 缓存配置
CACHE_DURATION=300000  # 缓存时间（5分钟）
MAX_POSTS=50          # 最大文章数量
```

**重要配置说明**:

- `BOT_TOKEN`: 在 Telegram 中通过 @BotFather 创建机器人获取
- `CHANNEL_ID`: 将机器人添加为频道管理员后，通过 API 获取（通常是 -100 开头的数字）

### **3. server.js - 主服务器文件**

javascript

```
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const path = require('path');
const axios = require('axios');

const app = express();
const PORT = process.env.PORT || 3000;

// 中间件配置
app.use(cors());
app.use(express.json());
app.use(express.static('public'));

// 缓存机制let cachedPosts = [];
let lastFetchTime = 0;
const CACHE_DURATION = parseInt(process.env.CACHE_DURATION) || 300000;

/**
 * 获取 Telegram 频道内容
 * 支持两种API方式：getChatHistory 和 getUpdates
 */async function fetchTelegramPosts() {
    const BOT_TOKEN = process.env.BOT_TOKEN;
    const CHANNEL_ID = process.env.CHANNEL_ID;

    if (!BOT_TOKEN || !CHANNEL_ID) {
        throw new Error('请配置 BOT_TOKEN 和 CHANNEL_ID 环境变量');
    }

    try {
        console.log('正在从 Telegram 频道获取内容...');

// 方法1: 使用 getChatHistory APItry {
            const url = `https://api.telegram.org/bot${BOT_TOKEN}/getChatHistory`;
            const response = await axios.get(url, {
                params: {
                    chat_id: CHANNEL_ID,
                    limit: 20
                },
                timeout: 10000
            });

            if (response.data.ok && response.data.result) {
                return processBotAPIData(response.data.result);
            }
        } catch (error) {
            console.log('getChatHistory 失败:', error.message);
        }

// 方法2: 回退到 getUpdates APItry {
            const url = `https://api.telegram.org/bot${BOT_TOKEN}/getUpdates`;
            const response = await axios.get(url, {
                timeout: 10000
            });

            if (response.data.ok && response.data.result) {
                return processGetUpdatesData(response.data.result);
            }
        } catch (error) {
            console.log('getUpdates 失败:', error.message);
        }

        throw new Error('所有 API 方法都失败了');

    } catch (error) {
        console.error('获取 Telegram 内容失败:', error.message);
        throw error;
    }
}

/**
 * 处理 getChatHistory 返回的数据
 */function processBotAPIData(messages) {
    const posts = [];

    messages.forEach((item) => {
        const message = item.message || item.channel_post;
        if (!message) return;

        const post = {
            id: message.message_id,
            text: message.text || message.caption || '(无文字内容)',
            time: new Date(message.date * 1000).toLocaleString('zh-CN', {
                year: 'numeric',
                month: '2-digit',
                day: '2-digit',
                hour: '2-digit',
                minute: '2-digit',
                hour12: false
            }),
            image: null
        };

// 处理图片消息if (message.photo && message.photo.length > 0) {
            const photo = message.photo[message.photo.length - 1];
            post.image = `https://api.telegram.org/bot${process.env.BOT_TOKEN}/getFile?file_id=${photo.file_id}`;
        }

// 处理文档消息if (message.document) {
            post.document = message.document.file_name;
        }

        posts.push(post);
    });

    return posts.sort((a, b) => b.id - a.id);
}

/**
 * 处理 getUpdates 返回的数据
 */function processGetUpdatesData(updates) {
    const posts = [];

    updates.forEach((update) => {
        const message = update.channel_post;
        if (!message) return;

        const post = {
            id: message.message_id,
            text: message.text || message.caption || '(无文字内容)',
            time: new Date(message.date * 1000).toLocaleString('zh-CN', {
                year: 'numeric',
                month: '2-digit',
                day: '2-digit',
                hour: '2-digit',
                minute: '2-digit',
                hour12: false
            }),
            image: null
        };

// 标记图片消息if (message.photo && message.photo.length > 0) {
            post.hasImage = true;
            post.image = `图片消息 ID: ${message.message_id}`;
        }

        posts.push(post);
    });

    return posts.sort((a, b) => b.id - a.id);
}

// API 路由 - 获取文章列表
app.get('/api/posts', async (req, res) => {
    try {
        const now = Date.now();

// 检查缓存是否过期if (now - lastFetchTime > CACHE_DURATION || cachedPosts.length === 0) {
            console.log('缓存过期，重新获取数据...');
            cachedPosts = await fetchTelegramPosts();
            lastFetchTime = now;
        }

        res.json({
            ok: true,
            channel: 'Kadriyeblog',
            posts: cachedPosts,
            count: cachedPosts.length,
            cached: lastFetchTime
        });

    } catch (error) {
        console.error('API 错误:', error.message);
        res.status(500).json({
            ok: false,
            error: error.message,
            posts: cachedPosts.length > 0 ? cachedPosts : []
        });
    }
});

// 手动刷新缓存
app.post('/api/refresh', async (req, res) => {
    try {
        cachedPosts = await fetchTelegramPosts();
        lastFetchTime = Date.now();

        res.json({
            ok: true,
            message: '缓存刷新成功',
            postsCount: cachedPosts.length
        });
    } catch (error) {
        res.status(500).json({
            ok: false,
            error: error.message
        });
    }
});

// 健康检查接口
app.get('/api/health', (req, res) => {
    res.json({
        ok: true,
        status: 'running',
        cachedPosts: cachedPosts.length,
        lastFetch: new Date(lastFetchTime).toISOString(),
        environment: process.env.NODE_ENV
    });
});

// 提供前端页面
app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

// 启动服务器
app.listen(PORT, () => {
    console.log(`🚀 Telegram Blog 服务器已启动`);
    console.log(`📍 访问地址: <http://localhost>:${PORT}`);
    console.log(`📱 频道: Kadriyeblog`);
    console.log(`💾 环境: ${process.env.NODE_ENV || 'development'}`);

// 启动时预加载数据fetchTelegramPosts().then(posts => {
        cachedPosts = posts;
        lastFetchTime = Date.now();
        console.log(`✅ 初始数据加载完成，共 ${posts.length} 篇文章`);
    }).catch(error => {
        console.log('❌ 初始数据加载失败:', error.message);
    });
});

// 优雅关闭处理
process.on('SIGINT', () => {
    console.log('\\n👋 正在关闭服务器...');
    process.exit(0);
});
```

### **4. public/index.html - 前端主页面**

html

```
<!DOCTYPE html><html lang="zh-CN" data-theme="dark"><head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"><title>Kadriyeblog - Telegram频道</title><link rel="stylesheet" href="style.css"></head><body><div class="container"><header class="header"><div class="channel-info"><img src="<https://pic1.imgdb.cn/item/68d80e9ec5157e1a883d35de.png>" alt="Kadriyeblog" class="avatar"><div class="channel-meta"><h1>Kadriyeblog</h1><p id="channel-desc">Telegram频道内容聚合</p><div class="stats"><span id="posts-count">加载中...</span><span id="last-update"></span></div></div></div><button id="theme-toggle" class="theme-btn">🌙 深色模式</button></header><div class="content-wrapper"><aside class="sidebar"><div class="toc"><h3>文章目录</h3><ul id="toc-list"></ul></div><div class="actions"><button id="refresh-btn" class="action-btn">🔄 刷新</button><button id="scroll-top" class="action-btn">⬆️ 回顶部</button></div></aside><main class="main-content"><div class="posts-container"><div id="loading" class="loading">正在加载文章...</div><div id="posts-list" class="posts-list"></div></div></main></div></div><script src="app.js"></script></body></html>
```

### **5. public/style.css - 样式文件**

css

```
:root {
    --bg-primary: #1a1a1a;
    --bg-secondary: #2d2d2d;
    --bg-card: #363636;
    --text-primary: #ffffff;
    --text-secondary: #b0b0b0;
    --accent-color: #007acc;
    --border-color: #404040;
    --success-color: #4CAF50;
    --error-color: #f44336;
}

[data-theme="light"] {
    --bg-primary: #ffffff;
    --bg-secondary: #f5f5f5;
    --bg-card: #ffffff;
    --text-primary: #333333;
    --text-secondary: #666666;
    --accent-color: #007acc;
    --border-color: #e0e0e0;
}

* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background-color: var(--bg-primary);
    color: var(--text-primary);
    line-height: 1.6;
    transition: all 0.3s ease;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 20px;
}

/* 头部样式 */.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 20px 0;
    border-bottom: 1px solid var(--border-color);
    margin-bottom: 30px;
}

.channel-info {
    display: flex;
    align-items: center;
    gap: 15px;
}

.avatar {
    width: 80px;
    height: 80px;
    border-radius: 50%;
    border: 3px solid var(--accent-color);
}

.channel-meta h1 {
    font-size: 24px;
    margin-bottom: 5px;
}

.channel-meta p {
    color: var(--text-secondary);
    margin-bottom: 8px;
}

.stats {
    display: flex;
    gap: 15px;
    font-size: 14px;
    color: var(--text-secondary);
}

.theme-btn {
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    color: var(--text-primary);
    padding: 10px 15px;
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.3s ease;
}

.theme-btn:hover {
    background: var(--accent-color);
}

/* 内容布局 */.content-wrapper {
    display: grid;
    grid-template-columns: 280px 1fr;
    gap: 30px;
    min-height: 70vh;
}

/* 侧边栏 */.sidebar {
    background: var(--bg-secondary);
    padding: 20px;
    border-radius: 12px;
    height: fit-content;
    position: sticky;
    top: 20px;
}

.toc h3 {
    margin-bottom: 15px;
    font-size: 16px;
    color: var(--text-primary);
}

#toc-list {
    list-style: none;
    max-height: 400px;
    overflow-y: auto;
}

#toc-list li {
    margin-bottom: 8px;
}

#toc-list a {
    color: var(--text-secondary);
    text-decoration: none;
    font-size: 14px;
    padding: 5px 0;
    display: block;
    transition: color 0.3s ease;
}

#toc-list a:hover {
    color: var(--accent-color);
}

.actions {
    margin-top: 20px;
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.action-btn {
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    color: var(--text-primary);
    padding: 10px;
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.3s ease;
    font-size: 14px;
}

.action-btn:hover {
    background: var(--accent-color);
}

/* 文章列表 */.posts-list {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.post-card {
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    border-radius: 12px;
    padding: 20px;
    transition: all 0.3s ease;
}

.post-card:hover {
    border-color: var(--accent-color);
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.post-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 12px;
}

.post-time {
    color: var(--text-secondary);
    font-size: 14px;
}

.post-id {
    background: var(--accent-color);
    color: white;
    padding: 2px 8px;
    border-radius: 12px;
    font-size: 12px;
}

.post-content {
    margin-bottom: 15px;
}

.post-text {
    white-space: pre-line;
    line-height: 1.6;
}

.post-image {
    margin-top: 15px;
}

.post-image img {
    max-width: 100%;
    border-radius: 8px;
    border: 1px solid var(--border-color);
}

/* 加载状态 */.loading {
    text-align: center;
    padding: 40px;
    color: var(--text-secondary);
    font-size: 16px;
}

.error {
    background: var(--error-color);
    color: white;
    padding: 20px;
    border-radius: 8px;
    text-align: center;
}

.success {
    background: var(--success-color);
    color: white;
    padding: 10px;
    border-radius: 6px;
    margin-bottom: 15px;
    text-align: center;
}

/* 响应式设计 */@media (max-width: 768px) {
    .content-wrapper {
        grid-template-columns: 1fr;
    }

    .sidebar {
        position: static;
        order: 2;
    }

    .main-content {
        order: 1;
    }

    .header {
        flex-direction: column;
        gap: 15px;
        text-align: center;
    }

    .channel-info {
        flex-direction: column;
        text-align: center;
    }
}
```

### **6. public/app.js - 前端逻辑**

javascript

```
class TelegramBlog {
    constructor() {
        this.API_BASE = '/api/posts';
        this.init();
    }

    init() {
        this.initTheme();
        this.bindEvents();
        this.loadPosts();
    }

// 初始化主题initTheme() {
        const savedTheme = localStorage.getItem('theme') || 'dark';
        document.documentElement.setAttribute('data-theme', savedTheme);
        this.updateThemeButton(savedTheme);
    }

// 更新主题按钮updateThemeButton(theme) {
        const btn = document.getElementById('theme-toggle');
        btn.textContent = theme === 'dark' ? '☀️ 浅色模式' : '🌙 深色模式';
    }

// 绑定事件bindEvents() {
// 主题切换
        document.getElementById('theme-toggle').addEventListener('click', () => {
            const currentTheme = document.documentElement.getAttribute('data-theme');
            const newTheme = currentTheme === 'dark' ? 'light' : 'dark';
            document.documentElement.setAttribute('data-theme', newTheme);
            localStorage.setItem('theme', newTheme);
            this.updateThemeButton(newTheme);
        });

// 刷新按钮
        document.getElementById('refresh-btn').addEventListener('click', () => {
            this.refreshPosts();
        });

// 回顶部按钮
        document.getElementById('scroll-top').addEventListener('click', () => {
            window.scrollTo({ top: 0, behavior: 'smooth' });
        });
    }

// 加载文章async loadPosts() {
        const loading = document.getElementById('loading');
        const postsList = document.getElementById('posts-list');

        try {
            loading.style.display = 'block';
            postsList.innerHTML = '';

            const response = await fetch(this.API_BASE);
            const data = await response.json();

            if (data.ok) {
                this.renderPosts(data.posts);
                this.updateStats(data);
                this.showMessage('数据加载成功', 'success');
            } else {
                throw new Error(data.error || '加载失败');
            }
        } catch (error) {
            this.showError(error.message);
        } finally {
            loading.style.display = 'none';
        }
    }

// 刷新文章async refreshPosts() {
        try {
            const response = await fetch('/api/refresh', { method: 'POST' });
            const data = await response.json();

            if (data.ok) {
                this.showMessage('数据刷新成功', 'success');
                setTimeout(() => this.loadPosts(), 500);
            } else {
                throw new Error(data.error || '刷新失败');
            }
        } catch (error) {
            this.showError(error.message);
        }
    }

// 渲染文章列表renderPosts(posts) {
        const postsList = document.getElementById('posts-list');
        const tocList = document.getElementById('toc-list');

        if (posts.length === 0) {
            postsList.innerHTML = '<div class="error">暂无文章内容</div>';
            tocList.innerHTML = '<li>暂无目录</li>';
            return;
        }

// 渲染文章
        postsList.innerHTML = posts.map(post => `
            <article class="post-card" id="post-${post.id}">
                <div class="post-header">
                    <span class="post-time">${post.time}</span>
                    <span class="post-id">#${post.id}</span>
                </div>
                <div class="post-content">
                    <div class="post-text">${this.escapeHtml(post.text)}</div>
                    ${post.image ? `
                        <div class="post-image">
                            <img src="${post.image}" alt="文章图片" onerror="this.style.display='none'">
                        </div>
                    ` : ''}
                </div>
            </article>
        `).join('');

// 渲染目录
        tocList.innerHTML = posts.map(post => `
            <li>
                <a href="#post-${post.id}" title="${post.text.substring(0, 50)}...">
                    ${post.time.split(' ')[0]} #${post.id}
                </a>
            </li>
        `).join('');
    }

// 更新统计信息updateStats(data) {
        document.getElementById('posts-count').textContent = `共 ${data.count} 篇文章`;
        document.getElementById('last-update').textContent = `最后更新: ${new Date().toLocaleTimeString()}`;
    }

// 显示消息showMessage(message, type = 'success') {
        const existingMsg = document.querySelector('.message');
        if (existingMsg) existingMsg.remove();

        const msg = document.createElement('div');
        msg.className = `message ${type}`;
        msg.textContent = message;

        document.querySelector('.main-content').prepend(msg);
        setTimeout(() => msg.remove(), 3000);
    }

// 显示错误showError(message) {
        const postsList = document.getElementById('posts-list');
        postsList.innerHTML = `
            <div class="error">
                <h3>加载失败</h3>
                <p>${message}</p>
                <button onclick="blog.loadPosts()" class="action-btn" style="margin-top: 10px;">重试</button>
            </div>
        `;
    }

// HTML 转义escapeHtml(unsafe) {
        return unsafe
            .replace(/&/g, "&amp;")
            .replace(/</g, "&lt;")
            .replace(/>/g, "&gt;")
            .replace(/"/g, "&quot;")
            .replace(/'/g, "&#039;")
            .replace(/\\n/g, '<br>');
    }
}

// 初始化应用const blog = new TelegramBlog();
```

------

## **完整部署步骤**

### **1. 服务器环境准备**

bash

```
# 更新系统sudo apt update && sudo apt upgrade -y

# 安装 Node.js（如果未安装）curl -fsSL <https://deb.nodesource.com/setup_18.x> | sudo -E bash -
sudo apt-get install -y nodejs

# 验证安装node --version
npm --version
```

### **2. 项目部署**

bash

```
# 创建项目目录mkdir telegram-blog
cd telegram-blog

# 创建 package.json 文件（复制上面的内容）nano package.json

# 安装依赖npm install

# 创建环境变量文件nano .env

# 创建项目结构mkdir public

# 创建服务器文件nano server.js

# 创建前端文件nano public/index.html
nano public/style.css
nano public/app.js
```

### **3. 配置 Nginx 反向代理**

bash

```
# 安装 Nginxsudo apt install nginx -y

# 创建 Nginx 配置文件sudo nano /etc/nginx/sites-available/telegram-blog
```

配置文件内容：

nginx

```
server {
    listen 80;
    server_name your-domain.com;# 替换为你的域名location / {
        proxy_pass <http://localhost:3001>;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;

# 超时设置proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

# 静态文件缓存location ~* \\.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

启用配置：

bash

```
# 启用网站配置sudo ln -s /etc/nginx/sites-available/telegram-blog /etc/nginx/sites-enabled/

# 测试配置sudo nginx -t

# 重启 Nginxsudo systemctl restart nginx
```

### **4. 使用 PM2 管理进程**

bash

```
# 安装 PM2npm install -g pm2

# 启动应用
pm2 start server.js --name telegram-blog

# 设置开机自启
pm2 startup
pm2 save

# 查看状态
pm2 status
pm2 logs telegram-blog
```

### **5. 防火墙配置**

bash

```
# 启用防火墙sudo ufw enable

# 放行端口sudo ufw allow 80
sudo ufw allow 22
sudo ufw allow 3001

# 检查状态sudo ufw status
```

------

## **功能特性**

✅ **自动缓存机制** - 5分钟缓存减少 API 调用

✅ **双主题支持** - 暗黑/浅色模式一键切换

✅ **响应式设计** - 完美支持桌面和移动端

✅ **实时刷新** - 手动刷新获取最新内容

✅ **错误处理** - 完善的错误提示和重试机制

✅ **健康检查** - 内置服务状态监控接口

✅ **性能优化** - 静态资源缓存和压缩

------

## **故障排除**

### **常见问题解决**

1. **端口占用问题**

bash

```
# 检查端口占用sudo lsof -i :3001
# 杀死占用进程sudo kill -9 <PID>
```

1. **Nginx 配置错误**

bash

```
# 测试配置sudo nginx -t
# 查看错误日志sudo tail -f /var/log/nginx/error.log
```

1. **应用启动失败**

bash

```
# 检查 PM2 日志
pm2 logs telegram-blog
# 重启应用
pm2 restart telegram-blog
```

1. **API 获取失败**

- 检查 Bot Token 和 Channel ID 是否正确
- 确认机器人已添加为频道管理员
- 验证频道是否为公开频道

------

## **维护命令**

bash

```
# 查看应用状态
pm2 status

# 查看实时日志
pm2 logs telegram-blog

# 重启应用
pm2 restart telegram-blog

# 停止应用
pm2 stop telegram-blog

# 重新加载 Nginxsudo systemctl reload nginx

# 检查服务状态sudo systemctl status nginx
```


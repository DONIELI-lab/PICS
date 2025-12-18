# ContentCreator 网站部署指南

本指南将详细介绍如何将 ContentCreator 项目部署到各种环境中。

## 目录

1. [环境准备](#环境准备)
2. [项目构建](#项目构建)
3. [部署选项](#部署选项)
   - [静态托管服务](#静态托管服务)
   - [VPS/云服务器](#vps云服务器)
   - [Docker 容器化部署](#docker-容器化部署)
4. [配置与优化](#配置与优化)
5. [常见问题与解决方案](#常见问题与解决方案)

## 环境准备

在部署之前，请确保您的环境满足以下要求：

1. **Node.js 环境**
   - 推荐版本: 18.x 或更高
   - 检查方式: `node -v`

2. **包管理器**
   - 本项目使用 pnpm: `npm install -g pnpm`
   - 检查方式: `pnpm -v`

3. **Git**
   - 用于克隆项目和管理代码
   - 检查方式: `git --version`

## 项目构建

1. **克隆项目代码**
   ```bash
   git clone <项目仓库地址>
   cd ContentCreator
   ```

2. **安装依赖**
   ```bash
   pnpm install
   ```

3. **构建生产版本**
   ```bash
   pnpm build
   ```
   
   构建完成后，项目会在 `dist` 目录下生成静态文件。

4. **验证构建结果**
   ```bash
   pnpm preview
   ```
   
   此命令将启动一个本地服务器，用于预览生产版本的网站。您可以通过 `http://localhost:4173` 访问。

## 部署选项

### 静态托管服务

ContentCreator 是一个纯静态网站，适合部署到各种静态托管服务上。以下是几个常用平台的部署步骤：

#### Vercel

1. **访问 [Vercel](https://vercel.com/) 并登录**
2. **点击 "New Project"**
3. **导入您的 Git 仓库**
4. **配置项目设置**
   - 构建命令: `pnpm build`
   - 输出目录: `dist`
5. **点击 "Deploy" 开始部署**
6. **部署完成后，您将获得一个可访问的 URL**

#### Netlify

1. **访问 [Netlify](https://www.netlify.com/) 并登录**
2. **点击 "Add new site" -> "Import an existing project"**
3. **连接您的 Git 仓库**
4. **配置构建设置**
   - 构建命令: `pnpm build`
   - 发布目录: `dist`
5. **点击 "Deploy site"**
6. **部署完成后，您将获得一个随机生成的 URL，也可以配置自定义域名**

#### GitHub Pages

1. **在项目根目录创建 `.github/workflows/deploy.yml` 文件**
2. **添加以下内容**:
   ```yaml
   name: Deploy to GitHub Pages

   on:
     push:
       branches: [ main ]  # 或者您的主分支名称
     pull_request:
       branches: [ main ]  # 用于PR预览

   # 允许从fork的仓库中运行此工作流，同时保留写访问权限
   permissions:
     contents: write
     pages: write
     id-token: write

   # 环境配置
   environment:
     name: github-pages
     url: ${{ steps.deployment.outputs.page_url }}

   jobs:
     build-and-deploy:
       runs-on: ubuntu-latest
       steps:
         # 检出代码
         - name: Checkout
           uses: actions/checkout@v4
           with:
             fetch-depth: 0  # 获取完整的提交历史

         # 设置 Node.js 环境
         - name: Setup Node.js
           uses: actions/setup-node@v4
           with:
             node-version: '18'  # 使用 Node.js 18
             cache: 'pnpm'       # 缓存 pnpm 依赖

         # 安装 pnpm
         - name: Install pnpm
           uses: pnpm/action-setup@v2
           with:
             version: latest  # 使用最新版本的 pnpm

         # 安装依赖并构建项目
         - name: Install dependencies and build
           run: |
             pnpm install --frozen-lockfile  # 确保依赖版本一致
             pnpm build                      # 构建项目

         # 可选：构建结果检查
         - name: Verify build output
           run: |
             if [ ! -d "dist" ]; then
               echo "构建失败：dist 目录不存在"
               exit 1
             fi
             echo "构建成功，dist 目录包含 $(ls -la dist | wc -l) 个文件"

         # 部署到 GitHub Pages
         - name: Deploy to GitHub Pages
           uses: JamesIves/github-pages-deploy-action@v4
           with:
             folder: dist            # 构建输出目录
             branch: gh-pages        # 部署到的分支
             commit-message: 'Deploying to GitHub Pages'  # 自定义提交消息
             clean: true             # 清理目标分支上的旧文件
             single-commit: true     # 只创建一个提交

         # 输出部署状态
         - name: Output deployment status
           if: success()
           run: echo "✅ 部署成功！访问地址: ${{ steps.deployment.outputs.page_url }}"
   ```
3. **提交并推送更改到 GitHub**
4. **在仓库设置中，启用 GitHub Pages 并选择 `gh-pages` 分支**

### VPS/云服务器

如果您希望部署到自己的服务器，可以按照以下步骤操作：

1. **准备服务器环境**
   - 安装 Node.js 和 Nginx
   - 配置防火墙，允许 80 和 443 端口

2. **上传构建文件**
   ```bash
   # 使用 scp 命令上传文件
   scp -r dist/* user@your-server-ip:/var/www/contentcreator
   ```

3. **配置 Nginx**
   ```bash
   # 创建 Nginx 配置文件
   sudo nano /etc/nginx/sites-available/contentcreator
   ```
   
   添加以下内容：
   ```nginx
   server {
       listen 80;
       server_name your-domain.com;  # 替换为您的域名

       root /var/www/contentcreator;
       index index.html;

       location / {
           try_files $uri $uri/ /index.html;
       }
   }
   ```

4. **启用配置并重启 Nginx**
   ```bash
   sudo ln -s /etc/nginx/sites-available/contentcreator /etc/nginx/sites-enabled/
   sudo nginx -t  # 检查配置是否正确
   sudo systemctl restart nginx
   ```

5. **(可选) 配置 HTTPS**
   - 使用 Certbot 申请免费的 SSL 证书
   ```bash
   sudo apt install certbot python3-certbot-nginx
   sudo certbot --nginx -d your-domain.com
   ```

### Docker 容器化部署

使用 Docker 可以简化部署过程并确保环境一致性：

1. **在项目根目录创建 `Dockerfile`**
   ```dockerfile
   # 使用 Node.js 官方镜像作为构建环境
   FROM node:18-alpine AS build

   # 设置工作目录
   WORKDIR /app

   # 复制 package.json 和 pnpm-lock.yaml
   COPY package.json pnpm-lock.yaml ./

   # 安装 pnpm
   RUN npm install -g pnpm

   # 安装依赖
   RUN pnpm install

   # 复制项目文件
   COPY . .

   # 构建项目
   RUN pnpm build

   # 使用 Nginx 作为生产服务器
   FROM nginx:alpine

   # 复制构建结果到 Nginx
   COPY --from=build /app/dist /usr/share/nginx/html

   # 复制自定义 Nginx 配置
   COPY nginx.conf /etc/nginx/conf.d/default.conf

   # 暴露 80 端口
   EXPOSE 80

   # 启动 Nginx
   CMD ["nginx", "-g", "daemon off;"]
   ```

2. **创建 `nginx.conf` 文件**
   ```nginx
   server {
       listen 80;
       server_name localhost;

       location / {
           root /usr/share/nginx/html;
           index index.html index.htm;
           try_files $uri $uri/ /index.html;
       }

       error_page 500 502 503 504 /50x.html;
       location = /50x.html {
           root /usr/share/nginx/html;
       }
   }
   ```

3. **构建和运行 Docker 容器**
   ```bash
   # 构建 Docker 镜像
   docker build -t contentcreator .

   # 运行 Docker 容器
   docker run -p 80:80 contentcreator
   ```

4. **(可选) 使用 Docker Compose**
   创建 `docker-compose.yml` 文件：
   ```yaml
   version: '3'
   services:
     contentcreator:
       build: .
       ports:
         - "80:80"
       restart: always
   ```
   
   然后运行：
   ```bash
   docker-compose up -d
   ```

## 配置与优化

### API 配置

要使用真实的AI功能，您需要配置API密钥：

1. 在应用中点击"配置API"按钮
2. 输入您的API密钥（如OpenAI、Anthropic或Google的API密钥）
3. 选择您要使用的AI模型
4. 保存配置

配置会保存在浏览器的本地存储中，下次打开应用时会自动加载。

### 环境变量

本项目支持以下环境变量，您可以在构建时设置：

```bash
# API 基础 URL (如果需要连接自定义后端服务)
VITE_API_BASE_URL=https://api.yourdomain.com

# 调试模式
VITE_DEBUG=false
```

### 性能优化

1. **图片优化**
   - 使用适当尺寸的图片
   - 压缩图片以减小文件大小
   - 考虑使用 WebP 格式图片

2. **缓存策略**
   - 配置浏览器缓存
   - 利用 CDN 分发静态资源

3. **代码分割**
   - 虽然当前项目结构简单，但随着项目增长，可以考虑实现代码分割

## 常见问题与解决方案

### 路由问题

**问题**: 在使用客户端路由时，直接访问子路由会返回 404 错误。

**解决方案**: 确保服务器配置了回退路由，将所有请求指向 `index.html`。

### 静态资源加载失败

**问题**: 部署后，CSS、JS 等静态资源无法加载。

**解决方案**:
- 检查资源路径是否正确
- 确保构建时设置了正确的 `base` 路径
- 检查服务器权限设置

### 构建失败

**问题**: 在 CI/CD 环境中构建失败。

**解决方案**:
- 确保 CI 环境安装了 pnpm
- 检查 Node.js 版本是否兼容
- 清理缓存后重新构建

---

希望本指南能帮助您成功部署 ContentCreator 项目！如有其他问题，请随时咨询。
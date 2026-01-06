# Mini 照片社区 - 技术架构文档

> 版本: 1.0
> 更新日期: 2026-01-06

## 技术栈概览

| 层面 | 技术选型 | 版本 |
|------|----------|------|
| 前端框架 | Next.js (App Router) | 16.x |
| UI 组件 | shadcn/ui + Base UI | Base UI v1.0.0 |
| 样式方案 | Tailwind CSS | 4.x |
| 后端服务 | Supabase | - |
| 数据库 | PostgreSQL (Supabase) | - |
| 图片存储 | Supabase Storage | MVP 阶段 |
| EXIF 解析 | exifr | - |
| 地理编码 | 腾讯地图 LBS | 逆地理编码 API |
| 部署平台 | Vercel | - |

---

## 架构决策记录

### 1. 整体架构: Next.js + Supabase

**决策**: 采用 Next.js 16 + Supabase 的全栈方案

**对比方案**:

| 维度 | Next.js + Supabase | Next.js + PocketBase | Next.js + 自建后端 |
|------|-------------------|----------------------|-------------------|
| 数据库 | PostgreSQL (托管) | SQLite (自托管) | PostgreSQL/MySQL |
| 横向扩展 | 支持 | 不支持 | 支持 |
| 免费额度 | 500MB DB, 1GB 存储 | 完全免费 | 需分别付费 |
| 部署复杂度 | 低 | 中 | 高 |

**选择理由**:
1. Supabase 免费层足够 MVP 验证（500MB 数据库、1GB 存储、5GB 出站）
2. PostgreSQL 原生支持地理数据查询 (PostGIS)
3. 开源方案，数据可控，未来可自托管
4. PocketBase 不支持横向扩展，用户增长后迁移成本高

**参考来源**: [Supabase vs Firebase vs PocketBase 2025](https://www.supadex.app/blog/supabase-vs-firebase-vs-pocketbase-which-one-should-you-choose-in-2025)

---

### 2. 前端框架: Next.js 16

**决策**: 使用 Next.js 16.x (App Router)

**版本信息** (截至 2025.12):
- Next.js 16.1 (2025.12.18) - 最新稳定版
- Next.js 16 (2025.10.21) - Cache Components、Turbopack 稳定版

**关键特性**:
- Turbopack 稳定版: 启动和热更新性能大幅提升
- Cache Components: 新的缓存编程模型
- React Compiler 支持
- React 19 原生支持

**参考来源**: [Next.js 官方博客](https://nextjs.org/blog)

---

### 3. UI 组件库: shadcn/ui + Base UI

**决策**: 使用 shadcn/ui，底层选择 Base UI（而非 Radix UI）

**重要背景**:
- Radix UI 已不再积极维护，原始创建者称其为 "liability"
- Base UI 于 2025.12 发布 v1.0.0 稳定版
- Base UI 由 Radix、Material UI、Floating UI 创建者联合打造
- shadcn/ui 官方已支持初始化时选择 Base UI

**对比**:

| 维度 | Base UI | Radix UI |
|------|---------|----------|
| 维护状态 | 活跃 | 不再积极维护 |
| 稳定版本 | v1.0.0 (2025.12) | - |
| 组件覆盖 | 更完善 (含 multi-select 等) | 基础 |
| 长期支持 | MUI 团队保障 | 不确定 |

**参考来源**:
- [Radix UI 维护状态分析](https://dev.to/mashuktamim/is-your-shadcn-ui-project-at-risk-a-deep-dive-into-radixs-future-45ei)
- [Base UI vs Radix UI](https://preblocks.com/blog/radix-ui-vs-base-ui)

---

### 4. 图片存储: Supabase Storage → Cloudflare R2

**决策**: MVP 阶段使用 Supabase Storage，用户增长后迁移至 Cloudflare R2

**成本对比** (1TB 存储 + 20TB 月流量):

| 服务 | 成本 |
|------|------|
| Supabase Storage | ~$50+ |
| Cloudflare R2 | **$15** |
| AWS S3 | ~$1,723 |

**R2 优势**:
- 出站流量完全免费
- 全球 300+ CDN 节点
- S3 兼容，迁移简单

**参考来源**: [Cloudflare R2 vs S3](https://www.vantage.sh/blog/cloudflare-r2-aws-s3-comparison)

---

### 5. EXIF 解析: exifr

**决策**: 前端使用 exifr 库进行 EXIF 解析

**对比**:

| 维度 | exifr | ExifReader |
|------|-------|------------|
| HEIC 性能 | 快 30 倍 | 良好 |
| 包体积 | ~15KB | ~4KB (最小配置) |
| 解析方式 | 智能识别文件结构 | 标准解析 |

**选择理由**:
1. HEIC 解析快 30 倍（iPhone 用户常用格式）
2. 智能读取文件结构，不暴力扫描全部字节
3. API 简洁易用

**参考来源**: [exifr GitHub](https://github.com/MikeKovarik/exifr)

---

### 6. 地理编码: 腾讯地图 LBS

**决策**: 使用腾讯地图逆地理编码 API

**国内三大地图 API 对比**:

| 维度 | 高德地图 | 腾讯地图 | 百度地图 |
|------|----------|----------|----------|
| 逆地理编码配额 | 30万次/日 | **300万次/日** | 300万次/日 |
| 坐标系 | GCJ-02 | GCJ-02 | BD-09 |
| 商用授权 | 5万/年起 | 5万/年起 | 5万/年起 |

**选择理由**:
1. 配额充足（300万次/日，高德仅 30万次/日）
2. 微信生态整合优势
3. GCJ-02 坐标系与 WGS84 转换方便

**坐标系转换流程**:
```
照片 EXIF (WGS84) → 转换为 GCJ-02 → 腾讯逆地理编码 API → 可读地址
```

**参考来源**: [2025年国内地图API排名](https://www.explinks.com/blog/pr-2025-domestic-map-api-rankings/)

---

## 系统架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        客户端 (浏览器)                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Next.js    │  │   exifr     │  │  Tailwind + shadcn  │  │
│  │  App Router │  │  EXIF 解析   │  │     UI 组件         │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        Vercel (部署)                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Next.js Server (SSR/API)               │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│    Supabase     │ │    Supabase     │ │   腾讯地图 LBS   │
│   PostgreSQL    │ │    Storage      │ │   逆地理编码     │
│   + Auth        │ │   (→ R2)        │ │                 │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## 数据模型设计 (初版)

### users 表
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  username TEXT UNIQUE,
  avatar_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### photos 表
```sql
CREATE TABLE photos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  image_url TEXT NOT NULL,
  thumbnail_url TEXT,

  -- EXIF 数据
  taken_at TIMESTAMPTZ,          -- 拍摄时间
  latitude DOUBLE PRECISION,      -- GPS 纬度 (GCJ-02)
  longitude DOUBLE PRECISION,     -- GPS 经度 (GCJ-02)
  location_name TEXT,             -- 逆地理编码结果

  -- 元数据
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 地理位置索引 (用于地图功能)
CREATE INDEX idx_photos_location ON photos (latitude, longitude);

-- 时间索引 (用于时间线)
CREATE INDEX idx_photos_taken_at ON photos (taken_at DESC);
```

---

## Supabase 免费层限制

| 资源 | 限制 |
|------|------|
| 数据库大小 | 500 MB |
| 文件存储 | 1 GB |
| 出站带宽 | 5 GB |
| 月活用户 (MAU) | 50,000 |
| 活跃项目数 | 2 个 |
| 项目休眠 | 1 周无活动后暂停 |

**参考来源**: [Supabase Pricing](https://supabase.com/pricing)

---

## 后续迭代规划

### MVP 后优化项
1. 图片存储迁移至 Cloudflare R2（降低流量成本）
2. 引入图片压缩/缩略图生成（Cloudflare Images 或自建）
3. 考虑 PostGIS 扩展支持更复杂的地理查询

### 第二期功能技术准备
- 点赞/评论: Supabase Realtime 订阅
- 消息通知: Supabase Edge Functions + 推送服务

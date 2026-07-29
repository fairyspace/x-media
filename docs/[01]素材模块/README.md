# [01] 素材模块设计稿

Created: 2026-07-29
Status: Draft

---

## 1. 用途

素材模块负责管理所有媒体资产，包括：

- 用户主动上传的图片/视频/音频
- AI 生成结果回写的素材
- 画布引用的参考图、角色卡、首尾帧
- 素材的元数据、标签、分享与生命周期

它是画布模块和 AI 网关模块的资产中台。

---

## 2. 最重要 3 点

### 2.1 数据结构设计

核心表设计：

`sql
CREATE TABLE pms_asset (
  id               BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '主键',
  entity_id        BIGINT UNSIGNED  NOT NULL COMMENT '法律实体ID',
  user_id          BIGINT UNSIGNED  NOT NULL COMMENT '拥有者用户ID',
  project_id       BIGINT UNSIGNED      NULL COMMENT '所属项目ID',
  	ype             VARCHAR(32)      NOT NULL COMMENT '素材类型：image/video/audio',
  oss_key          VARCHAR(512)     NOT NULL COMMENT '对象存储Key',
  ucket           VARCHAR(128)         NULL COMMENT '存储桶',
  content_type     VARCHAR(128)         NULL COMMENT 'MIME类型',
  size             BIGINT UNSIGNED      NULL COMMENT '文件大小(字节)',
  digest           VARCHAR(64)          NULL COMMENT '文件摘要',
  status           TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '1:正常 2:归档 3:删除',
  created_at       DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at       DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_entity_user (entity_id, user_id),
  KEY idx_project_type (project_id, 	ype)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='素材主表';
`

`sql
CREATE TABLE pms_asset_meta (
  id               BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '主键',
  entity_id        BIGINT UNSIGNED  NOT NULL COMMENT '法律实体ID',
  sset_id         BIGINT UNSIGNED  NOT NULL COMMENT '素材ID',
  width            INT UNSIGNED        NULL COMMENT '宽度(px)',
  height           INT UNSIGNED        NULL COMMENT '高度(px)',
  duration         DECIMAL(10,3)       NULL COMMENT '时长(秒)',
  ps              DECIMAL(6,2)        NULL COMMENT '帧率',
  model            VARCHAR(128)        NULL COMMENT '来源模型',
  prompt           TEXT                NULL COMMENT '生成提示词',
  	ags             JSON                NULL COMMENT '标签',
  extra            JSON                NULL COMMENT '扩展属性',
  created_at       DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at       DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_entity_asset (entity_id, sset_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='素材元数据表';
`

`sql
CREATE TABLE pms_asset_share (
  id               BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT COMMENT '主键',
  entity_id        BIGINT UNSIGNED  NOT NULL COMMENT '法律实体ID',
  sset_id         BIGINT UNSIGNED  NOT NULL COMMENT '素材ID',
  shared_by        BIGINT UNSIGNED  NOT NULL COMMENT '分享人用户ID',
  shared_to        BIGINT UNSIGNED      NULL COMMENT '被分享用户ID',
  permission       VARCHAR(32)      NOT NULL DEFAULT 'read' COMMENT '权限：read/write',
  expire_at        DATETIME(3)         NULL COMMENT '分享过期时间',
  created_at       DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_entity_asset (entity_id, sset_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
COMMENT='素材分享表';
`

设计说明：

- pms_asset.oss_key 是唯一存储指针，前端按此读取资源。
- pms_asset_meta.tags 和 extra 用 JSON 存储灵活字段，便于后续检索。
- pms_asset_share 先做私有分享，MVP 不强制公开只读。

### 2.2 数据流转过程

`mermaid
sequenceDiagram
    participant Frontend
    participant AssetAPI
    participant AssetService
    participant OSS
    participant Database

    Frontend->>AssetAPI: 请求上传签名
    AssetAPI->>AssetService: 生成上传参数
    AssetService->>OSS: 申请签名URL
    OSS-->>AssetService: 返回签名URL
    AssetService-->>AssetAPI: 返回上传凭证
    AssetAPI-->>Frontend: 返回签名URL

    Frontend->>OSS: 直传文件
    OSS-->>Frontend: 上传成功
    Frontend->>AssetAPI: 上传完成回调
    AssetAPI->>AssetService: 写入资产与元数据
    AssetService->>Database: 保存 Asset/AssetMeta
    Database-->>AssetService: 返回资产ID
    AssetService-->>AssetAPI: 返回资产信息
    AssetAPI-->>Frontend: 更新素材库
`

关键流转：

- 大文件不走后端代理，前端直传 OSS，后端只保存 oss_key。
- 上传后必须回调落库，否则会出现文件存在但数据库无记录。
- 画布引用素材只保存 sset_id，不重复存储文件路径。

### 2.3 总体架构

`mermaid
flowchart LR
    A[前端画布/素材库] --> B[后端 Asset API]
    B --> C[AssetService]
    C --> D[(Database)]
    C --> E[OSS]
    B --> F[任务模块]
    F --> C
`

参考 open-ai-canvas：

- 后端服务：D:\GoWorkSpace\open-ai-canvas\backend\internal
- 部署配置：D:\GoWorkSpace\open-ai-canvas\docker-compose.server.yml

建议：

- 素材服务只暴露签名、回调、查询三类核心接口。
- 图片/视频预处理（人脸检测、背景分割）可先由任务模块完成后回写元数据，不放在素材模块内。

---

## 3. 接口清单（MVP）

| 方法   | 路径                      | 说明         |
|--------|---------------------------|--------------|
| POST   | /api/v1/assets/upload-url | 获取上传签名 |
| POST   | /api/v1/assets/callback   | 上传完成回调 |
| GET    | /api/v1/assets            | 素材列表     |
| GET    | /api/v1/assets/{id}       | 素材详情     |
| DELETE | /api/v1/assets/{id}       | 删除素材     |
| POST   | /api/v1/assets/{id}/share | 创建分享     |

---

## 4. 依赖模块

| 模块         | 依赖说明       |
|--------------|----------------|
| 用户模块     | 素材归属与权限 |
| 画布模块     | 画布引用素材   |
| 任务调度模块 | 生成结果入库   |
| 存储模块     | OSS 直传与签名 |

---

## 5. 与 open-ai-canvas 的映射

| 我们的模块 | open-ai-canvas 对应              |
|------------|----------------------------------|
| 素材模块   | 资源同步、私有 OSS、资源归属校验 |
| 上传签名   | 前端直传 OSS 方案                |
| 素材回写   | 任务完成后回写画布与素材库       |

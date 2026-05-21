# keep-hring-bot 数据统计架构

本文档用于把 Telegram + n8n + AI Agent 的招聘初筛流程落地到 MySQL，解决这些问题：

- 哪些人咨询过
- 哪些人正在初筛
- 哪些人通过初筛
- 哪些人没通过初筛，以及没通过原因
- 哪些人收到下载链接
- 后续哪些人点击下载、注册、激活

## 1. 总体架构

当前流程：

```text
Telegram Trigger
-> AI Agent
-> Send a text message
```

建议改为：

```text
Telegram Trigger
-> MySQL: Get Candidate
-> Code: Build Agent Input
-> AI Agent
-> Code: Parse Agent JSON
-> MySQL: Upsert Candidate
-> MySQL: Insert Events
-> Telegram: Send Message
```

核心原则：

1. `candidates` 表保存每个 Telegram 用户当前的初筛状态。
2. `candidate_events` 表保存每个关键动作，用于统计漏斗。
3. AI Agent 必须输出 JSON，不能只输出自然语言。
4. 是否通过初筛以结构化字段为准，不以聊天文本为准。

## 2. MySQL 表结构

### 2.1 候选人状态表

`candidates` 表用于查询“这个人现在是什么状态”。

```sql
CREATE TABLE candidates (
  telegram_user_id BIGINT NOT NULL,
  telegram_chat_id BIGINT NOT NULL,
  username VARCHAR(255) NULL,
  first_name VARCHAR(255) NULL,
  last_name VARCHAR(255) NULL,
  language_code VARCHAR(32) NULL,

  first_seen_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  last_message_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

  country VARCHAR(64) NULL,
  age_passed TINYINT(1) NULL,
  english_passed TINYINT(1) NULL,
  has_android TINYINT(1) NULL,
  internet_ok TINYINT(1) NULL,
  hours_ok TINYINT(1) NULL,
  us_time_ok TINYINT(1) NULL,
  commission_ok TINYINT(1) NULL,
  interested TINYINT(1) NULL,

  screening_step VARCHAR(64) NOT NULL DEFAULT 'not_started',
  screening_status VARCHAR(64) NOT NULL DEFAULT 'not_started',
  rejection_reason VARCHAR(128) NULL,

  passed_at DATETIME NULL,
  rejected_at DATETIME NULL,
  download_link_sent_at DATETIME NULL,
  download_clicked_at DATETIME NULL,
  registered_at DATETIME NULL,
  activated_at DATETIME NULL,

  raw_profile JSON NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  PRIMARY KEY (telegram_user_id),
  KEY idx_candidates_status (screening_status),
  KEY idx_candidates_country (country),
  KEY idx_candidates_rejection_reason (rejection_reason),
  KEY idx_candidates_last_message_at (last_message_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 2.2 候选人事件表

`candidate_events` 表用于统计漏斗。候选人状态可能会被覆盖，但事件记录不会丢。

```sql
CREATE TABLE candidate_events (
  id BIGINT NOT NULL AUTO_INCREMENT,
  telegram_user_id BIGINT NOT NULL,
  telegram_chat_id BIGINT NULL,
  event_name VARCHAR(128) NOT NULL,
  event_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  message_id BIGINT NULL,
  dedupe_key VARCHAR(255) NULL,
  metadata JSON NULL,

  PRIMARY KEY (id),
  UNIQUE KEY uk_candidate_events_dedupe_key (dedupe_key),
  KEY idx_candidate_events_user (telegram_user_id),
  KEY idx_candidate_events_name_time (event_name, event_time),
  KEY idx_candidate_events_time (event_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## 3. 状态枚举

### 3.1 screening_status

```text
not_started   还没开始初筛
in_progress   初筛中
passed        已通过初筛
rejected      未通过初筛
unknown       对话信息不足，暂时无法判断
```

### 3.2 screening_step

```text
not_started
experience
country
age
english
android
internet
hours
us_time
commission
interest
download_sent
rejected
```

### 3.3 rejection_reason

```text
unsupported_country
under_18
english_not_qualified
no_android
unstable_internet
hours_not_enough
us_time_unavailable
rejects_commission
not_interested
```

## 4. 关键事件

建议先记录这些事件：

```text
consulted              首次咨询
screening_started      开始初筛
screening_passed       通过初筛
screening_rejected     未通过初筛
download_link_sent     已发送下载链接
download_clicked       点击下载链接
registered             完成注册
activated              完成激活
```

第一阶段如果没有下载中转链接和 App 回传，可以先实现到：

```text
consulted
screening_started
screening_passed
screening_rejected
download_link_sent
```

## 5. n8n 节点设计

### 5.1 Telegram Trigger

从 Telegram Trigger 中取这些字段：

```text
telegram_user_id = message.from.id
telegram_chat_id = message.chat.id
username = message.from.username
first_name = message.from.first_name
last_name = message.from.last_name
language_code = message.from.language_code
message_id = message.message_id
message_text = message.text
```

### 5.2 MySQL: Get Candidate

在 AI Agent 之前查询候选人当前状态：

```sql
SELECT *
FROM candidates
WHERE telegram_user_id = {{$json.message.from.id}}
LIMIT 1;
```

如果查询不到，后面按新用户处理。

### 5.3 Code: Build Agent Input

把 Telegram 消息和候选人状态一起传给 AI Agent。

推荐传入格式：

```text
Current candidate state:
{{ JSON.stringify($json.candidate || {}) }}

Telegram profile:
{{ JSON.stringify($json.telegram_profile || {}) }}

User message:
{{ $json.message_text }}
```

这样 Agent 能知道这个人之前已经问到哪一步。

### 5.4 AI Agent 输出 JSON

AI Agent 必须固定输出 JSON：

```json
{
  "reply": "Thanks. Are you at least 18 years old?",
  "candidate_update": {
    "country": "Kenya",
    "age_passed": null,
    "english_passed": null,
    "has_android": null,
    "internet_ok": null,
    "hours_ok": null,
    "us_time_ok": null,
    "commission_ok": null,
    "interested": null,
    "screening_step": "age",
    "screening_status": "in_progress",
    "rejection_reason": null
  },
  "events": ["screening_started"]
}
```

通过初筛时：

```json
{
  "reply": "Great, you seem to meet the basic requirements. Please download the app here and follow the next-step instructions:\n\nhttps://h5.keeptochat.com/keep/chattertool_prod_v175.apk",
  "candidate_update": {
    "screening_step": "download_sent",
    "screening_status": "passed",
    "rejection_reason": null
  },
  "events": ["screening_passed", "download_link_sent"]
}
```

未通过初筛时：

```json
{
  "reply": "Thank you for your time. Unfortunately, this role is only open to candidates from Kenya, the Philippines, or Nigeria.",
  "candidate_update": {
    "country": "Ghana",
    "screening_step": "rejected",
    "screening_status": "rejected",
    "rejection_reason": "unsupported_country"
  },
  "events": ["screening_rejected"]
}
```

### 5.5 Code: Parse Agent JSON

AI Agent 后面加 Code 节点，做三件事：

1. 解析 Agent JSON。
2. 如果 Agent 输出不是合法 JSON，走兜底逻辑。
3. 整理成 MySQL 节点容易使用的字段。

兜底逻辑建议：

```text
screening_status = unknown
screening_step = current step 不变
reply = Agent 原始输出或固定兜底文案
```

### 5.6 MySQL: Upsert Candidate

将候选人状态写入 `candidates` 表：

```sql
INSERT INTO candidates (
  telegram_user_id,
  telegram_chat_id,
  username,
  first_name,
  last_name,
  language_code,
  last_message_at,
  country,
  age_passed,
  english_passed,
  has_android,
  internet_ok,
  hours_ok,
  us_time_ok,
  commission_ok,
  interested,
  screening_step,
  screening_status,
  rejection_reason,
  passed_at,
  rejected_at,
  download_link_sent_at,
  raw_profile
)
VALUES (
  :telegram_user_id,
  :telegram_chat_id,
  :username,
  :first_name,
  :last_name,
  :language_code,
  NOW(),
  :country,
  :age_passed,
  :english_passed,
  :has_android,
  :internet_ok,
  :hours_ok,
  :us_time_ok,
  :commission_ok,
  :interested,
  :screening_step,
  :screening_status,
  :rejection_reason,
  CASE WHEN :screening_status = 'passed' THEN NOW() ELSE NULL END,
  CASE WHEN :screening_status = 'rejected' THEN NOW() ELSE NULL END,
  CASE WHEN :screening_step = 'download_sent' THEN NOW() ELSE NULL END,
  :raw_profile
)
ON DUPLICATE KEY UPDATE
  telegram_chat_id = VALUES(telegram_chat_id),
  username = VALUES(username),
  first_name = VALUES(first_name),
  last_name = VALUES(last_name),
  language_code = VALUES(language_code),
  last_message_at = NOW(),

  country = COALESCE(VALUES(country), country),
  age_passed = COALESCE(VALUES(age_passed), age_passed),
  english_passed = COALESCE(VALUES(english_passed), english_passed),
  has_android = COALESCE(VALUES(has_android), has_android),
  internet_ok = COALESCE(VALUES(internet_ok), internet_ok),
  hours_ok = COALESCE(VALUES(hours_ok), hours_ok),
  us_time_ok = COALESCE(VALUES(us_time_ok), us_time_ok),
  commission_ok = COALESCE(VALUES(commission_ok), commission_ok),
  interested = COALESCE(VALUES(interested), interested),

  screening_step = VALUES(screening_step),
  screening_status = VALUES(screening_status),
  rejection_reason = VALUES(rejection_reason),

  passed_at = COALESCE(passed_at, VALUES(passed_at)),
  rejected_at = COALESCE(rejected_at, VALUES(rejected_at)),
  download_link_sent_at = COALESCE(download_link_sent_at, VALUES(download_link_sent_at)),
  raw_profile = VALUES(raw_profile),
  updated_at = NOW();
```

在 n8n MySQL 节点里，如果不方便使用命名参数，可以先用 Code 节点把字段整理成标准 JSON，然后在 MySQL 节点里用表达式填充。

### 5.7 MySQL: Insert Events

每个事件插入一条记录。为了避免重复，使用 `dedupe_key`。

推荐 `dedupe_key`：

```text
telegram_user_id + ':' + event_name
```

对于只应该发生一次的事件，例如 `consulted`、`screening_passed`、`download_link_sent`，这样可以去重。

插入 SQL：

```sql
INSERT INTO candidate_events (
  telegram_user_id,
  telegram_chat_id,
  event_name,
  event_time,
  message_id,
  dedupe_key,
  metadata
)
VALUES (
  :telegram_user_id,
  :telegram_chat_id,
  :event_name,
  NOW(),
  :message_id,
  :dedupe_key,
  :metadata
)
ON DUPLICATE KEY UPDATE
  event_time = event_time;
```

### 5.8 Telegram: Send Message

最后发送 Agent JSON 里的 `reply`：

```text
Chat ID:
{{$node["Telegram Trigger"].json["message"]["chat"]["id"]}}

Text:
{{$json.reply}}
```

## 6. 咨询事件记录

`consulted` 事件可以在 Telegram Trigger 后立即写入。

```sql
INSERT INTO candidate_events (
  telegram_user_id,
  telegram_chat_id,
  event_name,
  event_time,
  message_id,
  dedupe_key,
  metadata
)
VALUES (
  :telegram_user_id,
  :telegram_chat_id,
  'consulted',
  NOW(),
  :message_id,
  CONCAT(:telegram_user_id, ':consulted'),
  :metadata
)
ON DUPLICATE KEY UPDATE
  event_time = event_time;
```

同时可以先 upsert 一次候选人基础信息：

```sql
INSERT INTO candidates (
  telegram_user_id,
  telegram_chat_id,
  username,
  first_name,
  last_name,
  language_code,
  first_seen_at,
  last_message_at,
  screening_status,
  raw_profile
)
VALUES (
  :telegram_user_id,
  :telegram_chat_id,
  :username,
  :first_name,
  :last_name,
  :language_code,
  NOW(),
  NOW(),
  'in_progress',
  :raw_profile
)
ON DUPLICATE KEY UPDATE
  telegram_chat_id = VALUES(telegram_chat_id),
  username = VALUES(username),
  first_name = VALUES(first_name),
  last_name = VALUES(last_name),
  language_code = VALUES(language_code),
  last_message_at = NOW(),
  raw_profile = VALUES(raw_profile),
  updated_at = NOW();
```

## 7. 下载统计

当前系统提示词里直接发送 APK：

```text
https://h5.keeptochat.com/keep/chattertool_prod_v175.apk
```

这种方式只能统计“链接已发送”，不能准确统计“谁点击下载”。

如果要统计点击下载，建议改成中转链接：

```text
https://your-domain.com/download?uid=<telegram_user_id>&source=telegram
```

服务端收到请求后：

1. 写入 `download_clicked` 事件。
2. 更新 `candidates.download_clicked_at`。
3. 302 跳转到真实 APK 地址。

如果 App 可以在注册或激活时回传来源，再写入：

```text
registered
activated
```

这样漏斗可以从“咨询”一直看到“激活”。

## 8. 常用查询

### 8.1 通过初筛的人

```sql
SELECT *
FROM candidates
WHERE screening_status = 'passed'
ORDER BY passed_at DESC;
```

### 8.2 未通过初筛的人

```sql
SELECT *
FROM candidates
WHERE screening_status = 'rejected'
ORDER BY rejected_at DESC;
```

### 8.3 按拒绝原因统计

```sql
SELECT rejection_reason, COUNT(*) AS total
FROM candidates
WHERE screening_status = 'rejected'
GROUP BY rejection_reason
ORDER BY total DESC;
```

### 8.4 漏斗统计

```sql
SELECT event_name, COUNT(DISTINCT telegram_user_id) AS users
FROM candidate_events
WHERE event_name IN (
  'consulted',
  'screening_started',
  'screening_passed',
  'screening_rejected',
  'download_link_sent',
  'download_clicked',
  'registered',
  'activated'
)
GROUP BY event_name;
```

### 8.5 每日咨询人数

```sql
SELECT DATE(event_time) AS day, COUNT(DISTINCT telegram_user_id) AS users
FROM candidate_events
WHERE event_name = 'consulted'
GROUP BY DATE(event_time)
ORDER BY day DESC;
```

### 8.6 每日通过初筛人数

```sql
SELECT DATE(passed_at) AS day, COUNT(*) AS users
FROM candidates
WHERE screening_status = 'passed'
  AND passed_at IS NOT NULL
GROUP BY DATE(passed_at)
ORDER BY day DESC;
```

## 9. 推荐上线顺序

第一阶段：

1. 创建 `candidates` 表。
2. 创建 `candidate_events` 表。
3. n8n 在 Telegram Trigger 后写入 `consulted`。
4. AI Agent 改成输出 JSON。
5. AI Agent 后写入 `candidates`。
6. 查询通过和未通过初筛名单。

第二阶段：

1. 加 `download_link_sent` 事件。
2. 加中转下载链接。
3. 统计 `download_clicked`。

第三阶段：

1. App 注册时回传来源。
2. App 激活时回传来源。
3. 统计完整漏斗：咨询 -> 通过初筛 -> 点击下载 -> 注册 -> 激活。

## 10. 最小可用版本

如果想先最快跑起来，只需要：

1. `candidates` 表。
2. AI Agent 输出 JSON。
3. 每轮对话后 upsert `screening_status` 和 `rejection_reason`。

最小查询：

```sql
SELECT telegram_user_id, username, country, screening_status, rejection_reason, updated_at
FROM candidates
ORDER BY updated_at DESC;
```

这个版本已经能回答：

```text
哪些人通过了初筛？
哪些人没通过初筛？
为什么没通过？
```

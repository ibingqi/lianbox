# keep-hring-bot 最小版 candidates 表实施方案

本文档只实现一个目标：用一张 MySQL `candidates` 表记录 Telegram 候选人的初筛状态。

最小版先回答三个问题：

- 哪些人咨询过
- 哪些人通过初筛
- 哪些人未通过初筛，以及原因

不做独立事件表，不做下载点击统计，不做 App 注册/激活回传。后续需要漏斗统计时，再从本方案升级到 `candidate_events` 表。

## 1. 最小架构

当前 n8n 流程：

```text
Telegram Trigger
-> AI Agent
-> Send a text message
```

建议改成：

```text
Telegram Trigger
-> Code: Normalize Telegram Input
-> MySQL: Upsert Candidate Profile
-> MySQL: Get Candidate
-> Code: Build Agent Input
-> AI Agent
-> Code: Parse Agent JSON
-> MySQL: Update Candidate Screening
-> Telegram: Send Message
```

为什么需要 AI Agent 前面查一次 MySQL：

- Agent 需要知道这个用户之前已经问到哪一步。
- MySQL 里的状态才是后台查询的准数据。
- n8n 的 Simple Memory 可以继续保留，但不要把它当统计数据库。

## 2. MySQL 建表语句

直接在 MySQL 执行：

```sql
CREATE TABLE IF NOT EXISTS candidates (
  telegram_user_id BIGINT NOT NULL COMMENT 'Telegram user id, one row per candidate',
  telegram_chat_id BIGINT NULL COMMENT 'Telegram chat id, private chat is usually same as user id',
  username VARCHAR(255) NULL,
  first_name VARCHAR(255) NULL,
  last_name VARCHAR(255) NULL,
  language_code VARCHAR(32) NULL,

  country VARCHAR(64) NULL,
  age_passed TINYINT(1) NULL COMMENT '1=yes, 0=no, NULL=unknown',
  english_passed TINYINT(1) NULL COMMENT '1=yes, 0=no, NULL=unknown',
  has_android TINYINT(1) NULL COMMENT '1=yes, 0=no, NULL=unknown',
  internet_ok TINYINT(1) NULL COMMENT '1=yes, 0=no, NULL=unknown',
  hours_ok TINYINT(1) NULL COMMENT '1=yes, 0=no, NULL=unknown',
  us_time_ok TINYINT(1) NULL COMMENT '1=yes, 0=no, NULL=unknown',
  commission_ok TINYINT(1) NULL COMMENT '1=yes, 0=no, NULL=unknown',
  interested TINYINT(1) NULL COMMENT '1=yes, 0=no, NULL=unknown',

  screening_step VARCHAR(64) NOT NULL DEFAULT 'not_started' COMMENT 'current question or final step',
  screening_status VARCHAR(64) NOT NULL DEFAULT 'not_started' COMMENT 'not_started/in_progress/passed/rejected/unknown',
  rejection_reason VARCHAR(128) NULL COMMENT 'reason when screening_status = rejected',

  first_seen_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  last_message_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  last_message_id BIGINT NULL,
  last_user_message TEXT NULL,
  last_agent_reply TEXT NULL,

  passed_at DATETIME NULL,
  rejected_at DATETIME NULL,
  download_link_sent_at DATETIME NULL,

  raw_profile JSON NULL,
  agent_raw_output JSON NULL,
  parse_error TEXT NULL,

  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  PRIMARY KEY (telegram_user_id),
  KEY idx_candidates_status (screening_status),
  KEY idx_candidates_step (screening_step),
  KEY idx_candidates_country (country),
  KEY idx_candidates_rejection_reason (rejection_reason),
  KEY idx_candidates_last_message_at (last_message_at),
  KEY idx_candidates_passed_at (passed_at),
  KEY idx_candidates_rejected_at (rejected_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

字段说明：

```text
telegram_user_id       唯一用户，主键
screening_status       当前是否通过、拒绝、进行中
screening_step         当前问到哪一步
rejection_reason       未通过原因
passed_at              首次通过初筛时间
rejected_at            首次未通过初筛时间
download_link_sent_at  首次发送下载链接时间
last_user_message      最近一条用户消息，方便排查
last_agent_reply       最近一条 bot 回复，方便排查
agent_raw_output       Agent 原始 JSON 输出，方便排查
parse_error            Agent 输出不是合法 JSON 时记录错误
```

## 3. 状态口径

### 3.1 screening_status

只允许 Agent 输出这些值：

```text
not_started
in_progress
passed
rejected
unknown
```

含义：

```text
not_started   已咨询，但还没进入初筛
in_progress   初筛中
passed        已通过初筛
rejected      未通过初筛
unknown       Agent 无法判断，或输出异常
```

### 3.2 screening_step

建议使用这些值：

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

建议使用这些值：

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

## 4. n8n 节点细化

### 4.1 Telegram Trigger

保持你现在已有的 Telegram Trigger。

需要用到的数据：

```text
message.from.id
message.chat.id
message.from.username
message.from.first_name
message.from.last_name
message.from.language_code
message.message_id
message.text
```

### 4.2 Code: Normalize Telegram Input

在 Telegram Trigger 后加一个 Code 节点。

Mode 选：

```text
Run Once for Each Item
```

代码：

```javascript
const msg = $json.message || {};
const from = msg.from || {};
const chat = msg.chat || {};

return {
  json: {
    telegram_user_id: from.id ? Number(from.id) : null,
    telegram_chat_id: chat.id ? Number(chat.id) : null,
    username: from.username || null,
    first_name: from.first_name || null,
    last_name: from.last_name || null,
    language_code: from.language_code || null,
    message_id: msg.message_id ? Number(msg.message_id) : null,
    message_text: msg.text || "",
    raw_profile: JSON.stringify(from || {})
  }
};
```

### 4.3 MySQL: Upsert Candidate Profile

这个节点用于确保每个咨询用户在 `candidates` 表里有一行记录。

MySQL 节点选择：

```text
Operation: Execute SQL
```

SQL：

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
  last_message_id,
  last_user_message,
  screening_status,
  raw_profile
)
VALUES (
  $1,
  $2,
  $3,
  $4,
  $5,
  $6,
  NOW(),
  NOW(),
  $7,
  $8,
  'in_progress',
  $9
)
ON DUPLICATE KEY UPDATE
  telegram_chat_id = VALUES(telegram_chat_id),
  username = VALUES(username),
  first_name = VALUES(first_name),
  last_name = VALUES(last_name),
  language_code = VALUES(language_code),
  last_message_at = NOW(),
  last_message_id = VALUES(last_message_id),
  last_user_message = VALUES(last_user_message),
  raw_profile = VALUES(raw_profile),
  updated_at = NOW();
```

Query Parameters 填：

```text
{{ $json.telegram_user_id }},
{{ $json.telegram_chat_id }},
{{ $json.username }},
{{ $json.first_name }},
{{ $json.last_name }},
{{ $json.language_code }},
{{ $json.message_id }},
{{ $json.message_text }},
{{ $json.raw_profile }}
```

注意：

- n8n MySQL 节点建议用 Query Parameters，不要直接把用户消息拼进 SQL。
- 这样能避免用户消息里有引号、换行等内容导致 SQL 出错。
- 如果你的 n8n MySQL 节点不接受 `$1`、`$2` 这种占位符，优先使用节点里的 Query Parameters/Values 配置；实在不行再用表达式版本，但要确认文本值会被正确转义。

### 4.4 MySQL: Get Candidate

这个节点读取候选人当前状态，给 AI Agent 使用。

MySQL 节点选择：

```text
Operation: Execute SQL
```

SQL：

```sql
SELECT
  telegram_user_id,
  telegram_chat_id,
  username,
  first_name,
  last_name,
  language_code,
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
  last_user_message
FROM candidates
WHERE telegram_user_id = $1
LIMIT 1;
```

Query Parameters：

```text
{{ $json.telegram_user_id }}
```

如果你的 n8n MySQL 节点不接受 `$1` 占位符，也可以用表达式版本：

```sql
SELECT
  telegram_user_id,
  telegram_chat_id,
  username,
  first_name,
  last_name,
  language_code,
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
  last_user_message
FROM candidates
WHERE telegram_user_id = {{ $json.telegram_user_id }}
LIMIT 1;
```

如果这个节点执行后丢失了上一节点的 `message_text`，可以在后面 `Build Agent Input` 节点里用节点引用读取：

```text
$node["Normalize Telegram Input"].json.message_text
```

### 4.5 Code: Build Agent Input

把候选人状态和用户最新消息拼成 Agent 输入。

Mode 选：

```text
Run Once for Each Item
```

代码：

```javascript
const candidate = $json || {};
const input = $node["Normalize Telegram Input"].json;

return {
  json: {
    telegram_user_id: input.telegram_user_id,
    telegram_chat_id: input.telegram_chat_id,
    message_id: input.message_id,
    message_text: input.message_text,
    candidate,
    agent_input: [
      "Current candidate state:",
      JSON.stringify(candidate, null, 2),
      "",
      "Latest user message:",
      input.message_text
    ].join("\n")
  }
};
```

AI Agent 的 user message/input 使用：

```text
{{ $json.agent_input }}
```

## 5. AI Agent 输出要求

需要在 AI Agent 的系统提示词后面追加这段规则。

```text
Output format requirement:

You must always return valid JSON only. Do not wrap it in Markdown. Do not add any text outside the JSON.

The JSON schema must be:
{
  "reply": "message to send to the candidate",
  "candidate_update": {
    "country": null,
    "age_passed": null,
    "english_passed": null,
    "has_android": null,
    "internet_ok": null,
    "hours_ok": null,
    "us_time_ok": null,
    "commission_ok": null,
    "interested": null,
    "screening_step": "not_started | experience | country | age | english | android | internet | hours | us_time | commission | interest | download_sent | rejected",
    "screening_status": "not_started | in_progress | passed | rejected | unknown",
    "rejection_reason": null
  }
}

Rules:
- Set a field to true only when the candidate clearly satisfies it.
- Set a field to false only when the candidate clearly fails it.
- Set a field to null when it is still unknown.
- If any hard requirement fails, set screening_status to "rejected", screening_step to "rejected", and rejection_reason to the exact reason.
- If all hard requirements pass and the candidate confirms interest, set screening_status to "passed" and screening_step to "download_sent".
- If the download link is included in reply, screening_step must be "download_sent".
- Do not send the download link before screening_status is "passed".
```

示例：初筛中

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
  }
}
```

示例：通过初筛

```json
{
  "reply": "Great, you seem to meet the basic requirements. Please download the app here and follow the next-step instructions:\n\nhttps://h5.keeptochat.com/keep/chattertool_prod_v175.apk",
  "candidate_update": {
    "country": "Kenya",
    "age_passed": true,
    "english_passed": true,
    "has_android": true,
    "internet_ok": true,
    "hours_ok": true,
    "us_time_ok": true,
    "commission_ok": true,
    "interested": true,
    "screening_step": "download_sent",
    "screening_status": "passed",
    "rejection_reason": null
  }
}
```

示例：未通过初筛

```json
{
  "reply": "Thank you for your time. Unfortunately, this role is only open to candidates from Kenya, the Philippines, or Nigeria.",
  "candidate_update": {
    "country": "Ghana",
    "age_passed": null,
    "english_passed": null,
    "has_android": null,
    "internet_ok": null,
    "hours_ok": null,
    "us_time_ok": null,
    "commission_ok": null,
    "interested": null,
    "screening_step": "rejected",
    "screening_status": "rejected",
    "rejection_reason": "unsupported_country"
  }
}
```

## 6. Code: Parse Agent JSON

AI Agent 后面加 Code 节点。

Mode 选：

```text
Run Once for Each Item
```

代码：

```javascript
const input = $node["Normalize Telegram Input"].json;

function getAgentText(json) {
  if (typeof json === "string") return json;
  if (typeof json.output === "string") return json.output;
  if (typeof json.text === "string") return json.text;
  if (typeof json.response === "string") return json.response;
  if (typeof json.message === "string") return json.message;
  return JSON.stringify(json);
}

const rawText = getAgentText($json);

let parsed;
let parseError = null;

try {
  parsed = JSON.parse(rawText);
} catch (error) {
  parseError = error.message;
  parsed = {
    reply: rawText || "This question needs to be confirmed with the administrator. You can ask your mentor during the trial period.",
    candidate_update: {
      country: null,
      age_passed: null,
      english_passed: null,
      has_android: null,
      internet_ok: null,
      hours_ok: null,
      us_time_ok: null,
      commission_ok: null,
      interested: null,
      screening_step: "not_started",
      screening_status: "unknown",
      rejection_reason: null
    }
  };
}

const update = parsed.candidate_update || {};

function boolToTinyInt(value) {
  if (value === true) return 1;
  if (value === false) return 0;
  return null;
}

const allowedStatuses = ["not_started", "in_progress", "passed", "rejected", "unknown"];
const allowedSteps = [
  "not_started",
  "experience",
  "country",
  "age",
  "english",
  "android",
  "internet",
  "hours",
  "us_time",
  "commission",
  "interest",
  "download_sent",
  "rejected"
];

const screeningStatus = allowedStatuses.includes(update.screening_status)
  ? update.screening_status
  : "unknown";

const screeningStep = allowedSteps.includes(update.screening_step)
  ? update.screening_step
  : "not_started";

return {
  json: {
    telegram_user_id: input.telegram_user_id,
    telegram_chat_id: input.telegram_chat_id,
    message_id: input.message_id,
    reply: parsed.reply || "",

    country: update.country || null,
    age_passed: boolToTinyInt(update.age_passed),
    english_passed: boolToTinyInt(update.english_passed),
    has_android: boolToTinyInt(update.has_android),
    internet_ok: boolToTinyInt(update.internet_ok),
    hours_ok: boolToTinyInt(update.hours_ok),
    us_time_ok: boolToTinyInt(update.us_time_ok),
    commission_ok: boolToTinyInt(update.commission_ok),
    interested: boolToTinyInt(update.interested),
    screening_step: screeningStep,
    screening_status: screeningStatus,
    rejection_reason: update.rejection_reason || null,

    agent_raw_output: JSON.stringify(parsed),
    parse_error: parseError
  }
};
```

## 7. MySQL: Update Candidate Screening

这个节点把 Agent 判断结果写回 `candidates`。

MySQL 节点选择：

```text
Operation: Execute SQL
```

SQL：

```sql
UPDATE candidates
SET
  country = COALESCE($1, country),
  age_passed = COALESCE($2, age_passed),
  english_passed = COALESCE($3, english_passed),
  has_android = COALESCE($4, has_android),
  internet_ok = COALESCE($5, internet_ok),
  hours_ok = COALESCE($6, hours_ok),
  us_time_ok = COALESCE($7, us_time_ok),
  commission_ok = COALESCE($8, commission_ok),
  interested = COALESCE($9, interested),

  screening_step = COALESCE($10, screening_step),
  screening_status = CASE
    WHEN screening_status IN ('passed', 'rejected') AND $11 IN ('in_progress', 'unknown', 'not_started')
      THEN screening_status
    ELSE COALESCE($11, screening_status)
  END,
  rejection_reason = COALESCE($12, rejection_reason),

  passed_at = CASE
    WHEN $11 = 'passed' AND passed_at IS NULL THEN NOW()
    ELSE passed_at
  END,
  rejected_at = CASE
    WHEN $11 = 'rejected' AND rejected_at IS NULL THEN NOW()
    ELSE rejected_at
  END,
  download_link_sent_at = CASE
    WHEN $10 = 'download_sent' AND download_link_sent_at IS NULL THEN NOW()
    ELSE download_link_sent_at
  END,

  last_agent_reply = $13,
  agent_raw_output = $14,
  parse_error = $15,
  updated_at = NOW()
WHERE telegram_user_id = $16;
```

Query Parameters：

```text
{{ $json.country }},
{{ $json.age_passed }},
{{ $json.english_passed }},
{{ $json.has_android }},
{{ $json.internet_ok }},
{{ $json.hours_ok }},
{{ $json.us_time_ok }},
{{ $json.commission_ok }},
{{ $json.interested }},
{{ $json.screening_step }},
{{ $json.screening_status }},
{{ $json.rejection_reason }},
{{ $json.reply }},
{{ $json.agent_raw_output }},
{{ $json.parse_error }},
{{ $json.telegram_user_id }}
```

如果你的 n8n MySQL 节点支持 `Insert or Update` 操作，也可以不用这段 `UPDATE` SQL，而是把这些字段映射到节点字段里：

```text
Matching Column: telegram_user_id
Values to Send: Define Below for Each Column
```

但最小版更推荐使用上面的 SQL，因为它保留了这些保护逻辑：

这个 SQL 的保护逻辑：

- 已经 `passed` 的人，不会被后续异常输出覆盖成 `unknown`。
- 已经 `rejected` 的人，不会被后续异常输出覆盖成 `in_progress`。
- `passed_at`、`rejected_at`、`download_link_sent_at` 只记录第一次发生时间。

## 8. Telegram: Send Message

最后的 Telegram Send Message 节点：

```text
Chat ID:
{{ $json.telegram_chat_id }}

Text:
{{ $json.reply }}
```

如果你的 Telegram 节点接不到 `$json.reply`，确认它连接的是 `Parse Agent JSON` 节点或 `Update Candidate Screening` 节点后仍保留输入数据。

## 9. 常用查询

### 9.1 查询所有候选人

```sql
SELECT
  telegram_user_id,
  username,
  first_name,
  country,
  screening_status,
  screening_step,
  rejection_reason,
  passed_at,
  rejected_at,
  download_link_sent_at,
  updated_at
FROM candidates
ORDER BY updated_at DESC;
```

### 9.2 通过初筛的人

```sql
SELECT
  telegram_user_id,
  username,
  first_name,
  country,
  passed_at,
  download_link_sent_at
FROM candidates
WHERE screening_status = 'passed'
ORDER BY passed_at DESC;
```

### 9.3 未通过初筛的人

```sql
SELECT
  telegram_user_id,
  username,
  first_name,
  country,
  rejection_reason,
  rejected_at,
  last_user_message
FROM candidates
WHERE screening_status = 'rejected'
ORDER BY rejected_at DESC;
```

### 9.4 初筛中的人

```sql
SELECT
  telegram_user_id,
  username,
  first_name,
  country,
  screening_step,
  last_message_at,
  last_user_message
FROM candidates
WHERE screening_status = 'in_progress'
ORDER BY last_message_at DESC;
```

### 9.5 按拒绝原因统计

```sql
SELECT
  rejection_reason,
  COUNT(*) AS total
FROM candidates
WHERE screening_status = 'rejected'
GROUP BY rejection_reason
ORDER BY total DESC;
```

### 9.6 每日咨询人数

最小版没有事件表，所以用 `first_seen_at` 统计首次咨询：

```sql
SELECT
  DATE(first_seen_at) AS day,
  COUNT(*) AS total
FROM candidates
GROUP BY DATE(first_seen_at)
ORDER BY day DESC;
```

### 9.7 每日通过初筛人数

```sql
SELECT
  DATE(passed_at) AS day,
  COUNT(*) AS total
FROM candidates
WHERE passed_at IS NOT NULL
GROUP BY DATE(passed_at)
ORDER BY day DESC;
```

### 9.8 每日未通过初筛人数

```sql
SELECT
  DATE(rejected_at) AS day,
  COUNT(*) AS total
FROM candidates
WHERE rejected_at IS NOT NULL
GROUP BY DATE(rejected_at)
ORDER BY day DESC;
```

## 10. 上线检查清单

上线前检查：

```text
1. MySQL 已创建 candidates 表
2. n8n 已配置 MySQL credential
3. Telegram Trigger 能收到 message
4. Normalize Telegram Input 输出 telegram_user_id
5. Upsert Candidate Profile 能成功写入一行
6. Get Candidate 能查回这一行
7. AI Agent 输出合法 JSON
8. Parse Agent JSON 能得到 reply 和 screening_status
9. Update Candidate Screening 能更新 candidates
10. Telegram Send Message 发送的是 reply
```

测试对话建议：

```text
测试 1：用户来自 Ghana
预期：screening_status = rejected, rejection_reason = unsupported_country

测试 2：用户来自 Kenya，但未满 18
预期：screening_status = rejected, rejection_reason = under_18

测试 3：用户来自 Philippines，满足所有硬条件，并愿意继续
预期：screening_status = passed, screening_step = download_sent, download_link_sent_at 不为空

测试 4：Agent 输出异常
预期：screening_status = unknown, parse_error 不为空
```

## 11. 后续升级点

最小版的限制：

- 只能知道“下载链接是否发送”，不能知道“用户是否点击下载”。
- 只能保留最近一条用户消息和最近一条 bot 回复。
- 统计漏斗时只能用状态时间字段，不能还原完整事件流。

后续升级时再加：

```text
candidate_events 表
download 中转链接
App 注册/激活回传
```

## 12. 参考

- n8n MySQL node supports `Execute SQL`, `Select`, `Insert`, `Update`, and `Insert or Update`: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.mysql/
- n8n MySQL node supports Query Parameters for safer SQL values: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.mysql/
- n8n Code node supports JavaScript and Python workflow code: https://docs.n8n.io/code/code-node/

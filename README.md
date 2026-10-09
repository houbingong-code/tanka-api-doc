# 修复团队双向接口文档

状态：2026-10-09 对接评审稿，双方确认后实施。本文列出探卡平台与修复团队各自需要提供的接口、报文、签名和重试规则；业务边界见[双向接口契约](repair-team-integration-contract.md)。示例域名、条码、卡号、价格、机构及图片地址均为演示数据。

## 1. 接口清单与职责

探卡平台生成唯一 `barcode`，打印在装有多张卡的袋上。修复团队按条码接收预评任务并回传整袋逐卡预评内容；用户在探卡 App 选择方向后，平台分别下发修复、送评、寄回任务。团队回传逐卡修复结果、单卡拟退金额和履约事实；用户选择、付款、退款核对与执行、评级出分由平台管理。本期不要求收袋清点回调，不传 PDF 文件，也不提供媒体上传接口。

| 提供方 | 方法与路径 | 用途 |
| --- | --- | --- |
| 修复团队 | `POST /openapi/assessment/create` | 创建预评袋单 |
| 修复团队 | `POST /openapi/assessment/query` | 查询预评袋单 |
| 修复团队 | `POST /openapi/reports/query` | 补查完整预评或修复报告内容 |
| 修复团队 | `POST /openapi/tasks/repair/create` | 创建修复任务 |
| 修复团队 | `POST /openapi/tasks/grading/create` | 创建送评任务 |
| 修复团队 | `POST /openapi/tasks/return/create` | 创建统一寄回任务 |
| 修复团队 | `POST /openapi/tasks/query` | 查询任务执行结果 |
| 探卡平台 | `POST /partner/repair-team/assessment/report` | 接收整袋预评内容 |
| 探卡平台 | `POST /partner/repair-team/repair/report` | 接收逐卡修复结果及单卡拟退金额 |
| 探卡平台 | `POST /partner/repair-team/task/event` | 接收履约事件 |
| 探卡平台 | `POST /partner/repair-team/events/query` | 查询回调处理结果 |

调用方将接口表中的路径拼接在提供方交付的对应环境 Base URL 后；沙箱与正式环境使用各自的 Base URL 和密钥。Base URL 及其他联调配置见第 8 节，均不写入业务请求体。

## 2. 公共协议与参数

所有接口只接受 HTTPS `POST`，请求体为 UTF-8 JSON，`Content-Type: application/json; charset=utf-8`，不接受表单、文件、Base64 图片或压缩请求体。金额为人民币元的固定两位小数字符串，例如 `"120.00"`；时间为包含时区的 RFC 3339 字符串，例如 `"2026-10-09T16:00:00+08:00"`。所有字段名大小写敏感；未知字段不能用来改变约定的业务状态或金额。

| 位置 | 字段 | 必填 | 说明 |
| --- | --- | --- | --- |
| 请求体 | `schema_version` | 所有接口 | 字符串，当前固定为 `"1"` |
| 请求体 | `barcode` | 袋单、报告、任务与任务事件 | 平台生成的全局唯一袋条码，按原值比较，不作为鉴权凭据 |
| 请求体 | `request_id` | 平台调用团队的创建接口 | 平台生成；同一业务写操作重试时保持不变，不用于查询 |
| 请求体 | `task_id` | 修复／送评／寄回任务及关联报告、事件 | 平台生成的任务号；同一任务不因重试改变 |
| 请求体 | `event_id` | 团队推送报告或事件 | 团队生成；同一回调重试时保持不变，不用于查询 |
| 请求头 | `X-Partner-Timestamp` | 所有接口 | Unix 秒时间戳的十进制字符串 |
| 请求头 | `X-Partner-Signature` | 所有接口 | 按第 3 节计算的 64 位小写十六进制 HMAC-SHA256 |

`partner_card_code` 由团队在预评时生成，在同一 `barcode` 下唯一并保持稳定；不能用卡片名称、数组序号或图片地址替代。`issue_no` 是单张卡内从 1 开始的缺陷序号，只关联卡损与证据图，不用于计价或退款。数组中的对象顺序不构成身份。单个送评任务只包含一条四级评级路径；寄回按袋统一下发。

`request_id` 由平台在本对接的写操作中全局唯一生成；`event_id` 由团队在本对接的回调中全局唯一生成。两者均为不含空白的非空字符串。`barcode`、`task_id` 和团队卡号均按大小写敏感的原始字符串比较；不能去掉前导零或自行转换大小写。

成功响应统一采用以下结构，`data` 的具体字段见各接口。HTTP 200 且 `code=OK` 表示请求被持久化接收或查询成功，不表示报告已向用户发布、修复完成、物流寄出、付款或退款成功。

```json
{"code":"OK","message":"accepted","data":{"barcode":"BAG001","status":"accepted"}}
```

失败响应仍返回 JSON，例如：

```json
{"code":"INVALID_ARGUMENT","message":"expected_card_count must be positive","data":{}}
```

## 3. 签名与验签

双方按调用方向分别持有独立共享密钥：平台调用团队接口使用平台出站密钥，团队调用平台接口使用团队出站密钥。沙箱与生产密钥隔离。正式密钥建议为安全随机生成的 32 字节，以 Base64 交付并解码成原始字节后作为 HMAC 密钥；下方测试向量使用可见 ASCII 字节便于核算。正式密钥只通过双方约定的安全渠道交付，不写进 URL、JSON、日志或业务请求体。

签名使用**实际发送的原始 HTTP 请求体字节**，不解析后重新序列化 JSON。按以下顺序计算；`concat_bytes(a, b)` 表示把 `b` 的字节紧接在 `a` 后面，不写入逗号、空格或其他分隔符：

```text
prefix_bytes = UTF8("POST\n" + path + "\n" + timestamp + "\n")
signing_bytes = concat_bytes(prefix_bytes, raw_body_bytes)
signature = lowercase_hex(HMAC_SHA256(secret_bytes, signing_bytes))
```

`path` 是接口表中以 `/` 开头的请求路径；本期接口均不使用 URL 查询参数。`timestamp` 是请求头 `X-Partner-Timestamp` 的原始十进制字符串；`secret_bytes` 是双方交付的 Base64 密钥解码后的字节。第三行时间戳后有一个 LF（`\n`），随后立即是请求体的第一个字节；请求体末尾不额外增加换行。将 64 位小写十六进制 `signature` 放入 `X-Partner-Signature`。`Content-Type` 不参与签名，但仍须按第 2 节发送。代理若改写路径或请求体空白，验签将失败。

### 3.1 从请求体生成签名

以下是**离线测试向量**，时间戳已过期，不可直接发送到在线接口。假设调用方要查询条码 `BAG001`，把下面这段 JSON 作为请求体原样发送。示例没有空格、缩进或末尾换行；如果实际发送的字节不同，签名也会不同。

```text
secret: demo-secret-for-test-only
method: POST
path: /openapi/assessment/query
timestamp: 1700000000
raw body: {"schema_version":"1","barcode":"BAG001"}
```

这次 HMAC 的完整输入是下面四行之间的 UTF-8 字节：前三行末尾各有一个 LF，JSON 末尾没有 LF。域名和 `Content-Type` 不在输入中。

```text
POST
/openapi/assessment/query
1700000000
{"schema_version":"1","barcode":"BAG001"}
```

用示例密钥的 ASCII 字节计算 HMAC-SHA256，得到 `28df5cbd3d00a7ca6eaf4c26b3f0ffea71245cdfa51ae8262c73f4fe8ae54e7c`。对应的完整 HTTP 请求形状如下；`partner.example.com` 只是占位域名：

```http
POST /openapi/assessment/query HTTP/1.1
Host: partner.example.com
Content-Type: application/json; charset=utf-8
X-Partner-Timestamp: 1700000000
X-Partner-Signature: 28df5cbd3d00a7ca6eaf4c26b3f0ffea71245cdfa51ae8262c73f4fe8ae54e7c

{"schema_version":"1","barcode":"BAG001"}
```

下面用 Python 标准库复算测试向量；其他语言按同样的字节顺序计算即可。生产环境把 `secret` 换成 Base64 配置**解码后的原始密钥字节**，并用当前 Unix 秒时间戳。

```python
import hashlib
import hmac

secret = b"demo-secret-for-test-only"
method = "POST"
path = "/openapi/assessment/query"
timestamp = "1700000000"
raw_body = b'{"schema_version":"1","barcode":"BAG001"}'

prefix = f"{method}\n{path}\n{timestamp}\n".encode("utf-8")
signature = hmac.new(secret, prefix + raw_body, hashlib.sha256).hexdigest()
assert signature == "28df5cbd3d00a7ca6eaf4c26b3f0ffea71245cdfa51ae8262c73f4fe8ae54e7c"
print(signature)
```

### 3.2 接收方验签

接收方从 HTTP 请求读取两个签名头，并**在解析 JSON 之前**保留收到的原始请求体字节。先判断时间戳是否在允许的前后 300 秒内，再使用本方向、本环境的共享密钥按同一公式重算签名，并以常量时间比较。以下示例把 `now` 固定为测试时间；实际服务端应传当前 Unix 秒。

```python
import hashlib
import hmac

def verify(method, path, timestamp, signature, raw_body, secret, now):
    if not timestamp.isdecimal() or abs(now - int(timestamp)) > 300:
        return False
    if len(signature) != 64 or any(c not in "0123456789abcdef" for c in signature):
        return False
    prefix = f"{method}\n{path}\n{timestamp}\n".encode("utf-8")
    expected = hmac.new(secret, prefix + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)

secret = b"demo-secret-for-test-only"
body = b'{"schema_version":"1","barcode":"BAG001"}'
signature = "28df5cbd3d00a7ca6eaf4c26b3f0ffea71245cdfa51ae8262c73f4fe8ae54e7c"
path = "/openapi/assessment/query"

assert verify("POST", path, "1700000000", signature, body, secret, 1700000000)
assert not verify("POST", path, "1700000000", signature, body, secret, 1700000301)
assert not verify("POST", path, "1700000000", signature,
                  b'{"schema_version":"1","barcode":"BAG002"}', secret, 1700000000)
print("验签示例通过")
```

第二个断言说明时间戳超过 300 秒会被拒绝；第三个断言说明即使只改条码，请求体签名也会失效。服务端应在验签成功后才校验 `schema_version`、`request_id`／`event_id` 和业务内容；签名正确只证明请求持有共享密钥且报文未被改动，不代表业务请求必然有效。

团队回调平台时使用同一公式，但路径换成实际回调路径，密钥换成团队出站密钥。不能仅依赖来源 IP 或条码信任回调。密钥轮换期间可同时尝试当前与上一把密钥，旧密钥的停用时间由部署通知给出，不需要请求携带密钥 ID。

响应依赖 HTTPS 传输保护，不额外附加响应签名。上述 300 秒及密钥交付编码是本稿的联调默认值，双方评审如需调整，应在启用前同步更新文档与实现，不能单边改变。

业务写请求重试须保持 `request_id` 或 `event_id` 和**原始请求体不变**，但使用新的时间戳与签名。该简化方案不提供一次性 nonce 防重放：有效时间窗口内的相同写报文可能被再次发送，接收方必须按业务键及请求体内容执行幂等处理。`request_id` 或 `event_id` 相同且内容相同，返回原处理结果；键相同但内容不同返回 HTTP 409。查询接口只读，但被捕获的有效签名报文在窗口内可重复查询，因此双方必须保护 HTTPS 链路和代理日志中的签名及报告内容。网络超时视为结果未决，先查询或按原业务键重试，不另建任务。

## 4. 公共错误码与处理

| HTTP | `code` | 含义及调用方动作 |
| --- | --- | --- |
| 400 | `INVALID_ARGUMENT` | 缺失字段、类型／格式／金额／数组校验失败；修正报文，不按原报文无限重试 |
| 401 | `UNAUTHORIZED` | 缺签名头、密钥配置错误、时间偏差、签名不符；排查配置与时钟，不透露内部密钥信息 |
| 404 | `NOT_FOUND` | 查询的条码、任务或事件不存在；调用方先核对业务键 |
| 409 | `SCHEMA_VERSION_UNSUPPORTED` | 不支持请求体中的主版本 |
| 409 | `IDEMPOTENCY_CONFLICT` | 同一 `request_id`／`event_id` 被用于不同内容 |
| 409 | `BARCODE_CONFLICT` | 同一条码已创建不同内容的预评单 |
| 409 | `STATE_CONFLICT` | 卡片／任务状态不允许当前动作，或卡片不属于该条码／任务 |
| 503 | `TEMPORARILY_UNAVAILABLE` | 暂时无法持久化或处理；用同一业务键补查或重试 |

普通业务核查问题（例如报告数量与预计数量不符、图片暂不可用、单卡拟退金额超过该卡实付修复费）在**已安全持久化**时返回 HTTP 200，并在事件查询中显示 `held` 与原因代码；不得因反复推送而自动覆盖已发布或已付款数据。接收方必须保留回调原文摘要、接收时间和处理状态；日志中不能记录签名密钥、完整收件电话或敏感图片访问凭据。

## 5. 探卡平台调用修复团队

本节接口由修复团队提供，探卡平台调用。创建接口成功应在持久化业务键后应答；团队内部异步执行结果由查询接口或回调提供。

### 5.1 `POST /openapi/assessment/create`

创建一袋预评单。`barcode` 是团队侧唯一业务键；不要求团队先返回实点卡片或卡号。请求体：

```json
{
  "schema_version":"1","request_id":"bag-create-BAG001","barcode":"BAG001",
  "platform_order_no":"PA202610090001","expected_card_count":1,
  "handover_note":"条码以外的实物信息以实际预评为准"
}
```

| 字段 | 必填 | 校验 |
| --- | --- | --- |
| `platform_order_no` | 是 | 平台预评订单号，非空字符串 |
| `expected_card_count` | 是 | 正整数，仅供最终报告核对；不触发逐卡回执 |
| `handover_note` | 否 | 交接说明，字符串；不作为卡片清单 |

响应 `data`：

```json
{"barcode":"BAG001","status":"accepted"}
```

相同 `request_id` 和报文重试返回原结果；同一 `barcode` 对应不同创建内容返回 `BARCODE_CONFLICT`，不得生成第二张预评单。

### 5.2 `POST /openapi/assessment/query`

请求 `{"schema_version":"1","barcode":"BAG001"}`。响应 `data`：

```json
{"barcode":"BAG001","status":"assessing","report_ready":false}
```

`status` 为 `accepted`、`assessing`、`report_ready` 之一；`report_ready=true` 只表示团队已有整袋内容，平台仍须校验回调或使用报告补查接口读取内容。查询不存在的条码返回 `NOT_FOUND`。

### 5.3 `POST /openapi/reports/query`

平台在报告回调超时或漏收时补查。预评请求 `{"schema_version":"1","report_type":"assessment","barcode":"BAG001"}`；修复请求还须带 `task_id`：

```json
{"schema_version":"1","report_type":"repair","barcode":"BAG001","task_id":"TASK001"}
```

`report_type` 仅为 `assessment` 或 `repair`。未找到报告时响应 `data` 为 `{"found":false}`；找到时 `data` 必须含 `found=true`、原回调的 `event_id` 和 `report` 对象。`report` 必须是第 6.1 或 6.2 节定义的**完整原始报告业务内容**，包括相同的 `schema_version`、`event_id`、`barcode`、`cards`，不能只返回摘要或 PDF 地址。查询不得创建新报告、改写原事件号或修改报价。若团队内部存在更正，返回其明确标记为最新待处理的完整内容；平台仍按更正规则核查，不直接覆盖已付款快照。

### 5.4 `POST /openapi/tasks/repair/create`

仅在相应修复费支付确认后下发。用户按卡购买修复；任务只列出已付款卡片，不传项目序号或项目报价。

```json
{
  "schema_version":"1","request_id":"repair-TASK001","task_id":"TASK001",
  "barcode":"BAG001","cards":["BAG001-01"]
}
```

`cards` 是非空、无重复的团队卡号数组；每张卡必须属于该条码，且平台已按预评时冻结的单卡修复总价确认付款。团队不得自行增加收费项目后直接开工。响应 `data`：

```json
{"task_id":"TASK001","partner_task_id":"RT001","status":"accepted"}
```

`partner_task_id` 仅用于团队任务对账，不替代平台 `task_id`。相同 `task_id` 但不同卡片清单返回 `STATE_CONFLICT`；团队受理不等于已开工或已修复。

### 5.5 `POST /openapi/tasks/grading/create`

用户直接送评或修复后转送评，且平台已确认送评付款条件时下发。单个任务的所有卡共用一条四级评级路径及单价快照；任务可包含多张同方向卡。

```json
{
  "schema_version":"1","request_id":"grading-TASK002","task_id":"TASK002",
  "barcode":"BAG001","platform_grading_order_no":"GO202610090001",
  "cards":["BAG001-01"],
  "grading_selection":{
    "agency_id":1,"agency_code":"A01","agency_name":"评级机构",
    "service_id":11,"service_code":"S01","service_name":"送评",
    "type_id":21,"type_code":"T01","type_name":"单卡评级",
    "tier_id":31,"tier_code":"R01","tier_name":"常规档",
    "unit_price":"548.00"
  }
}
```

`platform_grading_order_no`、`cards` 与 `grading_selection` 必填；四级 ID、代码、名称、单价是平台下单时固定的路径快照。团队不得自行切换机构、服务、类型或档位。评级机构寄送地址**不来自**四级价格目录；正式启用此任务前需单独交付地址及团队业务映射。成功响应 `{"task_id":"TASK002","partner_task_id":"GT002","status":"accepted"}`。

### 5.6 `POST /openapi/tasks/return/create`

平台确认应寄回卡片已具备统一发货条件后，按袋下发一批寄回指令。团队不得仅因单卡被用户标记寄回就自行拆分发货。

```json
{
  "schema_version":"1","request_id":"return-TASK003","task_id":"TASK003",
  "barcode":"BAG001","return_batch_id":"RB001","cards":["BAG001-01"],
  "recipient":{
    "name":"收件人","phone":"13800000000","province":"广东省",
    "city":"深圳市","district":"南山区","address":"示例详细地址"
  }
}
```

`return_batch_id`、非空且无重复的卡号数组、收件人姓名／电话／省市区／详细地址均必填。收件地址仅用于本任务，团队不得据此修改用户地址簿。成功响应 `{"task_id":"TASK003","partner_task_id":"BT003","status":"accepted"}`；实际承运商和运单号须由 `/partner/repair-team/task/event` 回传。

### 5.7 `POST /openapi/tasks/query`

请求 `{"schema_version":"1","task_id":"TASK003"}`。响应 `data` 示例：

```json
{
  "task_id":"TASK003","partner_task_id":"BT003","task_type":"return",
  "status":"in_progress","latest_event_type":"return_shipped",
  "latest_event_at":"2026-10-11T10:00:00+08:00"
}
```

`task_type` 为 `repair`、`grading`、`return`；`status` 为 `accepted`、`in_progress`、`completed`、`rejected`。`completed` 只表示团队侧该任务履约完成，不替代平台的用户签收、退款成功或管理员录入评级出分。任务不存在返回 `NOT_FOUND`；查询不能创建新任务。

## 6. 修复团队调用探卡平台

本节接口由探卡平台提供，修复团队调用。回调在通过签名及基础格式校验后先持久化，再异步处理业务；HTTP 200 是接收回执，处理结果可用第 6.4 节查询。不得直接调用客户端或后台接口代用户选择、确认付款、发起退款或填写评级出分。

### 6.1 `POST /partner/repair-team/assessment/report`

整袋所有卡预评完成后，按条码一次性推送完整结构化内容。没有 `assessment_report_id`、`revision`、`assessment_item_id`、PDF 或媒体上传字段。

```json
{
  "schema_version":"1","event_id":"assessment-event-001","barcode":"BAG001",
  "completed_at":"2026-10-09T16:00:00+08:00","reported_card_count":1,
  "cards":[{
    "partner_card_code":"BAG001-01","card_sequence":1,
    "name":"卡牌名称","card_type":"宝可梦",
    "damage_summary":"毛边、露白、油污",
    "inspection_image_url":"https://images.example.com/cards/001-inspection.jpg",
    "inspection_image_caption":"卡牌及居中检测图",
    "centering":{
      "left_percent":"51.1","right_percent":"48.9",
      "top_percent":"47.3","bottom_percent":"52.7",
      "left_right_evaluation":"优秀（55/45以内）",
      "top_bottom_evaluation":"优秀（55/45以内）"
    },
    "assessment_score":"8.0","grading_recommendation":"谨慎送评",
    "score_breakdown":{
      "centering_score":"10.0","condition_score":"8.0","era_multiplier":"1.00"
    },
    "issues":[
      {"issue_no":1,"side":"back","area":"zone_2","name":"毛边",
       "description":"背面二区边缘存在毛边",
       "evidence_image_nos":["1.1"]},
      {"issue_no":2,"side":"front","area":"zone_1","name":"污渍",
       "description":"正面一区存在污渍",
       "evidence_image_nos":["1.2"]}
    ],
    "evidence_images":[
      {"image_no":"1.1","image_url":"https://images.example.com/issues/001.jpg",
       "caption":"背面二区毛边"},
      {"image_no":"1.2","image_url":"https://images.example.com/issues/002.jpg",
       "caption":"正面一区污渍"}
    ],
    "repair_supported":true,"repair_quote_total":"200.00"
  }]
}
```

| 字段 | 必填 | 约束及业务含义 |
| --- | --- | --- |
| `completed_at`、`reported_card_count`、`cards` | 是 | 完成时间；卡数为正整数且等于 `cards.length`，`cards` 为整袋全量结果且不能为空 |
| `partner_card_code`、`card_sequence`、`name` | 每张卡 | 团队卡号同袋唯一且后续不变；序号为 1 至 `reported_card_count` 的整数，同袋不重复；卡名非空 |
| `card_type`、`damage_summary` | 每张卡 | 卡牌类型、卡损总述，供报告展示；无明显卡损也须给出明确说明 |
| `inspection_image_url`、`inspection_image_caption` | 每张卡 | 卡牌及居中检测合成图的 HTTPS 地址与图注；地址必填，图注可选 |
| `centering` | 可测量时 | L/R、T/B 四个百分比为不含 `%` 的数字字符串，两个评价为非空字符串；无法测量时可不传 |
| `assessment_score`、`grading_recommendation` | 否 | 第三方预评意见，不是评级机构正式出分或承诺 |
| `score_breakdown` | 有预评分时 | `centering_score`、`condition_score`、`era_multiplier` 均为数字字符串，分别对应居中、卡况、时代倍率，仅供展示 |
| `issues` | 每张卡 | 数组，可为空；每个 `issue_no` 在当前卡内唯一且为正整数 |
| `issues[].side/area/name/description` | 每个缺陷 | 位置与描述；`side` 为 `front` 或 `back`，其余为非空字符串 |
| `issues[].evidence_image_nos` | 每个缺陷 | 引用当前卡 `evidence_images[].image_no` 的数组，可为空；同图可被多个缺陷引用 |
| `evidence_images` | 每张卡 | 证据图数组，可为空；每项含卡内唯一 `image_no`、HTTPS `image_url` 和非空 `caption`，供图号及逐图说明展示 |
| `repair_supported` | 每张卡 | 布尔值，表示是否可购买修复 |
| `repair_unsupported_reason` | 不可修复时 | 非空字符串，说明不可修复原因 |
| `repair_quote_total` | 每张卡 | 单卡修复总价，两位小数字符串；可修复时大于 `"0.00"`，不可修复时为 `"0.00"`，不拆分项目报价 |

`card_sequence` 表示报告中的“第几组／第几张”，不从团队卡号后缀推断。例如第007组且卡牌顺序为007/013时，该卡传 `card_sequence:7`，整袋传 `reported_card_count:13` 并包含全部13张卡；客户端将序号和总数补前导零后显示。证据图 `image_no` 可传 `7.1`、`7.2`，图号及图注与缺陷位置一起展示。本期只按样本接收卡牌及居中检测图、卡损证据图，不要求独立的正面或背面原图。团队报告的 `reported_card_count` 与数组长度不一致属于请求格式错误；与平台预期卡数不一致则保留回调并进入 `held` 核查，不自动删卡或补卡。团队卡号与平台已建立的映射冲突时也进入核查。平台发布前校验图片引用的可用性；图片访问域名、授权及有效期须在联调资料中确定，不能让后端任意抓取未经允许的 URL。

样本截图展示的单卡修复总价对应 `repair_quote_total`。平台按该卡冻结总价收款；后续若部分修复，团队在修复报告中直接回传该卡拟退总额，平台不根据卡损描述或证据图拆算项目金额。

成功响应 `data`：

```json
{"event_id":"assessment-event-001","status":"received"}
```

`received` 仅表示已持久化；平台完成校验后可能变为 `applied` 或 `held`。同一 `event_id`、同内容重复推送返回原回执；同 ID 不同内容返回 `IDEMPOTENCY_CONFLICT`。同条码报告更正须生成新 `event_id` 并推送整袋全量内容，不能只推变更卡；已有用户选择、付款或下发任务时，更正先 `held`，不得自动改价。

### 6.2 `POST /partner/repair-team/repair/report`

团队完成平台已下发的修复任务后，按 `task_id + barcode` 推送最终逐卡结果和单卡拟退总额，不推 PDF 文件。团队回传拟退金额不等于直接发起退款；平台校验并执行退款。

```json
{
  "schema_version":"1","event_id":"repair-event-001","task_id":"TASK001",
  "barcode":"BAG001","completed_at":"2026-10-10T16:00:00+08:00",
  "cards":[{
    "partner_card_code":"BAG001-01","status":"partially_completed",
    "result_summary":"部分毛边已修复，撞边露白未修复",
    "score_before":"8.0","score_after":"8.5",
    "refund_amount":"80.00","refund_reason":"撞边露白未修复",
    "report_entries":[{
      "entry_no":"1.1","status":"repaired",
      "title":"月亮伊布 背面2区毛边",
      "original_defect_description":"背面2区选区存在毛边缺陷。",
      "before_image_url":"https://images.example.com/repair/001-before.jpg",
      "after_image_url":"https://images.example.com/repair/001-after.jpg",
      "repair_explanation":"针对该缺陷完成局部修复与外观整理。"
    },{
      "entry_no":"1.4","status":"not_repaired",
      "title":"月亮伊布 正面1区撞边露白",
      "original_defect_description":"正面1区存在撞边露白缺陷。",
      "before_image_url":"https://images.example.com/repair/004-original.jpg",
      "treatment_conclusion":"撞边露白",
      "damage_cause":"边缘受到外力碰撞，印刷层磨损，露出纸基颜色。",
      "repair_recommendation":"为保持长期保存状态，建议不进行补色处理。"
    }]
  }]
}
```

`completed_at`、`cards`、每张卡的 `partner_card_code`、`status`、`result_summary`、`refund_amount` 与 `report_entries` 必填；`cards` 必须覆盖该任务中的全部卡片，每张恰好一次。`score_before`、`score_after` 是修复前后评估值，可不提供，不是正式评级分。逐条展示字段如下：

| 字段 | 必填 | 约束及展示含义 |
| --- | --- | --- |
| `report_entries` | 每张卡 | 非空数组，按报告展示顺序包含该卡全部修复对比与未修复缺陷；不代表收费项目 |
| `entry_no`、`status`、`title` | 每条 | `entry_no` 为该卡内唯一的展示编号字符串，如 `1.1`；`status` 仅为 `repaired` 或 `not_repaired`；`title` 为非空展示标题，可含卡名、位置及缺陷名 |
| `original_defect_description`、`before_image_url` | 每条 | 原缺陷描述和修复前／原缺陷证据图；描述非空，图片为可访问的 HTTPS 地址 |
| `after_image_url`、`repair_explanation` | `repaired` 时 | 与本条 `before_image_url` 一一配对的修复后 HTTPS 图片和非空修复说明；不得依赖两个独立图片数组的位置配对 |
| `treatment_conclusion`、`damage_cause`、`repair_recommendation` | `not_repaired` 时 | 非空的处理结论、损伤原因、修复建议；不提供 `after_image_url`，不得用原图冒充修复后图片 |

卡级 `status` 仅为 `completed`、`partially_completed`、`not_completed`，分别要求 `report_entries` 全部为 `repaired`、两种条目均存在、全部为 `not_repaired`。这些条目仅供展示和解释处理结果，不含条目单价或条目退款金额，也不用于计算卡级退款。卡级拟退金额 `refund_amount` 为两位小数字符串，平台校验 `0 ≤ refund_amount ≤ 该卡实付修复费`。`completed` 时拟退 `"0.00"`；`not_completed` 时拟退全部实付修复费；`partially_completed` 时拟退金额须大于零且小于实付修复费。拟退金额大于零时 `refund_reason` 必填。示例若该卡实付 `"200.00"` 元，则单卡拟退 `"80.00"` 元。金额、状态、展示条目或卡片归属冲突时保留回调并进入 `held` 核查，不能自动发起退款。报告已接收与渠道退款成功是不同状态。

成功响应 `{"event_id":"repair-event-001","status":"received"}`。相同 `event_id`、相同内容重试返回原回执；同一任务出现新事件且内容变化时保留原结果，可能进入 `held` 人工复核，不得静默重算既有退款。

### 6.3 `POST /partner/repair-team/task/event`

团队只回传已存在任务的履约事实。`event_type` 支持 `accepted`、`rejected`、`repair_started`、`grading_shipped`、`grading_received_back`、`return_shipped`、`exception`。事件发生时间使用 `occurred_at`，同一任务事件可乱序到达，平台按业务状态校验而非仅按接收顺序推进。

```json
{
  "schema_version":"1","event_id":"ship-event-001","task_id":"TASK003",
  "barcode":"BAG001","event_type":"return_shipped",
  "occurred_at":"2026-10-11T10:00:00+08:00",
  "partner_card_codes":["BAG001-01"],
  "shipment":{"carrier_code":"SF","tracking_no":"123456789012"}
}
```

`partner_card_codes` 为该任务内发生该事件的非空、无重复卡号数组。`grading_shipped`、`return_shipped` 必须传 `shipment.carrier_code` 和 `shipment.tracking_no`；`rejected`、`exception` 必须传非空 `reason_code` 和 `reason_message`。未知任务可先持久化为 `held` 等待关联或核查；已知任务但条码、卡片归属冲突时返回 `STATE_CONFLICT`，绝不自动创建新订单。寄回发货不等于用户已签收；寄往评级机构不等于已出分。成功响应 `{"event_id":"ship-event-001","status":"received"}`。

### 6.4 `POST /partner/repair-team/events/query`

团队在回执丢失、超时或需核对报告发布／任务事件状态时调用。请求 `{"schema_version":"1","event_id":"ship-event-001"}`。响应 `data` 示例：

```json
{"event_id":"ship-event-001","status":"applied","reason_code":""}
```

`status` 为 `received`（已持久化）、`processing`（处理中）、`applied`（已应用）、`held`（待核查）、`rejected`（最终拒绝）之一。`held`、`rejected` 时 `reason_code` 必须是非空的稳定机器码；建议首批使用 `CARD_COUNT_MISMATCH`、`IDENTITY_MISMATCH`、`REFUND_AMOUNT_INVALID`、`IMAGE_UNAVAILABLE`、`TASK_NOT_FOUND`、`STATE_CONFLICT`。响应不回显收件电话、签名或图片访问凭据。未知事件返回 `NOT_FOUND`；该查询不重新处理事件。

## 7. 幂等、补偿与联调验收

| 业务对象 | 唯一键与重复行为 | 结果未决时的恢复 |
| --- | --- | --- |
| 预评袋单 | `barcode` 唯一；`request_id` 固定；相同请求返回原结果，条码内容冲突返回 409 | 查询 `/openapi/assessment/query`，不存在再用原 `request_id` 重试 |
| 修复／送评／寄回任务 | `task_id` 唯一；创建报文不可在重试时改变 | 查询 `/openapi/tasks/query`，不存在再用原 `request_id` 与 `task_id` 重试 |
| 报告／履约回调 | `event_id` 唯一；相同内容重投返回原回执 | 查询 `/partner/repair-team/events/query`，不存在再用原 `event_id` 重投 |
| 报告更正 | 新 `event_id` 加整袋预评全量内容或对应修复任务全量结果 | 原内容保留；影响已选择、已付款或已退款记录时进入人工核查 |

双方在本地事务中保存待发送任务或回调及幂等键，事务提交后发起 HTTP 调用；接收方在 ACK 前持久化原文摘要和状态。查询接口不产生新的订单、任务或报告。系统异常不得以新条码、新任务号或新事件号“重试”同一业务操作。

联调至少覆盖以下场景：

1. 签名测试向量、JSON 空白变化、过期时间戳，以及有效时间窗口内重复投递只产生一次业务效果。
2. 相同业务键重试、同键不同内容冲突、调用超时补查、回调乱序与任务不存在。
3. 整袋报告卡数不符、图片失效不发布报告、单卡修复总价冻结；修复报告中前后图片不配对、未修复条目伪造修复后图片、展示编号重复或卡级与条目状态不符时进入核查。
4. 单卡全部未修复全额退款、部分修复按单卡拟退金额退款、拟退金额超出实付金额时拦截，以及同袋混合修复、直接送评、统一寄回。

## 8. 联调配置

双方在联调前交换并确认以下配置。地址和密钥均不放入业务请求体；示例域名不得用于正式调用。

| 配置 | 需要确认的内容 |
| --- | --- |
| 接口地址 | 双方各自提供沙箱与正式环境的 HTTPS Base URL，例如 `https://partner.example.com`；Base URL 不含路径、查询串或末尾 `/`，调用方在其后拼接第 1 节对应提供方的接口路径 |
| 签名密钥 | 每个调用方向、每个环境分别交付独立密钥，约定安全交付及轮换方式；使用时先对 Base64 配置解码 |
| 图片访问 | 图片域名、访问授权方式、有效期与报告发布后的长期可用方案 |
| 送评映射 | 评级机构实际寄送地址来源及四级评级选择对应的团队业务代码 |
| 运行参数 | 请求超时、限流、重试间隔、事件保留时间和双方服务器时钟校准方式 |

上述配置不改变接口报文结构；如果需要修改字段或签名规则，双方应先更新并确认同一版接口文档。

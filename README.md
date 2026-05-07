# ozon
AI自动上品测试


好的，我为你详细规划 **全自动采集 + 分析 + 上架** 方案的技术实现步骤，以及每个环节的潜在问题和调试方法。这个方案适合有一定编程基础（Python）的卖家，或愿意请人开发的团队。

---

## 一、整体架构

```
定时触发 (cron/任务计划)
    │
    ▼
模块1：数据采集 ──→ 商品原始数据 (CSV/DB)
    │
    ▼
模块2：AI分析筛选 ──→ 待上架商品列表
    │
    ▼
模块3：素材生成 ──→ 标题/描述/图片URL
    │
    ▼
模块4：Ozon API 上架 ──→ 成功/失败日志
    │
    ▼
结束，发送通知（微信/邮件）
```

---

## 二、模块详细实现步骤

### ✅ 模块1：数据采集

**目标**：从Ozon或其他平台获取热销商品数据（至少包含商品名称、价格、销量、图片URL）

#### 实现方式（推荐顺序）
1. **Ozon Partner API 采集自有类目热销榜**（安全、合法）  
   - 注册Ozon卖家账号，获取 API-Key 和 Client-Id  
   - 调用 `GET /v1/analytics/stock`（库存分析）或 `GET /v1/analytics/top-products`（如果有）  
   - 如果没有直接接口，可用 `GET /v1/product/list` 获取自己的商品列表，再通过销量排序

2. **第三方数据服务**（推荐新手）  
   购买 MPStats / SellerFox 的Excel导出功能，每天下载一次，然后本地脚本读取

3. **爬虫（高风险，不推荐）**  
   - 使用 `requests` + `BeautifulSoup` 采集Ozon搜索结果页  
   - 需要轮换User-Agent、代理IP，降低频率（比如 1请求/5秒）  
   - 容易被封IP，甚至封店，建议放弃

#### 可能遇到的问题
- **API 调用权限不足**：某些数据接口需要申请开通，或只对高级卖家开放。  
  解决：调用 `GET /v1/category/tree` 测试API是否可用，若返回403需联系Ozon支持。
- **数据格式不统一**：字段名、单位、价格可能带符号。  
  解决：用Python清洗，如 `price = float(price_str.replace('₽', '').replace(' ', ''))`
- **数据量小**：API可能只返回部分商品（如前100）。  
  解决：通过分页参数 `page` 和 `page_size` 循环获取更多。

#### 调试方法
```python
import requests
url = "https://api-seller.ozon.ru/v1/product/list"
headers = {"Client-Id": "xxx", "Api-Key": "xxx"}
payload = {"page": 1, "page_size": 10, "filter": {"visibility": "ALL"}}
resp = requests.post(url, headers=headers, json=payload)
print(resp.status_code, resp.text)   # 查看原始返回
```
如果返回 `{"code": 401, "message": "Invalid token"}`，说明API密钥错误。

---

### ✅ 模块2：AI分析筛选

**目标**：从采集到的商品中，筛选出“利润高、竞争小、适合你店铺”的商品。

#### 实现方式
- **阈值规则**：销量≥100，评分≥4.5，利润率≥30%  
- **AI评分**：调用 OpenAI API 或 本地模型，输入商品标题、描述，让AI输出1-5分的“爆款潜力值”
- **训练模型**（高级）：如果你有历史爆款数据，可用 `sklearn` 训练一个分类器

**简单两步骤**：
1. 先用规则过滤掉明显不行的商品
2. 对剩余商品调用OpenAI API进行打分，筛选分数≥4的商品

#### 示例代码（OpenAI调用）
```python
import openai
openai.api_key = "sk-xxx"

def evaluate_product(title, description):
    prompt = f"""你是跨境电商选品专家。判断以下商品是否适合在Ozon俄罗斯销售。返回0-5整数分，5分表示爆款潜力很大。只返回数字。
标题：{title}
描述：{description}"""
    response = openai.Completion.create(engine="gpt-3.5-turbo-instruct", prompt=prompt, max_tokens=2)
    return int(response.choices[0].text.strip())
```

#### 可能遇到的问题
- **API调用成本**：每个商品几美分，每天几百个品成本可控  
- **分数不稳定**：相同商品两次调用可能不同分。解决：设置阈值取多数投票，或用结构化输出。
- **中文vs俄语**：建议将标题翻译成俄语再给AI分析（更符合平台语境）。先用谷歌翻译API或googletrans库。

#### 调试
打印每次的prompt和响应：
```python
print(f"Prompt: {prompt}\nResponse: {response.choices[0].text}")
```
如果AI一直返回“1”或“2”，检查prompt是否明确要求只输出数字。

---

### ✅ 模块3：素材生成

**目标**：为筛选出的商品生成俄语标题、描述、图片。

#### 实现方式
- **标题生成**：使用OpenAI API（或本地模型），提供种子关键词（品牌+品类+属性）
- **描述生成**：模板 + AI润色。例如：“[品牌] [产品] – 高品质[材料]，[适合场景]，[核心卖点]。送[赠品]。”
- **图片处理**：从货源网站（如1688）下载主图，用`PIL`添加俄语文字或调整尺寸；或直接用原图（确保版权）

#### 示例提示词（俄语）
```
Ты эксперт по маркетингу на Ozon. Создай заголовок (до 150 символов) и описание (200-300 символов) для товара.  
Товар: {product_name}  
Характеристики: {features}  
Ключевые слова: {keywords}  
Верни в формате: Заголовок: ... Описание: ...
```

#### 可能遇到的问题
- **生成内容质量差**：需要多轮调prompt，加入示例。
- **图片版权**：不要直接复制他人图片，否则被投诉。建议用无版权图库或自己拍摄。
- **俄语语法错误**：可以用 `deep-translator` 或 调用Yandex翻译API校验。

#### 调试
单独测试一个样本，观察生成的标题长度是否超过150字符（Ozon限制）。用Python计算长度 `len(title.encode('utf-8'))`。

---

### ✅ 模块4：Ozon API 上架

**目标**：将生成的商品数据通过API发布到Ozon店铺。

#### 实现方式
- 调用 `POST /v2/product/create` 创建商品（先创建，再更新库存/价格）
- 图片上传需先调用 `POST /v2/product/import/images` 上传到Ozon CDN，获得URL后填入商品创建请求
- 每次上架后记录 `offer_id`，避免重复创建

#### 核心代码片段
```python
def create_product(product_data):
    url = "https://api-seller.ozon.ru/v2/product/create"
    payload = {
        "offer_id": product_data["sku"],
        "name": product_data["title"],
        "description": product_data["description"],
        "price": product_data["price"],
        "images": product_data["image_urls"],
        # 其他必填字段：category_id, vat, attributes等
    }
    resp = requests.post(url, headers=headers, json=payload)
    if resp.status_code != 200:
        log_error(resp.text)
        return False
    return True
```

#### 可能遇到的问题
- **缺少必填字段**：Ozon要求某些品类必须有特定属性（如品牌、尺寸）。提前从API `GET /v1/category/attribute` 获取该类目需要的属性列表。
- **图片上传失败**：图片URL必须可公网访问，且大小<5MB。建议先用 `requests.head` 检测URL有效性。
- **频率限制**：Ozon API默认每秒最多2次请求。加入 `time.sleep(0.5)` 或使用 `ratelimit` 库。

#### 调试
启用详细的日志，记录每个API请求的 request 和 response。
```python
import logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
# 在请求前后记录
logger.info(f"Request: {payload}")
logger.info(f"Response: {resp.text}")
```
查看失败时的错误码（如400表示参数错误，500表示服务器问题）。

---

## 三、部署与调度

### 方案选择
- **Windows任务计划程序**：适合在自己电脑上跑，但需要24小时开机  
- **云服务器（Linux）**：使用cron每天执行一次  
  ```bash
  # 每天凌晨3点运行
  0 3 * * * cd /home/ozon_bot && python main.py >> logs.txt 2>&1
  ```
- **无服务器函数（AWS Lambda）**：更便宜，但需要注意执行时间限制（15分钟）

### 环境依赖
创建 `requirements.txt`：
```
requests
pandas
openai
pillow
python-dotenv
```
使用虚拟环境。

### 通知机制
脚本结束时发送微信通知（Server酱）或邮件：
```python
import requests
def send_wechat(msg):
    requests.get(f"https://sc.ftqq.com/你的key.send?text={msg}")
```

---

## 四、常见问题与调试指南（汇总表）

| 模块 | 典型错误 | 可能原因 | 调试方法 |
|------|----------|----------|----------|
| 采集 | `HTTP 403` | API密钥失效或IP被禁 | 检查密钥，尝试更换网络代理 |
| 采集 | 数据为空 | 分页参数错误 | 打印请求参数，手动在Postman测试 |
| 分析 | OpenAI返回空 | API额度用完或网络问题 | 检查余额，添加重试机制 |
| 分析 | 分数都是5分 | prompt没有约束输出范围 | 明确要求1-5整数 |
| 素材生成 | 标题长度超限 | Ozon限制150字符，中文/俄文都算1个 | 自动截断并提示 |
| 上架 | `{"code": 403, "message": "access denied"}` | API密钥无商品创建权限 | 在Ozon后台确认API角色是否包含“商品管理” |
| 上架 | 图片上传失败 | 图片URL需要公网可达 | 下载图片再上传到临时OSS或直接用base64? Ozon不支持base64，必须URL |
| 整体 | 脚本异常停止 | 未捕获异常 | 用 `try...except` 包裹主逻辑，记录异常到文件 |

---

## 五、调试实用工具

- **Postman**：测试单个API请求，保存为代码片段
- **ngrok**：如果你需要本地调试点，可以用ngrok暴露本地的webhook
- **curl**：快速测试Ozon API
  ```bash
  curl -X POST https://api-seller.ozon.ru/v2/product/create \
    -H "Client-Id: xxx" -H "Api-Key: xxx" \
    -H "Content-Type: application/json" \
    -d '{"offer_id":"test"}'
  ```
- **Python调试**：使用 `pdb` 或 `ipdb` 设置断点
  ```python
  import pdb; pdb.set_trace()
  ```

---

## 六、建议开发顺序（降低风险）

1. **先手动**：用Postman或curl成功调用一次“创建商品”API，确认账号权限正常。
2. **再写脚本**：写一个最简单的Python脚本，固定一个商品数据，能通过API上架。
3. **逐步集成**：加入素材生成（先只用模板填充，不用AI）→ 加入AI筛选（先用规则，后加AI）→ 加入采集模块。
4. **小批量运行**：每天只处理5-10个商品，观察一周无异常后增加数量。

---

## 七、最终提醒

> 全自动方案维护成本不低，建议你至少保持每周查看一次日志，关注API变动和平台政策。  
> 如果觉得开发成本过高，可以在猪八戒/Fiverr上雇佣一位Python开发者，预算大概2000-5000元搭建基础框架。

如果你需要，我可以为你提供一个最小可运行的脚本框架（采集+AI分析+上架），填入密钥就能跑通一个商品。告诉我是否要这个模板。

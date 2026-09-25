# 野外观测记录匿名化 API

为自然保护站公开发布野外观测记录而设计的匿名化服务（Python + FastAPI）。
调用方提交一批 JSON 观测记录、正整数 **k**，以及最多 **3 个**待匿名字段的
泛化层级映射；服务对**全部记录统一**选择每个字段的泛化层级，使发布后每一种
字段值组合（等价组）都至少出现 **k** 次，并在所有可行方案中最小化层级序号之和。

服务为纯内存、无状态处理：**原始记录不写日志、不落盘、不挂载数据卷**。

## 一条命令启动

```bash
docker compose up --build
```

启动后：

- API 地址与端口：`http://localhost:8000`
- 匿名化接口：`POST http://localhost:8000/api/v1/anonymize`
- 健康检查：`GET http://localhost:8000/api/v1/health`
- 交互式文档（Swagger UI）：`http://localhost:8000/docs`

如需改用其他宿主机端口（容器内固定监听 8000）：

```bash
PORT=9000 docker compose up --build
# 此时 API 地址为 http://localhost:9000
```

也可以不使用 Compose：

```bash
docker build -t wildlife-anonymizer .
docker run --rm -p 8000:8000 wildlife-anonymizer
```

## 必要配置

| 配置 | 位置 | 默认值 | 说明 |
| --- | --- | --- | --- |
| 宿主机端口 | Compose 环境变量 `PORT` | `8000` | `PORT=9000 docker compose up --build` 映射到 9000 |
| 容器内端口 | 容器环境变量 `PORT` | `8000` | 一般无需修改；容器以非 root 用户 `nobody` 运行 |

无数据库、无密钥、无其他依赖。容器以只读根文件系统运行，不挂载任何卷。

## 请求格式

`POST /api/v1/anonymize`，`Content-Type: application/json`：

```json
{
  "k": 3,
  "records": [
    {"id": "obs-001", "species": "金丝猴", "reserve": "神农架-老君山", "observer": "志愿者A"},
    {"id": "obs-002", "species": "金丝猴", "reserve": "神农架-老君山", "observer": "志愿者B"}
  ],
  "fields": {
    "species": {
      "金丝猴": ["金丝猴", "灵长类", "哺乳动物", "*"],
      "林麝":   ["林麝",   "有蹄类", "哺乳动物", "*"],
      "云豹":   ["云豹",   "食肉目", "哺乳动物", "*"]
    },
    "reserve": {
      "神农架-老君山": ["神农架-老君山", "神农架", "湖北", "*"],
      "神农架-大九湖": ["神农架-大九湖", "神农架", "湖北", "*"],
      "卧龙-邓生":     ["卧龙-邓生",     "卧龙",   "四川", "*"],
      "卧龙-耿达":     ["卧龙-耿达",     "卧龙",   "四川", "*"]
    }
  }
}
```

### 字段说明

- `k`：正整数，发布后每种匿名字段值组合至少出现的记录数。
- `records`：原始观测记录数组，每条为任意 JSON 对象。未列入 `fields` 的字段
  （如示例中的 `id`、`observer`）原样返回，由调用方自行决定是否提交。
- `fields`：最多 3 个待匿名字段。每个字段给出“原值 → 层级路径”的映射：
  - 路径为字符串数组，**1～4 级**，同一字段内所有路径长度必须一致；
  - 路径首项必须等于原值本身（即 **0 级 = 原值**，不泛化）；
  - 路径末项必须是该字段**统一的通配符**（如上例的 `"*"`，即最高 1 级泛化）；
  - 中间各级为该原值在对应层级的泛化值。
  - 层级序号：`0` 为原值，`1` 起逐级泛化，例如 4 级路径的层级为 `0/1/2/3`。

### 选层规则

对所有记录使用同一组层级：

1. 枚举每个字段的全部层级组合（最多 `4³ = 64` 种）；
2. 只保留泛化后每个等价组记录数 `>= k` 的可行组合；
3. 最小化层级序号之和；
4. 和相同时，按**字段名排序后**的层级序号做**字典序**裁决
   （例如字段排序后为 `[reserve, species]`，`(2,3)` 优先于 `(3,2)`）。

## 响应格式

```json
{
  "generalized_records": [
    {"id": "obs-001", "species": "哺乳动物", "reserve": "神农架", "observer": "志愿者A"}
  ],
  "selected_levels": {"reserve": 1, "species": 2},
  "groups": [
    {"values": {"reserve": "神农架", "species": "哺乳动物"}, "count": 4},
    {"values": {"reserve": "卧龙",   "species": "哺乳动物"}, "count": 4}
  ],
  "k": 3,
  "total_records": 8
}
```

- `generalized_records`：泛化后的记录（与输入等长、等序；非匿名字段原样保留）。
- `selected_levels`：每个字段选中的层级序号。
- `groups`：发布后的各字段值组合及数量，`count` 均 `>= k`；按组在记录中
  首次出现的顺序返回。

仓库内 `examples/request.json` 为可直接使用的完整示例：

```bash
curl -s http://localhost:8000/api/v1/anonymize \
  -H 'Content-Type: application/json' \
  --data @examples/request.json
```

## 422 校验错误

以下情况返回 HTTP `422 Unprocessable Entity`（FastAPI 的请求体结构错误同样为
422），错误说明在响应体 `detail` 中：

- 记录总数小于 `k`；
- `k` 不是正整数（含 `0`、负数、布尔值）；
- 待匿名字段超过 3 个；
- 映射缺值：记录中出现的某原值（或非字符串值）在字段映射中不存在；
- 层级路径为空、超过 4 级、同字段路径不等长；
- 路径首项不等于原值，或最高级没有收敛为同一个统一通配符；
- 泛化值或通配符为空字符串。

注：记录总数 `>= k` 时必然存在可行方案（所有字段选到统一通配符级后全部记录
并入同一组）。

## 数据安全

- 匿名化完全在请求处理过程中于内存完成，服务不保存任何请求内容；
- 不配置 Docker 数据卷，容器根文件系统只读；
- 请勿在反向代理/网关层开启会记录请求体的访问日志。

## 本地开发（不使用 Docker）

需要 Python 3.11+：

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements-dev.txt
uvicorn app.main:app --reload
pytest
```

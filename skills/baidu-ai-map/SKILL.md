---
name: baidu-ai-map
description: 百度地图Map Agent Plan，官方为 Agent专属AI地图技能，直连 place、direction、geocoding、reverse_geocoding、weather 五大核心能力，大模型一键调用地图服务。
license: MIT
version: 1.0.0
homepage: https://lbs.baidu.com
repository: https://github.com/baidu-maps/map-skills
metadata:
  openclaw:
    requires:
      bins: ["curl"]
      env: BAIDU_MAP_AUTH_TOKEN
    primaryEnv: BAIDU_MAP_AUTH_TOKEN
---

# 百度地图服务 Agent Plan

提供大模型直接调用百度地图能力，支持语义化 AI 搜索、智能路线规划、天气查询、地理编码与逆地理编码。

## 使用准则

### 准则 1：API 端点

所有能力统一使用：

> **Base URL**: `https://api.map.baidu.com/`

### 准则 2：SK 凭证安全处理

SK（Service Key）是调用所有 API 的必须凭证：

1. 优先读取环境变量 `BAIDU_MAP_AUTH_TOKEN`。
2. 如果为空，提示用户前往 [百度地图 Agent Plan](https://lbs.baidu.com/apiconsole/agentplan) 申请。
3. 设置环境变量：

```bash
export BAIDU_MAP_AUTH_TOKEN="你的SK"
```

### 准则 3：统一鉴权方式（参数传入）

调用所有 API 时，统一传入参数 `baidu_map_auth_token`：

- GET：放在 query 参数中
- POST：也放在 query 参数中（不放 body）

示例：

```bash
curl --get "https://api.map.baidu.com/agent_plan/v1/place" \
  --data-urlencode "user_raw_request=帮我找北京可带宠物的咖啡馆" \
  --data-urlencode "region=北京市" \
  --data-urlencode "baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN"
```

## 全局参数与行为约束

1. `user_raw_request` 必须是完整的用户需求，不可压缩为关键词。
2. `user_raw_request` 出现“我附近”等非明确地点时，可代替用户推理为具体地点。
3. `user_raw_request` 需保留定语/约束词，例如“评分最高”“最近”“最便宜”“3公里内”。
4. 不得编造坐标；`center` / `location` / `refer_pois` 仅可来自用户明确提供或可信来源。
5. 统一使用参数 `baidu_map_auth_token` 鉴权（GET/POST 都放 query）。
6. 经纬度至少保留小数点后 6 位。
7. 所有工具返回坐标类型统一为 `gcj02`。
8. 禁止使用 grep/sed/awk/jq/python脚本 正则裁剪响应后再推断字段，这会造成百度地图 Agent Plan 巨额的token消耗；短时间内遇到相同或高度相似请求时，应优先基于结果直接推理，避免重复发起请求。
9. `direction` 可能返回两类结果：路线结果，或起终点澄清结果。当返回起终点澄清结果时，应先根据用户需求推理最合理候选；若仍无法唯一确定，应引导用户在候选中选择后再继续规划。

## 工具详解

### 1. Place（语义化AI地点检索）

#### API

`GET /agent_plan/v1/place`

#### 参数输入（给模型）

Required:
- `user_raw_request`: 用户原始需求，原样完整传入，不可压缩为关键词；保留约束词（如“评分最高”“最近”“3公里内”）
- `region`: 城市或区域限制

Optional:
- `center`: 检索中心点和排序参考点（`lat,lng`，gcj02）
- `sort`: `distance` 或 `relevance`（默认 `relevance`）

Rules:
- `sort=distance` 时，`center` 必传
- `center` 只能来自用户明确提供或可信来源，禁止推测/编造
- 经纬度至少保留小数点后 6 位

#### 鉴权

- GET：query 参数 `baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN`

#### 示例

```bash
# 1) 帮我查一下八达岭长城附近的五星级酒店
curl --get "https://api.map.baidu.com/agent_plan/v1/place" \
  --data-urlencode "user_raw_request=帮我查一下八达岭长城附近的五星级酒店" \
  --data-urlencode "region=延庆区" \
  --data-urlencode "sort=relevance" \
  --data-urlencode "baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN"

# 2) 离我最近的火锅店（distance 排序）
curl --get "https://api.map.baidu.com/agent_plan/v1/place" \
  --data-urlencode "user_raw_request=离我最近的火锅店" \
  --data-urlencode "region=北京市" \
  --data-urlencode "center=40.056800,116.308300" \
  --data-urlencode "sort=distance" \
  --data-urlencode "baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN"
```

### 2. Direction（语义化AI路线规划）

#### API

`POST /agent_plan/v1/direction`

`Content-Type: application/x-www-form-urlencoded`

#### 参数输入

Required:
- `user_raw_request`: 用户原始需求，包含起点和终点；保留路线约束词，需要推理并改写用户原始的交通方式
- `location`: 用户当前位置坐标（`lat,lng`，gcj02）

Optional:
- `refer_pois`: 地点精确映射，格式 `地点名称:uid,纬度,经度;地点名称:uid,纬度,经度`

Rules:
- 当前仅支持驾车、步行、骑行、公交路线规划。`user_raw_request` 中出现其他交通方式时，需要改写到这四种交通方式
- `location` 当起点有歧义（如同名地标、模糊起点）时，会基于用当前位置推理最合理的起点位置
- `refer_pois` 默认不传，用于同名地点消歧，仅在query中明确有歧义地点时传入
- `refer_pois` / `location` 的经纬度至少保留小数点后 6 位，禁止推测/编造，可以调用Plcae或Geocoding获取

#### 鉴权

- POST：query 参数 `baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN`

#### 示例

```bash
# 1) 帮我规划从故宫到颐和园的驾车路线
curl -X POST "https://api.map.baidu.com/agent_plan/v1/direction?baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "user_raw_request=帮我规划从故宫到颐和园的驾车路线" \
  --data-urlencode "location=39.914590,116.403770"

# 2) “我家”别名映射
curl -X POST "https://api.map.baidu.com/agent_plan/v1/direction?baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "user_raw_request=步行去我家附近最近的中餐厅" \
  --data-urlencode "location=40.056800,116.308300" \
  --data-urlencode "refer_pois=我家:fbc88a21464370106e3e1b52,40.092180,116.345310"

# 2) 交通方式推理改写：从王府井打车去三里屯要多久
curl -X POST "https://api.map.baidu.com/agent_plan/v1/direction?baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "user_raw_request=从王府井驾车去三里屯要多久" \
  --data-urlencode "location=39.914590,116.403770"
```

### 3. Geocoding（地理编码）

#### API

`GET /agent_plan/v1/geocoding`

#### 参数输入

Required:
- `address`: 要解析的完整地址

Optional:
- `region`: 城市/区域提示（减少歧义）

Rules:
- 地址越完整，解析越稳定

#### 鉴权

- GET：query 参数 `baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN`

#### 示例

```bash
curl --get "https://api.map.baidu.com/agent_plan/v1/geocoding" \
  --data-urlencode "address=北京市海淀区上地十街10号百度大厦" \
  --data-urlencode "region=北京市" \
  --data-urlencode "baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN"
```

### 4. Reverse Geocoding（逆地理编码）

#### API

`GET /agent_plan/v1/reverse_geocoding`

#### 参数输入

Required:
- `location`: `lat,lng` 格式坐标（gcj02）

Rules:
- 经纬度至少保留小数点后 6 位

#### 鉴权

- GET：query 参数 `baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN`

#### 示例

```bash
curl --get "https://api.map.baidu.com/agent_plan/v1/reverse_geocoding" \
  --data-urlencode "location=40.056800,116.308300" \
  --data-urlencode "baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN"
```

### 5. Weather（天气查询）

#### API

`GET /agent_plan/v1/weather`

#### 参数输入

Optional:
- `region`: 行政区划名称
- `location`: `lat,lng` 格式坐标（gcj02）

Rules:
- `region` 与 `location` 至少传一个
- `location` 传入时，经纬度至少保留小数点后 6 位

#### 鉴权

- GET：query 参数 `baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN`

#### 示例

```bash
# 1) 按坐标查询天气
curl --get "https://api.map.baidu.com/agent_plan/v1/weather" \
  --data-urlencode "location=38.766230,116.432130" \
  --data-urlencode "baidu_map_auth_token=$BAIDU_MAP_AUTH_TOKEN"
```

## 错误处理

1. 如果缺少 token，提示用户执行：  
   申请地址：https://lbs.baidu.com/apiconsole/agentplan  
   `export BAIDU_MAP_AUTH_TOKEN="<YOUR_BAIDU_MAP_AUTH_TOKEN>"`
2. 如果缺少必填参数，直接指出缺的是哪个参数。
3. 如果涉及 `region` 参数且有歧义，提示用户补充确认城市或区域。

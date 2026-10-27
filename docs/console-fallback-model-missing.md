# 回退路径上的 11102 不得升级为 (账号, 模型) 冷却

issue #245：15 个号的池子跑几小时后全部软冷却、所有请求 503，日志里 `11-128` 与
`11102` 混在一起。看上去像「换接口重试」，实际效果是**把每个账号锁 6 小时**。

## 链条

1. `/v2/chat/completions` 判渠道校验 → 400 `11-128`；
2. 0006 的兜底换到 `/console/chat/completions`；
3. `/console` 本来就不提供这个模型 → 400 `{"code":11102,"msg":"model [...] service info not found"}`；
4. `11102` 走 `BlockModelBackoff`，`modelBlockBaseTTL = 6h`，写进 `modelCooldowns`；
5. 选号侧 `healthyForModel` 自动避开该账号该模型 → 逐个账号重复 → 整池 503。

第 3 步的 11102 与「主路径自己报 11102」语义不同：它只是**这条路径没有这个模型**，
因为我们是因为渠道校验才换过去的。它不构成「该账号缺这个模型」的证据。

## 修法

新增错误种类 `ErrFallbackModelMissing`（`fallback_model_missing`），只在
`ChatStreamContext` 的候选路径循环里、`i > 0`（即回退路径）且分类结果为
`ErrModelBlocked` 时启用：

- **主路径的 11102 不变**：仍然是 `ErrModelBlocked`，(账号, 模型) 负缓存照写——
  那是「该后端确实没有这个模型」的确定性答复。
- handler 侧零动作（不冷却 / 不熔断 / 不 `NoteError`），**仍然轮转**：换个账号
  可能压根不需要回退（渠道校验与账号灰度有关）。
- 没有降级成 `ErrClient`：`ErrClient` 会喂连败计数（`NoteFailures`，issue #114），
  而连败是**账号健康信号**——这里失败的是路径，不是账号。若照 `ErrClient` 处理，
  渠道校验期间每个账号都会攒连败，攒够阈值照样临时出池。

## 与 #246 的关系

`0017` 修的是**触发源**（DSH 的 system 首句指纹）；本条修的是**放大器**。
两者独立：即使指纹都被净化，其它客户端/其它句式仍可能触发渠道校验，只要回退路径
仍写 6 小时模型冷却，故障就会放大成整池 503。所以两条都要有，且这条不能靠「先把
指纹补全」来替代——指纹是黑名单，永远补不完。

## 验证

- `go build ./...`、`go test ./...`（Go 1.22.5，与 `go.mod` / 发布 CI 同版本）全过。
- 客户端层用例：`/v2` 回 `11-128`、`/console` 回 `11102` → 分类为
  `ErrFallbackModelMissing`，且确实走了两条路径；主路径单独回 `11102` 仍是
  `ErrModelBlocked`（防止降级写宽）。
- handler 层用例：同一现场下断言 `RateLimitedModels` 为空、`ConsecutiveFails == 0`、
  账号不 `cooling` / 不 `disabled`、响应不是 200（本次请求仍失败，但账号回池）。

## 补丁与资产

实现位于 `dev/upstream-patches/0018-console-fallback-model-missing.patch`，基线为
`upstream-src` 资产 + `0016` + `0017`。重打包后（312 个文件，**待维护者上传**）：

```text
size:   907152
sha256: b16c081dc70bb1608846190640401f4fac375f4a9f78e2643d062790abf1da8a
```

> 与 `0017` 一样：合并后必须先重打包并覆盖上传 `upstream-src` 载体 Release，
> 否则发布门禁会报「仍有补丁需要现场应用」并阻断发布。

## 0031：`/console` 排到首位时同样不得冷却模型

`0030` 引入渠道校验路径记忆（issue #296）之后，上面说的「回退路径」不再等价于
「索引大于 0」。账号在 `/v2` 上被判 11-128 后 10 分钟内，`chatPaths` 会把它改排成
`/console` → `/v2`，于是 `/console` 可能落在 `i == 0`。

### 链条

1. 账号在 `/v2` 被判渠道校验 → `markChannelBlocked`，10 分钟内 `/console` 排首位；
2. 该账号下一次请求的 `i == 0` 就是 `/console`；
3. `/console` 不提供这个模型 → 400 `11102`；
4. 旧守卫 `if i > 0 && kind == ErrModelBlocked` 只吃 `i > 0`，**判不到** →
   `ErrModelBlocked` → `BlockModelBackoff`，写 6 小时（封顶 24 小时）的
   (账号, 模型) 冷却；
5. 选号侧避开该账号该模型 → 10 分钟内被打过标记的账号逐个中招 → 整组显示
   「模型受限」、该模型请求全 503，而 `/v2` 主路径其实是好的。

现场特征：面板上一批账号同时出现「模型受限」徽章，`state.json` 里对应
`model_cooldowns` 的 `reason` 是 `11102 model not available`、`until` 相差不到一分钟；
同一个模型在另一个分组（跑着没有 #296 的旧版二进制）上完全正常。

### 修法

把判据从「位置」换成「路径」：

```go
if kind == ErrModelBlocked && (i > 0 || path == chatConsolePath) {
```

- `i > 0` 保留原语义：换到第二路径拿到的 `11102` 一律请求级。
- `path == chatConsolePath` 覆盖重排：`/console` 无论排第几都不升级冷却——它是兜底
  路径，`11102` 只说明「这条路径没有这个模型」，不构成账号缺模型的证据。
- `/v2` 在 `i == 0` 时仍是 `ErrModelBlocked`（`TestPrimaryPathModelMissingStaysModelBlocked`
  钉着）：主路径自己报 `11102` 是确定性答复。

### 为什么 CI 没测出来

两半各测了一半：`TestChannelBlockedReordersPaths` 只验重排生效，
`TestConsoleFallbackModelMissingIsRequestLevel` 只验 `/v2` → `/console` 顺序下的
`11102`。而且 httptest 假后端不会自发产生 11-128，`markChannelBlocked` 在 CI 里
只能靠手工调用才会走到——组合场景没有用例覆盖。

### 验证

- `go build ./...`、`go vet ./...`、`go test ./...` 全过。
- 新增回归 `TestConsoleFirstPathModelMissingStaysRequestLevel`：先 `markChannelBlocked`
  让 `/console` 排到首位，断言路径顺序 `/console` → `/v2` 且分类为
  `ErrFallbackModelMissing`。
- 现网：整池探测该模型 19 个账号全 `ok`，`state.json` 里 `11102` 与 `model_cooldowns`
  归零，重启后不再复现。

## 补丁与资产（0031）

实现位于 `dev/upstream-patches/0031-console-first-path-model-missing.patch`，基线为
`upstream-src` 资产（v2.11.0，`0030` 已在内）。重打包后（317 个文件，**待维护者上传**）：

```text
size:   927932
sha256: 1def9bae3f6d4f88f6f6d848eeb4f6488143bcf5acf8d67282ecafc2dcfe5c6a
```

> 与前面几个补丁一样：合并后必须先重打包并覆盖上传 `upstream-src` 载体 Release，
> 否则发布门禁会报「仍有补丁需要现场应用」并阻断发布。

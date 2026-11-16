# 交付说明：旧式存储（legacy store）重用检测修复

## 问题

同一存储实例被多个限流实例（limiter）共享时，会同时污染多条业务线的计数：共享存储的 `init` 会重复执行、`windowMs` 互相覆盖，计数按同一 key 累加/重置互相踩踏，一条业务线的流量洪峰会吃掉另一条业务线的配额预算，防线静默失效。

新式（`Store` / `increment`）存储在共享时会触发 `ERR_ERL_STORE_REUSE` 告警；旧式（`LegacyStore` / `incr` 回调）存储被共享时告警哑火，两代存储防护不对称。

## 根因

`source/rate-limit.ts` 的 `parseOptions()` 中：

```ts
store: promisifyStore(notUndefinedOptions.store ?? new MemoryStore(validations))
```

`promisifyStore()` 遇到旧式存储时，会为**每一个 limiter 创建一个新的 `PromisifiedStore` 包装实例**。而 `source/validations.ts` 的 `unsharedStore()` 用 `Set<Store>` 记录并比对**传给中间件的那个对象**，也就是包装层。

于是：

- 旧式存储被两个 limiter 使用时 → 生成两个不同的包装对象 → `Set` 中不命中 → 不告警。
- 新式存储不经包装、原对象透传 → 命中 → 正常告警。

包装层遮蔽了底层存储的身份，这是两代存储防护不对称的根因。

## 修复方案

### 包装层（`source/rate-limit.ts`）

在 `PromisifiedStore` 构造时，通过 **非可枚举的 Symbol 属性** 暴露它代理的旧式存储实例：

```ts
Object.defineProperty(this, underlyingStoreSymbol, {
	value: legacyStore,
	enumerable: false,
})
```

同时用**读取型 getter** 透传 `localKeys` / `prefix`。这两项是另一个校验 `singleCount` 读取的存储元数据，此前包装层同样会遮蔽它们（旧式存储即使自身声明 `localKeys: true`，包装后也变成 `undefined`，导致串联两个各自独立的旧式存储时误报 `ERR_ERL_DOUBLE_COUNT`）。使用 getter 而非构造时拷贝，保证与底层存储始终同步，不产生陈旧副本。

### 校验层（`source/validations.ts`）

- 新增导出常量 `underlyingStoreSymbol`，作为包装层与校验层之间约定的身份通道。
- 新增 `resolveStoreIdentity(store)`：若对象带有该 Symbol 则返回其底层实例，否则返回对象本身（新式存储、直接传入的存储、用户自定义适配器都安全地落回自身）。
- `unsharedStore()` 以及内部注册表 `usedStores` 全部改用**底层身份**比对，告警文案里的类名与 `localKeys` 提示也取自底层实例（因此旧式存储告警会正确显示其真实类名，而不是 `PromisifiedStore`）。

## 包装层与校验层的分工（设计理由）

**包装层只负责"如实申报底层身份"，校验层独享比较与告警逻辑。**

理由：

1. **职责单一**。包装层唯一职责是让旧式回调 API 适配成 Promise API；它不应了解"什么算重用"这种校验策略。身份申报是纯数据，不引入策略。
2. **策略可演进而不动包装层**。将来若变更告警条件（例如结合 `prefix` 判断是否真冲突），只需改 `unsharedStore`，无需触碰适配代码。
3. **默认安全**。`resolveStoreIdentity` 对没有该 Symbol 的对象回退到自身，所以新式存储、用户自建适配器、直接传入的对象行为完全不变，不会被此次改动波及。
4. **非可枚举 Symbol 而非普通字段**。Symbol 不会与用户存储的自有属性冲突，`enumerable: false` 让它不进入 `Object.entries()`、`util.inspect` 的常规输出与 `debug` 日志，避免出现在面向用户的诊断信息里；同时它又不是真正的 `#private` 字段，校验层可以读取。
5. **不用 `constructor.name` 猜测**。按包装层与校验层共同持有的 Symbol 取身份，是显式契约，比按类名/鸭子类型推断更可靠，也不受压缩、继承、同名类影响。

## 行为矩阵（修复后）

| 存储形态 | 单 limiter 独用 | 各 limiter 各配一存 | 同一实例被多 limiter 共享 |
|---|---|---|---|
| 新式 `Store` | 不告警 | 不告警 | `ERR_ERL_STORE_REUSE` |
| 旧式 `LegacyStore`（修复前） | 不告警 | 不告警 | **静默放过（缺陷）** |
| 旧式 `LegacyStore`（修复后） | 不告警 | 不告警 | `ERR_ERL_STORE_REUSE` |
| 兼容型（同时实现两接口） | 不告警 | 不告警 | `ERR_ERL_STORE_REUSE` |

检测按底层存储身份进行；独用路径与各自配存路径均不误报；`validate: { unsharedStore: false }` 仍可单独关闭该检查。

## 测试

`test/library/validation-test.ts` — `unsharedStore` 单元层对照：

- 同一底层存储被两个不同适配器包裹 → 告警（新增）
- 两个适配器包裹各自独立的底层存储 → 不告警（新增）
- 适配器包裹无 `localKeys` 的共享存储 → 告警且含 "unique prefix" 提示（新增）
- 单存储单 limiter 独用 → 不告警（新增）
- 既有的直接存储共享/独用、有/无 `localKeys` 用例保持通过

`test/library/middleware-test.ts` — 端到端（走真实 `rateLimit()`）对照：

- 新式 / 旧式 / 兼容型存储被两个 limiter 共享 → 均告警（新增，其中旧式一条在修复前失败）
- 新式 / 旧式 / 兼容型存储各配各存 → 均不告警（新增）
- 共享旧式存储时显式关闭 `unsharedStore` → 不告警（新增）
- 两个各自独立、`localKeys: true` 的旧式存储串联 → 不误报 `ERR_ERL_DOUBLE_COUNT`（新增，验证元数据透传）

验证方式：临时回退源码修复后，新测试中"旧式存储共享"与"旧式存储元数据透传"两条如期失败，其余对照用例通过；恢复修复后全部通过。

## 结果

- 全量库测试：8 个套件、**265 个测试全部通过**（基线 253，新增 12）。
- `biome check source test/library`：通过（无格式/规则问题）。
- `tsc --noEmit`：通过。
- 未改动 `singleCount` 的判定逻辑、未改变任何既有公开 API 签名或默认行为。

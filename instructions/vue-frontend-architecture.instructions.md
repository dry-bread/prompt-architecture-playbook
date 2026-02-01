---
applyTo: '**/*.vue,**/*.ts,**/*.js'
---

# UniApp（Vue）大型项目代码结构设计规范（Instructions）

本规范用于指导 UniApp（Android/iOS）项目的代码组织与开发流程。
目标：结构清晰、边界稳定、可扩展、易维护、可测试。

================================================
一、写代码前必须先做：结构规划
================================================

1) 先判断本次需求属于：
- 现有 Feature（在该 Feature 内开发）
- 新 Feature（必须在 src/ 下创建新的顶层文件夹，与 common/ 同级）

2) 每个 Feature 是 src/ 下的**独立顶层文件夹**，与 common/ 同级。
   不要创建 features/ 文件夹来包裹所有 Feature！
   
   正确：src/device/、src/files/、src/preview/
   错误：src/features/device/、src/features/files/

3) 设立 common/ 作为项目级共享目录，但必须满足：
- 只有“几乎所有 Feature 都会用到”的内容才能放入 common/
- 不能为了图方便把业务逻辑塞进 common/
- 若仅少数 Feature 复用，则放回各自 Feature 内，或建立显式的 shared-feature（可选）并保持边界清晰

================================================
二、目录组织（UniApp 推荐结构）
================================================

【重要说明】每个 Feature 是 src/ 下的**独立顶层文件夹**，与 common/ 同级。
**不要**把所有 Feature 放在一个 features/ 文件夹下！

src/
  pages/                    # UniApp 页面路由入口（保持薄，仅做路由和组装）
  
  # ========== Feature 模块（每个都是独立顶层文件夹）==========
  device/                   # 示例 Feature：设备管理
    components/             # Feature 私有组件
    composables/            # Feature 私有逻辑（useXxx）
    stores/                 # Feature 私有状态（Pinia store）
    services/               # 外部交互：API/本地存储/系统能力封装
      clients/              # 纯请求/纯桥接（HTTP/uni.*）
      dataProviders/        # 数据转换、聚合、缓存策略
    interfaces/             # 类型：domain / dto
    utils/                  # Feature 内工具（可选）
    index.ts                # Feature public API（只导出允许外部使用的入口）
  
  files/                    # 示例 Feature：文件管理
    composables/
    stores/
    services/
    interfaces/
    index.ts
  
  preview/                  # 示例 Feature：预览功能
    ...
  
  settings/                 # 示例 Feature：设置
    ...
  
  # ========== 共享模块 ==========
  common/                   # 项目级共享（严格控制，与各 Feature 同级）
    components/             # 通用 UI 组件（无业务）
    composables/            # 通用 hooks（无业务）
    stores/                 # 全局通用 store（少量且稳定）
    services/               # 通用服务（日志、埋点、网络、权限等）
    interfaces/             # 通用类型
    utils/                  # 通用工具函数
    index.ts                # common 公共导出

【目录结构示意图】
src/
├── device/               ← Feature（与 common 同级）
├── files/                ← Feature
├── preview/              ← Feature  
├── settings/             ← Feature
├── common/               ← 共享模块
├── pages/                ← 页面路由入口
├── locales/              ← 国际化
├── App.vue
└── main.ts

规则：
- Feature 之间禁止互相深度引用（禁止从别人的内部路径 import）
- 跨 Feature 只能通过：
  - common/
  - 或对方 feature 的 index.ts（public API）显式导出
- 页面 pages/ 尽量“薄”：负责路由参数、组合 Feature 页面/组件，不承载业务逻辑
【导入路径示例】
```typescript
// ✅ 正确：从 feature 的 public API 导入
import { useDeviceStore, deviceClient } from '@/device';
import { useFileList } from '@/files';
import { storage } from '@/common';

// ❌ 错误：深度引用内部路径
import { useDeviceStore } from '@/device/stores/useDeviceStore';
import { discoveryClient } from '@/device/services/clients/discoveryClient';

// ❌ 错误：使用 features/ 包裹文件夹（不存在这个层级）
import { useDeviceStore } from '@/features/device';
```
================================================
三、职责分层（Vue 版本：Component + Composable + Store + Service）
================================================

1) Component（UI 组件）
允许：
- 渲染 UI
- 接收 props / emit events
- 仅持有“纯 UI 状态”（弹窗开关、选中态、动画态等）

禁止：
- 直接请求 API
- 编写核心业务逻辑（规则计算、流程编排）
- 承担跨组件共享状态

2) Composable（useXxx 组合式逻辑）
职责：
- 聚合页面/组件行为逻辑（事件处理、派发 action、组装 computed）
- 调用 Store / Service
- 把页面需要的 state/computed/actions 组织成可复用单元

规则：
- 复杂逻辑优先放 composables，而不是塞进 .vue script 里
- composable 要保持边界清晰：只服务一个 Feature（除非明确是 common composable）

3) Store（Pinia）
职责：
- 管理“跨组件共享”的业务状态（单一事实来源）
- 暴露 actions（业务动作）与 getters（派生数据）

规则：
- UI 不直接改业务状态，统一通过 store actions
- store 应尽量不依赖具体 UI 组件
- 大状态拆分：一个 Feature 可以有多个 store（按领域划分）

4) Service（外部交互）
职责：
- 与外部系统交互：HTTP、WebSocket、uni.*、本地存储、权限、文件、蓝牙等
- clients/：只做“纯调用”（不做业务）
- dataProviders/：做数据转换、聚合、缓存、容错、重试策略（不做 UI）

禁止：
- 持有 UI 状态
- 直接操作组件

================================================
四、双向绑定（v-model）的使用准则
================================================

Vue 双向绑定很方便，但在大型复杂项目里必须“限流”。

允许使用 v-model 的典型场景：
- 表单输入：Input/Picker/Switch/Slider 等
- 组件封装的受控输入（自定义 v-model）

不建议 / 禁止滥用的场景：
- 把业务状态到处 v-model 直连 store（会导致“谁在改状态”难追踪）
- 用 watch 链式驱动业务流程（会演化成隐式状态机、难调试）

推荐模式：
- 表单组件内部可以 v-model 绑定到本地 ref
- 提交/确认时再调用 store action 更新业务状态
- 若必须实时同步，也要通过显式 action（onInput → store.updateXxx）

================================================
五、开发流程（必须按顺序）
================================================

1) 先定“Feature 归属 + 目录位置”
- 新 Feature：在 src/ 下创建顶层文件夹（如 src/newFeature/），写 index.ts 入口
- 旧 Feature：在原目录扩展（如 src/device/）

2) 先写 UI（组件/页面）
- UI 初期可用 mock 数据/静态数据
- 组件仅保留 UI state

3) 再写 composable（useXxx）
- 把事件处理、数据组装、调用 store/service 的逻辑集中到 useXxx

4) 再写 store（Pinia）
- 设计 state / getters / actions
- actions 是业务唯一入口

5) 再写 service（clients + dataProviders）
- client：纯请求/纯桥接
- provider：转换与聚合，返回 domain model

6) 最后把 UI 接起来
- UI → composable → store actions → service → store state → UI 更新

================================================
六、命名与文件约定（强制）
================================================

- Feature：featureX（小写驼峰或短横线统一即可）
- 组件：XxxCard.vue / XxxList.vue（PascalCase）
- composable：useXxx.ts
- store：useXxxStore.ts（Pinia 标准习惯）
- service：
  - clients：XxxClient.ts（纯调用）
  - providers：XxxProvider.ts（转换聚合）
- 类型：
  - interfaces/domain/*.ts
  - interfaces/dto/*.ts（可选）
- index.ts：只导出允许外部使用的 API（public surface）

================================================
七、数据流向（强制单向）
================================================

用户操作
 → Component（emit）
 → Composable（useXxx 处理交互）
 → Store action（业务入口）
 → Service（外部交互）
 → Store state 更新
 → Component 通过响应式自动更新

规则：
- Component 不直接请求 API
- Component 不直接改业务 state（尤其不要直接改 store.state 的深层字段）
- 业务变更统一走 store actions

================================================
八、Checklist（写完必须自检）
================================================

- [ ] 这次改动是否放在正确的 Feature 目录？
- [ ] 是否把“少数复用”错误地放进了 common/？
- [ ] 组件是否只包含 UI 逻辑？
- [ ] 业务逻辑是否集中在 composable/store？
- [ ] API/系统能力是否封装在 service（client/provider）？
- [ ] 是否避免了跨 Feature 深度 import？
- [ ] v-model 是否只用于输入/表单，而非隐式业务编排？
- [ ] 数据流是否清晰：UI → action → service → state → UI？

若任一项不满足，必须先重构再继续开发。

---
applyTo: '**/*.tsx,**/*.ts'
---

# 前端代码结构设计规范

## 一、文件夹结构规划

在编写代码之前，首先考虑整体架构。如果是一个新的 Feature，应为其创建独立的文件夹。如果不是新feature，则应将在现有 Feature 文件夹中编写。

### 1.1 顶层目录规则

```
src/
├── featureA/           # Feature A（功能模块）
├── featureB/           # Feature B（功能模块）
├── common/             # 项目级共享资源
│   ├── components/     # 通用 UI 组件
│   ├── hooks/          # 通用 Hooks
│   ├── utils/          # 工具函数
│   ├── interfaces/     # 通用类型定义
│   └── services/       # 通用服务
└── ...
```

**引用规则：**
- ✅ Feature 可以引用 `common/` 中的内容
- ❌ Feature 之间**不能**相互引用
- ⚠️ 只有**所有** Feature 都需要使用的资源，才放入 `common/`

**如何强制执行引用规则：**

使用 **dependency-cruiser** 工具在 CI 中检查依赖关系。配置文件 `.dependency-cruiser.js`：

```javascript
export default {
    options: {
        doNotFollow: { path: "node_modules" },
    },
    forbidden: [
        {
            name: "no-cross-feature-import",
            comment: "Feature 之间不能相互引用，只能引用 common 文件夹",
            severity: "error",
            from: {
                // 匹配 src 下的一级目录（Feature）
                path: "^src/([^/]+)/",
            },
            to: {
                // 禁止引用其他 Feature（排除自身和 common）
                path: "^src/(?!$1|common)",
            },
        },
    ],
};
```

在 `package.json` 中添加检查脚本：

```json
{
    "scripts": {
        "depcheck": "depcruise --output-type err-long --config .dependency-cruiser.js src"
    }
}
```

运行 `yarn depcheck` 即可检查依赖违规。

### 1.2 Feature 目录结构

```
feature/
├── components/          # UI 组件
├── viewModels/          # ViewModel 类（状态 + 业务逻辑）
├── services/
│   ├── dataProviders/   # 数据获取 + 转换逻辑
│   └── clients/         # 纯 HTTP / API 客户端
├── interfaces/          # 领域类型 & API 类型定义
└── utils/               # Feature 内部工具函数
```

---

## 二、组件与逻辑职责分离

### 2.1 核心原则

| 层级 | 职责 |
|------|------|
| **Component** | 渲染 UI、订阅 ViewModel 状态 |
| **ViewModel** | 管理状态、封装业务逻辑 |
| **Service** | 数据获取、转换、API 调用 |

### 2.2 开发流程

1. **先制作 UI 组件**：组件内仅保留与 UI 相关的状态（如 `isOpen`、`activeTab`）
2. **创建 ViewModel 类**：将业务逻辑、数据状态封装在 Class 中
3. **使用 Context 注入**：在 Feature 顶层组件实例化 ViewModel，通过 Context 向下传递
4. **使用 RxJS 桥接**：通过 `useObservableValue` 让 ViewModel 状态驱动 UI 更新

---

## 三、ViewModel 编写规范

### 3.1 状态声明（使用 RxJS BehaviorSubject）

```typescript
import { BehaviorSubject, Observable } from 'rxjs';

export class JobViewModel {
    // 私有 BehaviorSubject，命名以 _ 开头，以 $ 结尾
    private _loading$ = new BehaviorSubject<boolean>(false);
    private _data$ = new BehaviorSubject<Job[]>([]);
    private _error$ = new BehaviorSubject<Error | null>(null);

    // 暴露 Observable（用于组件订阅）
    public get loading$(): Observable<boolean> {
        return this._loading$;
    }

    // 暴露同步 getter（用于获取当前值）
    public get loading(): boolean {
        return this._loading$.value;
    }

    public get data$(): Observable<Job[]> {
        return this._data$;
    }

    public get data(): Job[] {
        return this._data$.value;
    }

    // 业务方法
    public async fetchData(): Promise<void> {
        this._loading$.next(true);
        try {
            const result = await this.dataProvider.getJobs();
            this._data$.next(result);
        } catch (err) {
            this._error$.next(err as Error);
        } finally {
            this._loading$.next(false);
        }
    }

    public toggleItem(id: number): void {
        const current = this._expandedMap$.value;
        this._expandedMap$.next({
            ...current,
            [id]: !current[id],
        });
    }
}
```

### 3.2 命名约定

| 类型 | 命名规则 | 示例 |
|------|----------|------|
| 私有 Observable | `_xxx$` | `_loading$`, `_data$` |
| 公开 Observable | `xxx$` | `loading$`, `data$` |
| 同步 getter | `xxx` | `loading`, `data` |
| ViewModel 类 | `XxxViewModel` | `JobViewModel` |

---

## 四、Context 与 ViewModel 注入

### 4.1 创建 Context

```typescript
// viewModels/featureContext.ts
import * as React from 'react';
import { JobViewModel } from './jobViewModel';
import { LogViewModel } from './logViewModel';

export interface IFeatureContextValue {
    jobViewModel: JobViewModel;
    logViewModel: LogViewModel;
}

export const FeatureContext = React.createContext<IFeatureContextValue | undefined>(undefined);

// 便捷 Hook
export function useFeatureContext(): IFeatureContextValue {
    const value = React.useContext(FeatureContext);
    if (!value) {
        throw new Error('useFeatureContext must be used within FeatureContext.Provider');
    }
    return value;
}

// 工厂函数
export function createFeatureContext(artifactId: string): IFeatureContextValue {
    return {
        jobViewModel: new JobViewModel(artifactId),
        logViewModel: new LogViewModel(artifactId),
    };
}
```

### 4.2 在顶层组件中实例化并注入

```tsx
// components/FeaturePage.tsx
import React, { useState, useEffect } from 'react';
import { FeatureContext, createFeatureContext } from '../viewModels/featureContext';
import { FeatureContent } from './FeatureContent';

export const FeaturePage: React.FC<{ artifactId: string }> = ({ artifactId }) => {
    const [context, setContext] = useState<IFeatureContextValue | undefined>();

    useEffect(() => {
        // 在顶层实例化所有 ViewModel
        const ctx = createFeatureContext(artifactId);
        setContext(ctx);
    }, [artifactId]);

    if (!context) {
        return <Loading />;
    }

    return (
        <FeatureContext.Provider value={context}>
            <FeatureContent />
        </FeatureContext.Provider>
    );
};
```

---

## 五、组件中使用 RxJS 状态

### 5.1 useObservableValue Hook

此 Hook 用于将 RxJS Observable 转换为 React 状态：

```typescript
// common/hooks/useObservableValue.ts
import * as React from 'react';
import { Observable } from 'rxjs';

export function useObservableValue<T>(
    observable: Observable<T> | undefined,
    initialValue?: () => T
): T | undefined {
    const [currentValue, setCurrentValue] = React.useState<T | undefined>(
        initialValue ? initialValue() : undefined
    );

    React.useEffect(() => {
        if (!observable) return undefined;

        let lastValue = currentValue;
        const subscription = observable.subscribe({
            next(value: T): void {
                if (lastValue !== value) {
                    lastValue = value;
                    setCurrentValue(value);
                }
            },
        });

        return () => subscription.unsubscribe();
    }, [observable]);

    return currentValue;
}
```

### 5.2 在组件中订阅 ViewModel 状态

```tsx
// components/JobList.tsx
import React, { useCallback } from 'react';
import { useFeatureContext } from '../viewModels/featureContext';
import { useObservableValue } from 'src/common/hooks/useObservableValue';

export const JobList: React.FC = () => {
    // 1. 从 Context 获取 ViewModel
    const { jobViewModel } = useFeatureContext();

    // 2. 订阅 ViewModel 的响应式状态
    const loading = useObservableValue(jobViewModel.loading$, () => jobViewModel.loading);
    const jobs = useObservableValue(jobViewModel.data$, () => jobViewModel.data);
    const error = useObservableValue(jobViewModel.error$, () => jobViewModel.error);

    // 3. 通过 ViewModel 方法触发状态变更
    const handleRefresh = useCallback(() => {
        jobViewModel.fetchData();
    }, [jobViewModel]);

    const handleToggle = useCallback((id: number) => {
        jobViewModel.toggleItem(id);
    }, [jobViewModel]);

    // 4. 纯 UI 渲染
    if (loading) return <Spinner />;
    if (error) return <ErrorMessage error={error} />;

    return (
        <div>
            <Button onClick={handleRefresh}>Refresh</Button>
            {jobs.map(job => (
                <JobItem
                    key={job.id}
                    job={job}
                    onToggle={() => handleToggle(job.id)}
                />
            ))}
        </div>
    );
};
```

---

## 六、Service 层规范

### 6.1 Client（API 客户端）

仅负责 HTTP 请求，不包含业务逻辑：

```typescript
// services/clients/jobClient.ts
export class JobClient {
    constructor(private readonly baseUrl: string) {}

    async getJobs(): Promise<Response> {
        return fetch(`${this.baseUrl}/jobs`);
    }

    async getJobById(id: string): Promise<Response> {
        return fetch(`${this.baseUrl}/jobs/${id}`);
    }
}
```

### 6.2 DataProvider（数据提供者）

封装数据获取 + 转换逻辑：

```typescript
// services/dataProviders/jobDataProvider.ts
import { JobClient } from '../clients/jobClient';
import { Job, RawJobResponse } from '../../interfaces/job';

export class JobDataProvider {
    constructor(private readonly client: JobClient) {}

    async getJobs(): Promise<Job[]> {
        const response = await this.client.getJobs();
        if (!response.ok) {
            throw new Error(`Failed to fetch jobs: ${response.status}`);
        }
        const raw: RawJobResponse = await response.json();
        return this.transform(raw);
    }

    private transform(raw: RawJobResponse): Job[] {
        return raw.items.map(item => ({
            id: item.id,
            name: item.displayName,
            status: item.state,
            // ...
        }));
    }
}
```

---

## 七、数据流向总结

```
┌─────────────────────────────────────────────────────────────┐
│  用户操作                                                    │
│      ↓                                                      │
│  Component (调用 ViewModel 方法)                             │
│      ↓                                                      │
│  ViewModel (业务逻辑 → 调用 Service → 更新 BehaviorSubject)   │
│      ↓                                                      │
│  BehaviorSubject.next(newValue)                             │
│      ↓                                                      │
│  useObservableValue (订阅 → 触发 setState)                   │
│      ↓                                                      │
│  Component 重新渲染                                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 八、Checklist

开发新 Feature 时，检查以下事项：

- [ ] 是否为新 Feature 创建了独立文件夹？
- [ ] 文件夹结构是否包含 `components/`、`viewModels/`、`services/`、`interfaces/`？
- [ ] ViewModel 是否使用 `BehaviorSubject` 管理状态？
- [ ] ViewModel 是否同时暴露 `xxx$`（Observable）和 `xxx`（同步 getter）？
- [ ] 是否在 Feature 顶层组件创建 Context 并注入 ViewModel？
- [ ] 组件是否通过 `useObservableValue` 订阅状态？
- [ ] 组件是否只包含 UI 逻辑，业务逻辑是否都在 ViewModel 中？
- [ ] API 调用是否封装在 Service 层？

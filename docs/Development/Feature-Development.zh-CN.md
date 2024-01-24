# LobeChat 功能开发完全指南

本文档旨在指导开发者了解我们在 LobeChat 中的完整功能实现流程。

我们将以 sessionGroup 的实现为示例：[✨ feat: add session group manager](https://github.com/lobehub/lobe-chat/pull/1055) ， 通过以下六个主要部分来阐述完整的实现流程：

1. 数据模型 / 数据库定义
2. Service 实现 / Model 实现
3. 前端数据流 Store 实现
4. UI 实现与 action 绑定
5. 数据迁移
6. 数据导入导出

## 一、数据库部分

为了实现 Session Group 功能，首先需要在数据库层面定义相关的数据模型和索引。

定义一个新的 sessionGroup 表，分 4 步：

### 1. 建立数据模型 schema

在 `src/database/schema/sessionGroup.ts` 中定义 `DB_SessionGroup` 的数据模型：

```typescript
import { z } from 'zod';

export const DB_SessionGroupSchema = z.object({
  name: z.string(),
  sort: z.number().optional(),
});

export type DB_SessionGroup = z.infer<typeof DB_SessionGroupSchema>;
```

### 2. 创建数据库索引

> !\[\Note]
>
> 如果你不了解 LobeChat 所使用的数据库 IndexedDB 与 Dexie.js ，可以查阅 [本地数据库](./Local-Database.zh-CN) 部分了解相关前置知识。

由于要新增一个表，所以需要在在数据库 Schema 中，为 `sessionGroup` 表添加索引。

在 `src/database/core/schema.ts` 中添加 `dbSchemaV4`:

```diff
// ... 前面的一些实现

// ************************************** //
// ******* Version 3 - 2023-12-06 ******* //
// ************************************** //
// - Added `plugin` table

export const dbSchemaV3 = {
  ...dbSchemaV2,
  plugins:
    '&identifier, type, manifest.type, manifest.meta.title, manifest.meta.description, manifest.meta.author, createdAt, updatedAt',
};

+ // ************************************** //
+ // ******* Version 4 - 2024-01-21 ******* //
+ // ************************************** //
+ // - Added `sessionGroup` table

+ export const dbSchemaV4 = {
+   ...dbSchemaV3,
+   sessionGroups: '&id, name, sort, createdAt, updatedAt',
+   sessions: '&id, type, group, pinned, meta.title, meta.description, meta.tags, createdAt, updatedAt',
};
```

`sessions` 的定义是因为存在数据迁移的情况，详情查看第五节数据迁移。

### 3. 在本地 DB 中加入 sessionGroups 表

扩展本地数据库类以包含新的 `sessionGroups` 表：

```diff

import { dbSchemaV1, dbSchemaV2, dbSchemaV3, dbSchemaV4 } from './schemas';

interface LobeDBSchemaMap {
  files: DB_File;
  messages: DB_Message;
  plugins: DB_Plugin;
+ sessionGroups: DB_SessionGroup;
  sessions: DB_Session;
  topics: DB_Topic;
}

// Define a local DB
export class LocalDB extends Dexie {
  public files: LobeDBTable<'files'>;
  public sessions: LobeDBTable<'sessions'>;
  public messages: LobeDBTable<'messages'>;
  public topics: LobeDBTable<'topics'>;
  public plugins: LobeDBTable<'plugins'>;
+ public sessionGroups: LobeDBTable<'sessionGroups'>;

  constructor() {
    super(LOBE_CHAT_LOCAL_DB_NAME);
    this.version(1).stores(dbSchemaV1);
    this.version(2).stores(dbSchemaV2);
    this.version(3).stores(dbSchemaV3);
+   this.version(4).stores(dbSchemaV4);

    this.files = this.table('files');
    this.sessions = this.table('sessions');
    this.messages = this.table('messages');
    this.topics = this.table('topics');
    this.plugins = this.table('plugins');
+   this.sessionGroups = this.table('sessionGroups');
  }
}
```

如此一来，你就可以通过在 `Application` -> `Storage` -> `IndexedDB` 中查看到 `LOBE_CHAT_DB` 里的 `sessionGroups` 表了。

![](https://github.com/lobehub/lobe-chat/assets/28616219/aea50f66-4060-4a32-88c8-b3c672d05be8)

> \[!Note]
>
> 由于本节只关注 schema 定义，因此不展开迁移部分的实现。

### 4. 定义 Model

在构建 LobeChat 应用时，Model 负责与数据库的交互，它定义了如何读取、插入、更新和删除数据库的数据，定义具体的业务逻辑。

在 `src/database/model/sessionGroup.ts` 中定义 `SessionGroupModel`：

```typescript
import { BaseModel } from '@/database/core';
import { DB_SessionGroup, DB_SessionGroupSchema } from '@/database/schemas/sessionGroup';
import { nanoid } from '@/utils/uuid';

class _SessionGroupModel extends BaseModel {
  constructor() {
    super('sessions', DB_SessionGroupSchema);
  }

  async create(name: string, sort?: number, id = nanoid()) {
    return this._add({ name, sort }, id);
  }

  // ... 其他 CRUD 方法的实现
}

export const SessionGroupModel = new _SessionGroupModel();
```

## 二、Service 部分

在 `sessionService` 中实现 Session Group 相关的请求逻辑：

```typescript
class SessionService {
  // ... 省略 session 业务逻辑

  // ************************************** //
  // ***********  SessionGroup  *********** //
  // ************************************** //

  async createSessionGroup(name: string, sort?: number) {
    const item = await SessionGroupModel.create(name, sort);
    if (!item) {
      throw new Error('session group create Error');
    }

    return item.id;
  }

  // ... 其他 SessionGroup 相关的实现
}
```

把 sessionGroup 的实现放到 sessionService 的考虑是为了让会话 Session 领域更加内聚。如果未来对 sessionGroup 有了更多的要求，可以考虑再单独拆除。例如 chat 就是从 session 中拆出来的一个独立域。

## 三、Store Action 部分

在 LobeChat 应用中，Store 是用于管理应用前端状态的模块。其中的 Action 是触发状态更新的函数，通常会调用服务层的方法来执行实际的数据处理操作，然后更新 Store 中的状态。我们采用了 `zustand` 作为 Store 模块的底层依赖，对于状态管理的详细实践介绍，可以查阅 [状态管理最佳实践](./State-Management-Intro.zh-CN)

### sessionGroup CRUD

会话组的 CRUD 操作是管理会话组数据的核心行为。在 `src/store/session/slice/sessionGroup` 中，我们将实现与会话组相关的状态逻辑，包括添加、删除、更新会话组及其排序。

以下是 `action.ts` 文件中需要实现的 `SessionGroupAction` 接口方法：

```ts
export interface SessionGroupAction {
  // 增加会话组
  addSessionGroup: (name: string) => Promise<string>;
  // 删除会话组
  removeSessionGroup: (id: string) => Promise<void>;
  // 更新会话的会话组 ID
  updateSessionGroupId: (sessionId: string, groupId: string) => Promise<void>;
  // 更新会话组名称
  updateSessionGroupName: (id: string, name: string) => Promise<void>;
  // 更新会话组排序
  updateSessionGroupSort: (items: SessionGroupItem[]) => Promise<void>;
}
```

以 `addSessionGroup` 方法为例，我们首先调用 `sessionService` 的 `createSessionGroup` 方法来创建新的会话组，然后使用 `refreshSessions` 方法来刷新 sessions 状态：

```ts
export const createSessionGroupSlice: StateCreator<
  SessionStore,
  [['zustand/devtools', never]],
  [],
  SessionGroupAction
> = (set, get) => ({
  // 实现添加会话组的逻辑
  addSessionGroup: async (name) => {
    // 调用服务层的 createSessionGroup 方法并传入会话组名称
    const id = await sessionService.createSessionGroup(name);
    // 调用 get 方法获取当前的 Store 状态并执行 refreshSessions 方法刷新会话数据
    await get().refreshSessions();
    // 返回新创建的会话组 ID
    return id;
  },
  // ... 其他 action 实现
});
```

通过以上的实现，我们可以确保在添加新的会话组后，应用的状态会及时更新，且相关的组件会收到最新的状态并重新渲染。这种方式提高了数据流的可预测性和可维护性，同时也简化了组件之间的通信。

### Sessions 分组逻辑改造

本次需求改造需要对 Sessions 进行升级，从原来的单一列表变成了三个不同的分组：`pinnedSessions`（置顶列表）、`customSessionGroups`（自定义分组）和 `defaultSessions`（默认列表）。

为了处理这些分组，我们需要改造 `useFetchSessions` 的实现逻辑。以下是关键的改动点：

1. 使用 `sessionService.getSessionsWithGroup` 方法负责调用后端接口 `SessionModel.queryWithGroups()` 来获取分组数据；
2. 将获取后的数据保存为三到不同的状态字段中：`pinnedSessions`、`customSessionGroups` 和 `defaultSessions`；

#### `useFetchSessions` 方法

该方法在 `createSessionSlice` 中定义，如下所示：

```typescript
export const createSessionSlice: StateCreator<
  SessionStore,
  [['zustand/devtools', never]],
  [],
  SessionAction
> = (set, get) => ({
  // ... 其他方法
  useFetchSessions: () =>
    useSWR<ChatSessionList>(FETCH_SESSIONS_KEY, sessionService.getSessionsWithGroup, {
      onSuccess: (data) => {
        set(
          {
            customSessionGroups: data.customGroup,
            defaultSessions: data.default,
            isSessionsFirstFetchFinished: true,
            pinnedSessions: data.pinned,
            sessions: data.all,
          },
          false,
          n('useFetchSessions/onSuccess', data),
        );
      },
    }),
});
```

在成功获取数据后，我们使用 `set` 方法来更新 `customSessionGroups`、`defaultSessions`、`pinnedSessions` 和 `sessions` 状态。这将保证状态与最新的会话数据同步。

#### `SessionModel.queryWithGroups` 方法

此方法是 `sessionService.getSessionsWithGroup` 调用的核心方法，它负责查询和组织会话数据，代码如下：

```typescript
class _SessionModel extends BaseModel {
  // ... 其他方法

  async queryWithGroups(): Promise<ChatSessionList> {
    const groups = await SessionGroupModel.query();
    const customGroups = await this.queryByGroupIds(groups.map((item) => item.id));
    const defaultItems = await this.querySessionsByGroupId(SessionDefaultGroup.Default);
    const pinnedItems = await this.getPinnedSessions();

    const all = await this.query();
    return {
      all,
      customGroup: groups.map((group) => ({
        ...group,
        children: customGroups[group.id],
      })),
      default: defaultItems,
      pinned: pinnedItems,
    };
  }
}
```

方法 `queryWithGroups` 首先查询所有会话组，然后基于这些组的 ID 查询自定义会话组，同时查询默认和固定的会话。最后，它返回一个包含所有会话和按组分类的会话列表对象。

## 四、UI 部分

在 UI 组件中绑定 Store Action 实现交互逻辑，例如 `CreateGroupModal`：

```tsx
const CreateGroupModal = () => {
  // ... 其他逻辑

  const [updateSessionGroup, addCustomGroup] = useSessionStore((s) => [
    s.updateSessionGroupId,
    s.addSessionGroup,
  ]);

  return (
    <Modal
      onOk={async () => {
        // ... 其他逻辑
        const groupId = await addCustomGroup(name);
        await updateSessionGroup(sessionId, groupId);
      }}
    >
      {/* ... */}
    </Modal>
  );
};
```

## 五、数据迁移部分

一般的迭代不太需要关心数据迁移问题，而一旦遇到旧的数据结构无法满足新的需求时，就需要考虑数据迁移的问题了。

在本次实现中，session 的 group 字段可以说是典型案例。

之前通过将其标记为 `pinned` 和 `default` 来区分置顶与分组。但是如果存在多个分组时，这种模式就无法满足了。

```
before   pin:  group = abc
after    pin:  group = pinned
after  unpin:  group = default
```

可以看到，如果取消置顶， unpin 之后 group 是无法回到 abc 的，因为没有其他存放这个字段。 因此我们需要一个新的字段 `pinned` 来标记是否置顶，而 `group` 字段则专心用来标记分组。

这样一来，就需要将旧的数据迁移到新的数据结构中，核心的一条迁移逻辑为：

- 当用户的 `group` 字段为 `pinned` 时，将其 `pinned` 字段置为 `true`，同时将 group 设为 `default`;

而数据迁移需要考虑两块：

- 配置文件迁移
- 数据库迁移

我建议优先实现配置文件迁移，后实现数据库迁移。因为配置文件迁移相对数据库迁移更加易于复现与测试。

### 配置文件迁移

LobeChat 的配置文件迁移管理在 `src/migrations/index.ts` 中，

```ts
// 当前最新的版本号
export const CURRENT_CONFIG_VERSION = 3;

// 历史记录版本升级模块
const ConfigMigrations = [
  /**
   * 2024.01.22
   * from `group = pinned` to `pinned:true`
   */
  MigrationV2ToV3,
  /**
   * 2023.11.27
   * 从单 key 数据库转换为基于 dexie 的关系型结构
   */
  MigrationV1ToV2,
  /**
   * 2023.07.11
   * just the first version, Nothing to do
   */
  MigrationV0ToV1,
];
```

同时，配置文件的版本号和数据库版本号不一定完全一致对应，原因是 db 的版本升级可能不一定存在需要数据迁移的情况。（比如新增了一个表，或者新增了一个字段），而配置文件的版本升级则一定存在数据迁移的情况。

本次迁移配置文件的逻辑在 `src/migrations/FromV2ToV3.ts` 中：

```ts
import type { Migration, MigrationData } from '@/migrations/VersionController';

import { V2ConfigState, V2Session } from '../FromV1ToV2/types/v2';
import { V3ConfigState, V3Session } from './types/v3';

export class MigrationV2ToV3 implements Migration {
  // 指定从该版本开始向上升级
  version = 2;

  migrate(data: MigrationData<V2ConfigState>): MigrationData<V3ConfigState> {
    const { sessions } = data.state;

    return {
      ...data,
      state: {
        ...data.state,
        sessions: sessions.map((s) => this.migrateSession(s)),
      },
    };
  }

  migrateSession = (session: V2Session): V3Session => {
    return {
      ...session,
      group: 'default',
      pinned: session.group === 'pinned',
    };
  };
}
```

实现可以看到非常简单。但重要的是，我们需要保证迁移的正确性，因此需要编写测试用例来保证迁移的正确性，测试用例写在 `migrations.test.ts` 中，需要搭配 `fixtures` 来固化测试数据。

```ts
import { MigrationData, VersionController } from '@/migrations/VersionController';

import { MigrationV1ToV2 } from '../FromV1ToV2';
import inputV1Data from '../FromV1ToV2/fixtures/input-v1-session.json';
import inputV2Data from './fixtures/input-v2-session.json';
import outputV3DataFromV1 from './fixtures/output-v3-from-v1.json';
import outputV3Data from './fixtures/output-v3.json';
import { MigrationV2ToV3 } from './index';

describe('MigrationV2ToV3', () => {
  let migrations;
  let versionController: VersionController<any>;

  beforeEach(() => {
    migrations = [MigrationV2ToV3];
    versionController = new VersionController(migrations, 3);
  });

  it('should migrate data correctly through multiple versions', () => {
    const data: MigrationData = inputV2Data;

    const migratedData = versionController.migrate(data);

    expect(migratedData.version).toEqual(outputV3Data.version);
    expect(migratedData.state.sessions).toEqual(outputV3Data.state.sessions);
    expect(migratedData.state.topics).toEqual(outputV3Data.state.topics);
    expect(migratedData.state.messages).toEqual(outputV3Data.state.messages);
  });

  it('should work correct from v1 to v3', () => {
    const data: MigrationData = inputV1Data;

    versionController = new VersionController([MigrationV2ToV3, MigrationV1ToV2], 3);

    const migratedData = versionController.migrate(data);

    expect(migratedData.version).toEqual(outputV3DataFromV1.version);
    expect(migratedData.state.sessions).toEqual(outputV3DataFromV1.state.sessions);
    expect(migratedData.state.topics).toEqual(outputV3DataFromV1.state.topics);
    expect(migratedData.state.messages).toEqual(outputV3DataFromV1.state.messages);
  });
});
```

需要测试：

- 正常的单次迁移（v2 -> v3)
- 完整的迁移 (v1 -> v3)

### 数据库迁移

数据库迁移需要在 LocalDB 类中进行书写，文件位于： `src/database/core/db.ts` ：

```diff
export class LocalDB extends Dexie {
  public files: LobeDBTable<'files'>;
  public sessions: LobeDBTable<'sessions'>;
  public messages: LobeDBTable<'messages'>;
  public topics: LobeDBTable<'topics'>;
  public plugins: LobeDBTable<'plugins'>;
  public sessionGroups: LobeDBTable<'sessionGroups'>;

  constructor() {
    super(LOBE_CHAT_LOCAL_DB_NAME);
    this.version(1).stores(dbSchemaV1);
    this.version(2).stores(dbSchemaV2);
    this.version(3).stores(dbSchemaV3);
    this.version(4)
      .stores(dbSchemaV4)
+     .upgrade((trans) => this.upgradeToV4(trans));

    this.files = this.table('files');
    this.sessions = this.table('sessions');
    this.messages = this.table('messages');
    this.topics = this.table('topics');
    this.plugins = this.table('plugins');
    this.sessionGroups = this.table('sessionGroups');
  }

+  /**
+   * 2024.01.22
+   *
+   * DB V3 to V4
+   * from `group = pinned` to `pinned:true`
+   */
+  upgradeToV4 = async (trans: Transaction) => {
+    const sessions = trans.table('sessions');
+    await sessions.toCollection().modify((session) => {
+      // translate boolean to number
+      session.pinned = session.group === 'pinned' ? 1 : 0;
+      session.group = 'default';
+    });
+  };
}
```

## 六、数据导入导出

在 LobeChat 中，数据导入导出功能是为了确保用户可以在不同设备之间迁移他们的数据。这包括会话、话题、消息和设置等数据。在本次的 Session Group 功能实现中，我们也需要对数据导入导出进行处理，以确保当完整导出的数据在其他设备上可以一模一样恢复。

数据导入导出的核心实现在 `src/service/config.ts` 的 `ConfigService` 中，其中的关键方法如下：

| 方法名称              | 描述             |
| --------------------- | ---------------- |
| `importConfigState`   | 导入配置数据     |
| `exportAgents`        | 导出所有助理数据 |
| `exportSessions`      | 导出所有会话数据 |
| `exportSingleSession` | 导出单个会话数据 |
| `exportSingleAgent`   | 导出单个助理数据 |
| `exportSettings`      | 导出设置数据     |
| `exportAll`           | 导出所有数据     |

### 数据导出

在 LobeChat 中，当用户选择导出数据时，会将当前的会话、话题、消息和设置等数据打包成一个 JSON 文件并提供给用户下载。这个 JSON 文件的标准结构如下：

```json
{
  "exportType": "sessions",
  "state": {
    "sessions": [],
    "topics": [],
    "messages": []
  },
  "version": 3
}
```

其中：

- `exportType`： 标识导出数据的类型，目前有 `sessions`、 `agent` 、 `settings` 和 `all` 四种；
- `state`： 存储实际的数据，不同 `exportType` 的数据类型也不同；
- `version`： 标识数据的版本。

在 Session Group 功能实现中，我们需要在 `state` 字段中添加 `sessionGroups` 数据。这样，当用户导出数据时，他们的 Session Group 数据也会被包含在内。

以导出 sessions 为例，导出数据的相关实现代码修改如下：

```diff
class ConfigService {
  // ... 省略其他

  exportSessions = async () => {
    const sessions = await sessionService.getSessions();
+   const sessionGroups = await sessionService.getSessionGroups();
    const messages = await messageService.getAllMessages();
    const topics = await topicService.getAllTopics();

-   const config = createConfigFile('sessions', { messages, sessions, topics });
+   const config = createConfigFile('sessions', { messages, sessionGroups, sessions, topics });

    exportConfigFile(config, 'sessions');
  };
}
```

### 数据导入

数据导入的功能是通过 `ConfigService.importConfigState` 来实现的。当用户选择导入数据时，他们需要提供一个由 符合上述结构规范的 JSON 文件。`importConfigState` 方法接受配置文件的数据，并将其导入到应用中。

在 Session Group 功能实现中，我们需要在导入数据的过程中处理 `sessionGroups` 数据。这样，当用户导入数据时，他们的 Session Group 数据也会被正确地导入。

以下是 `importConfigState` 中导入实现的变更代码：

```diff
class ConfigService {
  // ... 省略其他代码

+ importSessionGroups = async (sessionGroups: SessionGroupItem[]) => {
+   return sessionService.batchCreateSessionGroups(sessionGroups);
+ };

  importConfigState = async (config: ConfigFile): Promise<ImportResults | undefined> => {
    switch (config.exportType) {
      case 'settings': {
        await this.importSettings(config.state.settings);

        break;
      }

      case 'agents': {
+       const sessionGroups = await this.importSessionGroups(config.state.sessionGroups);

        const data = await this.importSessions(config.state.sessions);
        return {
+         sessionGroups: this.mapImportResult(sessionGroups),
          sessions: this.mapImportResult(data),
        };
      }

      case 'all': {
        await this.importSettings(config.state.settings);

+       const sessionGroups = await this.importSessionGroups(config.state.sessionGroups);

        const [sessions, messages, topics] = await Promise.all([
          this.importSessions(config.state.sessions),
          this.importMessages(config.state.messages),
          this.importTopics(config.state.topics),
        ]);

        return {
          messages: this.mapImportResult(messages),
+         sessionGroups: this.mapImportResult(sessionGroups),
          sessions: this.mapImportResult(sessions),
          topics: this.mapImportResult(topics),
        };
      }

      case 'sessions': {
+       const sessionGroups = await this.importSessionGroups(config.state.sessionGroups);

        const [sessions, messages, topics] = await Promise.all([
          this.importSessions(config.state.sessions),
          this.importMessages(config.state.messages),
          this.importTopics(config.state.topics),
        ]);

        return {
          messages: this.mapImportResult(messages),
+         sessionGroups: this.mapImportResult(sessionGroups),
          sessions: this.mapImportResult(sessions),
          topics: this.mapImportResult(topics),
        };
      }
    }
  };
}
```

上述修改的一个要点是先进行 sessionGroup 的导入，因为如果先导入 session 时，如果没有在当前数据库中查到相应的 SessionGroup Id，那么这个 session 的 group 会兜底修改为默认值。这样就无法正确地将 sessionGroup 的 ID 与 session 进行关联。

以上就是 LobeChat Session Group 功能在数据导入导出部分的实现。通过这种方式，我们可以确保用户的 Session Group 数据在导入导出过程中能够被正确地处理。

## 总结

以上就是 LobeChat Session Group 功能的完整实现流程。开发者可以参考本文档进行相关功能的开发和测试。

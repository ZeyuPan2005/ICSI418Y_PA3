# PA3 代码讲解与录像提纲

## 先理解数据如何流动

浏览器里的 React 负责表单和列表；Express 负责接收 HTTP 请求、检查输入；MongoDB 负责保存数据。React 不直接连接 Atlas，也不会读取 `.env`。

以创建项目为例：

1. `CreateProject.jsx` 用 `useState` 保存名称、描述、状态、负责人 ID。
2. 用户点击 Create project，组件调用 `api.js` 的 `request()`，发送 `POST /api/projects`。`JSON.stringify` 将 JavaScript 对象变成请求正文。
3. `app.js` 的 `express.json()` 把请求正文解析成 `req.body`。`projects.js` 检查必填字段、状态和 ID。
4. 后端把负责人 ID 字符串转换成 MongoDB `ObjectId`，再查找 `users`，确认用户存在。
5. 同一个 MongoDB 事务依次插入 `Projects` 和负责人的 `ProjectMemberships` 记录。任意一步失败，就不会留下只有项目、没有负责人关系的半成品。
6. 后端在事务提交后返回 HTTP 201 和新项目。React 将返回的项目加入列表，并选中它；成员组件随后请求该项目的成员。

## 为什么需要 ProjectMemberships

`Projects.project_lead_id` 表示负责人，是一个用户引用。它本身不代表全部成员。

例如有 Alice、Bob、Casey 三个用户，以及 Event Planner、Study Group 两个项目。Casey 加入两个项目时，会有两条关系记录：

```text
{ project_id: EventPlanner的ObjectId, user_id: Casey的ObjectId }
{ project_id: StudyGroup的ObjectId,   user_id: Casey的ObjectId }
```

同一个用户 ID 可以出现在不同项目的关系中，同一个项目 ID 也可以对应多个用户 ID，这就是多对多关系。成员表中不复制姓名，因为姓名可能重复或变化。

MongoDB 中实际存储的是 `ObjectId`；浏览器通过 JSON 接收到它们的字符串形式。这两个层次要区分。

## 推荐阅读源码的顺序

| 文件 | 学习重点 |
| --- | --- |
| `server/server.js` | 读取原有 `.env`，连接 `pa2`，建立唯一索引后启动 9000 端口 |
| `server/app.js` | 保留 PA2 的 `/signup` 和 `/login`；注册 PA3 路由和统一错误处理 |
| `server/projects.js` | 六个新 API；输入验证、ObjectId 转换、事务、成员增删 |
| `client/src/api.js` | fetch、JSON、HTTP 错误处理、姓名显示辅助函数 |
| `client/src/App.jsx` | 获取用户和项目、切换页面、选中项目、保存共享状态 |
| `client/src/CreateProject.jsx` | 受控表单、提交、成功和错误反馈 |
| `client/src/ProjectList.jsx` | 用项目中的负责人 ID 匹配用户列表，显示可读姓名 |
| `client/src/ProjectMembers.jsx` | 根据项目加载成员；添加/删除后重新请求列表 |

`App` 向子组件传递数据和回调：例如 `onCreated(project)` 让创建表单通知父组件更新项目列表。注册成功后新增的 `onRegistered` 回调负责刷新用户选项，原有注册逻辑继续保留。

`useEffect` 在组件需要获取数据时发送 GET。切换项目时用项目 ID 作为 `key`，重置成员组件状态；同时 `AbortController` 取消旧请求，避免旧项目的响应覆盖当前项目的数据。

## 五个值得讲清楚的验证点

1. **必填字段和状态**：前端帮助用户纠错，后端仍独立校验。只有 Planning、Active、Completed 合法。
2. **ID 和存在性**：先检查 ID 格式，再转换 ObjectId，然后查询对应记录。格式错误返回 400，记录不存在返回 404。
3. **重复成员**：先查询是否存在相同关系；数据库的 `(project_id, user_id)` 唯一索引再保证并发请求不会插入两条。重复返回 409。
4. **保护负责人**：前端显示 Project Lead 标签、不显示 Remove；后端 DELETE 路由也比较负责人 ID，直接调用 API 仍无法删除负责人。
5. **错误与隐私**：公共用户接口只返回 ID、姓名和用户名；统一错误处理返回安全的 JSON 提示，不把数据库原始错误发给浏览器。

事务机制参考 MongoDB 官方文档：
https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/

## 录制前准备

- 启动后端和前端，确认没有启动报错。
- 在 Accounts 中准备三位用户，例如 Alice、Bob、Casey；使用演示密码。
- 准备一个项目，录像中再创建第二个项目，确保最终至少两个项目。
- 提前打开 Atlas 的 `pa2` 数据库、`Projects` 和 `ProjectMemberships`。
- 提前打开下面需要讲的源码文件；录屏不要展示 `.env` 或 Atlas URI。
- 自动化测试数据会清理，不会留作永久演示数据。

## 建议的 4 分 30 秒录像流程

| 时间 | 操作与讲解 |
| --- | --- |
| 0:00–0:35 | 展示 Signup 创建第三位用户，随后 Login 成功；说明沿用 PA2 |
| 0:35–1:10 | 创建第二个项目，填写名称、描述、状态并选择已注册负责人；展示项目列表与负责人姓名 |
| 1:10–1:40 | 查看成员，指出负责人自动加入且没有 Remove；把 Casey 加入第一个项目 |
| 1:40–2:10 | 切换第二个项目，也添加 Casey；解释同一个用户可以参与多个项目 |
| 2:10–2:35 | 再添加一次 Casey，展示重复提示；移除一个非负责人，观察列表立即更新 |
| 2:35–3:10 | 在 Atlas 展示 Projects 与 ProjectMemberships，指出关系字段是 ObjectId，展示同一用户的两条关系（如已删除其中一条，可先重新添加） |
| 3:10–4:10 | 展示 React 三个组件、fetch 调用、Express 路由；解释项目创建事务、重复检查和唯一索引、负责人删除保护 |
| 4:10–4:30 | 展示 README 的运行命令、测试结果，并结束录像 |

你不需要逐行朗读源码。重点是解释一次用户操作如何经过 React → Express → MongoDB → React，以及为什么成员关系要单独存储。

提交仍需你完成：录制 Zoom 视频、检查代码和 Git diff、提交并推送到你打算提交的仓库，然后在 Brightspace 提交 GitHub URL 和视频。实现过程没有自动提交或推送。

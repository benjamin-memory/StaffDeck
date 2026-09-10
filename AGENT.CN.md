# 仓库指南

## 项目结构与模块组织

StaffDeck 由 Python 3.11+ 的 FastAPI 服务与 React/TypeScript 控制台组合而成。后端应用代码位于 `backend/app/`；`backend/single_port_app.py` 等入口文件支持桌面端和单端口运行时。后端测试位于 `backend/tests/`，受支持的会话运行时为 Harness v2。前端代码位于 `frontend-enterprise/src/`，静态资源位于 `frontend-enterprise/public/`，测试文件以 `*.test.ts` 或 `*.test.tsx` 的形式与源码就近放置。开发生命周期工具脚本放在 `scripts/`，平台发布资源放在 `packaging/`。

## 构建、测试与开发命令

- `python3 -m venv backend/.venv && backend/.venv/bin/python -m pip install -e "backend[dev]"`：安装后端及测试依赖。
- `npm --prefix frontend-enterprise ci`：安装锁定版本的前端依赖。
- `scripts/dev_up.sh --detach`：构建前端并启动单端口应用；使用 `scripts/dev_status.sh` 和 `scripts/dev_down.sh` 查看状态或停止应用。
- `backend/.venv/bin/python -m pytest backend/tests`：运行后端测试套件。
- `backend/.venv/bin/ruff check backend`：检查 Python 代码风格。
- `npm --prefix frontend-enterprise test`：运行 Vitest；`npm --prefix frontend-enterprise run build`：执行 TypeScript 类型检查并进行 Vite 生产构建。
- 修改 UI 文案或 Vite 环境变量用法时，在 `frontend-enterprise` 目录下运行 `i18n:check` 和 `config:check`。

## 编码风格与命名规范

Python 使用四空格缩进、类型注解，函数/模块采用 `snake_case`，类采用 `PascalCase`；Ruff 面向 Python 3.11，行宽限制为 100 字符。TypeScript 启用严格模式，沿用现有的两空格缩进、单引号、语句末尾加分号的风格。React 组件使用 `PascalCase` 命名，Hook 以 `use...` 命名，测试文件按被测单元命名。前端导入优先使用 `@/` 别名。

## 测试指南

Python 测试文件命名为 `test_*.py`，前端测试文件命名为 `*.test.ts(x)`。对于行为变更，应添加有针对性的回归测试，尤其要覆盖权限、持久化、流式输出和渠道路由等方面。项目未配置数值化的覆盖率门槛，但变更涉及的代码路径仍应被测试覆盖。对于 UI 变更，还应在浏览器中验证受影响的路由和用户角色。

## 提交与拉取请求指南

遵循提交历史中简洁的 Conventional Commit 规范，例如 `feat(channels): add binding status` 或 `fix: reject unsafe avatar URLs`。保持提交聚焦单一目的。拉取请求应说明意图与风险、关联相关 issue、列出已运行的测试，并注明用于 UI 验证的路由和角色。可见的变更应附上截图，并保留工作区中与本次改动无关的变更。

## 安全与配置

将 `backend/.env.example` 复制为 `backend/.env`；切勿提交密钥或渠道凭据。使用强 `APP_SECRET` 值，并为外部凭据遵循最小权限原则。目前受支持的生产环境迁移路径基于 SQLite。

## Agent 技能

### Issue 跟踪

不要为本仓库创建或管理 issue。参见 `docs/agents/issue-tracker.md`。

### 分流标签

不使用 issue 分流标签。参见 `docs/agents/triage-labels.md`。

### 领域文档

使用单上下文（single-context）文档布局。参见 `docs/agents/domain.md`。

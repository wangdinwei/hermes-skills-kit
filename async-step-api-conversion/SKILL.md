---
name: async-step-api-conversion
description: 将 Flask 同步步骤 API 转为异步后台线程模式 — 防止 Cloudflare 524 超时。含后端线程+进度文件+通用轮询端点+前端 pollingCheck 全链路。
---

# 同步步骤 API → 异步转换

## 适用场景

Flask Web 应用中，步骤 API 执行 AI/ffmpeg 等耗时操作（>30秒），导致 Cloudflare/nginx 超时（524/504）。前端已使用 `step-task-manager.js` 的 `taskManager.run()`。

## 三层架构

```
前端: taskManager.run({ pollingCheck: true, ... })
  → POST /api/run-step/LXX      → 后端立即返回 {"success": true}
  → 轮询 /api/run-step/LXX/progress  → 读 .task_state.json
  → onSuccess(data)               → 完成回调
```

## 后端改造模板

### 1. 统一进度端点（一次性，已存在于 pipeline.py）

```python
@pipeline_bp.route('/run-step/<step_key>/progress', methods=['GET'])
def api_step_progress(step_key):
    # 从 .task_state.json 读取进度
```

白名单：`app.py` 的 `_PUBLIC_ROUTES` 加 `'pipeline.api_step_progress'`

### 2. 步骤 API 改造（每个步骤重复此模式）

```python
import threading

def api_run_LXX_impl():
    # 1. 参数解析 + 积分扣减（保持在主线程，可用 request/current_user）
    data = request.get_json()
    workspace = data.get("workspace", "").strip()
    # ...

    episode_dir = os.path.join(get_workspace_root(), workspace, "production", ...)
    step_dir = os.path.join(episode_dir, "LXX_目录名")
    os.makedirs(step_dir, exist_ok=True)

    # 2. 写初始进度
    def _write_state(status, message="", **kwargs):
        state = {"status": status, "message": message, "updated_at": datetime.now().isoformat(), **kwargs}
        with open(os.path.join(step_dir, ".task_state.json"), "w", encoding="utf-8") as f:
            json.dump(state, f, ensure_ascii=False)

    _write_state("running", "正在生成...")

    # 3. 打包 config（捕获所有局部变量，后台线程不能用 request）
    config = {"workspace": workspace, "project": project, "step_dir": step_dir, ...}

    # 4. 启动后台线程 + 立即返回
    threading.Thread(target=_run_LXX_background, args=(config,), daemon=True).start()
    return jsonify({"success": True, "message": "任务已提交"})

def _run_LXX_background(config):
    """后台线程：执行实际工作，写进度"""
    step_dir = config["step_dir"]

    def _write_bg(status, message="", **kwargs):
        state = {"status": status, "message": message, "updated_at": datetime.now().isoformat(), **kwargs}
        with open(os.path.join(step_dir, ".task_state.json"), "w") as f:
            json.dump(state, f, ensure_ascii=False)

    try:
        # 原同步代码全部移到这里
        result = _call_ai(...)
        _write_bg("done", "生成完成", result_field=result)
    except Exception as e:
        _write_bg("error", str(e))
```

## 前端改造

只需加一行：`pollingCheck: true`

```javascript
taskManager.run({
    action: 'generate-xxx',
    text: 'AI 正在生成...',
    runUrl: '/api/run-step/LXX',
    body: { ... },
    pollingCheck: true,  // ← 加这行
    onStop: _executeStop,
    onSuccess: function(data) {
        // data 来自 .task_state.json 的 done 状态
        // 兼容：data.success 或 data.status === 'done'
        if (data.success || data.status === 'done') {
            renderResult(data);
        }
    }
});
```

## 关键规则

| 规则 | 说明 |
|------|------|
| config 传参 | 后台线程不能用 `request`/`current_user`/`flask.g`，所有数据通过 config 字典传递 |
| 进度文件 | `.task_state.json` 必须含 `status` 字段：`running`/`done`/`error`/`idle` |
| pollingCheck | 前端加此标志后，`taskManager` 自动轮询，忽略 API 响应的数据 |
| onSuccess 数据来源 | 来自进度文件的 `done` 状态，不是 API 初始返回值 |
| daemon=True | 主进程退出时线程自动终止 |

## 踩坑记录

- **git checkout 危险**：本项目 git 仓库是旧版，`git checkout -- app.py` 会丢失数万行代码。恢复只能用 `releases/` 备份
- **write_file 危险**：绝不整体覆写 `base.html`/`step_base.html`，必须用 patch
- **白名单遗漏**：新增 Blueprint 路由需要加入 `_PUBLIC_ROUTES`，否则 401
- **L05 原 f-string bug**：`{element_id}` 被 Python 解析 → 改为 `{{element_id}}`
- **前后端 onSuccess 数据格式不匹配**：进度文件返回的是 `status: "done"` 而不是 `success: true`，前端需兼容

### 🔥 高危：API 返回格式错误导致"操作失败"

**根因**：`taskManager.run()` 检查 `data.success`，若 API 返回 `{"status": "started"}` 而非 `{"success": true}`，taskManager 立即进 `_handleError` → 显示"操作失败"→ 关闭 loading。用户看到 confirm → 瞬间闪 loading → 错误弹窗。

**铁律**：异步步骤 API 必须返回 `{"success": True, "message": "..."}`。

**隐蔽性**：用户以为 confirm 不消失、loading 不出现，实际上 confirm 正常关闭、loading 出现即灭（被 error 覆盖了）。

### 🔥 高危：子助手改造时漏 import

子助手修改步骤文件时可能漏掉原有 import（如 `from app.common import log_user_action`），导致 500。改造后必查：`threading`、`datetime`、`log_user_action` 是否都已导入。

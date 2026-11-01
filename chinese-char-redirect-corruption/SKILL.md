---
name: chinese-char-redirect-corruption
description: 诊断 Flask redirect() 中中文项目名未 URL 编码导致的字符损坏——跳转后 URL 中文错字、文件系统找不到项目。需要 quote() 修复。
category: devops
---

# 中文参数 redirect 字符损坏诊断

## 触发条件

用户创建中文名项目后，跳转到大纲页/首页时：
- URL 中的项目名和实际文件系统中的目录名**不同字**
- 页面显示"加载失败"或空状态
- outline API 返回 `{"ok": false, "error": "无大纲"}`
- 但项目目录明明存在

## 典型症状

```
URL 显示: p=%E6%9A%97%E6%B6%A6  → "暗润" (àn rùn)
文件系统: /data/.../production/暗涌/ → "暗涌" (àn yǒng)
```

两个汉字看似相同但 UTF-8 编码不同（字形接近但不同字）。

## 根因

Flask 的 `redirect()` 把 Python 字符串直接放入 HTTP `Location` 头。中文 UTF-8 字节在不同浏览器/代理层可能被重新解码为不同字符，导致：
1. 项目目录以正确名字创建（如"暗涌"）
2. redirect 未用 `quote()` 编码
3. 浏览器收到的 Location 头不含 % 编码，解析URL时重新解码成**另一个字**

## 排查步骤

### 1. 确认文件系统路径
普通用户的项目在 `/data/user-projects/<username>/production/<项目名>/`，管理员在 `/path/to/workspace

**⚠️ 注意 dual workspace：** `get_workspace_root()` 对普通用户返回 `/data/user-projects/`，对管理员返回 `/path/to/workspace

### 2. 核对 URL 参数
从浏览器地址栏复制参数，在终端解码对比：
```bash
python3 -c "from urllib.parse import unquote; print(unquote('暗润'))"
python3 -c "from urllib.parse import unquote; print(unquote('暗涌'))"
```

### 3. 查 API 响应
用 curl 调试 outline API：
```bash
curl -s "http://localhost:8889/api/project/outline?w=<workspace>&p=<project>&s=第1季"
```

### 4. 定位 redirect 源码
搜索项目中所有 `redirect(f"/..."` 模式，找到未编码的参数拼接：
```python
# ❌ 错误写法
return redirect(f"/outline?w={workspace}&p={project_name}&s=第1季")

# ✅ 正确写法
from urllib.parse import quote
return redirect(f"/outline?w={workspace}&p={quote(project_name)}&s=第1季")
```

## 修复

1. 在 redirect 处导入 `from urllib.parse import quote`
2. 对所有含中文的 query 参数包裹 `quote()`：
```python
return redirect(f"/outline?w={workspace}&p={quote(project_name)}&s={quote(season)}")
```

## 涉及文件（项目特有）
- `app/routes/auth.py` — `_bp_use_template` 路由 line 924
- `app/routes/api_endpoints.py` — `api_create_project_impl` redirect line

## 验证
1. 重启服务
2. 用中文名创建一个新项目
3. 检查 redirect 后 URL 中的中文参数是否正确编码（应为 `%XX%XX%XX` 格式）
4. 确认大纲页能正常加载并显示"待生成"状态

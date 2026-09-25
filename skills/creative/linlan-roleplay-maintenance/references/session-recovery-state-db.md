# state.db 会话查询与恢复配方（只读，网关运行时可用）

DB 路径：`~/.hermes/state.db`（profile 时为 `$HERMES_HOME/state.db`）。表：
- `sessions(id, source, parent_session_id, started_at, ended_at, title, message_count, session_key, chat_id, ...)` — 全部历史会话；`idx_sessions_parent` 索引父子链。
- `messages(id, session_id, role, content, tool_calls, timestamp, ...)` — 消息；FTS 搜索中文易失效，定位靠 browse/sqlite 而非 query。
- `gateway_routing` + `~/.hermes/sessions/sessions.json` — 现行路由（该聊天当前绑到哪个 session），不是历史清单。

## 全会话列表（找 pre-new 目标）
```sql
SELECT s.id, COALESCE(s.parent_session_id,'-'),
       datetime(s.started_at,'unixepoch','localtime'),
       datetime((SELECT MAX(m.timestamp) FROM messages m WHERE m.session_id=s.id),'unixepoch','localtime'),
       (SELECT COUNT(*) FROM messages m WHERE m.session_id=s.id)
FROM sessions s ORDER BY s.started_at;
```
解读：`/new` 生成的行 `parent_session_id` = 被替换的旧会话。连续 N 次 /new = N 行链到同一个旧会话；链末端的旧会话即"new 之前那个 session"。

## 读某会话尾部（macOS 无 tac：先 DESC 取 N 条再 ASC 排回）
```sql
SELECT datetime(m.timestamp,'unixepoch','localtime') || ' [' || m.role || '] ' ||
       COALESCE(substr(replace(m.content,char(10),' ⏎ '),1,400), '<'||m.tool_name||'>')
FROM (SELECT * FROM messages WHERE session_id='<SESSION_ID>' ORDER BY id DESC LIMIT 18) m
ORDER BY m.id ASC;
```

## 恢复动作（由用户执行，勿在跑着的网关上热改 gateway_routing/sessions.json 绑回去）
- Telegram 内：`/sessions` 浏览列表选中目标。
- CLI/桌面：`hermes --resume <session_id>`（`--resume` 按 ID 或标题）。
- 原地续：把旧会话尾部拉齐（上一条查询），在本会话重建场景继续；旧会话若含已废弃设定（如改龄前 canon），原地续比跳回更干净。

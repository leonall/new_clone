# 前端接入指南：OpenClaw Responses API（流式 + 步骤展示）

## 概览

从 Chat Completions 切换到 Responses API，前端可以实时看到 agent 的每个执行步骤（读 skill、搜索、生成文件等），效果类似 Claude Cowork。

- **旧 endpoint**：`POST /v1/chat/completions`（只返回最终文本）
- **新 endpoint**：`POST /v1/responses`（SSE 流式，每步可见）

---

## 接入参数

```
URL:    http://your-server:19789/v1/responses
Method: POST
Auth:   Authorization: Bearer public-skill-demo-2026
```

---

## 请求格式

```json
{
  "model": "openclaw",
  "stream": true,
  "input": "帮我做一份中国新能源汽车行业的竞争格局分析"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `model` | string | 固定填 `"openclaw"` 或 `"openclaw:main"` |
| `stream` | boolean | `true` 开启 SSE 流式；`false` 等待完整响应 |
| `input` | string | 用户输入 |
| `user` | string（可选） | 用户唯一 ID，同一 user 共享 session 上下文 |
| `instructions` | string（可选） | 追加到 system prompt |

---

## SSE 事件类型

| 事件 | 触发时机 | 用途 |
|------|----------|------|
| `response.created` | 请求开始 | 初始化 UI，显示 loading |
| `response.output_item.added` | 新步骤开始 | 渲染步骤卡片 |
| `response.output_text.delta` | 文字流式输出 | 追加到当前文本区域 |
| `response.output_text.done` | 当前文本段完成 | 步骤完成 |
| `response.output_item.done` | 一个 item 完成 | 更新步骤状态为 ✅ |
| `response.completed` | 全部结束 | 隐藏 loading，展示结果 |
| `response.failed` | 出错 | 显示错误信息 |

---

## 前端代码

### 1. 发起请求 + 解析 SSE 流

```javascript
async function callAgent(userMessage) {
  const response = await fetch('http://your-server:19789/v1/responses', {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer public-skill-demo-2026',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'openclaw',
      stream: true,
      input: userMessage,
    }),
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n');
    buffer = lines.pop(); // 保留不完整的行

    let currentEvent = '';
    for (const line of lines) {
      if (line.startsWith('event: ')) {
        currentEvent = line.slice(7).trim();
      } else if (line.startsWith('data: ')) {
        const raw = line.slice(6).trim();
        if (raw === '[DONE]') { onDone(); return; }
        try {
          const data = JSON.parse(raw);
          handleEvent(currentEvent, data);
        } catch (e) {}
      }
    }
  }
}
```

### 2. 事件处理

```javascript
function handleEvent(event, data) {
  switch (event) {
    case 'response.created':
      showLoading();
      break;
    case 'response.output_item.added':
      addStepCard(data);       // 新步骤开始，渲染卡片
      break;
    case 'response.output_text.delta':
      appendText(data.delta);  // 文字流式追加
      break;
    case 'response.output_item.done':
      markStepDone(data);      // 步骤打勾
      break;
    case 'response.completed':
      hideLoading();
      break;
    case 'response.failed':
      showError(data.error?.message || '请求失败');
      break;
  }
}
```

### 3. 步骤卡片渲染

```javascript
const stepLabels = {
  'web_search': '🔍 搜索资料...',
  'web_fetch':  '📄 读取页面...',
  'read':       '📂 读取文件...',
  'exec':       '⚙️ 执行脚本...',
  'write':      '💾 写入文件...',
};

function addStepCard(data) {
  const name = data?.item?.name || data?.item?.type || 'unknown';
  const label = stepLabels[name] || `🔧 ${name}...`;
  const id = data?.item?.id || Date.now();

  const card = document.createElement('div');
  card.className = 'step-card step-running';
  card.id = `step-${id}`;
  card.innerHTML = `<span class="icon">⏳</span><span>${label}</span>`;
  document.getElementById('steps-container').appendChild(card);
}

function markStepDone(data) {
  const card = document.getElementById(`step-${data?.item?.id}`);
  if (card) {
    card.classList.replace('step-running', 'step-done');
    card.querySelector('.icon').textContent = '✅';
  }
}

function appendText(delta) {
  document.getElementById('output').insertAdjacentText('beforeend', delta);
}
```

### 4. CSS

```css
.step-card {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  border-radius: 8px;
  margin: 4px 0;
  font-size: 14px;
  transition: all 0.3s;
}
.step-running {
  background: #f0f4ff;
  border-left: 3px solid #4a6cf7;
  color: #4a6cf7;
}
.step-done {
  background: #f0fff4;
  border-left: 3px solid #22c55e;
  color: #16a34a;
}
```

---

## 非流式模式（不需要展示过程时）

```javascript
const res = await fetch('http://your-server:19789/v1/responses', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer public-skill-demo-2026',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ model: 'openclaw', stream: false, input: userMessage }),
});

const result = await res.json();
const text = result.output
  ?.filter(i => i.type === 'message')
  ?.flatMap(i => i.content)
  ?.filter(c => c.type === 'output_text')
  ?.map(c => c.text)
  ?.join('') || '';
```

---

## 注意事项

1. **CORS**：浏览器直接请求需要后端代理，或在 gateway 配置 CORS 允许来源
2. **超时**：生成 PPT 等重型任务可能需要 60s+，不要设太短的 timeout
3. **Session 上下文**：传 `user` 字段让同一用户多次请求共享上下文
4. **文件下载**：PPT 生成后在服务器本地，下载需另行处理（静态文件服务方案）

---

## 快速验证（curl）

```bash
curl -N http://your-server:19789/v1/responses \
  -H 'Authorization: Bearer public-skill-demo-2026' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw",
    "stream": true,
    "input": "帮我做一份新能源汽车行业竞争分析"
  }'
```

看到 `event: response.output_item.added` 就说明接通了 ✅

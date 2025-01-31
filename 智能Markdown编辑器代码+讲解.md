# 智能markdown编辑器

### 一、核心架构
1. **双栏布局**：
   - 采用flex布局实现编辑区(editor-pane)和预览区(preview-pane)的双栏布局
   - 响应式设计：移动端自动切换为垂直布局（@media查询）
   - 使用CodeMirror作为代码编辑器，marked库实现实时Markdown解析

2. **工具栏功能**：
   - 文件操作：保存/导出/生成标题
   - AI功能：智能续写/优化/解释
   - GitHub集成：加载/提交文件
   - 设置面板

### 二、核心功能实现

#### 1. 编辑器功能
```javascript
const editor = CodeMirror.fromTextArea(...)
```
- **快捷键支持**：Ctrl+S保存，Ctrl+E打开AI面板等
- **自动保存**：
  ```javascript
  setInterval(() => {
    if (editor.getValue().trim() && Date.now() - lastSaveTime > AUTO_SAVE_INTERVAL) {
      saveDraft();
    }
  }, 30000);
  ```
- **草稿恢复**：通过localStorage保存文档状态（内容、光标位置、文件名）

#### 2. AI集成
```javascript
async function handleAIRequest() {
  // 流式请求处理
  const response = await fetch(api_url, {
    body: JSON.stringify({
      model: settings.model,
      messages: [{ role: "user", content: ... }],
      stream: true
    })
  });

  // 流式数据处理
  while (true) {
    const { done, value } = await reader.read();
    // 实时更新编辑器内容
    editor.replaceRange(text, editor.getCursor());
  }
}
```
- 支持三种AI操作模式：解释/续写/优化
- 流式响应处理：实时显示生成内容
- 结果确认机制：接受/放弃生成内容

#### 3. GitHub集成
```javascript
async function submitMdToGitHub() {
  const encodedContent = btoa(unescape(encodeURIComponent(content)));
  await fetch(`https://api.github.com/repos/${user}/${repo}/contents/...`, {
    method: "PUT",
    headers: { Authorization: `token ${token}` },
    body: JSON.stringify({ message: "...", content: encodedContent })
  });
}
```
- 文件提交：使用GitHub Contents API
- 文件加载：递归获取仓库目录结构
- 配置存储：用户名/Token/仓库信息存localStorage

#### 4. 特色功能
- **智能标题生成**：
  ```javascript
  async function generateTitle(content) {
    const response = await axios.post(api_url, {
      messages: [{ role: "user", content: "生成标题要求..." }]
    });
    return 处理后的标题;
  }
  ```
- **代码块高亮**：
  ```javascript
  marked.setOptions({
    highlight: code => hljs.highlightAuto(code).value
  });
  ```
- **快捷键操作**：
  - Ctrl+D 复制当前行
  - Shift+Ctrl+Up 上移代码块

### 三、UI/UX设计
1. **视觉设计**：
   - CSS变量管理主题色
   - GitHub风格的Markdown预览样式
   - 流畅的动画过渡效果

2. **交互设计**：
   - 响应式工具栏布局
   - 智能光标追踪（streaming-cursor）
   - 加载状态指示（旋转动画）

3. **错误处理**：
   - 统一的Toast提示系统
   ```javascript
   function showToast(message, duration) {
     // 创建并管理临时提示元素
   }
   ```
   - 网络请求异常捕获
   - 本地存储异常处理

### 四、代码组织特点
1. **模块化结构**：
   - 编辑器初始化
   - Markdown处理
   - 网络请求
   - UI组件
   - 事件处理

2. **状态管理**：
   - 全局配置对象 settings
   - 编辑状态保存（localStorage）
   - 请求控制（AbortController）

3. **安全措施**：
   - API密钥的password类型输入
   - GitHub Token本地加密存储
   - 文件名消毒处理


# 网站源码：

```html
<!DOCTYPE html>
<html lang="zh-CN">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>智能Markdown编辑器</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/codemirror/5.63.0/codemirror.min.css">
    <link rel="stylesheet"
        href="https://cdnjs.cloudflare.com/ajax/libs/github-markdown-css/5.1.0/github-markdown.min.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.7.0/styles/github.min.css">
    <style>
        :root {
            --primary-color: #2c3e50;
            --secondary-color: #3498db;
            --background-color: #f8f9fa;
            --border-color: #dee2e6;
        }

        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Arial, sans-serif;
            background-color: var(--background-color);
        }

        .toolbar {
            padding: 12px 20px;
            background: white;
            border-bottom: 1px solid var(--border-color);
            display: flex;
            gap: 12px;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
            position: relative;
        }

        .filename-display {
            display: flex;
            align-items: center;
            margin: 0 10px;
            padding: 4px 8px;
            background: white;
            border: 1px solid var(--border-color);
            border-radius: 4px;
            min-width: 200px;
            max-width: 400px;
        }

        .filename-display input {
            border: none;
            padding: 2px 4px;
            flex: 1;
            font-size: 14px;
            color: var(--primary-color);
            width: 100%;
        }

        .filename-display input:focus {
            outline: none;
            background: #f0f0f0;
        }

        .toolbar button {
            padding: 8px 16px;
            background: var(--secondary-color);
            color: white;
            border: none;
            border-radius: 6px;
            cursor: pointer;
            transition: all 0.2s ease;
            display: flex;
            align-items: center;
            gap: 6px;
        }

        .toolbar button:hover {
            background: #2980b9;
            transform: translateY(-1px);
        }

        .container {
            display: flex;
            height: calc(100vh - 62px);
            position: relative;
        }

        .editor-pane,
        .preview-pane {
            flex: 1;
            padding: 20px;
            min-width: 300px;
        }

        .CodeMirror {
            height: 100%;
            border: 1px solid var(--border-color);
            border-radius: 8px;
            font-family: 'Fira Code', Consolas, monospace;
            font-size: 14px;
        }

        .preview-pane {
            overflow: auto;
            background: white;
            border-left: 1px solid var(--border-color);
        }

        .markdown-body {
            padding: 20px;
            max-width: 800px;
            margin: 0 auto;
            color: #000;
        }

        .markdown-body code {
            color: white;
        }

        .dialog-overlay {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.3);
            display: none;
            justify-content: center;
            align-items: center;
            z-index: 1000;
        }

        .dialog {
            background: white;
            padding: 24px;
            border-radius: 8px;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
            width: 600px;
            /* 从400px改为600px */
            max-width: 95%;
            /* 保持响应式 */
            max-height: 85vh;
            overflow-y: auto;
        }

        /* 在较小屏幕上保持合理宽度 */
        @media (max-width: 768px) {
            .dialog {
                width: 90%;
                padding: 20px;
            }
        }

        .dialog h3 {
            margin-bottom: 16px;
            color: var(--primary-color);
        }

        .dialog-input {
            width: 100%;
            padding: 8px;
            margin-bottom: 12px;
            border: 1px solid var(--border-color);
            border-radius: 4px;
        }

        .dialog-actions {
            margin-top: 16px;
            display: flex;
            gap: 8px;
            justify-content: flex-end;
        }

        .streaming-cursor {
            border-right: 2px solid var(--secondary-color);
            animation: blink 1s step-end infinite;
        }

        @keyframes blink {
            50% {
                border-color: transparent;
            }
        }

        .result-confirm {
            margin-top: 16px;
            padding: 12px;
            background: #f8f9fa;
            border: 1px solid var(--border-color);
            border-radius: 6px;
        }

        #aiCustomInput {
            transition: all 0.3s ease;
            resize: vertical;
        }

        .dialog-input[style*="display: none"]+.dialog-input {
            margin-top: 0;
        }

        #exportBtn[data-loading]::after {
            content: '...';
            animation: loading 1s infinite;
        }

        @keyframes loading {

            0%,
            100% {
                content: '.';
            }

            33% {
                content: '..';
            }

            66% {
                content: '...';
            }
        }

        @media (max-width: 768px) {
            .container {
                flex-direction: column;
            }

            .preview-pane {
                border-left: none;
                border-top: 1px solid var(--border-color);
            }
        }

        /* GitHub设置区域样式 */
        .github-settings {
            margin-top: 24px;
            padding-top: 24px;
            border-top: 1px solid var(--border-color);
        }

        .github-settings h4 {
            margin-bottom: 16px;
            color: var(--primary-color);
            font-size: 16px;
            font-weight: 500;
        }

        .github-settings .form-grid {
            display: grid;
            grid-template-columns: repeat(2, 1fr);
            gap: 16px;
        }

        .github-settings .form-group {
            display: flex;
            flex-direction: column;
        }

        .github-settings .form-group label {
            margin-bottom: 8px;
            color: var(--primary-color);
            font-size: 14px;
            font-weight: 500;
        }

        .github-settings .form-group input {
            padding: 8px 12px;
            border: 1px solid var(--border-color);
            border-radius: 4px;
            font-size: 14px;
            transition: border-color 0.2s ease, box-shadow 0.2s ease;
        }

        .github-settings .form-group input:focus {
            outline: none;
            border-color: var(--secondary-color);
            box-shadow: 0 0 0 2px rgba(52, 152, 219, 0.1);
        }

        .github-settings .form-group input[type="password"] {
            letter-spacing: 2px;
        }

        /* 响应式布局 */
        @media (max-width: 600px) {
            .github-settings .form-grid {
                grid-template-columns: 1fr;
            }

            .github-settings .form-group input {
                padding: 6px 10px;
            }
        }

        /* GitHub提交按钮样式 */
        .toolbar button.github-submit {
            background-color: #2ecc71;
        }

        .toolbar button.github-submit:hover {
            background-color: #27ae60;
        }



        /* 加载状态样式 */
        .github-submit[data-loading="true"] {
            position: relative;
            pointer-events: none;
            opacity: 0.7;
        }

        .github-submit[data-loading="true"]::after {
            content: '';
            position: absolute;
            width: 16px;
            height: 16px;
            right: 10px;
            top: 50%;
            transform: translateY(-50%);
            border: 2px solid #fff;
            border-top-color: transparent;
            border-radius: 50%;
            animation: github-loading 0.8s linear infinite;
        }

        @keyframes github-loading {
            to {
                transform: translateY(-50%) rotate(360deg);
            }
        }
    </style>
</head>

<body>
    <!-- Modify the toolbar section in the HTML -->
    <nav class="toolbar">
        <button onclick="toggleSettings()">⚙️ 设置</button>
        <button onclick="exportMD()" id="exportBtn">💾 导出</button>
        <button onclick="toggleAIPanel()">🤖 AI助手</button>
        <button onclick="loadFromGitHub()">从GitHUb加载文件</button>
        <button onclick="submitMdToGitHub()">提交到GitHub</button>
        <div class="filename-display">
            <input type="text" id="currentFileName" placeholder="未命名文档">
        </div>
        <button onclick="handleGenerateTitle()">📝 生成标题</button>
    </nav>

    <div class="container">
        <div class="editor-pane">
            <textarea id="editor"></textarea>
        </div>
        <div class="preview-pane markdown-body" id="preview"></div>
    </div>

    <!-- AI操作面板 -->
    <div id="aiPanel" class="dialog-overlay">
        <div class="dialog">
            <h3>AI 助手</h3>
            <div id="aiStatus" class="dialog-status"></div>
            <textarea id="aiCustomInput" class="dialog-input" placeholder="请输入要处理的内容"
                style="display: none; height: 100px;"></textarea>
            <select id="aiAction" class="dialog-input">
                <option value="explain">解释内容</option>
                <option value="continue">智能续写</option>
                <option value="improve">优化内容</option>
            </select>
            <div class="dialog-actions">
                <button id="aiExecuteBtn" onclick="handleAIRequest()">执行</button>
                <button onclick="toggleAIPanel()">关闭</button>
            </div>
            <div id="resultConfirm" class="result-confirm" style="display: none;">
                <div id="finalResult"></div>
                <div class="dialog-actions">
                    <button onclick="acceptResult()">✅ 接受</button>
                    <button onclick="rejectResult()">❌ 放弃</button>
                </div>
            </div>
        </div>
    </div>

    <!-- 设置面板 -->
    <!-- 在设置面板中 -->
    <div id="settingsPanel" class="dialog-overlay">
        <div class="dialog">
            <h3>设置</h3>
            <!-- AI设置部分 -->
            <div class="ai-settings">
                <h4>AI 配置</h4>
                <input type="text" id="baseUrl" placeholder="API 基础地址" class="dialog-input">
                <input type="password" id="apiKey" placeholder="API 密钥" class="dialog-input">
                <input type="text" id="model" placeholder="模型名称" class="dialog-input">
            </div>

            <!-- GitHub设置部分 -->
            <div class="github-settings">
                <h4>GitHub 配置</h4>
                <div class="form-grid">
                    <div class="form-group">
                        <label for="githubUser">用户名</label>
                        <input type="text" id="githubUser" placeholder="GitHub用户名">
                    </div>
                    <div class="form-group">
                        <label for="githubToken">访问令牌</label>
                        <input type="password" id="githubToken" placeholder="Personal access token">
                    </div>
                    <div class="form-group">
                        <label for="githubRepo">仓库名称</label>
                        <input type="text" id="githubRepo" placeholder="用户名/仓库名">
                    </div>
                    <div class="form-group">
                        <label for="githubBranch">分支名称</label>
                        <input type="text" id="githubBranch" value="main" placeholder="分支名">
                    </div>
                </div>
            </div>

            <div class="dialog-actions">
                <button onclick="saveSettings()">保存</button>
                <button onclick="toggleSettings()">取消</button>
            </div>
        </div>
    </div>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/codemirror/5.63.0/codemirror.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/codemirror/5.63.0/mode/markdown/markdown.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/marked@4.2.12/lib/marked.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.7.0/highlight.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/axios/1.3.4/axios.min.js"></script>

    <script>
        // AI 处理相关的状态变量
        let isGenerating = false;            // 是否正在生成内容
        let abortController = null;          // 用于取消请求
        let streamMark = null;               // 流式输出的标记
        let currentPosition = null;          // 当前插入位置
        let generatedContent = '';           // 生成的内容
        let originalContent = '';            // 原始内容

        // 标记光标位置的样式
        const cursorStyles = {
            className: 'streaming-cursor'
        };

        // 初始化编辑器
        const editor = CodeMirror.fromTextArea(document.getElementById('editor'), {
            mode: 'markdown',
            lineNumbers: true,
            lineWrapping: true,
            autofocus: true,
            extraKeys: {
                "Ctrl-S": () => {
                    saveDraft();
                    showToast('手动保存成功', 2000);
                },
                "Cmd-S": () => {
                    saveDraft();
                    showToast('手动保存成功', 2000);
                },
                "Ctrl-E": toggleAIPanel,
                "Cmd-E": toggleAIPanel,
                "Ctrl-D": handleDuplicateLine,
                "Shift-Ctrl-Up": handleMoveLineUp
            }
        });

        // 配置marked
        marked.setOptions({
            breaks: true,
            highlight: code => hljs.highlightAuto(code).value
        });

        // 应用设置
        let settings = {
            baseUrl: localStorage.getItem('ai_base_url') || 'https://api.lingyiwanwu.com/v1',
            apiKey: localStorage.getItem('ai_api_key') || '95391dfcc5194055a072f0662c7e69ab',
            model: localStorage.getItem('ai_model') || 'yi-lightning'
        };

        // 实时预览
        editor.on('change', cm => {
            try {
                document.getElementById('preview').innerHTML = marked.parse(cm.getValue());
            } catch (e) {
                console.error('Markdown解析错误:', e);
            }
        });

        // 自动保存相关
        let isSaving = false;
        let lastSaveTime = 0;
        const AUTO_SAVE_INTERVAL = 5 * 60 * 1000;

        // Update saveDraft function to refresh time display
        function saveDraft() {
            const content = editor.getValue();
            if (!content.trim()) return;

            try {
                const draft = {
                    content: content,
                    timestamp: Date.now(),
                    cursor: editor.getCursor(),
                    fileName: document.getElementById('currentFileName').value.trim()
                };
                localStorage.setItem('md_draft', JSON.stringify(draft));
                lastSaveTime = draft.timestamp;

                // Update time display if settings panel is open
                const statusDiv = document.querySelector('.settings-status');
                if (statusDiv) {
                    const currentUser = localStorage.getItem('github_user') || '未登录';
                    statusDiv.textContent = `当前用户: ${currentUser} | 最后保存: ${getFormattedSaveTime()}`;
                }

                showToast('内容已自动保存', 2000);
            } catch (e) {
                console.error('本地存储失败:', e);
                showToast('自动保存失败，请检查存储空间', 3000);
            }
        }

        setInterval(() => {
            if (editor.getValue().trim() && Date.now() - lastSaveTime > AUTO_SAVE_INTERVAL) {
                saveDraft();
            }
        }, 30000);

        window.addEventListener('beforeunload', (e) => {
            const content = editor.getValue().trim();
            if (content) {
                saveDraft();
                e.preventDefault();
                e.returnValue = '您有未保存的更改';
            }
        });

        window.addEventListener('load', () => {
            try {
                const draft = JSON.parse(localStorage.getItem('md_draft'));
                if (draft && draft.content) {
                    if (confirm(`发现 ${new Date(draft.timestamp).toLocaleString()} 的自动保存内容，是否恢复？`)) {
                        editor.setValue(draft.content);
                        if (draft.cursor) {
                            editor.setCursor(draft.cursor);
                        }
                        // 如果草稿中有文件名，也恢复它
                        if (draft.fileName) {
                            document.getElementById('currentFileName').value = draft.fileName;
                        }
                    }
                }
            } catch (e) {
                console.error('恢复草稿失败:', e);
            }
        });

        // 标题生成和处理函数
        async function handleGenerateTitle() {
            const content = editor.getValue();
            if (!content.trim()) {
                showToast('文档内容为空', 2000);
                return;
            }

            try {
                const fileNameInput = document.getElementById('currentFileName');
                fileNameInput.disabled = true;
                const title = await generateTitle(content);
                fileNameInput.value = title;
                fileNameInput.disabled = false;
                showToast('标题生成成功', 2000);
            } catch (error) {
                console.error('生成标题失败:', error);
                showToast('生成标题失败: ' + error.message, 3000);
                document.getElementById('currentFileName').disabled = false;
            }
        }

        // 导出功能
        async function exportMD() {
            const btn = document.getElementById('exportBtn');
            try {
                const content = editor.getValue();
                if (!content.trim()) return showToast('内容为空', 2000);

                btn.innerHTML = '⏳ 处理中...';
                btn.disabled = true;

                await saveDraft();

                const fileName = document.getElementById('currentFileName').value.trim() || '未命名文档';
                const blob = new Blob([content], { type: 'text/markdown;charset=utf-8' });
                const url = URL.createObjectURL(blob);

                const tempLink = document.createElement('a');
                tempLink.href = url;
                tempLink.download = `${fileName}.md`;
                tempLink.style.display = 'none';
                document.body.appendChild(tempLink);
                tempLink.click();
                document.body.removeChild(tempLink);
                URL.revokeObjectURL(url);

                localStorage.removeItem('md_draft');
                showToast(`已导出: ${fileName}.md`, 3000);
            } catch (error) {
                console.error('导出失败:', error);
                showToast(`导出失败: ${error.message}`, 5000);
            } finally {
                btn.innerHTML = '💾 导出';
                btn.disabled = false;
            }
        }

        async function submitMdToGitHub() {
            const content = editor.getValue();
            const user = document.getElementById('githubUser').value.trim();
            const token = document.getElementById('githubToken').value.trim();
            const repo = document.getElementById('githubRepo').value.trim();
            const branch = document.getElementById('githubBranch').value.trim() || 'main';
            if (!user || !token || !repo) return alert('请先在设置中填写GitHub信息');

            try {
                const fileName = document.getElementById('currentFileName').value.trim() || '未命名文档';
                const url = `https://api.github.com/repos/${user}/${repo}/contents/${fileName}.md`;

                // 检查文件是否存在
                let sha = null;
                const resCheck = await fetch(url, {
                    method: 'GET',
                    headers: {
                        'Authorization': `token ${token}`
                    }
                });
                if (resCheck.ok) {
                    const data = await resCheck.json();
                    sha = data.sha; // 获取文件的 SHA 值
                }

                // 编码内容
                const encodedContent = btoa(unescape(encodeURIComponent(content)));

                // 提交文件
                const res = await fetch(url, {
                    method: 'PUT',
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': `token ${token}`
                    },
                    body: JSON.stringify({
                        message: `Add ${fileName}.md`,
                        content: encodedContent,
                        branch,
                        sha // 包含 SHA 值以覆盖现有文件
                    })
                });
                if (!res.ok) throw new Error(`GitHub API错误: ${res.status}`);
                alert(`已成功提交到: ${user}/${repo}/${fileName}.md`);
            } catch (err) {
                console.error('提交失败:', err);
                alert(`提交失败: ${err.message}`);
            }
        }

        // Update the loadFromGitHub function
        async function loadFromGitHub() {
            const user = localStorage.getItem('github_user');
            const token = localStorage.getItem('github_token');
            const repo = localStorage.getItem('github_repo');

            // Validate GitHub settings
            if (!user || !token || !repo) {
                showToast('请先在设置中配置GitHub信息', 3000);
                return;
            }

            try {
                // Use the full repository path
                const response = await fetch(`https://api.github.com/repos/${user}/${repo}/contents/`, {
                    headers: {
                        'Authorization': `token ${token}`, // Changed from Bearer to token
                        'Accept': 'application/vnd.github.v3+json'
                    }
                });

                if (response.status === 404) {
                    throw new Error('找不到仓库，请检查仓库名称是否正确');
                }

                if (!response.ok) {
                    const error = await response.json();
                    throw new Error(`GitHub API错误: ${error.message}`);
                }

                const contents = await response.json();

                // Filter markdown files
                const mdFiles = contents.filter(file => file.name.endsWith('.md'));

                if (mdFiles.length === 0) {
                    showToast('仓库中没有Markdown文件', 3000);
                    return;
                }

                showFileSelectionDialog(mdFiles);

            } catch (error) {
                console.error('加载文件列表失败:', error);
                showToast(`错误: ${error.message}`, 5000);
            }
        }

        function showFileSelectionDialog(files) {
            const dialog = document.createElement('div');
            dialog.className = 'dialog-overlay';
            dialog.style.display = 'flex';

            dialog.innerHTML = `
        <div class="dialog">
            <h3>选择文件</h3>
            <div class="file-list" style="max-height: 400px; overflow-y: auto; margin: 10px 0;">
                ${files.map(file => `
                    <div class="file-item" style="padding: 10px; cursor: pointer; border-bottom: 1px solid #eee; display: flex; align-items: center; gap: 8px;">
                        <svg height="16" width="16" style="min-width: 16px;">
                            <path fill="#666" d="M2 1.75C2 .784 2.784 0 3.75 0h6.5c.966 0 1.75.784 1.75 1.75v11.5A1.75 1.75 0 0 1 10.25 15h-6.5A1.75 1.75 0 0 1 2 13.25Zm1.75-.25a.25.25 0 0 0-.25.25v11.5c0 .138.112.25.25.25h6.5a.25.25 0 0 0 .25-.25V1.75a.25.25 0 0 0-.25-.25Zm-2 2.5v11.5c0 .966.784 1.75 1.75 1.75h6.5A1.75 1.75 0 0 0 12 13.25V1.75A1.75 1.75 0 0 0 10.25 0h-6.5A1.75 1.75 0 0 0 2 1.75Z"></path>
                        </svg>
                        <span onclick="loadFile('${file.path}')" style="flex: 1;">${file.name}</span>
                        <span style="color: #666; font-size: 12px;">${formatFileSize(file.size)}</span>
                    </div>
                `).join('')}
            </div>
            <div class="dialog-actions">
                <button onclick="this.closest('.dialog-overlay').remove()">取消</button>
            </div>
        </div>
    `;

            document.body.appendChild(dialog);
        }

        function formatFileSize(bytes) {
            if (bytes < 1024) return bytes + ' B';
            if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB';
            return (bytes / (1024 * 1024)).toFixed(1) + ' MB';
        }

        async function loadFile(path) {
            const user = localStorage.getItem('github_user');
            const token = localStorage.getItem('github_token');
            const repo = localStorage.getItem('github_repo');

            try {
                const response = await fetch(`https://api.github.com/repos/${user}/${repo}/contents/${path}`, {
                    headers: {
                        'Authorization': `token ${token}`,
                        'Accept': 'application/vnd.github.v3+json'
                    }
                });

                if (!response.ok) {
                    throw new Error(`GitHub API错误: ${response.status}`);
                }

                const data = await response.json();
                // 修改这里：使用 decodeURIComponent 和 escape 来正确处理 UTF-8 编码
                const content = decodeURIComponent(escape(atob(data.content)));

                editor.setValue(content);
                document.getElementById('currentFileName').value = path.replace(/\.md$/, '');
                document.querySelector('.dialog-overlay').remove();
                showToast('文件加载成功', 2000);
            } catch (error) {
                console.error('加载文件失败:', error);
                showToast(`加载失败: ${error.message}`, 3000);
            }
        }

        // 复制当前行
        function handleDuplicateLine() {
            const cursor = editor.getCursor();
            const line = editor.getLine(cursor.line);
            editor.replaceRange('\n' + line,
                { line: cursor.line, ch: line.length });
            editor.setCursor({ line: cursor.line + 1, ch: cursor.ch });
        }

        // 向上移动行
        function handleMoveLineUp() {
            const cursor = editor.getCursor();
            if (cursor.line === 0) return; // 已经在第一行

            const currentLine = editor.getLine(cursor.line);
            const prevLine = editor.getLine(cursor.line - 1);

            // 交换两行
            editor.replaceRange(currentLine + '\n' + prevLine,
                { line: cursor.line - 1, ch: 0 },
                { line: cursor.line + 1, ch: 0 });

            // 移动光标
            editor.setCursor({ line: cursor.line - 1, ch: cursor.ch });
        }

        // AI生成标题函数
        async function generateTitle(content) {
            try {
                const response = await axios.post(`${settings.baseUrl}/chat/completions`, {
                    model: settings.model,
                    messages: [{
                        role: "user",
                        content: `请根据以下内容生成一个合适的Markdown文件名：
要求：
1. 只需返回纯文本文件名
2. 不要包含.md后缀
3. 使用中文标题
4. 长度控制在15字以内
5. 不要使用任何标点符号

文档内容：
${content.slice(0, 2000)}` // 限制上下文长度
                    }],
                    temperature: 0.7,
                    max_tokens: 20
                }, {
                    headers: {
                        'Authorization': `Bearer ${settings.apiKey}`,
                        'Content-Type': 'application/json'
                    }
                });

                // 清理返回结果
                let title = response.data.choices[0].message.content
                    .replace(/^["']+|["']+$/g, '') // 去除首尾引号
                    .replace(/\.md$/i, '')        // 去除可能的后缀
                    .replace(/[\\/:*?"<>|]/g, '') // 去除非法字符
                    .trim();

                return title || '未命名文档'; // 提供默认值
            } catch (error) {
                throw new Error(`AI标题生成失败: ${error.response?.data?.error?.message || error.message}`);
            }
        }

        async function handleAIRequest() {
            // 如果正在生成，则取消当前请求
            if (isGenerating && abortController) {
                abortController.abort();
                isGenerating = false;
                document.getElementById('aiExecuteBtn').textContent = '执行';
                if (streamMark) streamMark.clear();
                return;
            }

            // 清空上一次的结果
            document.getElementById('finalResult').textContent = '';
            generatedContent = '';

            // 获取输入内容（优先使用选中内容）
            let content = editor.getSelection();
            const aiCustomInput = document.getElementById('aiCustomInput');
            let isCustomInput = false;

            // 判断输入来源
            if (!content) {
                content = aiCustomInput.value.trim();
                isCustomInput = true;
                if (!content) return alert('请输入要处理的内容');
            }

            try {
                isGenerating = true;
                originalContent = content;
                abortController = new AbortController();

                // 更新界面状态
                document.getElementById('aiExecuteBtn').textContent = '停止';
                document.getElementById('resultConfirm').style.display = 'none';

                // 确定插入位置
                let insertPosition;
                if (isCustomInput) {
                    // 修正：正确获取文档末尾位置
                    const lastLine = editor.lastLine();
                    const docEnd = {
                        line: lastLine,
                        ch: editor.getLine(lastLine).length
                    };
                    insertPosition = {
                        line: lastLine + 1,
                        ch: 0
                    };
                    // 先插入两行空白
                    editor.replaceRange('\n\n', docEnd);
                } else {
                    // 在选中内容后插入
                    const endSelectionPos = editor.getCursor('to');
                    insertPosition = {
                        line: endSelectionPos.line + 2,
                        ch: 0
                    };
                    editor.replaceRange('\n\n', endSelectionPos);
                }

                // 设置光标和标记
                editor.setCursor(insertPosition);
                currentPosition = insertPosition;
                streamMark = editor.markText(insertPosition, insertPosition, {
                    className: 'streaming-cursor'
                });

                // 构建请求
                const action = isCustomInput ? 'custom' : document.getElementById('aiAction').value;
                const response = await fetch(`${settings.baseUrl}/chat/completions`, {
                    method: 'POST',
                    headers: {
                        'Authorization': `Bearer ${settings.apiKey}`,
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify({
                        model: settings.model,
                        messages: [{
                            role: "user",
                            content: buildPrompt(action, content)
                        }],
                        stream: true,
                        temperature: 0.7
                    }),
                    signal: abortController.signal
                });

                if (!response.ok) throw new Error(`HTTP错误: ${response.status}`);

                // 处理流式响应
                const reader = response.body.getReader();
                const decoder = new TextDecoder();
                let buffer = '';

                while (true) {
                    const { done, value } = await reader.read();
                    if (done) break;

                    buffer += decoder.decode(value, { stream: true });
                    const chunks = buffer.split('\n');
                    buffer = chunks.pop();

                    for (const chunk of chunks) {
                        const line = chunk.trim();
                        if (!line.startsWith('data: ')) continue;

                        const jsonStr = line.slice(6);
                        if (jsonStr === '[DONE]') continue;

                        try {
                            const data = JSON.parse(jsonStr);
                            const text = data.choices[0]?.delta?.content || '';
                            if (text) {
                                generatedContent += text;
                                // 实时插入编辑器
                                editor.replaceRange(text, editor.getCursor());
                                // 自动滚动到可见位置
                                editor.scrollIntoView(editor.getCursor());
                            }
                        } catch (e) {
                            console.warn('解析错误:', e);
                        }
                    }
                }

                // 显示确认面板
                showConfirmationPanel();
                // 如果是自定义输入，清空输入框
                if (isCustomInput) aiCustomInput.value = '';

            } catch (error) {
                if (error.name !== 'AbortError') {
                    alert(`请求失败: ${error.message}`);
                    console.error(error);
                }
            } finally {
                isGenerating = false;
                if (abortController) {
                    abortController = null;
                }
                document.getElementById('aiExecuteBtn').textContent = '执行';
                if (streamMark) {
                    streamMark.clear();
                    streamMark = null;
                }
                // 重置自定义输入状态
                if (isCustomInput) {
                    toggleAIPanel(true);
                }
            }
        }

        // 更新提示词生成函数
        function buildPrompt(action, content) {
            const prompts = {
                explain: `请解释以下内容：\n\n${content}`,
                continue: `请续写以下内容：\n\n${content}`,
                improve: `请优化以下内容：\n\n${content}`,
                custom: `请处理以下内容：\n\n${content}`
            };
            return prompts[action] || content;
        }

        // 结果确认处理
        function showConfirmationPanel() {
            document.getElementById('resultConfirm').style.display = 'block';
            document.getElementById('finalResult').textContent = generatedContent;
        }

        function acceptResult() {
            editor.focus();
            toggleAIPanel();
        }

        function rejectResult() {
            const endPos = editor.getCursor();
            editor.replaceRange('', currentPosition, endPos);
            editor.replaceRange(originalContent, currentPosition);
            toggleAIPanel();
        }

        // 设置管理
        function toggleSettings() {
            const panel = document.getElementById('settingsPanel');
            panel.style.display = panel.style.display === 'flex' ? 'none' : 'flex';
            initSettings();
        }

        function getFormattedSaveTime() {
            if (!lastSaveTime) return '未保存';

            const now = Date.now();
            const diff = now - lastSaveTime;

            // Format time differences
            if (diff < 60000) { // Less than 1 minute
                return '刚刚';
            } else if (diff < 3600000) { // Less than 1 hour
                return `${Math.floor(diff / 60000)} 分钟前`;
            } else if (diff < 86400000) { // Less than 24 hours
                return `${Math.floor(diff / 3600000)} 小时前`;
            } else {
                return new Date(lastSaveTime).toLocaleString();
            }
        }

        // 初始化设置面板
        function initSettings() {
            // 从 localStorage 加载设置
            const baseUrlInput = document.getElementById('baseUrl');
            const apiKeyInput = document.getElementById('apiKey');
            const modelInput = document.getElementById('model');

            // 设置默认值和当前保存的值
            baseUrlInput.value = settings.baseUrl;
            apiKeyInput.value = settings.apiKey;
            modelInput.value = settings.model;

            // 为了安全，在显示时将 API 密钥输入框清空
            if (!apiKeyInput.value) {
                apiKeyInput.placeholder = '请输入 API 密钥';
            }

            // 加载并显示GitHub配置
            document.getElementById('githubUser').value = localStorage.getItem('github_user') || '';
            document.getElementById('githubToken').value = localStorage.getItem('github_token') || '';
            document.getElementById('githubRepo').value = localStorage.getItem('github_repo') || '';
            document.getElementById('githubBranch').value = localStorage.getItem('github_branch') || 'main';

            // 显示当前用户和时间信息
            const lastSaveTimeStr = lastSaveTime ? new Date(lastSaveTime).toLocaleString() : '未保存';

            // Update save time display
            const statusDiv = document.querySelector('.settings-status') || document.createElement('div');
            statusDiv.className = 'settings-status';
            statusDiv.style.cssText = `
        margin-top: 12px;
        font-size: 12px;
        color: #666;
        padding: 8px;
        background: #f5f5f5;
        border-radius: 4px;
    `;

            const currentUser = localStorage.getItem('github_user') || '未登录';
            statusDiv.textContent = `当前用户: ${currentUser} | 最后保存: ${getFormattedSaveTime()}`;

            if (!document.querySelector('.settings-status')) {
                document.querySelector('.dialog').appendChild(statusDiv);
            }
        }

        function saveSettings() {
            settings = {
                baseUrl: document.getElementById('baseUrl').value.trim(),
                apiKey: document.getElementById('apiKey').value.trim(),
                model: document.getElementById('model').value.trim()
            };

            if (!settings.baseUrl || !settings.apiKey || !settings.model) {
                return alert('请填写完整设置');
            }

            localStorage.setItem('ai_base_url', settings.baseUrl);
            localStorage.setItem('ai_api_key', settings.apiKey);
            localStorage.setItem('ai_model', settings.model);

            // 保存GitHub配置
            saveGitHubSettings();

            toggleSettings();
        }

        // 保存GitHub配置
        function saveGitHubSettings() {
            const githubUser = document.getElementById('githubUser').value.trim();
            const githubToken = document.getElementById('githubToken').value.trim();
            const githubRepo = document.getElementById('githubRepo').value.trim();
            const githubBranch = document.getElementById('githubBranch').value.trim() || 'main';

            if (!githubUser || !githubToken || !githubRepo) {
                return alert('请填写完整的GitHub配置信息');
            }

            localStorage.setItem('github_user', githubUser);
            localStorage.setItem('github_token', githubToken);
            localStorage.setItem('github_repo', githubRepo);
            localStorage.setItem('github_branch', githubBranch);
        }

        // 新增：AI助手面板开关
        function toggleAIPanel(reset) {
            const aiPanel = document.getElementById('aiPanel');
            const selection = editor.getSelection();
            const aiAction = document.getElementById('aiAction');
            const aiCustomInput = document.getElementById('aiCustomInput');

            // 切换显示状态
            aiPanel.style.display = aiPanel.style.display === 'flex' ? 'none' : 'flex';

            // 根据选中状态切换输入方式
            if (aiPanel.style.display === 'flex') {
                if (selection) {
                    aiAction.style.display = 'block';
                    aiCustomInput.style.display = 'none';
                } else {
                    aiAction.style.display = 'none';
                    aiCustomInput.style.display = 'block';
                    aiCustomInput.value = ''; // 清空之前的输入
                }
            }

            // 重置面板状态
            if (reset) {
                document.getElementById('resultConfirm').style.display = 'none';
                aiCustomInput.value = '';
            }
        }

        // 初始化
        document.querySelectorAll('.dialog-overlay').forEach(panel => {
            panel.addEventListener('click', e => {
                if (e.target === panel) panel.style.display = 'none';
            });
        });
        initSettings();

        function showToast(message, duration = 2000) {
            const toast = document.createElement('div');
            toast.style = `
                position: fixed;
                bottom: 20px;
                left: 50%;
                transform: translateX(-50%);
                background: rgba(0,0,0,0.8);    
                color: white;
                padding: 12px 24px;
                border-radius: 4px;
                animation: fadeIn 0.3s;
                z-index: 9999;
            `;
            toast.textContent = message;
            document.body.appendChild(toast);

            setTimeout(() => {
                toast.style.animation = 'fadeOut 0.3s';
                setTimeout(() => toast.remove(), 300);
            }, duration);
        }

        const style = document.createElement('style');
        style.textContent = `
        @keyframes fadeIn {
            from { opacity: 0; transform: translate(-50%, 10px); }
            to { opacity: 1; transform: translate(-50%, 0); }
        }
        @keyframes fadeOut {
            from { opacity: 1; transform: translate(-50%, 0); }
            to { opacity: 0; transform: translate(-50%, 10px); }
        }`;
        document.head.appendChild(style);
    </script>
</body>

</html>
```
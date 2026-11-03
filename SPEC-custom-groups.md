# Custom Groups Feature — Implementation Spec

这是一次澄清完成的需求，请按此规格实现，不要扩展范围、不要加额外功能。完成后本文件可删除。

## 目标

让用户把多个 host 的 tabs 聚合成一个"自定义组"显示在 dashboard 上。首个用例：AI 组（把 ChatGPT / Claude / Gemini / Kimi / DeepSeek / 等 15 个 AI 站点合并为一张卡片）。

同时重构配置机制：从 `config.local.js` 文件配置改为 `chrome.storage.local` + 内建 UI。

## 数据模型

存储位置：`chrome.storage.local` 的 `customGroups` 键。

```js
customGroups = [
  {
    key: string,          // 唯一标识，格式 'grp-' + Date.now().toString(36)；预置组用 'grp-ai-default'
    label: string,        // 用户可见名称
    matchers: [
      {
        host?: string,           // 精确 hostname，优先级高于 hostEndsWith
        hostEndsWith?: string,   // 后缀匹配（host 和 hostEndsWith 二选一，至少一个必填）
        pathPrefix?: string      // 可选，pathname 前缀匹配
      },
      ...
    ]
  },
  ...
]
```

## 首次种子数据

如果 `chrome.storage.local.customGroups` 不存在或为空数组，写入一个默认 AI 组：

```js
[{
  key: 'grp-ai-default',
  label: 'AI',
  matchers: [
    { host: 'chatgpt.com' },
    { host: 'chat.openai.com' },
    { host: 'claude.ai' },
    { host: 'code.claude.com' },
    { host: 'gemini.google.com' },
    { host: 'kimi.com' },
    { host: 'chat.deepseek.com' },
    { host: 'perplexity.ai' },
    { host: 'poe.com' },
    { host: 'x.com', pathPrefix: '/i/grok' },
    { host: 'chat.qwen.ai' },
    { host: 'chatglm.cn' },
    { host: 'yuanbao.tencent.com' },
    { host: 'yiyan.baidu.com' },
    { host: 'hailuo.ai' },
    { host: 'doubao.com' }
  ]
}]
```

## 代码改动清单

### extension/app.js

1. **加载 customGroups**：在启动时从 `chrome.storage.local` 读取，若空则写入上述默认种子。挂到模块级变量 `customGroups`。
2. **移除 `LOCAL_CUSTOM_GROUPS` 和 `LOCAL_LANDING_PAGE_PATTERNS` 两个分支**（约在第 1044 行、第 1071 行）。`LANDING_PAGE_PATTERNS` 的默认内容保留硬编码。
3. **重写 `matchCustomGroup(url)`**：以组为中心，遍历 group.matchers。

   ```js
   function matchCustomGroup(url) {
     try {
       const parsed = new URL(url);
       for (const group of customGroups) {
         for (const m of group.matchers) {
           const hostOk = m.host
             ? parsed.hostname === m.host
             : m.hostEndsWith
               ? parsed.hostname.endsWith(m.hostEndsWith)
               : false;
           if (!hostOk) continue;
           if (m.pathPrefix && !parsed.pathname.startsWith(m.pathPrefix)) continue;
           return group;
         }
       }
       return null;
     } catch { return null; }
   }
   ```

4. **分组时用 `group.key` 作 domain，保留 `label`**：

   ```js
   const g = matchCustomGroup(tab.url);
   if (g) {
     if (!groupMap[g.key]) groupMap[g.key] = { domain: g.key, label: g.label, tabs: [], isCustom: true };
     groupMap[g.key].tabs.push(tab);
     continue;
   }
   ```

5. **排序**：Homepages（`__landing-pages__`）最前，然后自定义组（`isCustom === true`，按 customGroups 数组顺序），最后普通 host 组按 tab 数降序。不再保留原来的"landing domain 优先"逻辑（那是为原来 landing group 成员的同域 tab 排序用的，自定义组的 matchers 已经覆盖此需求）。

6. **添加 Settings Modal 逻辑**：
   - 齿轮按钮点击 → 打开 modal，渲染当前 customGroups 状态
   - 每组一行：`[label input]  [删除按钮]`；点击展开 matcher 列表
   - 每个 matcher 一行：`[host 输入框]  [pathPrefix 输入框]  [删除按钮]`
   - host 输入框失焦时：若输入能被 `new URL()` 解析，抽取 `hostname` 并去掉前缀 `www.`；否则保留原值
   - "+ 添加 matcher" 按钮（组内）和 "+ 添加组" 按钮（底部）
   - 保存按钮 → 校验（label 非空、至少有一个 matcher、每个 matcher host 非空）→ 写入 `chrome.storage.local` → 关闭 modal → 调用 `fetchOpenTabs()` + 重渲
   - 取消按钮 → 直接关闭，不保存
   - 新增组时 `key = 'grp-' + Date.now().toString(36)`

7. **已有的 `customGroups` 和 `matchCustomGroup` 变量位置**需要调整为模块级（当前是在 render 函数内部闭包），因为 modal 保存后要能读写。

### extension/index.html

1. **移除** `<script src="config.local.js" onerror="...">` 行（第 130 行附近），以及注释 `<!-- Personal config ... -->`（第 128 行）。
2. **在 header 右上角添加齿轮按钮**，与已有的 "close all" 等按钮放一起。class `settings-btn`，aria-label `Settings`。SVG 齿轮图标 inline。
3. **添加 modal 骨架**（默认 hidden）：

   ```html
   <section id="settings-modal" class="settings-modal hidden" aria-hidden="true">
     <div class="settings-backdrop" data-action="close-settings"></div>
     <div class="settings-panel">
       <header class="settings-header">
         <h2>Custom Groups</h2>
         <button class="settings-close" data-action="close-settings" aria-label="Close">✕</button>
       </header>
       <div class="settings-body" id="settings-groups-list"></div>
       <footer class="settings-footer">
         <button class="settings-add-group" data-action="add-group">+ Add group</button>
         <div class="spacer"></div>
         <button class="settings-cancel" data-action="close-settings">Cancel</button>
         <button class="settings-save" data-action="save-settings">Save</button>
       </footer>
     </div>
   </section>
   ```

### extension/style.css

添加 `.settings-btn`、`.settings-modal`、`.settings-backdrop`、`.settings-panel`、`.settings-header`、`.settings-body`、`.settings-footer`、`.group-row`、`.group-row.expanded`、`.matcher-row`、`.matcher-input`、`.hidden` 等样式。

视觉风格保持与现有 dashboard 一致（边框、圆角、配色、字体大小）。modal 居中，背景半透明遮罩，panel 最大宽 640px、最大高 80vh 内滚。

### extension/.gitignore

保留 `extension/config.local.js` 这一行不动（无害，且能防御遗留文件被误提交）。

### AGENTS.md

把提到 `config.local.js` 的部分改写为"通过 dashboard 右上角齿轮按钮配置自定义组"。不要新增别的内容。

## 测试清单（人工验证）

1. 全新 profile 打开 new tab → 看到 AI 组卡片（含若干 AI 站的 tabs，或若当前没 AI tabs 则不显示该卡片但 storage 里已有数据）
2. 同时打开 claude.ai、chatgpt.com、kimi.com → 全部落入 AI 卡片
3. 打开 x.com（刷推首页）→ 独立的 x.com 组，不进 AI
4. 打开 x.com/i/grok → 进 AI 组
5. 点击齿轮 → modal 打开，能看到 AI 组和全部 matcher
6. 修改 AI label 为 "AI Tools" → save → 卡片 label 更新
7. 新建 "Work" 组，添加 matcher host=github.com pathPrefix=/myorg → save → github.com/myorg 的 tabs 进 Work 卡片，github.com 其他 tabs 仍在 github.com 卡
8. 在 host 输入框粘贴 `https://foo.com/bar` 失焦 → 变成 `foo.com`
9. 删除 AI 组 → save → 刷新 → AI 组消失，相关 tabs 回到各自 host 组
10. 校验：label 空 / matcher host 空时，save 提示错误（alert 或行内红字都行，简单即可）

## 实现原则

- KISS：不引入任何第三方库，不写 sortable / drag。
- 不做"系统组 vs 用户组"区分，预置组写入 storage 后和用户建的组完全对等。
- 不支持拖拽排序：自定义组之间顺序就按 customGroups 数组顺序。
- host 输入容错只做最小实现（URL 解析抽 hostname + 去 `www.`），不做更复杂的 punycode/正则。
- 保存是"全量覆盖"：UI 里的状态直接 set 到 storage，不做增量 diff。
- key 字段用户不可见、不可编辑。

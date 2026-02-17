# FreePS 国际化(i18n)工作计划

> **目标**: 实现中英双语支持，并根据浏览器语言自动切换。

## 一、项目阅读理解总结

### 1.1 项目概况
- **项目名称**: FreePS - 在线图像编辑器
- **技术栈**: HTML5 + Canvas，基于 [miniPaint](https://github.com/viliusle/miniPaint) 开发
- **部署方式**: 纯静态，主要文件为 `index.html` + `dist/bundle.js`

### 1.2 现有 i18n 基础
经分析发现，**miniPaint 已内置多语言翻译系统**：
- 使用 jQuery 的 `translate` 插件，通过 `class="trn"` 标记需翻译元素
- 支持多种语言，**已包含简体中文 ("zh")** 和 英语 ("en")
- 配置项 `config.LANG` 默认值为 `"en"`
- 翻译工具 `Tools_translate.translate(lang)` 可切换语言
- 语言可存于 Cookie (`language`) 或 URL 参数 (`?lang=zh`)

### 1.3 当前语言切换逻辑问题
`load_translations()` 中的逻辑**未正确使用**浏览器语言、URL 参数和 Cookie：
```javascript
// 当前逻辑（简化）
this.Helper.getCookie("language");  // 结果未使用
var e = this.Helper.get_url_parameters();
null != e.lang && e.lang.replace(...);  // 替换结果未赋值
this.Tools_translate.translate(i.Z.LANG);  // 固定使用默认 "en"
```

### 1.4 需翻译内容分布
| 位置 | 说明 |
|------|------|
| `index.html` | 静态文案：Preview, Colors, Information, Layers, Undo 等 |
| `dist/bundle.js` | 动态生成的 UI：菜单、工具提示、对话框、错误信息等 |
| 所有带 `class="trn"` 的元素 | 由 translate 插件自动处理 |

---

## 二、实现目标

1. **支持语言**: 中文(zh)、英语(en)
2. **自动切换**: 根据浏览器语言 (`navigator.language`) 自动选择
3. **优先级**: URL 参数 > Cookie > 浏览器语言 > 默认英语

---

## 三、详细工作计划

### 阶段一：修复语言检测逻辑（核心修改）

**任务 1.1** 修改 `dist/bundle.js` 中的 `load_translations` 逻辑

**当前代码片段**（需定位并替换）：
```
this.Helper.getCookie("language");var e=this.Helper.get_url_parameters();null!=e.lang&&e.lang.replace(/([^a-z]+)/gi,""),this.Tools_translate.translate(i.Z.LANG)
```

**替换为**：
```javascript
var _u=this.Helper.get_url_parameters(),_k=this.Helper.getCookie("language"),_n=(navigator.language||navigator.userLanguage||"").toLowerCase().split("-")[0];var _l=(_u&&_u.lang?_u.lang.replace(/([^a-z-]+)/gi,"").split("-")[0]:null)||_k||("zh"===_n?"zh":"en")||"en";_l="zh"===_l||"en"===_l?_l:"en";this.Helper.setCookie("language",_l);i.Z.LANG=_l;this.Tools_translate.translate(_l)
```

**逻辑说明**：
1. `_p` = URL 参数
2. `_c` = Cookie 中的语言
3. `_b` = 浏览器语言主代码 (如 zh-CN → zh)
4. `_l` = 最终语言，优先级：URL > Cookie > (浏览器为中文则 zh 否则 en) > en
5. 仅保留 zh 和 en 两种
6. 写入 Cookie 并调用 translate

**任务 1.2** 验证 bundle 中中文翻译完整性
- 检查 `trans_lang_codes` 是否包含 `"zh"`
- 抽样检查关键 UI 文案的中文翻译

---

### 阶段二：HTML 元数据与静态文案

**任务 2.1** 更新 `index.html`
- `<html lang="en">` → 改为由 JS 根据当前语言动态设置，或保留 `lang` 待 translate 执行后同步
- 考虑添加 `<html lang="zh">` 的初始化脚本（在 bundle 加载前根据 navigator 预判）

**任务 2.2** 可选：预加载语言检测脚本
在 `<head>` 中、`bundle.js` 之前添加：
```html
<script>
(function(){
  var b=(navigator.language||navigator.userLanguage||"").toLowerCase();
  if(b.indexOf("zh")===0&&!document.cookie.match(/language=/)){
    document.cookie="language=zh;path=/;max-age=31536000";
  }
})();
</script>
```
用于在首次访问时预先设置 Cookie，确保 bundle 加载后能读到正确语言（若采用 Cookie 优先策略可省略，因 bundle 内已修复逻辑）。

---

### 阶段三：完善与校验

**任务 3.1** 测试场景
- [ ] 浏览器语言为 zh-CN/zh-TW/zh → 界面显示中文
- [ ] 浏览器语言为 en/en-US 等 → 界面显示英文
- [ ] URL 带 `?lang=zh` → 强制中文
- [ ] URL 带 `?lang=en` → 强制英文
- [ ] 切换语言后 Cookie 正确，刷新后保持

**任务 3.2** 缺失翻译补充（如有）
- 对照英文界面逐项检查
- 对 bundle 中缺失的中文 key 进行补充（若可获取 miniPaint 源码则在其翻译文件中添加）

**任务 3.3** RTL 与无障碍
- 中文、英文均为 LTR，无需 RTL 特殊处理
- 确保 `aria-label` 等属性在切换语言时同步更新（如有）

---

## 四、技术风险与备选方案

| 风险 | 说明 | 备选方案 |
|------|------|----------|
| 修改 minified 代码 | bundle.js 被压缩，字符串替换需精确匹配 | 使用唯一、足够长的字符串进行 replace，替换后验证应用可正常启动 |
| 无源码构建 | 无法从源码重新打包 | 仅做最小化 patch，不改变 bundle 整体结构 |
| 翻译不完整 | 某些菜单或提示可能缺中文 | 在 miniPaint 上游或本仓库维护补充翻译映射表 |

---

## 五、文件修改清单

| 文件 | 修改类型 |
|------|----------|
| `dist/bundle.js` | 1 处字符串替换（load_translations 逻辑） |
| `index.html` | 可选：添加预检测脚本；调整 `lang` 属性逻辑 |
| `docs/I18N_WORK_PLAN.md` | 本工作计划（新建） |

---

## 六、实施顺序建议

1. **备份** `dist/bundle.js`
2. 执行 **阶段一**：修改 `load_translations` 逻辑
3. 本地/预发环境 **功能测试**
4. 执行 **阶段二**：按需调整 HTML
5. 执行 **阶段三**：完整测试与翻译补充
6. 提交代码并部署

---

## 七、后续扩展（可选）

- 在 UI 中增加语言切换入口（如设置菜单中的中/英切换）
- 支持更多语言（如日语、韩语，bundle 中已有部分）
- 若获取到 miniPaint 源码，将 i18n 改动合并回源码并建立正式构建流程

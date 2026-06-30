# Databasus 简体中文支持技术设计方案

**作者**: yanshaodong  
**日期**: 2026-06-30  
**版本**: v1.0

## 变更日志

| 日期 | 变更内容 | 作者 |
|------|---------|------|
| 2026-06-30 | 初始版本 - 分析项目现状并输出技术设计方案 | yanshaodong |

---

## 1. 项目现状分析

### 1.1 前端技术栈
- **框架**: React 19 + TypeScript
- **构建工具**: Vite
- **UI 组件库**: Ant Design 5.29.3
- **样式**: Tailwind CSS
- **国际化库**: ❌ **无**

### 1.2 国际化支持现状

#### 1.2.1 前端代码分析
- ❌ **无 i18n 库**: `package.json` 中未包含 `react-i18next`、`react-intl` 等国际化库
- ❌ **文本硬编码**: 所有 UI 文本直接写在 TSX 组件中（如 `Sign In`、`Sign Up`、`Databases` 等）
- ⚠️ **部分本地化**: 仅日期/时间格式基于浏览器 locale 自动适配（`getUserTimeFormat.ts`）
- ❌ **无语言包**: 没有独立的语言 JSON 文件或语言包模块

#### 1.2.2 后端代码分析
- ❌ **API 响应**: 错误消息和 API 响应均为英文硬编码
- ❌ **日志输出**: 日志消息均为英文（符合项目规范）
- ⚠️ **CLAUDE.md 规定**: "English only in code, comments, identifiers, log messages, API strings"

#### 1.2.3 Dockerfile 分析
- **多阶段构建**: 前端构建 → 后端构建 → 运行时镜像
- **静态文件嵌入**: 前端构建产物（`/frontend/dist`）被复制到 Go 二进制文件中（`/app/ui/build`）
- **运行时生成配置**: `start.sh` 动态生成 `runtime-config.js`，但不包含语言配置

### 1.3 关键发现

**核心问题**: 项目目前**不支持任何形式的国际化**，所有用户可见的文本都是英文硬编码。

---

## 2. 可行性分析：仅通过二次包装 Dockerfile 实现中文支持

### 2.1 结论

**❌ 不可行** - 仅通过二次包装 Dockerfile **无法**实现简体中文支持。

### 2.2 原因详解

#### 2.2.1 Dockerfile 的能力边界
Dockerfile 只能实现以下功能：
- ✅ 安装系统包（如中文字体、locale 配置）
- ✅ 设置环境变量
- ✅ 复制文件到容器
- ✅ 修改配置文件（如果应用支持）
- ✅ 运行脚本

**无法做到**：
- ❌ 修改已编译的前端 JavaScript 代码中的硬编码文本
- ❌ 动态替换前端 UI 字符串
- ❌ 修改嵌入在 Go 二进制文件中的前端资源

#### 2.2.2 前端资源的嵌入机制
根据 `Dockerfile` 第 53 行和第 178 行：
```dockerfile
# 构建阶段：复制前端构建产物
COPY --from=frontend-build /frontend/dist /app/ui/build

# 运行阶段：复制前端资源
COPY --from=backend-build /app/ui/build ./ui/build
```

前端资源在构建时被嵌入到 Go 二进制文件中，运行时无法直接修改。

#### 2.2.3 尝试的方案及限制

| 方案 | 可行性 | 限制 |
|------|--------|------|
| **方案1**: 通过环境变量控制语言 | ❌ 不可行 | 前端代码未读取语言环境变量 |
| **方案2**: 替换构建后的 JS 文件 | ⚠️ 理论可行但极不推荐 | JS 文件已压缩混淆，手动替换易出错且难以维护 |
| **方案3**: 使用 Nginx 反向代理 + 内容替换 | ⚠️ 部分可行 | 只能替换静态文本，无法处理动态生成的 UI（如 React 组件） |
| **方案4**: 在运行时挂载新的前端文件 | ❌ 不可行 | 前端资源已嵌入二进制文件，挂载不会被读取 |

---

## 3. 推荐技术方案

### 3.1 方案概述

要实现简体中文支持，需要**修改源码**并**重新构建镜像**。推荐采用以下方案：

### 3.2 方案A：完整国际化支持（推荐）

#### 3.2.1 技术栈
- **国际化库**: `react-i18next` (最受欢迎的 React i18n 库)
- **语言包格式**: JSON
- **支持语言**: 
  - `en-US` (默认)
  - `zh-CN` (简体中文)

#### 3.2.2 实施步骤

**步骤1**: 安装 i18n 依赖
```bash
cd frontend
pnpm add react-i18next i18next i18next-browser-languagedetector
```

**步骤2**: 创建语言包文件
```
frontend/src/locales/
  ├── en-US/
  │   ├── common.json
  │   ├── auth.json
  │   ├── databases.json
  │   └── ...
  └── zh-CN/
      ├── common.json
      ├── auth.json
      ├── databases.json
      └── ...
```

**步骤3**: 配置 i18n
```typescript
// frontend/src/i18n.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';

import enCommon from './locales/en-US/common.json';
import zhCommon from './locales/zh-CN/common.json';
// ... 导入其他语言包

i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    resources: {
      'en-US': { common: enCommon, /* ... */ },
      'zh-CN': { common: zhCommon, /* ... */ },
    },
    fallbackLng: 'en-US',
    interpolation: { escapeValue: false },
  });

export default i18n;
```

**步骤4**: 修改组件使用 `useTranslation` hook
```tsx
// 修改前
<Button>Sign In</Button>

// 修改后
import { useTranslation } from 'react-i18next';

const { t } = useTranslation();
<Button>{t('auth.signIn')}</Button>
```

**步骤5**: 添加语言切换器
```tsx
// 在用户界面添加语言选择器
<Select defaultValue="en-US" onChange={(lang) => i18n.changeLanguage(lang)}>
  <Option value="en-US">English</Option>
  <Option value="zh-CN">简体中文</Option>
</Select>
```

**步骤6**: 修改 Dockerfile 支持多语言
无需修改 Dockerfile，因为语言包会被打包到前端构建产物中。

#### 3.2.3 优点
- ✅ 完整的国际化支持
- ✅ 易于扩展其他语言
- ✅ 符合 React 最佳实践
- ✅ 支持动态切换语言

#### 3.2.4 缺点
- ❌ 需要修改大量源码（所有包含文本的组件）
- ❌ 工作量较大（预计需要修改 100+ 组件）
- ❌ 需要重新构建镜像

---

### 3.3 方案B：后端 API 国际化（辅助方案）

如果前端暂不支持国际化，可以先实现后端错误消息的中文化。

#### 3.3.1 实施步骤

**步骤1**: 添加 i18n 中间件
```go
// backend/internal/middleware/i18n.go
func I18nMiddleware() gin.HandlerFunc {
  return func(c *gin.Context) {
    lang := c.GetHeader("Accept-Language")
    if lang == "" {
      lang = "en-US"
    }
    c.Set("lang", lang)
    c.Next()
  }
}
```

**步骤2**: 创建语言包
```
backend/locales/
  ├── en-US.json
  └── zh-CN.json
```

**步骤3**: 修改错误消息
```go
// 修改前
c.JSON(400, gin.H{"error": "Database not found"})

// 修改后
c.JSON(400, gin.H{"error": i18n.T(c.GetString("lang"), "errors.databaseNotFound")})
```

#### 3.3.2 优点
- ✅ 后端错误消息可以中文化
- ✅ 工作量相对较小

#### 3.3.3 缺点
- ❌ 前端界面仍然是英文
- ❌ 用户体验不完整

---

## 4. 替代方案：不修改源码的临时解决方式

如果**必须**在不修改源码的情况下实现中文支持，可以考虑以下**临时方案**：

### 4.1 方案C：浏览器自动翻译（最简单）

**实施方法**:
1. 用户使用 Chrome 浏览器访问 Databasus
2. 右键点击页面 → 选择"翻译成中文"
3. Chrome 会自动翻译页面内容

**优点**:
- ✅ 零开发成本
- ✅ 无需修改任何代码

**缺点**:
- ❌ 翻译质量不稳定
- ❌ 每次刷新页面需要重新翻译
- ❌ 动态加载的内容可能无法翻译

### 4.2 方案D：自定义 CSS + JavaScript 注入（不推荐）

**实施方法**:
通过 Dockerfile 注入自定义 JavaScript，在浏览器端动态替换文本。

**步骤**:
1. 修改 `start.sh`，在 `index.html` 中注入自定义 JS
2. 编写 JS 脚本，遍历 DOM 并替换英文文本为中文

**示例**:
```bash
# 在 start.sh 中添加
cat > /app/ui/build/custom-translate.js <<'EOF'
document.addEventListener('DOMContentLoaded', () => {
  const translations = {
    'Sign In': '登录',
    'Sign Up': '注册',
    // ... 数千个翻译条目
  };
  
  document.querySelectorAll('*').forEach(el => {
    if (el.childNodes.length === 1 && el.childNodes[0].nodeType === 3) {
      const text = el.textContent.trim();
      if (translations[text]) {
        el.textContent = translations[text];
      }
    }
  });
});
EOF

# 修改 index.html 引入该脚本
sed -i 's/<\/body>/<script src="\/custom-translate.js"><\/script><\/body>/' /app/ui/build/index.html
```

**优点**:
- ✅ 无需修改源码
- ✅ 只需重新构建 Docker 镜像

**缺点**:
- ❌ **极难维护**：需要手动维护翻译字典（数千个条目）
- ❌ **性能问题**：遍历整个 DOM 树性能差
- ❌ **动态内容无法处理**：React 动态渲染的内容无法翻译
- ❌ **容易出错**：文本匹配可能误判
- ❌ **不符合生产标准**：属于 Hack 方案

**结论**: ⚠️ **不推荐**，仅作为最后手段。

---

## 5. 最终建议

### 5.1 推荐路径

| 优先级 | 方案 | 说明 |
|--------|------|------|
| ⭐⭐⭐ | **方案A**：完整国际化支持 | 虽然工作量较大，但是唯一符合生产标准的方案 |
| ⭐⭐ | **方案B + 方案A**：分阶段实施 | 先完成后端 API 国际化，再完成前端国际化 |
| ⭐ | **方案C**：浏览器自动翻译 | 作为临时解决方案，零成本 |

### 5.2 实施计划（方案A）

#### 阶段1：基础设施搭建（1-2天）
- [ ] 安装 `react-i18next` 依赖
- [ ] 创建 i18n 配置文件
- [ ] 创建英文和中文语言包框架
- [ ] 添加语言切换组件

#### 阶段2：核心组件国际化（3-5天）
- [ ] 认证相关组件（`SignIn`、`SignUp`、`ResetPassword` 等）
- [ ] 数据库管理组件
- [ ] 备份配置组件
- [ ] 导航栏和侧边栏

#### 阶段3：完整国际化（5-10天）
- [ ] 所有剩余组件
- [ ] 错误消息和提示
- [ ] 文档和注释（可选）

#### 阶段4：测试和优化（2-3天）
- [ ] 语言切换测试
- [ ] 中文显示效果优化
- [ ] 浏览器兼容性测试

**总计工作量**: 约 11-20 个工作日

---

## 6. 技术细节

### 6.1 Ant Design 国际化

Ant Design 组件库本身支持国际化，需要同步配置：

```tsx
// App.tsx
import { ConfigProvider } from 'antd';
import zhCN from 'antd/locale/zh_CN';
import enUS from 'antd/locale/en_US';

function App() {
  const { i18n } = useTranslation();
  
  const antdLocale = i18n.language === 'zh-CN' ? zhCN : enUS;
  
  return (
    <ConfigProvider locale={antdLocale}>
      {/* 应用内容 */}
    </ConfigProvider>
  );
}
```

### 6.2 日期格式化国际化

项目已有 `getUserTimeFormat.ts`，需要扩展到支持中文日期格式：

```typescript
// 修改后
export const getUserTimeFormat = () => {
  const locale = i18n.language; // 使用 i18n 语言
  const { dateFormat } = getLocaleDateFormat(locale);
  const is12Hour = getIs12HourFormat(locale);
  
  return {
    use12Hours: is12Hour,
    format: is12Hour ? `${dateFormat} h:mm A` : `${dateFormat} HH:mm`,
  };
};
```

### 6.3 Dockerfile 优化

虽然 Dockerfile 不需要大改，但可以考虑以下优化：

```dockerfile
# 添加语言环境变量
ENV APP_LANGUAGE=en-US

# 如果需要支持中文排序等 locale 功能，可以安装 locales
RUN apt-get update && apt-get install -y --no-install-recommends \
  locales \
  && sed -i '/zh_CN.UTF-8/s/^# //g' /etc/locale.gen \
  && locale-gen \
  && rm -rf /var/lib/apt/lists/*
  
ENV LANG=zh_CN.UTF-8
ENV LANGUAGE=zh_CN:zh
ENV LC_ALL=zh_CN.UTF-8
```

---

## 7. 结论

1. **仅通过二次包装 Dockerfile 无法实现简体中文支持**，因为项目目前没有国际化基础设施。

2. **推荐采用方案A**（完整国际化支持），虽然工作量较大，但是唯一符合生产标准的方案。

3. **如果时间紧迫**，可以先采用方案C（浏览器自动翻译）作为临时解决方案。

4. **不推荐方案D**（JavaScript 注入），因为维护成本极高且容易出错。

---

## 8. 附录

### 8.1 相关文件清单

**需要修改的前端文件**（部分示例）：
- `frontend/src/App.tsx`
- `frontend/src/pages/AuthPageComponent.tsx`
- `frontend/src/features/*/ui/*.tsx` (约 100+ 组件)

**需要新增的文件**：
- `frontend/src/i18n.ts`
- `frontend/src/locales/en-US/*.json`
- `frontend/src/locales/zh-CN/*.json`
- `frontend/src/shared/ui/LanguageSwitcherComponent.tsx`

### 8.2 参考资料

- [react-i18next 官方文档](https://react.i18next.com/)
- [Ant Design 国际化文档](https://ant.design/docs/react/i18n)
- [i18next 官方文档](https://www.i18next.com/)

---

**文档结束**

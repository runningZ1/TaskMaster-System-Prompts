# Task Master Prompt管理系统

此目录包含 Task Master 中所有 AI 驱动功能的集中式Prompt模板。

## 概述

Prompt管理系统提供：
- **集中存储**：所有Prompt都位于一个位置 (`/src/prompts`)
- **JSON 模式验证**：使用 AJV 进行全面验证，并提供详细的错误报告
- **版本控制**：随时间跟踪Prompt的更改
- **变体支持**：针对不同上下文（研究模式、复杂性级别等）的不同Prompt
- **模板变量**：通过变量替换进行动态Prompt生成
- **IDE 集成**：VS Code IntelliSense 和验证支持

## 目录结构

```
src/prompts/
├── README.md                # 此文件
├── schemas/                 # 用于验证的 JSON 模式
│   ├── README.md           # 模式文档
│   ├── prompt-template.schema.json  # 主模板模式
│   ├── parameter.schema.json        # 参数验证模式
│   └── variant.schema.json          # Prompt变体模式
├── parse-prd.json          # PRD 解析Prompt
├── expand-task.json        # 任务扩展Prompt
├── add-task.json           # 任务创建Prompt
├── update-tasks.json       # 批量任务更新Prompt
├── update-task.json        # 单任务更新Prompt
├── update-subtask.json     # 子任务更新Prompt
├── analyze-complexity.json # 复杂性分析Prompt
└── research.json           # 研究查询Prompt
```

## Schema 验证

所有Prompt模板都将根据位于 `/src/prompts/schemas/` 中的 JSON 模式进行验证。验证系统：

- **结构验证**：确保必需字段和正确的嵌套
- **参数类型检查**：验证参数类型、模式和范围
- **模板语法**：验证 Handlebars 语法和变量引用
- **语义版本控制**：强制执行正确的版本格式
- **交叉引用验证**：确保参数与模板变量匹配

### 验证功能
- **必需字段**：`id`、`version`、`description`、`prompts.default`
- **类型安全**：字符串、数字、布尔值、数组、对象验证
- **模式匹配**：字符串参数的正则表达式验证
- **范围验证**：数字参数的最小值/最大值
- **枚举约束**：分类参数的受限值集

## 开发工作流程

### 设置开发环境
1. **VS Code 集成**：模式会自动配置 IntelliSense
2. **依赖项**：验证需要 `ajv` 和 `ajv-formats`
3. **文件监听**：对模板的更改会触发自动验证

### 创建新Prompt
1. 在 `/src/prompts/` 中创建新的 `.json` 文件
2. 遵循模式结构（参见模板结构部分）
3. 使用正确的类型和验证定义参数
4. 创建包含模板变量的系统和用户Prompt
5. 在提交之前使用 PromptManager 进行测试

### 修改现有Prompt
1. 遵循语义版本控制更新 `version` 字段
2. 尽可能保持向后兼容
3. 使用使用该Prompt的现有代码进行测试
4. 如果参数发生更改，请更新文档

## Prompt模板参考

### 1. parse-prd.json
**目的**：将产品需求文档解析为结构化任务
**变体**：`default`，`research`（启用研究模式时）

**必需参数**：
- `numTasks` (number)：要生成的目标任务数量
- `nextId` (number)：任务的起始 ID
- `prdContent` (string)：PRD 文件的内容
- `prdPath` (string)：PRD 文件的路径
- `defaultTaskPriority` (string)：生成任务的默认优先级

**可选参数**：
- `research` (boolean)：启用研究模式以获取最新最佳实践（默认值：false）

**用法**：由 `task-master parse-prd` 命令用于将 PRD 文档转换为可操作的任务列表。

### 2. add-task.json
**目的**：根据用户描述生成新任务
**变体**：`default`，`research`（启用研究模式时）

**必需参数**：
- `prompt` (string)：用户的任务描述
- `newTaskId` (number)：新任务的 ID

**可选参数**：
- `existingTasks` (array)：现有任务列表作为上下文
- `gatheredContext` (string)：从代码库分析中收集的上下文
- `contextFromArgs` (string)：来自手动参数的附加上下文
- `priority` (string)：任务优先级（high/medium/low，默认值：medium）
- `dependencies` (array)：任务依赖项 ID
- `useResearch` (boolean)：使用研究模式（默认值：false）

**用法**：由 `task-master add-task` 命令用于在 AI 协助下创建新任务。

### 3. expand-task.json
**目的**：使用三种复杂的策略将任务分解为详细的子任务
**变体**：`complexity-report`（存在 `expansionPrompt` 时）、`research`（启用研究模式时）、`default`（标准情况）

**必需参数**：
- `subtaskCount` (number)：要生成的子任务数量
- `task` (object)：要扩展的任务
- `nextSubtaskId` (number)：新子任务的起始 ID

**可选参数**：
- `additionalContext` (string)：用于扩展的附加上下文（默认值：""）
- `complexityReasoningContext` (string)：复杂性分析推理上下文（默认值：""）
- `gatheredContext` (string)：收集的项目上下文（默认值：""）
- `useResearch` (boolean)：使用研究模式（默认值：false）
- `expansionPrompt` (string)：来自复杂性报告的扩展Prompt

**变体选择策略**：
1. **complexity-report**：当 `expansionPrompt` 存在时使用（最高优先级）
2. **research**：当 `useResearch === true && !expansionPrompt` 时使用
3. **default**：标准回退策略

**用法**：由 `task-master expand` 命令用于根据可用上下文和复杂性分析，使用最合适的策略将复杂任务分解为可管理的子任务。

### 4. update-task.json
**目的**：使用新信息更新单个任务，支持完全更新和追加模式
**变体**：`default`，`append`（当 `appendMode` 为 true 时），`research`（启用研究模式时）

**必需参数**：
- `task` (object)：要更新的任务
- `taskJson` (string)：任务的 JSON 字符串表示
- `updatePrompt` (string)：要应用的更改描述

**可选参数**：
- `appendMode` (boolean)：是追加到详细信息还是进行完全更新（默认值：false）
- `useResearch` (boolean)：使用研究模式（默认值：false）
- `currentDetails` (string)：当前任务详细信息作为上下文（默认值："(No existing details)"）
- `gatheredContext` (string)：附加项目上下文

**用法**：由 `task-master update-task` 命令用于修改现有任务。

### 5. update-tasks.json
**目的**：根据新上下文或更改更新多个任务
**变体**：`default`，`research`（启用研究模式时）

**必需参数**：
- `tasks` (array)：要更新的任务数组
- `updatePrompt` (string)：要应用的更改描述

**可选参数**：
- `useResearch` (boolean)：使用研究模式（默认值：false）
- `projectContext` (string)：附加项目上下文

**用法**：由 `task-master update` 命令用于批量更新多个任务。

### 6. update-subtask.json
**目的**：通过仅生成新内容来向子任务追加信息
**变体**：`default`，`research`（启用研究模式时）

**必需参数**：
- `parentTask` (object)：父任务上下文
- `currentDetails` (string)：当前子任务详细信息（默认值："(No existing details)"）
- `updatePrompt` (string)：用户请求要添加的内容

**可选参数**：
- `prevSubtask` (object)：上一个子任务（如果有）
- `nextSubtask` (object)：下一个子任务（如果有）
- `useResearch` (boolean)：使用研究模式（默认值：false）
- `gatheredContext` (string)：附加项目上下文

**用法**：由 `task-master update-subtask` 命令用于记录子任务的进度和发现。

### 7. analyze-complexity.json
**目的**：分析任务复杂性并生成扩展建议
**变体**：`default`，`research`（启用研究模式时），`batch`（分析 >10 个任务时）

**必需参数**：
- `tasks` (array)：要分析的任务数组

**可选参数**：
- `gatheredContext` (string)：附加项目上下文
- `threshold` (number)：扩展建议的复杂性阈值（1-10，默认值：5）
- `useResearch` (boolean)：使用研究模式进行更深入的分析（默认值：false）

**用法**：由 `task-master analyze-complexity` 命令用于确定需要分解的任务。

### 8. research.json
**目的**：使用项目上下文执行 AI 驱动的研究
**变体**：`default`，`low`（简洁响应），`medium`（平衡），`high`（详细）

**必需参数**：
- `query` (string)：研究查询

**可选参数**：
- `gatheredContext` (string)：收集的项目上下文
- `detailLevel` (string)：详细级别（low/medium/high，默认值：medium）
- `projectInfo` (object)：包含以下属性的项目信息：
  - `root` (string)：项目根路径
  - `taskCount` (number)：相关任务数量
  - `fileCount` (number)：相关文件数量

**用法**：由 `task-master research` 命令用于获取上下文信息和指导。

## 模板结构

每个Prompt模板都是一个 JSON 文件，具有以下结构：

```json
{
  "id": "unique-identifier",
  "version": "1.0.0",
  "description": "What this prompt does",
  "metadata": {
    "author": "system",
    "created": "2024-01-01T00:00:00Z",
    "updated": "2024-01-01T00:00:00Z",
    "tags": ["category", "feature"],
    "category": "task"
  },
  "parameters": {
    "paramName": {
      "type": "string|number|boolean|array|object",
      "required": true|false,
      "default": "default value",
      "description": "Parameter description",
      "enum": ["option1", "option2"],
      "pattern": "^[a-z]+$",
      "minimum": 1,
      "maximum": 100
    }
  },
  "prompts": {
    "default": {
      "system": "System prompt template",
      "user": "User prompt template"
    },
    "variant-name": {
      "condition": "JavaScript expression",
      "system": "Variant system prompt",
      "user": "Variant user prompt",
      "metadata": {
        "description": "When to use this variant"
      }
    }
  }
}
```

## 模板功能

### 变量替换
使用 `{{variableName}}` 注入动态值：
```
"user": "Analyze these {{tasks.length}} tasks with threshold {{threshold}}"
```

### 条件语句
使用 `{{#if variable}}...{{/if}}` 进行条件内容：
```
"user": "{{#if useResearch}}Research and {{/if}}create a task"
```

### 辅助函数

#### 相等辅助函数
使用 `{{#if (eq variable "value")}}...{{/if}}` 进行字符串比较：
```
"user": "{{#if (eq detailLevel \"low\")}}Provide a brief summary{{/if}}"
"user": "{{#if (eq priority \"high\")}}URGENT: {{/if}}{{taskTitle}}"
```

`eq` 辅助函数支持基于参数值的简洁条件逻辑：
- 比较字符串：`(eq detailLevel "medium")`
- 与枚举值比较：`(eq status "pending")`
- 多个条件：`{{#if (eq level "1")}}First{{/if}}{{#if (eq level "2")}}Second{{/if}}`

#### 否定辅助函数
使用 `{{#if (not variable)}}...{{/if}}` 进行否定条件：
```
"user": "{{#if (not useResearch)}}Use basic analysis{{/if}}"
"user": "{{#if (not hasSubtasks)}}This task has no subtasks{{/if}}"
```

`not` 辅助函数支持简洁的否定条件逻辑：
- 否定布尔值：`(not useResearch)`
- 否定真值/假值：`(not emptyArray)`
- 比单独的布尔参数更简洁：无需 `notUseResearch` 标志

#### 数字比较辅助函数
使用 `{{#if (gt variable number)}}...{{/if}}` 进行大于比较：
```
"user": "generate {{#if (gt numTasks 0)}}approximately {{numTasks}}{{else}}an appropriate number of{{/if}} top-level development tasks"
"user": "{{#if (gt complexity 5)}}This is a complex task{{/if}}"
"system": "create {{#if (gt subtaskCount 0)}}exactly {{subtaskCount}}{{else}}an appropriate number of{{/if}} subtasks"
```

使用 `{{#if (gte variable number)}}...{{/if}}` 进行大于或等于比较：
```
"user": "{{#if (gte priority 8)}}HIGH PRIORITY{{/if}}"
"user": "{{#if (gte threshold 1)}}Analysis enabled{{/if}}"
"system": "{{#if (gte complexityScore 8)}}Use detailed breakdown approach{{/if}}"
```

数字比较辅助函数支持复杂的条件逻辑：
- **动态计数**：`{{#if (gt numTasks 0)}}exactly {{numTasks}}{{else}}an appropriate number of{{/if}}`
- **基于阈值的行为**：`(gte complexityScore 8)` 用于高复杂性处理
- **零检查**：`(gt subtaskCount 0)` 用于条件内容生成
- **小数支持**：`(gt score 7.5)` 用于分数比较
- **增强的Prompt复杂性**：实现与 GitHub 规范匹配的 `parse-prd` 和 `expand-task` 逻辑

### 循环
使用 `{{#each array}}...{{/each}}` 遍历数组：
```
"user": "Tasks:\n{{#each tasks}}- {{id}}: {{title}}\n{{/each}}"
```

### 特殊循环变量
在 `{{#each}}` 块中，您可以访问：
- `{{@index}}`：当前数组索引（0-based）
- `{{@first}}`：布尔值，第一项为 true
- `{{@last}}`：布尔值，最后一项为 true

```
"user": "{{#each tasks}}{{@index}}. {{title}}{{#unless @last}}\n{{/unless}}{{/each}}"
```

### JSON 序列化
使用 `{{{json variable}}}`（三花括号）将对象/数组序列化为 JSON：
```
"user": "Analyze these tasks: {{{json tasks}}}"
```

### 嵌套属性
使用点表示法访问嵌套属性：
```
"user": "Project: {{context.projectName}}"
```

## Prompt变体

变体允许根据条件使用不同的Prompt：

```json
{
  "prompts": {
    "default": {
      "system": "Default system prompt",
      "user": "Default user prompt"
    },
    "research": {
      "condition": "useResearch === true",
      "system": "Research-focused system prompt",
      "user": "Research-focused user prompt"
    },
    "high-complexity": {
      "condition": "complexityScore >= 8",
      "system": "Complex task handling prompt",
      "user": "Detailed breakdown request"
    }
  }
}
```

### 条件评估
条件是使用参数值作为上下文评估的 JavaScript 表达式：
- 简单比较：`useResearch === true`
- 数字比较：`threshold >= 5`
- 字符串匹配：`priority === 'high'`
- 复杂逻辑：`useResearch && threshold > 7`

## PromptManager 模块

PromptManager 在 `scripts/modules/prompt-manager.js` 中实现，并提供：
- **模板加载和缓存**：模板一次加载并缓存以提高性能
- **Schema 验证**：使用 AJV 进行全面验证，并提供详细的错误报告
- **变量替换**：Handlebars 样式的语法用于动态内容
- **变体选择**：根据条件自动选择
- **错误处理**：优雅的回退和详细的错误消息
- **单例模式**：每个项目根目录一个实例以提高效率

### 验证行为
- **Schema 可用**：完整的验证和详细的错误消息
- **Schema 缺失**：回退到基本的结构验证
- **无效模板**：抛出带有字段级详细信息的描述性错误
- **参数验证**：类型检查、模式匹配、范围验证

## 代码用法

### 基本用法
```javascript
import { getPromptManager } from '../prompt-manager.js';

const promptManager = getPromptManager();
const { systemPrompt, userPrompt, metadata } = promptManager.loadPrompt('add-task', {
  // 参数与模板的参数定义匹配
  prompt: 'Create a user authentication system',
  newTaskId: 5,
  priority: 'high',
  useResearch: false
});

// 与 AI 服务一起使用
const result = await generateObjectService({
  systemPrompt,
  prompt: userPrompt,
  // ... 其他 AI 参数
});
```

### 带变体
```javascript
// 研究变体将自动选择
const { systemPrompt, userPrompt } = promptManager.loadPrompt('expand-task', {
  useResearch: true,  // 触发研究变体
  task: taskObject,
  subtaskCount: 5
});
```

### 错误处理
```javascript
try {
  const result = promptManager.loadPrompt('invalid-template', {});
} catch (error) {
  if (error.message.includes('Schema validation failed')) {
    console.error('Template validation error:', error.message);
  } else if (error.message.includes('not found')) {
    console.error('Template not found:', error.message);
  }
}
```

## 添加新Prompt

1. **创建 JSON 文件**，遵循模板结构
2. **定义参数**，包含正确的类型、验证和描述
3. **创建Prompt**，包含清晰的系统和用户模板
4. **使用模板变量**进行动态内容
5. **添加变体**（如果需要）以适应不同的上下文
6. **使用 PromptManager 进行彻底测试**
7. **更新此文档**以包含新Prompt的详细信息

### 新Prompt示例
```json
{
  "id": "new-feature",
  "version": "1.0.0",
  "description": "Generate code for a new feature",
  "parameters": {
    "featureName": {
      "type": "string",
      "required": true,
      "pattern": "^[a-zA-Z][a-zA-Z0-9-]*$",
      "description": "Name of the feature to implement"
    },
    "complexity": {
      "type": "string",
      "required": false,
      "enum": ["simple", "medium", "complex"],
      "default": "medium",
      "description": "Feature complexity level"
    }
  },
  "prompts": {
    "default": {
      "system": "You are a senior software engineer.",
      "user": "Create a {{complexity}} {{featureName}} feature."
    }
  }
}
```

## 最佳实践

### 模板设计
1. **清晰的 ID**：使用 kebab-case，描述性标识符
2. **语义版本控制**：遵循 semver 进行版本管理
3. **全面的参数**：定义所有必需和可选参数
4. **类型安全**：使用正确的参数类型和验证
5. **清晰的描述**：记录每个Prompt和参数的作用

### 变量使用
1. **有意义的名称**：使用描述性变量名
2. **一致的模式**：遵循既定的命名约定
3. **安全默认值**：提供合理的默认值
4. **验证**：使用模式、枚举和范围进行验证

### 变体策略
1. **简单条件**：保持变体条件易于理解
2. **明确目的**：每个变体都应有独特的用例
3. **回退逻辑**：始终提供默认变体
4. **文档**：解释每个变体何时使用

### 性能
1. **缓存**：模板会自动缓存
2. **延迟加载**：模板仅在需要时加载
3. **最小变体**：不要创建不必要的变体
4. **高效条件**：保持条件评估快速

## 测试Prompt

### 验证测试
```javascript
// 测试 schema 验证
const promptManager = getPromptManager();
const results = promptManager.validateAllPrompts();
console.log(`Valid: ${results.valid.length}, Errors: ${results.errors.length}`);
```

### 集成测试
修改Prompt时，请确保测试：
- 变量替换与实际数据结构一起工作
- 变体选择根据条件正确触发
- AI 响应与预期行为保持一致
- 所有参数均已正确验证
- 错误处理适用于无效输入

### 快速测试
```javascript
// 测试Prompt加载和变量替换
const promptManager = getPromptManager();
const result = promptManager.loadPrompt('research', {
  query: 'What are the latest React best practices?',
  detailLevel: 'medium',
  gatheredContext: 'React project with TypeScript'
});
console.log('System:', result.systemPrompt);
console.log('User:', result.userPrompt);
console.log('Metadata:', result.metadata);
```

### 测试清单
- [ ] 模板根据 schema 进行验证
- [ ] 定义所有必需参数
- [ ] 变量替换正常工作
- [ ] 变体在正确条件下触发
- [ ] 错误消息清晰有用
- [ ] 重复使用的性能可接受

## 故障排除

### 常见问题

**Schema 验证错误**：
- 检查是否存在必需字段
- 验证参数类型是否与 schema 匹配
- 确保版本遵循语义版本控制
- 验证 JSON 语法

**变量替换问题**：
- 检查变量名是否与参数名匹配
- 验证嵌套属性访问语法
- 确保数组迭代语法正确
- 使用实际数据结构进行测试

**变体选择问题**：
- 验证条件语法是否为有效的 JavaScript
- 检查参数值是否与条件预期匹配
- 确保存在默认变体
- 使用调试日志记录测试条件评估

**性能问题**：
- 检查模板中是否存在循环引用
- 验证缓存是否正常工作
- 监视模板加载频率
- 考虑简化复杂条件

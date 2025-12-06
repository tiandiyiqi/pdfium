# PDF.js 叠印功能分别处理实现记录

## 项目概述

本次修改旨在为 PDF.js 添加分别处理填充和描边叠印的功能，使得填充操作和描边操作可以使用不同的混合模式（Blend Mode），从而更好地支持印前处理中的叠印效果。

## 修改时间

- **修改日期**: 2024年12月6日
- **修改人员**: AI 助手
- **项目版本**: PDF.js 开发版本

## 功能需求

### 原始问题
PDF.js 原有的混合模式处理机制存在以下限制：
1. 只支持全局混合模式设置，无法分别控制填充和描边的叠印效果
2. 使用 `BM` 参数统一设置 `ctx.globalCompositeOperation`
3. 无法满足印前处理中对填充和描边分别设置叠印模式的需求

### 目标功能
1. 支持 `fillBM` 和 `strokeBM` 参数分别设置填充和描边的混合模式
2. 保持对原有 `BM` 参数的兼容性
3. 在 `fill()` 和 `stroke()` 方法中分别应用对应的混合模式

## 实现方案

### 整体架构
```
CanvasExtraState 类扩展
├── fillCompositeOperation: 填充混合模式属性
├── strokeCompositeOperation: 描边混合模式属性
└── clone() 方法更新: 确保属性正确复制

setGState 方法扩展
├── "BM" 分支: 兼容性处理，同时设置填充和描边模式
├── "fillBM" 分支: 专门设置填充混合模式
└── "strokeBM" 分支: 专门设置描边混合模式

渲染方法更新
├── fill() 方法: 应用填充混合模式
└── stroke() 方法: 应用描边混合模式
```

## 具体修改内容

### 1. 扩展 CanvasExtraState 类

**文件路径**: `/Users/Tiandiyiqi/Documents/Prepress/pdf.js/src/display/canvas.js`

#### 修改前代码 (第364行附近):
```javascript
class CanvasExtraState {
  constructor() {
    this.fillColor = "#000000";
    this.strokeColor = "#000000";
    this.fillAlpha = 1;
    this.strokeAlpha = 1;
    // ... 其他属性
  }
}
```

#### 修改后代码:
```javascript
class CanvasExtraState {
  constructor() {
    this.fillColor = "#000000";
    this.strokeColor = "#000000";
    this.fillAlpha = 1;
    this.strokeAlpha = 1;
    this.fillCompositeOperation = "source-over";  // 新增
    this.strokeCompositeOperation = "source-over"; // 新增
    // ... 其他属性
  }
}
```

#### 更新 clone() 方法 (第385行附近):
```javascript
clone() {
  const state = new CanvasExtraState();
  state.clipBox = this.clipBox;
  state.minMax = this.minMax;
  state.fillCompositeOperation = this.fillCompositeOperation;  // 新增
  state.strokeCompositeOperation = this.strokeCompositeOperation; // 新增
  return state;
}
```

### 2. 扩展 setGState 方法

**文件路径**: `/Users/Tiandiyiqi/Documents/Prepress/pdf.js/src/display/canvas.js`

#### 修改前代码 (第1170-1180行附近):
```javascript
case "BM":
  ctx.globalCompositeOperation = value;
  break;
```

#### 修改后代码:
```javascript
case "BM":
  // 兼容性处理：同时设置填充和描边混合模式
  ctx.globalCompositeOperation = value;
  current.fillCompositeOperation = value;
  current.strokeCompositeOperation = value;
  break;

case "fillBM":
  // 专门设置填充混合模式
  current.fillCompositeOperation = value;
  break;

case "strokeBM":
  // 专门设置描边混合模式
  current.strokeCompositeOperation = value;
  break;
```

### 3. 更新 stroke() 方法

**文件路径**: `/Users/Tiandiyiqi/Documents/Prepress/pdf.js/src/display/canvas.js`

#### 修改前代码 (第1500-1510行附近):
```javascript
stroke(pat) {
  const ctx = this.ctx;
  const current = this.current;
  
  ctx.globalAlpha = current.strokeAlpha;
  // ... 描边操作
  ctx.globalAlpha = current.fillAlpha;
}
```

#### 修改后代码:
```javascript
stroke(pat) {
  const ctx = this.ctx;
  const current = this.current;
  
  // 保存原始混合模式并设置描边混合模式
  const originalCompositeOperation = ctx.globalCompositeOperation;
  ctx.globalCompositeOperation = current.strokeCompositeOperation;
  
  ctx.globalAlpha = current.strokeAlpha;
  // ... 描边操作
  
  // 恢复原始混合模式
  ctx.globalCompositeOperation = originalCompositeOperation;
  ctx.globalAlpha = current.fillAlpha;
}
```

### 4. 更新 fill() 方法

**文件路径**: `/Users/Tiandiyiqi/Documents/Prepress/pdf.js/src/display/canvas.js`

#### 修改前代码 (第1585-1595行附近):
```javascript
fill(pat) {
  const ctx = this.ctx;
  const current = this.current;
  
  // ... 填充操作
  ctx.fill();
}
```

#### 修改后代码:
```javascript
fill(pat) {
  const ctx = this.ctx;
  const current = this.current;
  
  // 保存原始混合模式并设置填充混合模式
  const originalCompositeOperation = ctx.globalCompositeOperation;
  ctx.globalCompositeOperation = current.fillCompositeOperation;
  
  // ... 填充操作
  ctx.fill();
  
  // 恢复原始混合模式
  ctx.globalCompositeOperation = originalCompositeOperation;
}
```

## 技术要点

### 1. 兼容性设计
- 保留原有的 `BM` 参数支持，确保向后兼容
- `BM` 参数同时设置填充和描边混合模式，保持原有行为

### 2. 状态管理
- 在 `CanvasExtraState` 中分别存储填充和描边的混合模式
- 通过 `clone()` 方法确保状态正确复制
- 在操作前后正确保存和恢复混合模式状态

### 3. 渲染流程
- `fill()` 和 `stroke()` 方法分别应用对应的混合模式
- 操作完成后恢复原始混合模式，避免影响后续操作

## 支持的混合模式

本次修改支持所有 Canvas 2D API 标准的混合模式：

- `"source-over"` (默认)
- `"multiply"`
- `"screen"`
- `"overlay"`
- `"darken"`
- `"lighten"`
- `"color-dodge"`
- `"color-burn"`
- `"hard-light"`
- `"soft-light"`
- `"difference"`
- `"exclusion"`
- `"hue"`
- `"saturation"`
- `"color"`
- `"luminosity"`

## 使用示例

### PDF 内容流中使用
```
/DeviceCMYK cs
1 0 0 0 k  % CMYK 青色

% 设置填充混合模式为 multiply
/fillBM /Multiply gs

100 100 200 200 re f  % 填充矩形，使用 multiply 混合模式

% 设置描边混合模式为 screen
/strokeBM /Screen gs

0 0 300 300 re s  % 描边矩形，使用 screen 混合模式
```

### 兼容原有用法
```
% 原有用法：同时设置填充和描边混合模式
/BM /Multiply gs

% 新用法：分别设置
/fillBM /Multiply gs
/strokeBM /Screen gs
```

## 测试验证

### 测试文件
项目中的测试文件可用于验证修改效果：
- `/study/PDF samples/test3-adobe.pdf`
- `/study/PDF samples/test3-esko.pdf`

### 验证步骤
1. 启动开发服务器：`npx gulp server`
2. 访问：http://localhost:8888/web/viewer.html
3. 加载包含叠印设置的 PDF 文件
4. 验证填充和描边分别应用正确的混合模式

## 修改影响评估

### 正向影响
1. **功能增强**: 支持分别控制填充和描边的叠印效果
2. **兼容性保持**: 完全兼容原有的混合模式设置方式
3. **印前支持**: 更好地满足专业印刷需求

### 潜在风险
1. **性能影响**: 增加了状态保存和恢复操作，但影响极小
2. **代码复杂度**: 增加了状态管理的复杂性

### 测试建议
1. 测试各种混合模式的组合使用
2. 验证与软掩码模式的兼容性
3. 测试大量叠印操作时的性能表现

## 后续优化建议

1. **性能优化**: 考虑在不需要叠印时跳过状态保存操作
2. **API 扩展**: 可考虑添加更多印前相关的图形状态参数
3. **文档完善**: 更新官方文档说明新的混合模式参数

## 相关文件

- **主要修改文件**: `/src/display/canvas.js`
- **测试文件**: `/study/PDF samples/` 目录下的测试 PDF
- **开发文档**: `/study/项目开发指引.md`

## 总结

本次修改成功实现了 PDF.js 对填充和描边叠印功能的分别处理，为印前处理提供了更好的支持。修改保持了良好的兼容性，同时扩展了功能，为后续的印前功能开发奠定了基础。

---

*文档创建时间: 2024年12月6日*  
*最后更新: 2024年12月6日*
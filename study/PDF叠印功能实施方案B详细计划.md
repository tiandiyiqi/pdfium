# PDF 叠印功能实施方案 B 详细实施计划

## 项目信息

- **方案名称**：方案 B - 渲染层分别处理
- **预计工期**：2-3 周（10-15 个工作日）
- **实施人员**：开发团队
- **开始日期**：待定
- **目标**：为所有 PDF 对象类型实现完整的叠印支持

---

## 阶段一：基础架构搭建（3-4 天）

### Day 1：叠印检测辅助函数实现

#### 任务 1.1：创建颜色空间获取函数

**文件**：`core/fpdfapi/render/cpdf_renderstatus.h` 和 `.cpp`

**新增函数**：

```cpp
// 在 cpdf_renderstatus.h 的 private 部分添加
CPDF_ColorSpace::Family GetObjectColorSpaceFamily(
    CPDF_PageObject* pObj,
    bool for_fill) const;
```

**实现逻辑**：

```cpp
CPDF_ColorSpace::Family CPDF_RenderStatus::GetObjectColorSpaceFamily(
    CPDF_PageObject* pObj,
    bool for_fill) const {
  const CPDF_Color* pColor = for_fill
      ? pObj->color_state().GetFillColor()
      : pObj->color_state().GetStrokeColor();

  if (!pColor || pColor->IsNull() || pColor->IsPattern()) {
    return CPDF_ColorSpace::Family::kUnknown;
  }

  RetainPtr<CPDF_ColorSpace> pColorSpace = pColor->GetColorSpace();
  return pColorSpace ? pColorSpace->GetFamily()
                    : CPDF_ColorSpace::Family::kUnknown;
}
```

**测试要求**：

- 测试各种颜色空间（RGB、CMYK、Separation、DeviceN 等）
- 测试 Pattern 颜色（应返回 Unknown）
- 测试空颜色状态

**验收标准**：

- ✅ 函数正确返回颜色空间类型
- ✅ 处理边界情况（空颜色、Pattern 等）
- ✅ 单元测试通过

---

#### 任务 1.2：创建叠印检测函数

**文件**：`core/fpdfapi/render/cpdf_renderstatus.h` 和 `.cpp`

**新增函数**：

```cpp
// 在 cpdf_renderstatus.h 的 private 部分添加
bool ShouldApplyOverprintFill(CPDF_PageObject* pObj) const;
bool ShouldApplyOverprintStroke(CPDF_PageObject* pObj) const;
```

**实现逻辑**（参考图像对象的实现）：

```cpp
bool CPDF_RenderStatus::ShouldApplyOverprintFill(CPDF_PageObject* pObj) const {
  const CPDF_GeneralState& state = pObj->general_state();

  // 检查基本条件
  if (!state.HasRef()) {
    return false;
  }

  // 检查填充叠印是否启用
  if (!state.GetFillOP()) {
    return false;
  }

  // 检查叠印模式（0 = 标准叠印）
  if (state.GetOPMode() != 0) {
    return false;
  }

  // 检查混合模式（必须是 Normal）
  if (state.GetBlendType() != BlendMode::kNormal) {
    return false;
  }

  // 检查透明度（必须完全不透明）
  if (state.GetFillAlpha() != 1.0f) {
    return false;
  }

  // 检查颜色空间（必须是 CMYK、Separation 或 DeviceN）
  CPDF_ColorSpace::Family cs_family = GetObjectColorSpaceFamily(pObj, true);
  return (cs_family == CPDF_ColorSpace::Family::kDeviceCMYK ||
          cs_family == CPDF_ColorSpace::Family::kSeparation ||
          cs_family == CPDF_ColorSpace::Family::kDeviceN);
}

bool CPDF_RenderStatus::ShouldApplyOverprintStroke(CPDF_PageObject* pObj) const {
  const CPDF_GeneralState& state = pObj->general_state();

  // 检查基本条件
  if (!state.HasRef()) {
    return false;
  }

  // 检查描边叠印是否启用
  if (!state.GetStrokeOP()) {
    return false;
  }

  // 检查叠印模式（0 = 标准叠印）
  if (state.GetOPMode() != 0) {
    return false;
  }

  // 检查混合模式（必须是 Normal）
  if (state.GetBlendType() != BlendMode::kNormal) {
    return false;
  }

  // 检查透明度（必须完全不透明）
  if (state.GetStrokeAlpha() != 1.0f) {
    return false;
  }

  // 检查颜色空间（必须是 CMYK、Separation 或 DeviceN）
  CPDF_ColorSpace::Family cs_family = GetObjectColorSpaceFamily(pObj, false);
  return (cs_family == CPDF_ColorSpace::Family::kDeviceCMYK ||
          cs_family == CPDF_ColorSpace::Family::kSeparation ||
          cs_family == CPDF_ColorSpace::Family::kDeviceN);
}
```

**测试要求**：

- 测试各种条件组合（FillOP、StrokeOP、OPMode、透明度等）
- 测试各种颜色空间
- 测试边界情况

**验收标准**：

- ✅ 函数正确检测叠印条件
- ✅ 所有测试用例通过
- ✅ 代码注释完整

---

### Day 2：分别渲染函数框架

#### 任务 2.1：创建分别渲染路径函数

**文件**：`core/fpdfapi/render/cpdf_renderstatus.h` 和 `.cpp`

**新增函数**：

```cpp
// 在 cpdf_renderstatus.h 的 private 部分添加
bool DrawPathWithSeparateBlendModes(
    CPDF_PathObject* path_obj,
    const CFX_Matrix& mtObj2Device,
    bool fill_needs_darken,
    bool stroke_needs_darken);
```

**实现框架**：

```cpp
bool CPDF_RenderStatus::DrawPathWithSeparateBlendModes(
    CPDF_PathObject* path_obj,
    const CFX_Matrix& mtObj2Device,
    bool fill_needs_darken,
    bool stroke_needs_darken) {

  // 保存原始混合模式
  BlendMode original_blend = path_obj->general_state().GetBlendType();

  CFX_FillRenderOptions::FillType fill_type = path_obj->filltype();
  bool stroke = path_obj->stroke();

  CFX_Matrix path_matrix = path_obj->matrix() * mtObj2Device;
  if (!IsAvailableMatrix(path_matrix)) {
    return true;
  }

  bool result = true;

  // 渲染填充（如果需要）
  if (fill_type != CFX_FillRenderOptions::FillType::kNoFill) {
    BlendMode fill_blend = fill_needs_darken
        ? BlendMode::kDarken
        : original_blend;

    // 临时设置混合模式
    path_obj->mutable_general_state().SetBlendType(fill_blend);

    // 创建只填充的选项
    CFX_FillRenderOptions fill_options = GetFillOptionsForDrawPathWithBlend(
        options_.GetOptions(), path_obj, fill_type, false, type3_char_);

    uint32_t fill_argb = GetFillArgb(path_obj);

    // 调用 DrawPath，只填充
    result = device_->DrawPath(
        *path_obj->path().GetObject(), &path_matrix,
        path_obj->graph_state().GetObject(), fill_argb, 0, fill_options);

    if (!result) {
      path_obj->mutable_general_state().SetBlendType(original_blend);
      return false;
    }
  }

  // 渲染描边（如果需要）
  if (stroke) {
    BlendMode stroke_blend = stroke_needs_darken
        ? BlendMode::kDarken
        : original_blend;

    // 临时设置混合模式
    path_obj->mutable_general_state().SetBlendType(stroke_blend);

    // 创建只描边的选项
    CFX_FillRenderOptions stroke_options = GetFillOptionsForDrawPathWithBlend(
        options_.GetOptions(), path_obj,
        CFX_FillRenderOptions::FillType::kNoFill, true, type3_char_);

    uint32_t stroke_argb = GetStrokeArgb(path_obj);

    // 调用 DrawPath，只描边
    result = device_->DrawPath(
        *path_obj->path().GetObject(), &path_matrix,
        path_obj->graph_state().GetObject(), 0, stroke_argb, stroke_options);
  }

  // 恢复原始混合模式
  path_obj->mutable_general_state().SetBlendType(original_blend);

  return result;
}
```

**注意事项**：

- 需要检查 `GetFillOptionsForDrawPathWithBlend` 函数是否支持只填充或只描边的选项
- 可能需要创建新的辅助函数来生成选项

**测试要求**：

- 测试只填充的情况
- 测试只描边的情况
- 测试两者都有的情况
- 测试错误处理

**验收标准**：

- ✅ 函数框架完成
- ✅ 基本逻辑正确
- ✅ 代码可以编译通过

---

## 阶段二：路径对象实现（3-4 天）

### Day 3-4：修改 ProcessPath 函数

#### 任务 3.1：集成叠印检测到 ProcessPath

**文件**：`core/fpdfapi/render/cpdf_renderstatus.cpp`

**修改位置**：`ProcessPath` 函数（第 423-456 行）

**修改逻辑**：

```cpp
bool CPDF_RenderStatus::ProcessPath(CPDF_PathObject* path_obj,
                                    const CFX_Matrix& mtObj2Device) {
  CFX_FillRenderOptions::FillType fill_type = path_obj->filltype();
  bool stroke = path_obj->stroke();
  ProcessPathPattern(path_obj, mtObj2Device, &fill_type, &stroke);
  if (fill_type == CFX_FillRenderOptions::FillType::kNoFill && !stroke) {
    return true;
  }

  // If the option to convert fill paths to stroke is enabled for forced color,
  // set |fill_type| to FillType::kNoFill and |stroke| to true.
  CPDF_RenderOptions::Options& options = options_.GetOptions();
  if (options_.ColorModeIs(CPDF_RenderOptions::Type::kForcedColor) &&
      options.bConvertFillToStroke &&
      fill_type != CFX_FillRenderOptions::FillType::kNoFill) {
    stroke = true;
    fill_type = CFX_FillRenderOptions::FillType::kNoFill;
  }

  // ========== 新增：叠印检测逻辑 ==========
  bool fill_needs_overprint = false;
  bool stroke_needs_overprint = false;

  if (fill_type != CFX_FillRenderOptions::FillType::kNoFill) {
    fill_needs_overprint = ShouldApplyOverprintFill(path_obj);
  }
  if (stroke) {
    stroke_needs_overprint = ShouldApplyOverprintStroke(path_obj);
  }

  // 如果两者都需要叠印，使用统一的 darken 模式（优化路径）
  if (fill_needs_overprint && stroke_needs_overprint) {
    BlendMode original_blend = path_obj->general_state().GetBlendType();
    path_obj->mutable_general_state().SetBlendType(BlendMode::kDarken);

    uint32_t fill_argb = GetFillArgb(path_obj);
    uint32_t stroke_argb = GetStrokeArgb(path_obj);
    CFX_Matrix path_matrix = path_obj->matrix() * mtObj2Device;
    if (!IsAvailableMatrix(path_matrix)) {
      path_obj->mutable_general_state().SetBlendType(original_blend);
      return true;
    }

    bool result = device_->DrawPath(
        *path_obj->path().GetObject(), &path_matrix,
        path_obj->graph_state().GetObject(), fill_argb, stroke_argb,
        GetFillOptionsForDrawPathWithBlend(options, path_obj, fill_type, stroke,
                                           type3_char_));

    path_obj->mutable_general_state().SetBlendType(original_blend);
    return result;
  }

  // 如果只有一方需要叠印，使用分别渲染
  if (fill_needs_overprint || stroke_needs_overprint) {
    return DrawPathWithSeparateBlendModes(path_obj, mtObj2Device,
                                         fill_needs_overprint,
                                         stroke_needs_overprint);
  }
  // ========== 叠印检测逻辑结束 ==========

  // 原有逻辑：都不需要叠印
  uint32_t fill_argb = fill_type != CFX_FillRenderOptions::FillType::kNoFill
                           ? GetFillArgb(path_obj)
                           : 0;
  uint32_t stroke_argb = stroke ? GetStrokeArgb(path_obj) : 0;
  CFX_Matrix path_matrix = path_obj->matrix() * mtObj2Device;
  if (!IsAvailableMatrix(path_matrix)) {
    return true;
  }

  return device_->DrawPath(
      *path_obj->path().GetObject(), &path_matrix,
      path_obj->graph_state().GetObject(), fill_argb, stroke_argb,
      GetFillOptionsForDrawPathWithBlend(options, path_obj, fill_type, stroke,
                                         type3_char_));
}
```

**测试要求**：

- 创建包含路径对象叠印的测试 PDF
- 测试只填充叠印
- 测试只描边叠印
- 测试两者都叠印
- 测试都不叠印（原有逻辑）
- 对比 Adobe Acrobat 渲染结果

**验收标准**：

- ✅ 代码编译通过
- ✅ 基本功能测试通过
- ✅ 与 Adobe Acrobat 渲染结果一致

---

#### 任务 3.2：完善 DrawPathWithSeparateBlendModes

**文件**：`core/fpdfapi/render/cpdf_renderstatus.cpp`

**需要完善的内容**：

1. 检查 `GetFillOptionsForDrawPathWithBlend` 是否支持只填充/只描边
2. 如果不支持，需要创建辅助函数
3. 处理错误情况
4. 优化性能（减少重复计算）

**可能需要的辅助函数**：

```cpp
CFX_FillRenderOptions GetFillOnlyOptions(...);
CFX_FillRenderOptions GetStrokeOnlyOptions(...);
```

**测试要求**：

- 测试各种路径类型（矩形、圆形、复杂路径等）
- 测试各种填充模式（EvenOdd、Winding）
- 测试性能（确保无明显性能回归）

**验收标准**：

- ✅ 函数完整实现
- ✅ 所有测试用例通过
- ✅ 性能测试通过

---

### Day 5：路径对象测试和调试

#### 任务 5.1：单元测试

**文件**：创建 `core/fpdfapi/render/cpdf_renderstatus_overprint_unittest.cpp`

**测试内容**：

- `ShouldApplyOverprintFill` 函数测试
- `ShouldApplyOverprintStroke` 函数测试
- `GetObjectColorSpaceFamily` 函数测试
- `DrawPathWithSeparateBlendModes` 函数测试

#### 任务 5.2：集成测试

**测试内容**：

- 使用真实 PDF 文档测试
- 对比 Adobe Acrobat 渲染结果
- 测试各种边界情况

#### 任务 5.3：Bug 修复

- 修复发现的问题
- 优化代码
- 更新文档

**验收标准**：

- ✅ 所有测试通过
- ✅ 无已知 bug
- ✅ 代码审查通过

---

## 阶段三：文本对象实现（3-4 天）

### Day 6-7：修改 ProcessText 函数

#### 任务 6.1：创建文本分别渲染函数

**文件**：`core/fpdfapi/render/cpdf_renderstatus.h` 和 `.cpp`

**新增函数**：

```cpp
// 在 cpdf_renderstatus.h 的 private 部分添加
bool DrawTextWithSeparateBlendModes(
    CPDF_TextObject* text_obj,
    const CFX_Matrix& mtObj2Device,
    bool fill_needs_darken,
    bool stroke_needs_darken);
```

**实现思路**：

- 文本对象的渲染逻辑比路径对象复杂
- 需要处理文本渲染模式（Fill、Stroke、FillStroke 等）
- 需要调用 `DrawTextPathWithPattern` 或直接渲染
- 参考 `ProcessText` 的现有实现

**实现框架**：

```cpp
bool CPDF_RenderStatus::DrawTextWithSeparateBlendModes(
    CPDF_TextObject* text_obj,
    const CFX_Matrix& mtObj2Device,
    bool fill_needs_darken,
    bool stroke_needs_darken) {

  // 保存原始混合模式
  BlendMode original_blend = text_obj->general_state().GetBlendType();

  const TextRenderingMode text_render_mode = text_obj->text_state().GetTextMode();
  RetainPtr<CPDF_Font> pFont = text_obj->text_state().GetFont();

  // 确定需要渲染的内容
  bool is_fill = false;
  bool is_stroke = false;
  // ... 根据 text_render_mode 确定 ...

  bool result = true;

  // 渲染填充（如果需要）
  if (is_fill && fill_needs_darken) {
    text_obj->mutable_general_state().SetBlendType(BlendMode::kDarken);
    // ... 渲染填充文本 ...
    // 注意：需要只渲染填充，不渲染描边
  }

  // 渲染描边（如果需要）
  if (is_stroke && stroke_needs_darken) {
    text_obj->mutable_general_state().SetBlendType(BlendMode::kDarken);
    // ... 渲染描边文本 ...
    // 注意：需要只渲染描边，不渲染填充
  }

  // 恢复原始混合模式
  text_obj->mutable_general_state().SetBlendType(original_blend);

  return result;
}
```

**注意事项**：

- 文本对象的渲染逻辑复杂，需要仔细研究现有代码
- 可能需要修改 `DrawTextPathWithPattern` 函数
- 需要考虑 Type3 字体等特殊情况

**测试要求**：

- 测试各种文本渲染模式
- 测试 Type3 字体
- 测试复杂文本布局

**验收标准**：

- ✅ 函数框架完成
- ✅ 基本逻辑正确
- ✅ 代码可以编译通过

---

#### 任务 6.2：集成叠印检测到 ProcessText

**文件**：`core/fpdfapi/render/cpdf_renderstatus.cpp`

**修改位置**：`ProcessText` 函数（第 816 行开始）

**修改逻辑**（类似 ProcessPath）：

```cpp
bool CPDF_RenderStatus::ProcessText(CPDF_TextObject* textobj,
                                    const CFX_Matrix& mtObj2Device,
                                    CFX_Path* clipping_path) {
  // ... 现有代码 ...

  // ========== 新增：叠印检测逻辑 ==========
  bool fill_needs_overprint = false;
  bool stroke_needs_overprint = false;

  if (is_fill) {
    fill_needs_overprint = ShouldApplyOverprintFill(textobj);
  }
  if (is_stroke) {
    stroke_needs_overprint = ShouldApplyOverprintStroke(textobj);
  }

  // 如果两者都需要叠印，使用统一的 darken 模式
  if (fill_needs_overprint && stroke_needs_overprint) {
    BlendMode original_blend = textobj->general_state().GetBlendType();
    textobj->mutable_general_state().SetBlendType(BlendMode::kDarken);
    // ... 使用原有渲染逻辑 ...
    textobj->mutable_general_state().SetBlendType(original_blend);
    return result;
  }

  // 如果只有一方需要叠印，使用分别渲染
  if (fill_needs_overprint || stroke_needs_overprint) {
    return DrawTextWithSeparateBlendModes(textobj, mtObj2Device,
                                         fill_needs_overprint,
                                         stroke_needs_overprint);
  }
  // ========== 叠印检测逻辑结束 ==========

  // 原有逻辑：都不需要叠印
  // ... 现有代码 ...
}
```

**测试要求**：

- 创建包含文本对象叠印的测试 PDF
- 测试各种文本渲染模式
- 对比 Adobe Acrobat 渲染结果

**验收标准**：

- ✅ 代码编译通过
- ✅ 基本功能测试通过
- ✅ 与 Adobe Acrobat 渲染结果一致

---

### Day 8：文本对象测试和调试

#### 任务 8.1：单元测试

**测试内容**：

- `DrawTextWithSeparateBlendModes` 函数测试
- 各种文本渲染模式测试

#### 任务 8.2：集成测试

**测试内容**：

- 使用真实 PDF 文档测试
- 测试各种字体类型
- 测试复杂文本布局

#### 任务 8.3：Bug 修复

- 修复发现的问题
- 优化代码

**验收标准**：

- ✅ 所有测试通过
- ✅ 无已知 bug

---

## 阶段四：图像对象完善（2-3 天）

### Day 9-10：完善 ProcessImage

#### 任务 9.1：分析现有实现问题

**文件**：`core/fpdfapi/render/cpdf_imagerenderer.cpp`

**问题分析**：

1. 当前只检查了 `GetFillOP()`，未检查 `GetStrokeOP()`
2. 图像对象理论上可能有描边（虽然不常见）
3. 需要确保实现完整

**修改方案**：

- 保持现有实现（图像对象通常只有填充）
- 添加注释说明
- 如果需要，可以添加描边支持

#### 任务 9.2：代码审查和优化

**任务内容**：

- 审查现有代码
- 优化性能
- 添加注释
- 确保与路径/文本对象的实现一致

**测试要求**：

- 测试各种图像类型
- 测试 CMYK、Separation、DeviceN 颜色空间
- 对比 Adobe Acrobat 渲染结果

**验收标准**：

- ✅ 代码优化完成
- ✅ 测试通过
- ✅ 代码审查通过

---

## 阶段五：测试和优化（2-3 天）

### Day 11-12：全面测试

#### 任务 11.1：功能测试

**测试内容**：

1. **路径对象测试**

   - 只填充叠印
   - 只描边叠印
   - 两者都叠印
   - 各种颜色空间

2. **文本对象测试**

   - 只填充叠印
   - 只描边叠印
   - 两者都叠印
   - 各种文本渲染模式

3. **图像对象测试**

   - 各种颜色空间
   - 各种图像格式

4. **组合测试**
   - 多个对象组合
   - 复杂 PDF 文档

#### 任务 11.2：兼容性测试

**测试内容**：

- 测试各种设备驱动（Skia、AGG、Win32 等）
- 测试打印输出
- 测试不同平台

#### 任务 11.3：性能测试

**测试内容**：

- 对比实现前后的性能
- 测试大规模文档
- 确保无性能回归

**验收标准**：

- ✅ 所有功能测试通过
- ✅ 兼容性测试通过
- ✅ 性能测试通过（无明显回归）

---

### Day 13：代码审查和文档

#### 任务 13.1：代码审查

**审查内容**：

- 代码风格
- 代码注释
- 错误处理
- 性能优化

#### 任务 13.2：文档更新

**文档内容**：

- 代码注释
- 设计文档更新
- 用户文档（如果需要）

**验收标准**：

- ✅ 代码审查通过
- ✅ 文档更新完成

---

## 阶段六：可选扩展（1 周）

### Day 14-15：Shading 对象支持（可选）

#### 任务 14.1：分析 Shading 对象

**分析内容**：

- Shading 对象的渲染流程
- 是否需要叠印支持
- 实现复杂度

#### 任务 14.2：实现 Shading 叠印（如果需要）

**实现内容**：

- 类似路径对象的实现
- 测试和调试

---

### Day 16-17：Form 对象支持（可选）

#### 任务 16.1：分析 Form 对象

**分析内容**：

- Form 对象的渲染流程
- Form 对象内部的对象如何处理
- 实现复杂度

#### 任务 16.2：实现 Form 叠印（如果需要）

**实现内容**：

- Form 对象内部递归处理
- 测试和调试

---

## 风险应对计划

### 风险 1：设备驱动不支持分别渲染

**应对措施**：

- 实现回退逻辑
- 如果设备驱动不支持，回退到统一渲染
- 记录警告日志

### 风险 2：性能问题

**应对措施**：

- 优化渲染逻辑
- 减少重复计算
- 使用缓存机制

### 风险 3：边界情况处理

**应对措施**：

- 充分测试
- 参考 PDF 规范
- 对比 Adobe Acrobat 行为

---

## 交付物清单

### 代码交付物

1. ✅ 修改的文件：

   - `core/fpdfapi/render/cpdf_renderstatus.h`
   - `core/fpdfapi/render/cpdf_renderstatus.cpp`
   - `core/fpdfapi/render/cpdf_imagerenderer.cpp`（可选）

2. ✅ 新增的测试文件：

   - `core/fpdfapi/render/cpdf_renderstatus_overprint_unittest.cpp`

3. ✅ 测试 PDF 文档：
   - 包含各种叠印场景的测试 PDF

### 文档交付物

1. ✅ 代码注释
2. ✅ 设计文档更新
3. ✅ 测试报告

---

## 验收标准

### 功能验收

- ✅ 路径对象支持填充和描边分别叠印
- ✅ 文本对象支持填充和描边分别叠印
- ✅ 图像对象叠印功能完善
- ✅ 所有对象类型渲染结果与 Adobe Acrobat 一致

### 质量验收

- ✅ 所有单元测试通过
- ✅ 所有集成测试通过
- ✅ 代码审查通过
- ✅ 无已知 bug
- ✅ 性能测试通过（无明显回归）

---

## 时间表总结

| 阶段   | 任务         | 预计时间 | 开始日期 | 结束日期 |
| ------ | ------------ | -------- | -------- | -------- |
| 阶段一 | 基础架构搭建 | 3-4 天   | Day 1    | Day 4    |
| 阶段二 | 路径对象实现 | 3-4 天   | Day 3    | Day 5    |
| 阶段三 | 文本对象实现 | 3-4 天   | Day 6    | Day 8    |
| 阶段四 | 图像对象完善 | 2-3 天   | Day 9    | Day 10   |
| 阶段五 | 测试和优化   | 2-3 天   | Day 11   | Day 13   |
| 阶段六 | 可选扩展     | 1 周     | Day 14   | Day 17   |

**总计**：10-15 个工作日（2-3 周）

---

## 后续工作

1. **监控和优化**

   - 收集用户反馈
   - 性能监控
   - 持续优化

2. **扩展支持**

   - Shading 对象支持（如果需要）
   - Form 对象支持（如果需要）

3. **文档完善**
   - 用户文档
   - API 文档
   - 最佳实践文档

---

**计划制定时间**：2024 年 12 月
**版本**：1.0
**状态**：待实施

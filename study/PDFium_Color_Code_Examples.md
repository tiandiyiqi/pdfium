# PDFium 颜色处理代码示例

## 1. 颜色空间转换实现

### 1.1 CMYK 到 RGB 转换核心代码

```cpp
// 文件：core/src/fpdf_page_colors.cpp
// Adobe CMYK 到 sRGB 转换函数
void AdobeCMYK_to_sRGB1(FX_FLOAT c, FX_FLOAT m, FX_FLOAT y, FX_FLOAT k,
                        FX_FLOAT* r, FX_FLOAT* g, FX_FLOAT* b) {
    // 应用黑色分量
    FX_FLOAT cyan = (1.0f - c) * (1.0f - k);
    FX_FLOAT magenta = (1.0f - m) * (1.0f - k);
    FX_FLOAT yellow = (1.0f - y) * (1.0f - k);

    // 转换为RGB
    *r = cyan;
    *g = magenta;
    *b = yellow;

    // 限制颜色值范围
    *r = std::max(0.0f, std::min(1.0f, *r));
    *g = std::max(0.0f, std::min(1.0f, *g));
    *b = std::max(0.0f, std::min(1.0f, *b));
}
```

### 1.2 颜色空间类定义

```cpp
// 文件：core/include/fpdfapi/fpdf_color.h
class CPDF_ColorSpace {
public:
    enum Family {
        DEVICEGRAY = 1,
        DEVICERGB,
        DEVICECMYK,
        CALGRAY,
        CALRGB,
        LAB,
        ICCBASED,
        SEPARATION,
        DEVICEN,
        INDEXED,
        PATTERN
    };

    virtual ~CPDF_ColorSpace() {}

    // 获取颜色空间类型
    virtual Family GetFamily() const = 0;

    // 颜色分量数量
    virtual int CountComponents() const = 0;

    // 颜色转换
    virtual bool GetRGB(FX_FLOAT* pBuf, FX_FLOAT* R,
                       FX_FLOAT* G, FX_FLOAT* B) const = 0;
};
```

## 2. 透明度处理实现

### 2.1 透明度状态管理

```cpp
// 文件：core/include/fpdfapi/fpdf_graphstate.h
class CPDF_Transparency {
private:
    FX_FLOAT m_Alpha;           // Alpha值 (0.0-1.0)
    bool m_bIsolated;           // 是否隔离
    bool m_bKnockout;           // 是否挖空

public:
    CPDF_Transparency() : m_Alpha(1.0f), m_bIsolated(false), m_bKnockout(false) {}

    // 设置透明度
    void SetAlpha(FX_FLOAT alpha) {
        m_Alpha = std::max(0.0f, std::min(1.0f, alpha));
    }

    FX_FLOAT GetAlpha() const { return m_Alpha; }

    // 隔离设置
    void SetIsolated(bool isolated) { m_bIsolated = isolated; }
    bool IsIsolated() const { return m_bIsolated; }

    // 挖空设置
    void SetKnockout(bool knockout) { m_bKnockout = knockout; }
    bool IsKnockout() const { return m_bKnockout; }

    // 应用透明度到颜色
    FX_COLORREF ApplyToColor(FX_COLORREF color) const {
        FX_BYTE alpha = static_cast<FX_BYTE>(m_Alpha * 255);
        return (color & 0x00FFFFFF) | (alpha << 24);
    }
};
```

### 2.2 透明度组渲染

```cpp
// 透明度组渲染逻辑（简化版）
void RenderTransparencyGroup(CPDF_PageObject* pObj,
                            const CPDF_Transparency& trans) {
    // 创建临时表面用于透明度组渲染
    CFX_DIBitmap tempSurface;
    tempSurface.Create(pObj->GetWidth(), pObj->GetHeight(), FXDIB_Argb);

    // 渲染到临时表面
    RenderToSurface(pObj, &tempSurface);

    // 应用透明度
    if (trans.IsIsolated()) {
        // 隔离透明度组：独立处理
        ApplyIsolatedAlpha(&tempSurface, trans.GetAlpha());
    } else {
        // 非隔离：与背景混合
        ApplyNonIsolatedAlpha(&tempSurface, trans.GetAlpha());
    }

    // 挖空处理
    if (trans.IsKnockout()) {
        ApplyKnockoutEffect(&tempSurface);
    }

    // 合成到最终表面
    CompositeToFinalSurface(&tempSurface);
}
```

## 3. 叠印处理代码

### 3.1 通用状态中的叠印设置

```cpp
// 文件：core/include/fpdfapi/fpdf_page/fpdf_pageobj.h
class CPDF_GeneralStateData {
private:
    bool m_StrokeOP;            // 描边叠印标志（true：启用描边叠印）
    bool m_FillOP;              // 填充叠印标志（true：启用填充叠印）
    int m_OPMode;               // 叠印模式（0：标准叠印，1：保留底层）
    FX_DWORD m_OverprintFlags;  // 叠印颜色分量标志

public:
    CPDF_GeneralStateData() : m_StrokeOP(false), m_FillOP(false), m_OPMode(0), m_OverprintFlags(0) {}
};

// CPDF_GeneralState是CPDF_GeneralStateData的计数引用包装类
class CPDF_GeneralState : public CFX_CountRef<CPDF_GeneralStateData> {
public:
    // 获取描边叠印状态
    bool GetStrokeOP() const {
        return m_pObj ? m_pObj->m_StrokeOP : false;
    }

    // 设置描边叠印状态
    void SetStrokeOP(bool strokeOP) {
        if (m_pObj) {
            m_pObj->m_StrokeOP = strokeOP;
        }
    }

    // 获取填充叠印状态
    bool GetFillOP() const {
        return m_pObj ? m_pObj->m_FillOP : false;
    }

    // 设置填充叠印状态
    void SetFillOP(bool fillOP) {
        if (m_pObj) {
            m_pObj->m_FillOP = fillOP;
        }
    }

    // 获取叠印模式
    int GetOPMode() const {
        return m_pObj ? m_pObj->m_OPMode : 0;
    }

    // 设置叠印模式
    void SetOPMode(int opMode) {
        if (m_pObj) {
            m_pObj->m_OPMode = opMode;
        }
    }
};
```

### 3.2 叠印渲染实现逻辑

```cpp
// 文件：core/src/fpdfapi/fpdf_render/fpdf_render_image.cpp
if (m_BitmapAlpha == 255 && pGeneralState && pGeneralState->m_FillOP &&
    pGeneralState->m_OPMode == 0 && pGeneralState->m_BlendType == FXDIB_BLEND_NORMAL &&
    pGeneralState->m_StrokeAlpha == 1 && pGeneralState->m_FillAlpha == 1) {
    // 应用叠印效果 - 设置混合模式为DARKEN
    m_BlendType = FXDIB_BLEND_DARKEN;
}

// 文件：core/src/fpdfapi/fpdf_render/fpdf_render.cpp
if ((m_Options.m_Flags & RENDER_OVERPRINT) && pPageObj->m_Type == PDFPAGE_IMAGE && pGeneralState && pGeneralState->m_FillOP && pGeneralState->m_StrokeOP) {
    // 叠印渲染逻辑
    // 当启用叠印选项且同时启用了填充和描边叠印时执行特定渲染
    CPDF_Document* pDocument = NULL;
    CPDF_Page* pPage = NULL;

    // 获取当前文档和页面
    if (m_pContext->m_pPageCache) {
        pPage = m_pContext->m_pPageCache->GetPage();
        pDocument = pPage->m_pDocument;
    } else {
        pDocument = ((CPDF_ImageObject*)pPageObj)->m_pImage->GetDocument();
    }

    // 获取颜色空间
    CPDF_Dictionary* pPageResources = pPage ? pPage->m_pPageResources : NULL;
    CPDF_Object* pCSObj = ((CPDF_ImageObject*)pPageObj)->m_pImage->GetStream()->GetDict()->GetElementValue(FX_BSTRC("ColorSpace"));
    CPDF_ColorSpace* pColorSpace = pDocument->LoadColorSpace(pCSObj, pPageResources);

    if (pColorSpace) {
        int format = pColorSpace->GetFamily();
        // 对CMYK、SEPARATION或DEVICEN颜色空间应用DARKEN混合模式实现叠印
        if (format == PDFCS_DEVICECMYK || format == PDFCS_SEPARATION || format == PDFCS_DEVICEN) {
            blend_type = FXDIB_BLEND_DARKEN;
        }
        pDocument->GetPageData()->ReleaseColorSpace(pCSObj);
    }
}
```

## 3.3 叠印变量详细分析

### 3.3.1 变量定义与结构关系

**核心变量定义**：

- `m_StrokeOP`：布尔值，存储在 `CPDF_GeneralStateData` 类中，控制描边是否启用叠印
- `m_FillOP`：布尔值，存储在 `CPDF_GeneralStateData` 类中，控制填充是否启用叠印
- `m_OPMode`：整数，存储在 `CPDF_GeneralStateData` 类中，控制叠印操作模式（0：标准叠印，1：保留底层）

**结构关系**：

- `CPDF_GeneralState` 是 `CPDF_GeneralStateData` 的计数引用包装类
- `CPDF_GeneralState` 提供了安全的访问和修改接口（如 `GetStrokeOP()`、`SetStrokeOP()` 等）
- 这些变量通过 PDF 内容解析器在处理 PDF 操作符时被设置

### 3.3.2 实现逻辑与功能用途

**变量设置逻辑**（来自 fpdf_page_graph_state.cpp）：

- **`/OP` 操作符**：设置 `m_StrokeOP`，如果不存在 `/op` 操作符，同时设置 `m_FillOP`
- **`/op` 操作符**：单独设置 `m_FillOP`
- **`/OPM` 操作符**：设置 `m_OPMode`

**渲染实现逻辑**（来自 fpdf_render_image.cpp 和 fpdf_render.cpp）：

- **填充叠印触发条件**：

  ```cpp
  if (m_BitmapAlpha == 255 && pGeneralState && pGeneralState->m_FillOP &&
      pGeneralState->m_OPMode == 0 && pGeneralState->m_BlendType == FXDIB_BLEND_NORMAL &&
      pGeneralState->m_StrokeAlpha == 1 && pGeneralState->m_FillAlpha == 1) {
      m_BlendType = FXDIB_BLEND_DARKEN;
  }
  ```

- **完整叠印触发条件**：
  ```cpp
  if ((m_Options.m_Flags & RENDER_OVERPRINT) && pPageObj->m_Type == PDFPAGE_IMAGE &&
      pGeneralState && pGeneralState->m_FillOP && pGeneralState->m_StrokeOP) {
      // 对特定颜色空间应用DARKEN混合模式
  }
  ```

**功能用途**：

1. **精细控制叠印行为**：通过 `m_StrokeOP` 和 `m_FillOP` 分别控制描边和填充的叠印
2. **灵活的叠印模式**：通过 `m_OPMode` 选择标准叠印或保留底层模式
3. **条件性叠印应用**：仅在满足特定条件时（如透明度为 1、混合模式为正常等）应用叠印
4. **颜色空间感知**：仅对 CMYK、SEPARATION 和 DEVICEN 颜色空间应用叠印效果

### 3.3.3 变量间关系与程序流程

**变量间的关系**：

- `m_StrokeOP` 和 `m_FillOP` 是独立的布尔开关，分别控制描边和填充的叠印
- `m_OPMode` 控制叠印的具体行为模式，0 表示标准叠印（使用 DARKEN 混合模式），1 表示保留底层（不改变混合模式）
- 这些变量与其他图形状态变量（如 `m_StrokeAlpha`、`m_FillAlpha`、`m_BlendType`）协同工作

**在整体程序流程中的作用**：

1. **解析阶段**：PDF 内容解析器处理 `/OP`、`/op` 和 `/OPM` 操作符，设置相应的叠印变量
2. **渲染准备阶段**：渲染器检查叠印相关变量的状态
3. **渲染执行阶段**：根据叠印变量的组合状态，决定是否应用 DARKEN 混合模式来实现叠印效果
4. **最终合成阶段**：应用叠印效果的颜色与底层颜色进行合成

**与示例文档的对应关系**：
PDFium 的实现与示例文档中提到的 4 种叠印模式可以通过以下组合对应：

| 示例文档模式        | PDFium 控制组合                                     | 效果                     |
| ------------------- | --------------------------------------------------- | ------------------------ |
| 0：无叠印           | `m_StrokeOP=false` + `m_FillOP=false`               | 不应用叠印效果           |
| 1：仅描边叠印       | `m_StrokeOP=true` + `m_FillOP=false` + `m_OPMode=0` | 仅描边应用叠印效果       |
| 2：仅填充叠印       | `m_StrokeOP=false` + `m_FillOP=true` + `m_OPMode=0` | 仅填充应用叠印效果       |
| 3：描边和填充都叠印 | `m_StrokeOP=true` + `m_FillOP=true` + `m_OPMode=0`  | 描边和填充都应用叠印效果 |

### 3.3.4 最佳实践与注意事项

1. **颜色空间限制**：叠印效果仅对 CMYK、SEPARATION 和 DEVICEN 颜色空间有效
2. **透明度影响**：只有当透明度为 1.0 时，叠印效果才会被应用
3. **混合模式影响**：只有当混合模式为正常时，叠印效果才会被应用
4. **渲染选项**：需要启用 `RENDER_OVERPRINT` 渲染选项才能看到叠印效果
5. **性能考虑**：叠印处理会增加渲染复杂度，应仅在必要时使用

## 4. 混合模式实现

### 4.1 混合模式枚举

```cpp
// 文件：core/include/fpdfapi/fpdf_blendmode.h
enum BlendMode {
    BM_NORMAL,          // 正常
    BM_MULTIPLY,        // 正片叠底
    BM_SCREEN,          // 滤色
    BM_OVERLAY,         // 叠加
    BM_DARKEN,          // 变暗
    BM_LIGHTEN,         // 变亮
    BM_COLORDODGE,      // 颜色减淡
    BM_COLORBURN,       // 颜色加深
    BM_HARDLIGHT,       // 强光
    BM_SOFTLIGHT,       // 柔光
    BM_DIFFERENCE,      // 差值
    BM_EXCLUSION,       // 排除
    BM_HUE,             // 色相
    BM_SATURATION,      // 饱和度
    BM_COLOR,           // 颜色
    BM_LUMINOSITY       // 明度
};
```

### 4.2 混合模式计算函数

```cpp
// 混合模式核心计算（简化示例）
FX_COLORREF ApplyBlendMode(BlendMode mode,
                          FX_COLORREF src,
                          FX_COLORREF dst) {
    // 提取颜色分量
    FX_BYTE srcR = FXSYS_GetRValue(src);
    FX_BYTE srcG = FXSYS_GetGValue(src);
    FX_BYTE srcB = FXSYS_GetBValue(src);
    FX_BYTE dstR = FXSYS_GetRValue(dst);
    FX_BYTE dstG = FXSYS_GetGValue(dst);
    FX_BYTE dstB = FXSYS_GetBValue(dst);

    FX_BYTE resultR, resultG, resultB;

    switch (mode) {
        case BM_NORMAL:
            // 正常混合：直接使用源颜色
            resultR = srcR;
            resultG = srcG;
            resultB = srcB;
            break;

        case BM_MULTIPLY:
            // 正片叠底：src * dst / 255
            resultR = (srcR * dstR) / 255;
            resultG = (srcG * dstG) / 255;
            resultB = (srcB * dstB) / 255;
            break;

        case BM_SCREEN:
            // 滤色：255 - (255-src)*(255-dst)/255
            resultR = 255 - ((255 - srcR) * (255 - dstR)) / 255;
            resultG = 255 - ((255 - srcG) * (255 - dstG)) / 255;
            resultB = 255 - ((255 - srcB) * (255 - dstB)) / 255;
            break;

        case BM_OVERLAY:
            // 叠加：根据背景亮度选择正片叠底或滤色
            resultR = (dstR < 128) ?
                (2 * srcR * dstR / 255) :
                (255 - 2 * (255 - srcR) * (255 - dstR) / 255);
            resultG = (dstG < 128) ?
                (2 * srcG * dstG / 255) :
                (255 - 2 * (255 - srcG) * (255 - dstG) / 255);
            resultB = (dstB < 128) ?
                (2 * srcB * dstB / 255) :
                (255 - 2 * (255 - srcB) * (255 - dstB) / 255);
            break;

        // 其他混合模式的实现...
        default:
            resultR = srcR;
            resultG = srcG;
            resultB = srcB;
    }

    return FXRGB(resultR, resultG, resultB);
}
```

## 5. 实际应用示例

### 5.1 使用 SDK API 处理颜色

```cpp
// 使用PDFium SDK进行颜色处理的示例
#include "fpdfview.h"
#include "fpdfedit.h"

void ProcessPDFColors(const char* pdf_path) {
    // 初始化PDFium库
    FPDF_LIBRARY_CONFIG config;
    config.version = 2;
    config.m_pUserFontPaths = NULL;
    config.m_pIsolate = NULL;
    config.m_v8EmbedderSlot = 0;
    FPDF_InitLibraryWithConfig(&config);

    // 加载文档
    FPDF_DOCUMENT doc = FPDF_LoadDocument(pdf_path, NULL);
    if (!doc) {
        printf("无法加载PDF文档\n");
        return;
    }

    // 获取页面数量
    int page_count = FPDF_GetPageCount(doc);

    for (int i = 0; i < page_count; i++) {
        // 加载页面
        FPDF_PAGE page = FPDF_LoadPage(doc, i);

        // 获取页面尺寸
        double width = FPDF_GetPageWidth(page);
        double height = FPDF_GetPageHeight(page);

        // 创建位图用于渲染
        FPDF_BITMAP bitmap = FPDFBitmap_Create(width, height, 1);
        FPDFBitmap_FillRect(bitmap, 0, 0, width, height, 0xFFFFFFFF);

        // 渲染页面（应用所有颜色处理）
        FPDF_RenderPageBitmap(bitmap, page, 0, 0, width, height, 0, 0);

        // 处理完成，清理资源
        FPDFBitmap_Destroy(bitmap);
        FPDF_ClosePage(page);
    }

    FPDF_CloseDocument(doc);
    FPDF_DestroyLibrary();
}
```

### 5.2 自定义颜色处理回调

```cpp
// 自定义颜色处理回调示例
class CustomColorHandler {
public:
    // 颜色空间转换回调
    static bool CustomColorConvert(void* pThis,
                                  FX_FLOAT* pSrc,
                                  FX_FLOAT* pDst,
                                  int nSrcComponents) {
        // 自定义颜色转换逻辑
        if (nSrcComponents == 4) { // CMYK
            // 应用自定义CMYK到RGB转换
            CustomCMYKtoRGB(pSrc[0], pSrc[1], pSrc[2], pSrc[3],
                           &pDst[0], &pDst[1], &pDst[2]);
            return true;
        }
        return false; // 使用默认转换
    }

    // 透明度处理回调
    static void CustomAlphaBlend(FX_BYTE* pDest,
                                const FX_BYTE* pSrc,
                                int length,
                                FX_BYTE alpha) {
        // 自定义透明度混合算法
        for (int i = 0; i < length; i++) {
            pDest[i] = (pSrc[i] * alpha + pDest[i] * (255 - alpha)) / 255;
        }
    }
};
```

## 6. 性能优化技巧

### 6.1 颜色转换优化

```cpp
// 使用查表法加速颜色转换
class ColorConversionLUT {
private:
    FX_BYTE m_CMYKtoRGB[256][256][256][256]; // 4D查找表

public:
    ColorConversionLUT() {
        // 预计算所有可能的CMYK到RGB转换
        for (int c = 0; c < 256; c++) {
            for (int m = 0; m < 256; m++) {
                for (int y = 0; y < 256; y++) {
                    for (int k = 0; k < 256; k++) {
                        FX_FLOAT r, g, b;
                        AdobeCMYK_to_sRGB1(c/255.0f, m/255.0f, y/255.0f, k/255.0f, &r, &g, &b);
                        m_CMYKtoRGB[c][m][y][k] = FXRGB(r*255, g*255, b*255);
                    }
                }
            }
        }
    }

    FX_COLORREF Convert(FX_BYTE c, FX_BYTE m, FX_BYTE y, FX_BYTE k) {
        return m_CMYKtoRGB[c][m][y][k]; // 直接查表，避免浮点计算
    }
};
```

---

_这些代码示例展示了 PDFium 颜色处理的核心实现，开发者可以参考这些代码理解 PDFium 的内部工作机制。_

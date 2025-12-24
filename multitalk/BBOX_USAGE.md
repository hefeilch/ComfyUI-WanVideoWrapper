# MultiTalk Mask 节点使用说明

## 节点说明

现在有两个节点可用于创建 MultiTalk 所需的 mask：

1. **MultiTalkMaskFromBBoxOrMask** (推荐) - 支持两种模式：
   - 直接使用已有的 mask
   - 从 bbox 生成 mask

2. **BBoxToMultiTalkMask** (保留) - 仅支持从 bbox 生成 mask

## 节点位置

在 ComfyUI 节点菜单中：
- `WanVideoWrapper` → `MultiTalk Mask (BBox or Mask)` (推荐)
- `WanVideoWrapper` → `BBox to MultiTalk Mask` (保留，向后兼容)

## MultiTalkMaskFromBBoxOrMask 节点参数

### 必填参数

- **mode** (COMBO): 工作模式
  - `use_mask`: 使用输入的 mask（如果已有 mask）
  - `from_bbox`: 从 bbox 生成 mask（如果没有 mask）

### use_mask 模式参数

- **input_mask** (MASK): 第一个输入的 mask
  - 支持格式: `[num_masks, H, W]` 或 `[H, W]` 或 `[B, num_masks, H, W]`
  - 如果是单个 mask `[H, W]`，会自动添加维度

- **mask_2** (MASK, 可选): 第二个 mask
- **mask_3** (MASK, 可选): 第三个 mask
- **mask_4** (MASK, 可选): 第四个 mask

- **add_background_mask** (BOOLEAN): 是否添加背景 mask（仅在 from_bbox 模式有效）

### from_bbox 模式参数

- **bbox_1** (STRING): 第一个 bbox，格式为 `x1,y1,x2,y2`
  - `x1, y1`: 左上角坐标
  - `x2, y2`: 右下角坐标
  - 示例: `"0,0,416,480"` 表示从 (0,0) 到 (416,480) 的矩形区域

- **bbox_2** (STRING, 可选): 第二个 bbox
- **bbox_3** (STRING, 可选): 第三个 bbox
- **bbox_4** (STRING, 可选): 第四个 bbox

- **image** (IMAGE, 可选): 输入图像（用于自动获取尺寸）
  - 如果提供，节点会自动从图像获取宽度和高度
  - 如果未提供，需要使用 `width` 和 `height` 参数

- **width** (INT): 图像宽度（仅在未提供 `image` 时使用）
  - 默认值: 832
  - 范围: 64-4096

- **height** (INT): 图像高度（仅在未提供 `image` 时使用）
  - 默认值: 480
  - 范围: 64-4096

- **add_background_mask** (BOOLEAN): 是否包含背景 mask
  - `True`: 会创建一个背景 mask（所有 bbox 区域之外的区域）
  - `False`: 只创建说话者的 mask
  - 推荐启用，有助于模型更好地理解图像结构

## 输出

- **ref_target_masks** (MASK): 生成的 mask 张量
  - 形状: `[num_masks, height, width]`
  - 可以直接连接到 `MultiTalkWav2VecEmbeds` 节点的 `ref_target_masks` 输入

## 使用示例

### 示例 1: 使用已有 mask（use_mask 模式）

如果你已经有 mask（例如从其他节点生成）：

```
LoadMask → MultiTalkMaskFromBBoxOrMask (mode=use_mask) → MultiTalkWav2VecEmbeds
                ↓
            (ref_target_masks)
```

支持多种 mask 格式：
- 单个 mask: `[H, W]` → 自动转换为 `[1, H, W]`
- 多个 mask: `[num_masks, H, W]` → 直接使用
- Batch mask: `[B, num_masks, H, W]` → 取第一个 batch

### 示例 2: 从 bbox 生成 mask（from_bbox 模式）

假设图像尺寸为 832x480，两个说话者分别位于左右两侧：

```
mode: "from_bbox"
bbox_1: "0,0,416,480"      # 左侧说话者
bbox_2: "416,0,832,480"    # 右侧说话者
add_background_mask: True
```

### 示例 3: 使用图像自动获取尺寸

1. 连接 `LoadImage` 节点的输出到 `MultiTalkMaskFromBBoxOrMask` 的 `image` 输入
2. 设置 bbox 坐标（相对于图像尺寸）
3. 节点会自动从图像获取正确的尺寸

### 示例 4: 混合使用 mask 和 bbox

如果你有一些 mask，但还需要添加一个从 bbox 生成的 mask：

1. 使用 `use_mask` 模式处理已有的 mask
2. 使用 `from_bbox` 模式生成额外的 mask
3. 使用 `MaskComposite` 节点合并所有 mask

### 示例 5: 完整工作流

```
LoadImage → MultiTalkMaskFromBBoxOrMask (from_bbox) → MultiTalkWav2VecEmbeds
                ↓
            (ref_target_masks)
```

## 注意事项

### use_mask 模式

1. **mask 格式**: 支持多种格式，节点会自动处理
   - `[H, W]` → 自动转换为 `[1, H, W]`
   - `[num_masks, H, W]` → 直接使用
   - `[B, num_masks, H, W]` → 取第一个 batch
2. **尺寸一致性**: 如果多个 mask 尺寸不一致，会自动 resize 到第一个 mask 的尺寸
3. **mask 数量**: 最多支持 4 个输入 mask

### from_bbox 模式

1. **坐标格式**: bbox 必须使用 `x1,y1,x2,y2` 格式，用逗号分隔
2. **坐标范围**: 坐标会自动裁剪到图像范围内
3. **有效性检查**: 如果 bbox 的宽度或高度为 0，该 bbox 会被忽略
4. **mask 数量**: 
   - 如果有 N 个 bbox，且 `add_background_mask=True`，会生成 N+1 个 mask
   - 如果有 N 个 bbox，且 `add_background_mask=False`，会生成 N 个 mask

### 通用注意事项

1. **音频对应**: mask 的顺序应该与 `MultiTalkWav2VecEmbeds` 中音频的顺序对应
   - 第一个 mask 对应 `audio_1`
   - 第二个 mask 对应 `audio_2`
   - 以此类推
2. **输出格式**: 所有模式都输出 `[num_masks, H, W]` 格式的 mask
3. **mask 值**: mask 中的值应该是 0.0（背景）或 1.0（目标区域）

## 常见问题

**Q: 如何确定 bbox 坐标？**
A: 可以使用图像编辑软件（如 Photoshop、GIMP）查看像素坐标，或者使用 ComfyUI 中的其他节点（如 GroundingDINO）自动检测。

**Q: 如果 bbox 重叠会怎样？**
A: 每个 bbox 会创建独立的 mask。如果重叠，在重叠区域会有多个 mask 同时为 1.0。

**Q: 背景 mask 的作用是什么？**
A: 背景 mask 帮助模型区分说话者区域和背景区域，通常能提高生成质量。

## 技术细节

- mask 数据类型: `torch.float32`
- mask 值: 0.0（背景）或 1.0（目标区域）
- 输出形状: `[num_masks, height, width]`
- 与 MultiTalk 的兼容性: 完全兼容 `ref_target_masks` 格式要求


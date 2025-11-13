# QRCodeDetection — 二维码检测流程说明

本项目基于 OpenCV 的 WeChatQRCode 模型，对输入目录中的图片批量进行二维码检测与解码，并输出带检测框与文本标注的结果图片到 `output/`。本文档详细介绍整体流程、每一步做了什么处理、这样做的好处，以及如何运行与定制。


## 依赖与准备
- Python 3.8+（建议）
- `opencv-contrib-python>=4.5.1`（必须，包含 WeChatQRCode 模块；注意不是 `opencv-python`）
- 预训练模型（已包含）：
  - `models/detect.prototxt`, `models/detect.caffemodel`
  - `models/sr.prototxt`, `models/sr.caffemodel`

安装依赖：
```bash
pip install -r requirements.txt
```


## 快速开始
- 将待检测图片放入 `input/` 目录（支持 `.jpg/.jpeg/.png/.bmp/.gif`）。
- 运行：
```bash
python main.py
```
- 结果图片保存在 `output/`，文件名与输入一致，会叠加检测框和解码文本。
- Windows 可直接双击 `run.bat`。


## 整体流程（做了什么、为什么）
1) 输入与健壮性检查
- 文件存在性与大小限制（10MB，避免超大图拖慢或耗尽内存）
- 解码读取图片（支持含中文路径）
- 过大分辨率自动等比缩放到不超过 4000×4000（控制内存与耗时）

2) 图像质量评估与基础增强
- 亮度、对比度、清晰度（拉普拉斯方差）三项质检
- 若质量不足，进行轻量级亮度与对比度增强（`alpha=1.2, beta=10`）
- 目的：在不引入过多伪影的前提下，提高二维码模块与边缘可分辨性

3) 多分支预处理，提升鲁棒性
- 原图（优先级最高，避免不必要的过处理）
- 高斯降噪 + Otsu 全局二值化（对干净背景有效）
- 自适应阈值（Gaussian）二值化（对复杂/不均匀光照背景更稳健）
- LAB 通道 CLAHE 局部对比度增强（提升暗区细节，抑制过曝）
- 轻微锐化（增强边缘但强度受控，减少过度锐化伪影）
- 目的：不同图像条件下，至少有一个支路为 WeChatQRCode 提供最适输入

4) 基于 WeChatQRCode 的检测与解码
- 逐一尝试各个预处理分支进行 `detectAndDecode`
- 汇总、去重所有成功解码结果；选择点集更多的一支用于可视化
- 目的：最大化召回，同时保持结果简洁一致

5) 可视化与输出
- 在图像上绘制二维码四边形轮廓与解码文本
- 使用 `cv2.imencode(...).tofile(...)` 保存结果，兼容中文路径
- 批处理多个文件，控制台输出彩色提示，易于查看批量结果

6) 超时保护与边界控制
- 30 秒超时触发中断，避免异常图像或模型卡死
- 最多返回前 10 个二维码结果，防止极端场景刷屏


## 每步处理的好处（要点）
- 尺寸与大小限制：稳定内存/耗时，避免异常输入拖垮进程
- 质量评估：早发现“不可救”的图像，减少无效尝试；或有针对性地轻量增强
- 多分支预处理：适配不同噪声、光照、对比场景，提升整体召回率与稳定性
- 原图优先：保留最佳天然信号，避免过处理导致的信息丢失
- 可视化输出：便于排查失败样本和校准阈值、预处理强度
- 超时与上限：批处理可控、稳定，不因少量异常样本影响整体


## 关键实现节点（便于二开）
- 模型加载：`main.py:55`（WeChatQRCode 与模型路径）
- 文件大小限制：`main.py:23`
- 读图与中文路径支持：`main.py:32`
- 最大分辨率压缩：`main.py:42`
- 图像质量评估入口：`main.py:49`；实现：`main.py:193`
  - 亮度阈值：`main.py:201`
  - 对比度阈值：`main.py:207`
  - 模糊度阈值：`main.py:215`
- 轻量增强：`main.py:52`
- 预处理分支：
  - 灰度/降噪/Otsu：`main.py:77`, `main.py:79`
  - 自适应阈值：`main.py:84`
  - CLAHE：`main.py:93`
  - 锐化：`main.py:99`
- 分支检测循环：`main.py:116`
- 结果去重：`main.py:127`
- 绘制框与文本：`main.py:135`, `main.py:140`
- 输出路径与写盘：`main.py:151`, `main.py:154`
- 超时设置：`main.py:109`
- 返回数量上限：`main.py:179`
- 批处理入口与输入目录：`main.py:232`


## 参数与定制建议
- 更激进的增强（低光/低对比）：
  - 提高 `alpha` 或 `beta`（`main.py:52`），或增加高增益 CLAHE（`main.py:93`）
- 降低误检或抑制噪声：
  - 放宽锐化强度（`main.py:99`），或提高降噪核尺寸（`main.py:78`）
- 加速：
  - 减少预处理分支数量；下调分辨率阈值（`main.py:42`）；调低超时（`main.py:109`）
- 更多类型码制：
  - 目前使用 WeChatQRCode 专注二维码，如需一维码/其他码制，可并行集成 `pyzbar` 等


## 目录结构
```
QRCodeDetection/
├─ input/                # 待检测图片（示例：image.png）
├─ output/               # 结果图片输出目录
├─ models/               # WeChatQRCode 相关模型
│  ├─ detect.prototxt
│  ├─ detect.caffemodel
│  ├─ sr.prototxt
│  └─ sr.caffemodel
├─ main.py               # 主流程脚本
├─ requirements.txt      # 依赖
├─ run.bat               # Windows 一键运行
├─ create_venv_linux.sh  # Linux/macOS 快速环境脚本
└─ create_venv_windows.bat
```


## 常见问题
- ImportError: 没有 `cv2.wechat_qrcode`
  - 请确认安装的是 `opencv-contrib-python` 而非 `opencv-python`；版本 >= 4.5.1
- 无法读取/写入中文路径
  - 已采用 `imdecode`/`imencode(...).tofile(...)`，一般可兼容；若仍失败，请确认系统默认编码或路径权限
- 检测慢/卡住
  - 可降低超时、减少分支、或下调最大分辨率阈值
- 检测不到二维码
  - 检查是否过曝/过暗/模糊；尝试提高增强强度或更换清晰样本



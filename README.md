# 🚀 Manga-to-CBZ-Packager: 漫画/同人本子全自动批量重构打包工具（支持文件夹与 PDF 双向转 CBZ）

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

专为本地漫画党、同人本子收藏家、以及 Emby / Perfect Viewer / KuroReader 玩家打造的 PC 端全自动洗地打包神器！

本工具基于 Python 核心，可一键全盘递归扫描指定目录，实现两大硬核功能：
1. **文件夹 ➔ CBZ**：将散碎的 JPG/PNG/WebP 图片文件夹，全自动无损高压缩打包为标准的 `.cbz` 格式。
2. **PDF ➔ CBZ**：将排版死板、阅读器兼容极差的 PDF 本子，自动高速提取每一页为超清图片，并重新封装为标准的 `.cbz` 格式。

---

## ✨ 为什么选择 `.cbz` 格式？

- 📱 **全平台阅读器完美通刷**：不管是 iPad 的 ComicShare、安卓的拷贝漫画、还是 PC 端的各路看番神器，`.cbz` 都是公网公认的原生漫画格式，支持双页无缝裁剪、进度条秒拉。
- 🎬 **Emby 刮削不卡顿**：散碎图片文件夹会让 Emby 扫描到怀疑人生，而单个封装好的 `.cbz` 物理文件可以让 Emby 秒速完成元数据识别与海报墙铺设！

---

## 📦 核心处理脚本 (`manga_packager.py`)

本脚本完全采用本地化安全处理逻辑，不上传任何数据，保护你的私人“圣殿”绝对隐私。

### 1. 前置依赖安装（PC 端）
脚本中 PDF 高速提取图片依赖 `PyMuPDF` 库（速度比传统库快数十倍）。在命令行/终端中一键安装：
```bash
pip install PyMuPDF
```

### 2. 完整源码（直接复制并保存为 `manga_packager.py`）

```python
import os
import sys
import zipfile
import shutil
import fitz  # PyMuPDF 依赖库

# --- ⚙️ 核心参数配置区域 ---
# 填写你需要批量洗地、打包的漫画/本子根目录（支持深层递归嵌套）
INPUT_DIR = r"D:\Manga_Collections" 

# 是否在打包成功后，全自动删除原有的碎图片文件夹和 PDF（True 为删除，False 为保留防翻车）
DELETE_ORIGINAL = False

# --- 🚀 核心自动化逻辑 (无需修改) ---

def folder_to_cbz(folder_path):
    """将普通的图片文件夹打包为 .cbz"""
    parent_dir = os.path.dirname(folder_path)
    folder_name = os.path.basename(folder_path)
    cbz_path = os.path.join(parent_dir, f"{folder_name}.cbz")
    
    if os.path.exists(cbz_path):
        print(f"⚠️ 跳过: {cbz_path} 已存在。")
        return False

    print(f"📦 正在打包文件夹 ➔ {folder_name}.cbz ...")
    has_images = False
    
    with zipfile.ZipFile(cbz_path, 'w', zipfile.ZIP_DEFLATED) as cbz:
        for root, _, files in os.walk(folder_path):
            for file in sorted(files):
                if file.lower().endswith(('.png', '.jpg', '.jpeg', '.webp', '.bmp')):
                    has_images = True
                    file_full_path = os.path.join(root, file)
                    # 保持压缩包内的相对路径清爽
                    arc_name = os.path.relpath(file_full_path, folder_path)
                    cbz.write(file_full_path, arc_name)
                    
    if not has_images:
        os.remove(cbz_path)
        print(f"❌ 放弃: 文件夹 [{folder_name}] 内未检测到任何有效漫画图片。")
        return False
        
    print(f"🎯 成功: {folder_name}.cbz 打包落盘！")
    if DELETE_ORIGINAL:
        shutil.rmtree(folder_path)
        print(f"🧹 已自动擦除原有碎图片文件夹。")
    return True

def pdf_to_cbz(pdf_path):
    """高效提取 PDF 页面并封装为 .cbz"""
    dir_name = os.path.dirname(pdf_path)
    base_name = os.path.splitext(os.path.basename(pdf_path))[0]
    cbz_path = os.path.join(dir_name, f"{base_name}.cbz")
    
    if os.path.exists(cbz_path):
        print(f"⚠️ 跳过: {cbz_path} 已存在。")
        return False

    print(f"⚡ 正在撕裂并重构 PDF ➔ {base_name}.cbz ...")
    temp_extract_dir = os.path.join(dir_name, f"tmp_{base_name}")
    os.makedirs(temp_extract_dir, exist_ok=True)
    
    try:
        # 打开 PDF 物理档案
        doc = fitz.open(pdf_path)
        for page_num in range(len(doc)):
            page = doc.load_page(page_num)
            # 矩阵放大系数，确保提取出来的漫画字迹、网格细节绝对高清不糊
            pix = page.get_pixmap(matrix=fitz.Matrix(2, 2)) 
            img_path = os.path.join(temp_extract_dir, f"page_{page_num:04d}.jpg")
            pix.save(img_path)
            
        # 顺手关门
        doc.close()
        
        # 将临时提取出来的高清图无损压入 CBZ
        with zipfile.ZipFile(cbz_path, 'w', zipfile.ZIP_DEFLATED) as cbz:
            for root, _, files in os.walk(temp_extract_dir):
                for file in sorted(files):
                    file_full_path = os.path.join(root, file)
                    cbz.write(file_full_path, file)
                    
        print(f"🎯 成功: PDF 完美重构 ➔ {base_name}.cbz！")
        if DELETE_ORIGINAL:
            os.remove(pdf_path)
            print(f"🧹 已物理蒸发原始 PDF 文件。")
            
    except Exception as e:
        print(f"❌ 翻车: 处理 PDF 时遭遇未知结界: {str(e)}")
        if os.path.exists(cbz_path):
            os.remove(cbz_path)
    finally:
        # 无论成功失败，必定踩碎临时中转舱，防爆本地硬盘
        if os.path.exists(temp_extract_dir):
            shutil.rmtree(temp_extract_dir)

def main():
    if not os.path.exists(INPUT_DIR):
        print(f"❌ 错误: 配置的漫画根目录 [{INPUT_DIR}] 不存在，请检查路径！")
        sys.exit(1)
        
    print("==========================================================")
    print("🚀 赛博本子打包战车启动，开始全盘检索...")
    print("==========================================================")
    
    # 第一轮：专门对付散碎的 PDF 文件
    for root, _, files in os.walk(INPUT_DIR):
        for file in files:
            if file.lower().endswith('.pdf'):
                pdf_full_path = os.path.join(root, file)
                pdf_to_cbz(pdf_full_path)
                
    # 第二轮：专门清洗和打包散碎的图片文件夹
    # 为了防止递归冲突，先全量捕获需要打包的文件夹目标
    target_folders = []
    for root, dirs, _ in os.walk(INPUT_DIR):
        for d in dirs:
            # 过滤掉脚本运行过程中产生的临时文件夹
            if not d.startswith('tmp_'):
                target_folders.append(os.path.join(root, d))
                
    # 从最深层文件夹逆向开始打包，防止父子目录嵌套逻辑错乱
    for folder in reversed(target_folders):
        if os.path.exists(folder): # 再次确认该文件夹没有被上一步的级联操作干掉
            folder_to_cbz(folder)

    print("==========================================================")
    print("🏁 终场大结局！全量漫画资产已全自动重构为标准 CBZ 矩阵！")
    print("==========================================================")

if __name__ == "__main__":
    main()
```

---

## 📊 战果审计与清洗现场

当脚本在 PC 端全力开火时，你的命令行终端将实时吐出极度舒适的排障日志：

```text
==========================================================
🚀 赛博本子打包战车启动，开始全盘检索...
==========================================================
⚡ 正在撕裂并重构 PDF ➔ [同人志] 灵梦的赛博神社日常.cbz ...
🎯 成功: PDF 完美重构 ➔ [同人志] 灵梦的赛博神社日常.cbz！
📦 正在打包文件夹 ➔ [Manga] Re_从零开始的异世界生活_第01卷.cbz ...
🎯 成功: [Manga] Re_从零开始的异世界生活_第01卷.cbz 打包落盘！
==========================================================
🏁 终场大结局！全量漫画资产已全自动重构为标准 CBZ 矩阵！
==========================================================
```

原本杂乱不堪、混杂着 PDF 和上万张碎图片的文件夹，瞬间变成了一本本整整齐齐、随时可以双页流利翻阅的标准 `.cbz` 赛博实体书！

---

## 📄 许可证与免责声明

本项目基于 **[MIT License](LICENSE)** 协议开源。
请在运行脚本前将 `DELETE_ORIGINAL` 保持为 `False` 进行安全测试，确保生成的 `.cbz` 在你的漫画阅读器中能完美流畅读取后，再开启自动洗地抹除功能。作者不对任何因操作不当导致的原站本子资产丢失承担责任。
```

---
name: cotton-quotation-ocr
description: 识别微信截图中的棉花质检报告（棉包组批信息汇总表），提取价格、红字、断裂比强度并导出Excel
---
# 棉花质检报告图片OCR识别

Use when: 用户需要从微信截图/棉花包组批信息汇总表中识别价格、质量数据并导出Excel。

## 图片特征
- 微信截图，尺寸554×776（5张小图）到1276×1581（大图）不等
- 表格格式：棉包组批信息汇总表
- 红字包含关键价格信息（毛重单价或基差）
- 13张图片按格式分3类：图1-5(小图~550×780)、图6-9(中图~600×850)、图10-13(大图~730×1070)

## 完整工作流

### 1. 环境检查
```bash
which tesseract && tesseract --version | head -3
python3 -c "from PIL import Image; print('PIL ok')"
pip3 show pytesseract openpyxl 2>/dev/null | grep Name
```
- Python venv: `/home/hehao-linux/.hermes/hermes-agent/venv/bin/python3`
- 依赖: `pip3 install pytesseract openpyxl`

### 2. 图片路径
```
/mnt/f/cotton quotation/
```

### 3. 红字提取（关键步骤）
```python
from PIL import Image, ImageEnhance, ImageFilter, ImageOps
import pytesseract

def prep(img, scale=3):
    img2 = img.resize((img.width*scale, img.height*scale), Image.LANCZOS)
    gray = img2.convert('L')
    gray = ImageOps.autocontrast(gray, cutoff=2)
    gray = ImageEnhance.Contrast(gray).enhance(5)
    gray = ImageEnhance.Sharpness(gray).enhance(3)
    gray = gray.filter(ImageFilter.MedianFilter(size=3))
    return gray

# 提取红字（R>140, G<110, B<110）
img_rgb = img.convert('RGB')
red_mask = Image.new('RGB', img.size, (255,255,255))
px = img_rgb.load()
rm = red_mask.load()
for y in range(img.height):
    for x in range(img.width):
        r,g,b = px[x,y]
        if r>140 and g<110 and b<110:
            rm[x,y] = (255,0,0)
red_text = pytesseract.image_to_string(prep(red_mask,4), config='--oem 3 --psm 6 -l chi_sim+equ')
```
- 红字价格格式: `毛17700` = 毛重17700元/吨；`01+840基差` = ZCE_COTTON 01合约+840元基差

### 4. 表格区域分块OCR
```python
# 小图(554×780): 切 top(0~200) + bot(200~end)
# 中图(600×850): 切 left(0~w//2) + right(w//2~w)
# 大图(730×1070): 切 col1(0~w//3) + col2(w//3~2w//3) + col3(2w//3~w)
img = Image.open(path)
cropped = img.crop((x1,y1,x2,y2))
text = pytesseract.image_to_string(prep(cropped, 3), config='--oem 3 --psm 6 -l chi_sim+equ')
```

### 5. 字段解析正则
```python
# 批号
re.search(r'组批批号\s*[:：]\s*(\d+)', text)
# 毛重/净重/公重
re.search(r'合计毛重.*?([\d.]+)\s*吨', text)
re.search(r'合计净重.*?([\d.]+)\s*吨', text)
# 马克隆值（从红字）
re.search(r'马([\d.]+)', red_text)
# 断裂比强度
re.search(r'平均值[;:：]?\s*([\d.]+)', text)
re.search(r'最小值[;:：]?\s*([\d.]+)', text)
```

### 6. Excel导出
- 毛重单价(列19): 红色填充，白色粗体
- 基差(列20): 黄色填充，粗体
- 断裂比强度: 绿(≥30) 橙(27~29.9) 红(<27)

## 常见问题
- tesseract未安装: `sudo apt-get install tesseract-ocr tesseract-ocr-chi-sim`
- 图片分辨率太小(554×776): 放大4倍预处理；表格密集区域分块识别
- 中文OCR错误率高: 使用 `--psm 6 -l chi_sim+equ` 配置
- 微信截图表格线细: 用`ImageOps.autocontrast`+`Contrast.enhance(5)`增强后再OCR
- 颜色级OCR混淆: 认准`白棉X级`关键词

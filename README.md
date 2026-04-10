因为喜欢看书，发现有一款Delphi写的小说快捕好用，就用上了，显示研究了书源写法，
可惜很来软件没有开源又没人维护，不能处理https网站和分页网站的情况，自己有点编程基础就先简单做了1个通用小说下载器，根据目录页下载章节的小工具
很来。再后来就出现一款kotlin开发的安卓阅读的软件，比较好用，又是开源的，也在第1时间研究了书源写法，但是阅读的书源写法还是很复杂。
电脑上只能用虚拟机。虽然阅读有flutter开发，但是烂尾了，功能基本没实现。去年AI发展迅速就顺手用python把通用下载器的功能复刻借鉴快捕和阅读的优点，写了快捕Python版，
今年3月底的时候刚好想到flutter可以跨平台，然后就复现了快捕python版的功能，做了kuaibuflutter。功能倒是基本正常，但是有些地方做的不太好，于是开源出来。


这里再补充一下自动制作kuaibu专用书源的参考

# 自动书源生成器技术设计文档

## 1. 功能概述

自动书源生成器是一个用于快速生成 kuaibu 书源的工具。用户只需提供网站 URL，系统会自动：
- 分析网站的搜索功能
- 执行搜索并获取第一本书的详情页
- 获取目录页和正文页
- 提取 URL 模式并生成完整的书源规则

## 2. 核心流程

```
用户输入网站URL
    ↓
分析首页搜索功能
    ↓
执行搜索获取详情页
    ↓
获取目录页（判断详情页是否就是目录页）
    ↓
获取正文页（第一章）
    ↓
提取URL规则模式
    ↓
生成书源
```

## 3. 关键技术点

### 3.1 URL 模式分析

从右往左匹配 URL 路径段，最多检查 2 层：

**第1层匹配**：尝试 `书类_书号` 模式（下划线分隔）
- 例如：`/book/3_3311/` → 书类=3, 书号=3311

**第2层匹配**：尝试 `书类/书号` 模式（斜杠分隔）
- 例如：`/book/3/3311/` → 书类=book, 书号=3

**回落策略**：
- 如果两层都匹配失败，回落到只有书号
- 书类不能是固定词（如 book, novel, read 等）
- 书类不能包含下划线

### 3.2 目录页判断策略

详情页和目录页可能是同一个 URL，判断依据：

1. **检查是否有目录链接**：查找"目录"、"章节目录"等关键词
2. **检查分页导航**：检测分页组件
3. **检查章节链接数量**：匹配大量章节链接（>10 个）

如果详情页就是目录页，直接使用详情页 HTML 进行章节提取。

### 3.3 章节链接识别

遍历所有 `<a>` 标签，匹配章节链接的特征：
- 包含"第"和"章"
- 或者长度小于 30（短章节名）

找到第一个匹配的链接作为第一章。

### 3.4 kuaibu 格式搜索 URL

搜索 URL 使用特殊的 kuaibu 格式：
```
URL,{paramName={key}}
```

例如：
```
https://www.example.com/search.php,{searchkey={key}}
```

这种格式告诉 kuaibu：
- 实际搜索 URL 是什么
- 参数名是什么
- `{key}` 会被替换为用户输入的关键词

## 4. 算法设计

### 4.1 URL 模式提取算法

```python
def _extractDetailPattern(detail_url):
    # 1. 解析 URL 路径
    segments = detail_url.path_segments
    
    # 2. 从右往左取最多2层
    right_segments = segments[-2:]
    
    # 3. 第1层：检查下划线模式
    first = right_segments[0]
    if '_' in first:
        parts = first.split('_')
        if parts[0] 和 parts[1] 都非空:
            book_class = parts[0]
            book_id = parts[1]
            is_underscore = True
    
    # 4. 第2层：检查斜杠模式
    if not is_underscore and len(right_segments) >= 2:
        second = right_segments[1]
        first = right_segments[0]
        if first 非空 且 second 不是固定词 且 second 不含下划线:
            book_class = second
            book_id = first
    
    # 5. 构建模式
    pattern = base_url + path_segments.replace(book_id, '(书号)').replace(book_class, '(书类)')
```

### 4.2 目录页规则生成

```python
def _buildTocPattern(detail_url, book_pattern, toc_url):
    # 简单字符串替换
    return toc_url.replace(detail_url, book_pattern)
```

### 4.3 章节页规则生成

```python
def _buildChapterPattern(detail_url, book_pattern, chapter_url):
    # 1. 分离前段和后段
    chapter_suffix = chapter_url.replace(detail_url, '')
    
    # 2. 处理后段：把 . 左边替换为 (章号)
    dot_index = chapter_suffix.find('.')
    if dot_index >= 0:
        chapter_suffix = '(章号)' + chapter_suffix[dot_index:]
    else:
        chapter_suffix = '(章号)'
    
    # 3. 组合
    return book_pattern + chapter_suffix
```

## 5. 错误处理

### 5.1 搜索分析失败
- 尝试常见搜索 URL 模式（/s.php, /search.php, /search）
- 如果都失败，返回错误信息

### 5.2 详情页获取失败
- 检查搜索结果是否为空
- 检查 URL 是否有效
- 返回部分书源（只包含搜索功能）

### 5.3 目录页获取失败
- 检查详情页是否就是目录页
- 检查是否有目录链接
- 返回部分书源（包含已获取的规则）

### 5.4 正文页获取失败
- 检查章节链接是否有效
- 返回部分书源（包含已获取的规则）

## 6. 文件结构

```
auto_source_generator.py
├── AutoSourceGenerator 类
│   ├── analyze_search_function()      # 分析搜索功能
│   ├── fetch_search_results()         # 获取搜索结果
│   ├── fetch_toc_page()               # 获取目录页
│   ├── fetch_content_page()           # 获取正文页
│   ├── extract_url_patterns()         # 提取 URL 模式
│   ├── generate_source()              # 生成书源
│   └── start_generate()               # 主流程
└── HTMLTextExtractor 类               # HTML 文本提取器
```

## 7. 调试功能

系统会保存以下调试文件到 `debug_autosource` 目录：
- `01_首页.txt` - 首页 HTML
- `02_搜索页.txt` - 搜索结果页 HTML
- `03_详情页.txt` - 详情页 HTML
- `04_目录页.txt` - 目录页 HTML
- `05_正文页.txt` - 正文页 HTML

## 8. 示例

### 输入
```
网站URL: https://www.example.com/
搜索关键词: 我的
```

### 输出书源
```
网站名称=示例小说网
网站网址=https://www.example.com/
网站编码=UTF-8
简介页网址规则=https://www.example.com/book/(书类)/(书号)/
目录页网址规则=https://www.example.com/book/(书类)/(书号)/
章节页网址规则=https://www.example.com/book/(书类)/(书号)/(章号).html
目录章节排序方式=按HTML先后排列
搜索网址=https://www.example.com/search.php,{searchkey={key}}
搜索类型=POST
分类排行=
搜索状态=1
分类状态=0
```


其实分类排行也可以顺便生成，首页可以根据常用分类特征获取分类
识别分类范围，替换页码可以从右往左取最后网页扩展名加数字比如1.，替换为{page}.        
排行格式：分类1::网址1&&分类2::网址2
但是有些网站真正分类排行在二级链接

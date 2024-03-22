---
layout: post
title:  "jsontoexcel"
date:   2024-02-29 12:00:00 +0800
categories: linux
---

# 使用pandas将json解析成excel

```
import pandas as pd
from pandas import json_normalize
import json

# 读取 JSON 文件
with open('/xxx.json', 'r', encoding='utf-8') as file:
    data = json.load(file)

# 使用 json_normalize 将嵌套的 JSON 展平
flat_data = json_normalize(data)

# 创建一个 pandas 的 DataFrame
df = pd.DataFrame(flat_data)

# 将 DataFrame 保存为 Excel 文件
df.to_excel('xxx.xlsx', index=False)
```

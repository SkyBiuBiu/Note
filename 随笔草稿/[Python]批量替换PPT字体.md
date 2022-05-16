# [Python]批量替换PPT字体

[toc]

## 使用说明

1. 将脚本放置在需要批量修改的PPT文件夹根目录
2. 修改配置文件 conf.ini 中的字体
3. 执行脚本文件

​	exe文件 下载：[PPT换字体脚本.zip - 蓝奏云 (lanzouw.com)](https://wwu.lanzouw.com/ikzop032d2vc)


## 脚本代码

```python
from pptx import Presentation
import pptx_ea_font
import os
import configparser

def setFont(file):
   # 定义函数修改字体
   prs = Presentation(file)
   for slide in prs.slides:
      for shape in slide.shapes:
         if shape.has_text_frame:
            text_frame = shape.text_frame
            for paragraph in text_frame.paragraphs:
               for run in paragraph.runs:
                  pptx_ea_font.set_font(run, FONT)
   prs.save(file)


def traverse_directory_tree ():
   # 遍历目录树，更改后缀为.ppt和.pptx的文件
   path = os.getcwd()
   for root,dirs,files in os.walk(path):
      for file in files:
         if file.endswith(".pptx") or file.endswith(".ppt"):
            print(os.path.join(root,file) + " 的字体已被更改为：" + FONT)
            file = os.path.join(root,file)
            setFont(file)


if __name__ == "__main__":
   # 初始化对象
   conf = configparser.ConfigParser()
   conf.read("config.ini")
   print("目录下所有的PPT字体将设置为：" + conf.get("CONF", "FONT"))
   FONT = conf.get("CONF","FONT")
   # 执行命令
   traverse_directory_tree()

```



## 配置文件

```ini
[CONF]
FONT = 思源黑体 CN Regular
```


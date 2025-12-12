# 背景
Markdown 是一种轻量级标记语言，使人可以将时间都花在撰写内容而不是调整格式上，用来写技术文档再合适不过了。但markdown有个痛点---添加图片不方便。

markdown添加图片的方法：
- 方法1：插入本地图片（支持绝对路径和相对路径）
```
![avatar](/home/picture/1.png)
```
- 方法2：插入网络图片
```
![avatar](https://baidu.com/pic/doge.png)
```
如果图片在本地，markdown中插入本地图片，这样做缺点很多：
1 本地图片路径更改或丢失导致markdown找不到图片；
2 分享不灵活，mk文档发给别人时，找不到图片路径。

插入网络图片，需要自己制造图床，写文档时并不方便，另外看文档时需要网络环境。

还有一种办法，用base64转码工具，把图片转成一段字符串，然后引用这段base64编码即可，这样添加图片比较方便，但是需要一些多的动作。
- 方法3：图片转为base64
```
![avatar][doge] 

# 放在文章末尾
[doge]:data:image/png;base64,iVBORw0...... 
```

#  将markdown本地图片转为base64格式
写markdown文档时，如果一张图一张图转base64，但效率比较低，我们也可以写文档时先临时插入本地图片，写完之后通过脚本一次性生成base64的形式。

脚本有如下要求：

- 1 读取markdown文件，并用正则查找出所有的本地引用标签
- 2 替换图片标签`![image-name](url)` 为`![image-name][image-name]`
- 3 在md文件后面追加上base64编码 `[image-name]:图片base64编码`

.py脚本如下：

```
import sys
import re
import base64
import chardet

images_suffix = ["gif", "png", "jpg", "jpeg"]

appendix = []

def transform(lines):
    newlines = []
    pattern = re.compile(r'!\[(.*)\]\((.*)\)', re.I) # 正则匹配
    for line in lines:
        li = pattern.findall(line)
        if not li:
            newlines.append(line)
            continue
        for match in li:
            img_name = match[0]
            img_path = match[1]
            if not img_name or not img_path:
                newlines.append(line)
                continue
            if 'http' in img_path: # skip http img
                newlines.append(line)
                continue
            suffix = img_path.rsplit(".")[1]
            if suffix in images_suffix:
                try:
                    with open(img_path, 'rb') as f:
                        image_bytes = base64.b64encode(f.read())
                except:
                    newlines.append(line)
                    continue
                image_str = str(image_bytes)
                #print(image_str)
                base64_pre = 'data:image/' + suffix + ';base64,'
                real_image_str = base64_pre + image_str[2:len(image_str) - 1]
                appendix.append('[' + img_name + ']' + ':' + real_image_str + '\n\n\n')
                line = line.replace('(' + img_path + ')', '[' + img_name + ']')
        newlines.append(line)    
                
    return newlines

def md_2_img(markdown_file):
    if not markdown_file.endswith('.md'):
        return
    code = chardet.detect(open(markdown_file, 'rb').read())['encoding']
    with open(markdown_file, 'r',  encoding=code) as mk:
        lines = mk.readlines()
    newlines = transform(lines)
    with open(markdown_file, 'w',  encoding=code) as mk:
        mk.writelines(newlines)
    with open(markdown_file, 'a+',  encoding=code) as mk:
        mk.write("\n\n")
        mk.write("".join(appendix))

if __name__ == '__main__':
    markdown_file = sys.argv[1]
    print(markdown_file)
    md_2_img(markdown_file)
```
运行方式：

先用md的编译器直接ctrl+C、ctrl+V以本地方式写入图片，写完文档后.md文件同级目录下会有多个使用的图片，此时（最好copy一份一样的.md文件来运行脚本比较保险）打开Terminal终端（写这篇文档时我在用Anaconda Prompt终端，进入后直接用cd指令进入.md文件所在目录，把该.py脚本和.md文件放在一起，在终端中输入:python [你的脚本名].py [你的md名].md ，回车运行即会原地替换你的.md文件，你会发现.md文件所有的图片都以方法3的方式嵌入了图片，这就对本地图片没有任何依赖，支持上传到各大平台、网站进行展示（所有人都能看到图片）
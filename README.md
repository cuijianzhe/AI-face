# AI-face
这是一个基于python换脸的代码

# AI	换脸实现
> **科普：我们人眼看到连续画面的帧数为24 帧，大约 0.04秒，低于0.04就会卡成ppt。电影胶片是24帧 也就是每秒钟可以看到24张图像 低于这个数值就会感觉画面不流畅 所以以24帧为界限**
## 实现思路：
先把源视频文件转换成图片，在用API面部识别进行融合更换面部内容变成其他图形，并且利用软件完成对源文件音频的提取，再次把更换过的图片转换成为视频，并和音频进行融合。

1. 原视频转图片
2. 提取原视频音源
3. 图片面部识别并更换
4. 变化后的图片转视频
5. 音频和视频融合

## 环境：python3.7 + pycharm-2019.1 + ffmpeg 
[FFmpeg官网](http://ffmpeg.org/)
使用实例：
1. 提取音频：
**ffmpeg -i** 1.mp4  **-f mp3** 1.mp3
2.合成视频和音频
 **ffmpeg -i 没有声音.mp4 -i 提取生成的.mp3 -strict -2 -f mp4 合成的.mp4**

**需要的库文件：**

**opencv-python**
**pillow**（PIL）
**subprocess**

## Face++ 面部识别
在此使用旷视科技的人脸识别API进行完成。先对图片进行脸部识别并进行融合，看这里：
* [Face ++官方网址](https://www.faceplusplus.com.cn/)  
* [使用脸部识别API](https://console.faceplusplus.com.cn/documents/4888373)  
* [图像识别融合API](https://console.faceplusplus.com.cn/documents/20813963)

这就得注册并且拥有自己的API key进行API调用了
![image.png](https://img.hacpai.com/file/2019/09/image-127d14ee.png)

~~代码如下：~~
```python
import requests
import simplejson
import json
import base64

def find_face(imgfile):
    print("finding")
    http_url = 'https://api-cn.faceplusplus.com/facepp/v3/detect'
    data = {"api_key": 'nFnE8LQ32lteRvt99pA-kaMGCG9PRkGI',
            "api_secret": '0RD-G7z9LNEmd4WDdd7PJSmq7vQaIuTs', "image_file": imgfile, "return_landmark": 1}
    files = {"image_file": open(imgfile, "rb")}
    response = requests.post(http_url, data=data, files=files)
    req_con = response.content.decode('utf-8')   #decode将已编码的json字符串解码成python对象
    req_dict = json.JSONDecoder().decode(req_con)
    this_json = simplejson.dumps(req_dict)   #将Python对象编码成JSON字符串
    this_json2 = simplejson.loads(this_json)   #将已编码的 JSON 字符串解码为 Python 对象
    faces = this_json2['faces']
    # print(faces)
    list0 = faces[0]
    rectangle = list0['face_rectangle']
    # print(rectangle)

    return rectangle
# find_face()

def merge_face(img_url_1,img_url_2,img_url_3,number):
    ff1 = find_face(img_url_1)
    ff2 = find_face(img_url_2)
    rectangle1 = str(str(ff1['top']) + "," + str(ff1['left']) + "," + str(ff1['width']) + "," + str(ff1['height']))
    rectangle2 = str(ff2['top']) + "," + str(ff2['left']) + "," + str(ff2['width']) + "," + str(ff2['height'])
    url_add = "https://api-cn.faceplusplus.com/imagepp/v1/mergeface"
    f1 = open(img_url_1, 'rb')
    f1_64 = base64.b64encode(f1.read())
    f1.close()
    f2 = open(img_url_2, 'rb')
    f2_64 = base64.b64encode(f2.read())
    f2.close()
    data = {"api_key": 'nFnE8LQ32lteRvt99pA-kaMGCG9PRkGI', "api_secret": '0RD-G7z9LNEmd4WDdd7PJSmq7vQaIuTs',
            "template_base64": f1_64, "template_rectangle": rectangle1,
            "merge_base64": f2_64, "merge_rectangle": rectangle2, "merge_rate": number}
    response = requests.post(url_add, data=data)
    req_con = response.content.decode('utf-8')
    req_dict = json.JSONDecoder().decode(req_con)
    result = req_dict['result']
    imgdata = base64.b64decode(result)
    file = open(img_url_3, 'wb')
    file.write(imgdata)
    file.close() 
import os
def test():
    image1 = "guan.jpg"  #图2的脸会换到图1上
    image2 = "cjz.jpg"
    image3 = "1234.png"
    try:
        merge_face(image1,image2,image3,100)
    except:
        pass
test()
```
这是效果图：
![image.png](https://img.hacpai.com/file/2019/09/image-b02b1b33.png)
如果是我的脸换到朴信惠或者关晓彤任何一个人脸上会出现违和感，在此就不展示了。


## 把视频拆分成图片
```python
# 将视频拆分成图片
def video2txt_jpg():
    vc=cv2.VideoCapture("test.mp4")
    c=1
    if vc.isOpened():
        r, frame=vc.read()
        if not os.path.exists('images'):
            os.mkdir('images')
        os.chdir('images')
    else:
        r=False
    while r:
        cv2.imwrite(str(c) + '.jpg', frame)
        # txt2image()  # 同时转换为ascii图
        r, frame=vc.read()
        c+=1
    os.chdir('..')
    return vc
video2txt_jpg()
```
可以得出如下：
![image.png](https://img.hacpai.com/file/2019/08/image-75b47260.png)

## 把图片整合成视频：
```python
def jpg2video():
    fourcc=VideoWriter_fourcc(*"MJPG")
    fps = 24
    outfile_name = 'test11'
    images=os.listdir('images')
    im=Image.open('images/' + images[0])
    vw=cv2.VideoWriter(outfile_name + '.avi', fourcc, fps, im.size)

    os.chdir('images')
    for image in range(len(images)):
        # Image.open(str(image)+'.jpg').convert("RGB").save(str(image)+'.jpg')
        frame=cv2.imread(str(image + 1) + '.jpg')
        vw.write(frame)
        # print(str(image + 1) + '.jpg' + ' finished')
    os.chdir('..')
    vw.release()
```
如下：
![image.png](https://img.hacpai.com/file/2019/08/image-b1facb4e.png)

## 利用ffmpeg把视频文件中的音源提取出来

```python
# 调用ffmpeg获取mp3音频文件
def video2mp3():
    outfile_name= 'ccc' + '.mp3'
    subprocess.call('ffmpeg -i ' + "test.MP4" + ' -f mp3 ' + outfile_name, shell=True)
```
如下是提取过程：
![image.png](https://img.hacpai.com/file/2019/09/image-f27a0e1d.png)

## 将提取出来的音源和图片合成的视频整合
```python
def video_add_mp3():
    outfile_name= 'ccc-txt.mp4'
    subprocess.call('ffmpeg -i ' + 'test11.avi' + ' -i ' + "ccc.mp3" + ' -strict -2 -f mp4 ' + outfile_name, shell=True)
video_add_mp3()
```
![image.png](https://img.hacpai.com/file/2019/09/image-bbd6a7d6.png)
![image.png](https://img.hacpai.com/file/2019/09/image-e76d5543.png)

## 整合代码
到此分拆步骤完成，接下来整合成一个APP进行工作
```python
#!/usr/bin/python
# -*- coding: utf-8 -*-
# @Time  : 2019/9/1 8:50
# @Author : cuijianzhe
# @File  : AI换脸.py
# @Software: PyCharm

import requests
import simplejson
import json
import base64

import argparse
import os
import cv2
import subprocess
from cv2 import VideoWriter, VideoWriter_fourcc, imread, resize
from PIL import Image, ImageFont, ImageDraw
## 面部识别
def find_face(imgfile):
    print("finding")
    http_url = 'https://api-cn.faceplusplus.com/facepp/v3/detect'
    data = {"api_key": 'nFnE8LQ32lteRvt99pA-kaMGCG9PRkGI',
            "api_secret": '0RD-G7z9LNEmd4WDdd7PJSmq7vQaIuTs', "image_url": imgfile, "return_landmark": 1}
    files = {"image_file": open(imgfile, "rb")}
    response = requests.post(http_url, data=data, files=files)
    req_con = response.content.decode('utf-8')
    req_dict = json.JSONDecoder().decode(req_con)
    this_json = simplejson.dumps(req_dict)
    this_json2 = simplejson.loads(this_json)
    faces = this_json2['faces']
    list0 = faces[0]
    rectangle = list0['face_rectangle']
    # print(rectangle)
    return rectangle
#number表示换脸的相似度
def merge_face(image_url_1,image_url_2,image_url_3,number):
    ff1 = find_face(image_url_1)
    ff2 = find_face(image_url_2)
    rectangle1 = str(str(ff1['top']) + "," + str(ff1['left']) + "," + str(ff1['width']) + "," + str(ff1['height']))
    rectangle2 = str(ff2['top']) + "," + str(ff2['left']) + "," + str(ff2['width']) + "," + str(ff2['height'])
    url_add = "https://api-cn.faceplusplus.com/imagepp/v1/mergeface"
    f1 = open(image_url_1, 'rb')
    f1_64 = base64.b64encode(f1.read())
    f1.close()
    f2 = open(image_url_2, 'rb')
    f2_64 = base64.b64encode(f2.read())
    f2.close()
    data = {"api_key": 'nFnE8LQ32lteRvt99pA-kaMGCG9PRkGI', "api_secret": '0RD-G7z9LNEmd4WDdd7PJSmq7vQaIuTs',
            "template_base64": f1_64, "template_rectangle": rectangle1,
            "merge_base64": f2_64, "merge_rectangle": rectangle2, "merge_rate": number}
    response = requests.post(url_add, data=data)
    req_con = response.content.decode('utf-8')
    req_dict = json.JSONDecoder().decode(req_con)
    result = req_dict['result']
    imgdata = base64.b64decode(result)
    file = open(image_url_2, 'wb')
    file.write(imgdata)
    file.close()

def test(filename):
    image1 = "pxh.png"
    image2 = filename
    image = filename
    try:
        merge_face(image2,image1,image,90)
    except:
        pass

# 将视频拆分成图片
def video2txt_jpg(file_name):
    vc=cv2.VideoCapture(file_name)
    c=1
    if vc.isOpened():
        r, frame=vc.read()
        if not os.path.exists('Cache'):
            os.mkdir('Cache')
        os.chdir('Cache')
    else:
        r=False
    while r:
        cv2.imwrite(str(c) + '.jpg', frame)
        # txt2image()  # 同时转换为ascii图
        test('./Cache/'+str(c) + '.jpg')
        r, frame=vc.read()
        c+=1
    os.chdir('..')
    return vc

# 将图片合成视频
def jpg2video(outfile_name, fps):
    fourcc=VideoWriter_fourcc(*"MJPG")
    images=os.listdir('Cache')
    im=Image.open('Cache/' + images[0])
    vw=cv2.VideoWriter(outfile_name + '.avi', fourcc, fps, im.size)
    os.chdir('Cache')
    for image in range(len(images)):
        # Image.open(str(image)+'.jpg').convert("RGB").save(str(image)+'.jpg')
        frame=cv2.imread(str(image + 1) + '.jpg')
        vw.write(frame)
        # print(str(image + 1) + '.jpg' + ' finished')
    os.chdir('..')
    vw.release()

# 调用ffmpeg获取mp3音频文件
def video2mp3(file_name):
    outfile_name=file_name.split('.')[0] + '.mp3'
    subprocess.call('ffmpeg -i ' + file_name + ' -f mp3 ' + outfile_name, shell=True)
# 合成音频和视频文件
def video_add_mp3(file_name, mp3_file):
    outfile_name=file_name.split('.')[0] + '-txt.mp4'
    subprocess.call('ffmpeg -i ' + file_name + ' -i ' + mp3_file + ' -strict -2 -f mp4 ' + outfile_name, shell=True)

def find_face(imgfile):
    print("finding")
    http_url = 'https://api-cn.faceplusplus.com/facepp/v3/detect'
    data = {"api_key": 'nFnE8LQ32lteRvt99pA-kaMGCG9PRkGI',
            "api_secret": '0RD-G7z9LNEmd4WDdd7PJSmq7vQaIuTs', "image_url": imgfile, "return_landmark": 1}
    files = {"image_file": open(imgfile, "rb")}
    response = requests.post(http_url, data=data, files=files)
    req_con = response.content.decode('utf-8')
    req_dict = json.JSONDecoder().decode(req_con)
    this_json = simplejson.dumps(req_dict)
    this_json2 = simplejson.loads(this_json)
    faces = this_json2['faces']
    list0 = faces[0]
    rectangle = list0['face_rectangle']
    # print(rectangle)
    return rectangle
#number表示换脸的相似度
def merge_face(image_url_1,image_url_2,image_url_3,number):
    ff1 = find_face(image_url_1)
    ff2 = find_face(image_url_2)
    rectangle1 = str(str(ff1['top']) + "," + str(ff1['left']) + "," + str(ff1['width']) + "," + str(ff1['height']))
    rectangle2 = str(ff2['top']) + "," + str(ff2['left']) + "," + str(ff2['width']) + "," + str(ff2['height'])
    url_add = "https://api-cn.faceplusplus.com/imagepp/v1/mergeface"
    f1 = open(image_url_1, 'rb')
    f1_64 = base64.b64encode(f1.read())
    f1.close()
    f2 = open(image_url_2, 'rb')
    f2_64 = base64.b64encode(f2.read())
    f2.close()
    data = {"api_key": 'nFnE8LQ32lteRvt99pA-kaMGCG9PRkGI', "api_secret": '0RD-G7z9LNEmd4WDdd7PJSmq7vQaIuTs',
            "template_base64": f1_64, "template_rectangle": rectangle1,
            "merge_base64": f2_64, "merge_rectangle": rectangle2, "merge_rate": number}
    response = requests.post(url_add, data=data)
    req_con = response.content.decode('utf-8')
    req_dict = json.JSONDecoder().decode(req_con)
    result = req_dict['result']
    imgdata = base64.b64decode(result)
    file = open(image_url_3, 'wb')
    file.write(imgdata)
    file.close()
import os
def test(filename):
    image1 = "pxh.jpg"
    image2 = filename
    image3 = filename
    try:
        merge_face(image2,image1,image3,90)
    except:
        pass
def main():
    # test('./Cache/8.jpg')
    for root,dir,files in os.walk('./Cache'):
        for file in files:
            filename = os.path.join(root,file)
            # print(filename)
            test(filename)

if __name__ == '__main__':
    INPUT="test.mp4"
    FPS=24
    vc=video2txt_jpg(INPUT)
    FPS=vc.get(cv2.CAP_PROP_FPS)  # 获取帧率
    vc.release()
    main()
    jpg2video(INPUT.split('.')[0], FPS)
    print(INPUT, INPUT.split('.')[0] + '.mp3')
    video2mp3(INPUT)
    video_add_mp3(INPUT.split('.')[0] + '.avi', INPUT.split('.')[0] + '.mp3')
```

在视频截图中的最终效果看起来不怎么好，看来应该是面部特征越明显才能融合越好。不过目前是实现了从宋祖儿--->朴信惠换脸术，

![image.png](https://img.hacpai.com/file/2019/09/image-011b7f1c.png)

![image.png](https://img.hacpai.com/file/2019/09/image-79037cc6.png)


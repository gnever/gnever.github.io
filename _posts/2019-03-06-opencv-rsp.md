---
layout: post
title: opencv 同时处理多个rtsp流媒体
categories: opencv
description: opencv 同时处理多个rtsp流媒体
keywords: opencv rtsp
---

## 背景
场景内有7台摄像头，需要对摄像头拍摄范围内人员进行识别然后上报。
单进程单摄像头的处理方式由于图片预测逻辑比较耗时，会导致延迟会比较高，所以需要异步预测。
并且最好还要对多摄像头的管理有较好的机制。随添随用

## 思路
1 使用 multiprocessing 起动多个子进程，每个子进程分配唯一的 rtsp 地址
2 使用 Queue 避免对视频帧处理耗时导致获取到的帧延迟较大的情况

## 看代码
用到的核心库如下

``` python
import cv2
import multiprocessing
from multiprocessing import Process,Queue,Pool
```

逻辑部分伪代码

```python

cameraList = [
{'name':'xxx1', 'rtsp':'rtsp://username:pwd@192.168.5.1/main/Channels/2'},
{'name':'xxx2', 'rtsp':'rtsp://username:pwd@192.168.5.2/main/Channels/2'},
{'name':'xxx3', 'rtsp':'rtsp://username:pwd@192.168.5.3/main/Channels/2'},
....
]
step = 4 #设置每4s抓一次帧

#消费进程
def consumer(q, tmp) :
    while True:
        try:
            #发现堆积严重时释放队列内数据
            if q.qsize() > 100:
                while q.empty() == False:
                    if q.qsize() > 10:
                        q.get(True, 1)

            value = q.get()
            handler(value['index'], value['frame'], value['t'])#将摄像头索引和对应的帧进行处理
        except Exception,e:
            tlog('handler fail ' + e.message)

def readRTSP(q, index) :
    #打散
    time.sleep(index + random.randint(1,len(cameraList)))

    video_info =  cameraList[index]
    rtspUrl = video_info['rtsp']
    cap = cv2.VideoCapture(rtspUrl)
    fps = cap.get(cv2.CAP_PROP_FPS)

    i = 0
    while cap.isOpened():
        try:
            success,frame = cap.read()
            i += 1
            if success == True:
                if i % (fps * step) != 0:
                    cv2.waitKey(1)
                    continue

                i = 0
                q.put_nowait({'index':index, 't':int(time.time()), 'frame':frame})#将该帧推入队列
                cv2.waitKey(1)
                continue
        except:
            tlog('read fail')
    cap.release()

if __name__=='__main__':

    manager = multiprocessing.Manager()
    # 父进程创建Queue，并传给各个子进程：
    q = manager.Queue()
    p = Pool()

    #设置消费者数量
    for i in range(4):
        pr = p.apply_async(consumer,args=(q, 'consumer'))

    for i in range(len(cameraList)):
        pr = p.apply_async(readRTSP,args=(q, i))

    p.close()
    p.join()
```


+++
title = "2024-05-16 护照网上换发攻略"
date = "2024-05-16"
tags = ["misc"]
description = "换发流程和照片处理"
showFullContent = false
+++

2024年5月6日起，20城试点网上护照换发，正好需要更新护照，记录一下网上换发的流程。

## 护照网上换发流程
申请护照换发的网上入口：微信小程序-国家移民管理局政务服务平台-中国公民服务-证件换补发网上申请。

需提前准备好护照照片、签名照片、身份证照片。完成申请后，等待审批，审批通过后支持工本费，寄到家中到付。

## 照片合规处理
护照照片有严格的规格要求：格式为jpeg，20k~80k，宽高比3:4，图像大小(354,472)~(480,640)，以及一些隐式要求，如头部占比需足够大、头顶需有一定留白。

写了一个工具来处理。这个工具同样适用于签名照片和身份证照片，因为这两个照片也有尺寸的要求。

```python
from PIL import Image, ImageOps
import argparse

def Config(parser):
  parser.add_argument('-i', dest='input', default="./in.jpg", help='input photo path')
  parser.add_argument('-o', dest='output', default="./out.jpg", help='input photo path')
  parser.add_argument('--height', dest='height', default=640, help='desired output height')
  parser.add_argument('--width', dest='width', default=480, help='desired output width')
  parser.add_argument('--crop_ratio', dest='crop_ratio',type=float, default=0.8, help='crop_ratio = cropped_size/original_size')
  parser.add_argument('--crop_xoff', dest='crop_xoff', type=float, default=0.5, help='0: left-aligned, 0.5: mid, 1: right-aligned')
  parser.add_argument('--crop_yoff', dest='crop_yoff', type=float, default=0.5, help='0: up-aligned, 0.5: mid, 1: down-aligned')
  return parser.parse_args()

if __name__ == '__main__':
  args = Config(argparse.ArgumentParser())
  with Image.open(args.input) as im:
    width, height = im.size
    cropped_width = args.crop_ratio * width
    cropped_height = args.crop_ratio * height
    xdiff = width - cropped_width
    ydiff = height - cropped_height
    xoff = xdiff * args.crop_xoff
    yoff = ydiff * args.crop_yoff
    box = (xoff, yoff, width - xdiff + xoff, height - ydiff + yoff)
    print("input file {}: original size = ({},{}), cropped size = ({},{}), cropped region={}, desired size = ({},{})".format(args.input, width, height, cropped_width, cropped_height, box, args.width, args.height))
    cropped = im.crop(box)
    desired_size = (args.width, args.height)
    ImageOps.fit(cropped, desired_size).save(args.output)
```

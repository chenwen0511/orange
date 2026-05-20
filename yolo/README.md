# 参考
https://gitee.com/aspartamej/yolov10_om_infer

# 环境
source genpose2

安装 Ultralytics（YOLO 官方库）：

```bash
pip install ultralytics
```


# 权重下载
wget https://github.com/THU-MIG/yolov10/releases/download/v1.1/yolov10m.onnx


# 转换
yolo export model=yolov10m.pt format=onnx opset=11 simplify

参考：export.log

# 推理
yolov10n = YOLOv10('/home/stephen/01-code/Orange/yolo/yolov10m.onnx')


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
wget https://ultralytics.com/images/bus.jpg
python python/yolov10_onnxruntime.py

```
(genpose2) [root@orangepi yolov10_om_infer]# python python/yolov10_onnxruntime.py --iters 200 --warmup 10
onnxruntime cpuid_info warning: Unknown CPU vendor. cpuinfo_vendor value: 15

预处理: 9.996 ms  |  推理 session.run ×200: mean=492.748 ms, min=457.039 ms, max=901.233 ms, std=79.899 ms  |  后处理: 2.899 ms
端到端（单次预处理 + 单次推理 + 单次后处理）: 476.321 ms （推理取最后一次；若要看稳定推理请主要看上一行 mean）
已保存: result.jpg
```
```

(genpose2) [root@orangepi yolov10_om_infer]# python python/yolov10_onnxruntime.py --iters 200 --warmup 10 --npu-device-id 0
onnxruntime cpuid_info warning: Unknown CPU vendor. cpuinfo_vendor value: 15
onnxruntime 1.23.2 | 本机已编译 EP: ['AzureExecutionProvider', 'CPUExecutionProvider'] | 本会话实际使用: ['CPUExecutionProvider']（首选: CPUExecutionProvider）
提示: 当前为 CPU 推理，未走昇腾 NPU。若要用 NPU，需安装带 CANNExecutionProvider 的 ONNX Runtime（如昇腾/社区提供的 onnxruntime-cann 等，且与 CANN 版本匹配），或改用本仓库 OM + ACL 推理链路。


```

# ACL

docker pull swr.cn-north-4.myhuaweicloud.com/toolsmanhehe/pi0:1.0-opiaistation-py310-ubuntu22.04-aarch64

docker run --name pi0 -it -d \
--net=host \
--shm-size=10g \
--privileged=true \
--device=/dev/upgrade \
--device=/dev/davinci_manager \
--device=/dev/davinci0 \
--device=/dev/hisi_hdc \
--entrypoint=bash \
-v /etc/sys_version.conf:/etc/sys_version.conf \
-v /var/dmp_daemon:/var/dmp_daemon \
-v /usr/local/sbin/npu-smi:/usr/local/sbin/npu-smi \
-v /var/slogd:/var/slogd \
-v /usr/local/Ascend/driver/lib64:/usr/local/Ascend/driver/lib64 \
-v /models:/models \
-v /home/drv/:/home/drv/ \
-v /tmp:/tmp \
-v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime \
-v /home/stephen:/home/stephen \
swr.cn-north-4.myhuaweicloud.com/toolsmanhehe/pi0:1.0-opiaistation-py310-ubuntu22.04-aarch64
	
docker exec -it pi0 bash	
	
cd /models/
git clone https://modelers.cn/XLRJ/pi0.git	
	
python ./run_om_e2e.py \
--vlm-model-path /models/pi0/outputs/om/pi0_vlm.om \
--action-expert-model-path /models/pi0/outputs/om/pi0_action_expert_linux_aarch64.om
## Nexus设备升级和降级

- 升级

Step 1:查看当前交换机的版本

show module

show version   

Step 2:下载并上传推荐的NX-OS版本

Step 3:在交换机中校验镜像的MD5值

show file bootflash:<IMAGE-NAME> md5sum    ；例如：show file bootflash:nxos.9.3.10.bin md5sum

Step 4:在实际执行升级之前检查升级软件的影响

show install all impact nxos bootflash:nxos.9.3.10.bin

Step 5:保存配置

copy running-config startup-config

Step 6:升级Cisco NX-OS软件

install all nxos bootflash:nxos.9.3.10.bin

Step 7:验证交换机的版本

show module

show version  



- 降级

Step 1:查看当前交换机的版本

show module

show version   

Step 2:下载并上传推荐的NX-OS版本

Step 3:在交换机中校验镜像的MD5值

show file bootflash:<IMAGE-NAME> md5sum    ；例如：show file bootflash:nxos.9.3.10.bin md5sum

Step 4:检查任何硬件不兼容性

show install all impact nxos bootflash:nxos.9.3.10.bin

Step 5:保存配置

copy running-config startup-config

Step 6:安装Cisco NX-OS软件

install all nxos bootflash:nxos.9.3.10.bin

Step 7:验证交换机的版本

show module

show version  
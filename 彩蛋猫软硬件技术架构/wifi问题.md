一  mqtt
1 mqtt基本设置
  离线消息不重发，消息只发一次；离线自动重连；心跳默认60s，使用内存缓存；每次连接和断开需要清除cleanseesion
 
  本地2分钟内去重消息
2 mqtt服务保活
  内部服务互相启动
3 wifi保活方案
	
自动息屏情况下，设置WIFI永不断开，WIFI不会断开连接，如果息屏期间WIFI断开，不会自动重连上。就算持有WIFI锁，也不会自动连接上。如果主动通知扫描会自动连接上。
如果按电源键，WIFI自动断开，设置息屏永不断开就不起作用。只有持有WIFI锁才会保持连接。但中间WIFI自动断开，不会重连，认为连接没断开。通过定时判断是否连接，通知扫描WIFI，实现自动重连。

终极方案，持有WIFI锁，定时监测是否连接，实现关屏重连。

彩蛋猫一开始设计，闲时一小时就熄灭屏幕，不做特殊处理，会直接导致定时器，闹钟失效，依赖闹钟和定时器的业务全部失效。

影响了mqtt断开，mqtt的默认心跳timer失效。
影响了基于闹钟的数据同步服务。

针对息屏唤醒，加了物理按键唤醒。


设计上改为暗屏方案。

闲时一小时暗屏。

技术上添加了息屏WIFI永不断开，获得CPU锁，没加WIFI锁。mqtt改为自动重连。mqtt心跳改为handlerthread方案。不接受离线消息。


4 心跳保活方案

定时监测订阅事件结果

应用作为系统应用放到privapp目录下。

普通应用如果没启动过是收不到任何通知的。

系统应用可以接收任何通知。如果应用没启动过，从广播启动服务，服务要加白名单。或者一像素启动再启动服务。

5 apk升级

应用升级，脚本启动完成安装。
WIFI升级监测，下载。闲忙时候控制是否下载。

差分升级
rom打包后md5会不一样。
差分升级基础包要使用rom版本。

6 rom升级

7 时间同步问题
  系统每次开机时间都是2001-1-1 ，联网后会自动同步时间。
  实际运用中，时间同步出现时间同步失败或者时间相差几天。
  通过ping接口来同步服务时间，根据年份判断

8 充电
  需要保存充电开始时间。  换电池继续充电。 充电时间长短跟系统时间有关系，需要时间同步。
9 语音问题
  语音授权时间也和系统时间有关系

10 日志问题.

  分不同模块记录不同的日志。
  所有接口请求日志
  按键日志，页面状态日志。
  埋点日志。

  日志1小时上传策略。7天前自动删除。
  MQTT主动拉取日志 

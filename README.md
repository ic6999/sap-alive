【VPN-SAP BTP节点教程】

 https://www.youtube.com/watch?v=PGWiaf48xvU&list=WL&index=1&t=473s
 
老羅叔叔的數字生活指南：

https://www.youtube.com/watch?v=IgzVDiGQFMY

！您为期 90 天的 SAP BTP 试用分为多个间隔。 当前试用期将在 30 天后过期。 如果您在此期间不登录，您的账户将被暂停。如果您定期登录，我们会自动延长间隔。

！老罗：90天后用同一邮箱账号重新拉取老罗镜像，可再续90天。

@资源：

SAP官网：https://www.sap.com
临时邮箱：https://mail.cx/zh/
短信接码平台：https://lubansms.com
Cloud Foundry CLi 命令行客户端：https://github.com/cloudfoundry/cli/r...
UUID在线生成网站：https://tooltool.net/zh/uuid
BTP一键启动脚本：https://github.com/xiaolin-007/clash/...  

一．注册账号

选择新加坡更快

二．安装部署

①下载安装Cloud Foundry CLi 命令行客户端：[https://github.com/cloudfoundry/cli/r...](https://github.com/cloudfoundry/cli)

②打开macOS-终端/windows-powershell，以下均通过终端操作，结果会同步显示到BTP主控台。

1.登录SAP BTP主机

美国：

cf login -a https://api.cf.us10-001.hana.ondemand.com

新加坡：

cf login -a https://api.cf.us10-001.hana.ondemand.com

2. 拉取镜像（老罗提供）：
3. 
把 APP_NAME 换成你的应用名

cf push aaa --docker-image ghcr.io/uncleluogithub/mous:latest -m 512M --health-check-type port

3.设置 UUID

cf set-env iSAP UUID uuid

4.固化配置并重建

cf restage aaa

5) 获取二维码/链接
cf logs aaa --recent

复制节点：


【多环境多节点扩展】

-在全局帐号fde1f680trial下添加原默认帐号trail下的子帐号如t2（t2_t2-mcekz8t8），确认开启所有权限和空间配额。

-命令行以另一区域登录（如trail为新加坡，t2则为美国），输入主帐号同样的邮箱密码，默认分配名为dev的空间。

-用不同App名如s3拉取镜像，生成应用如t2/dev/s3。后续如第一个节点操作即可。

-每个子账号可继续添加应用。如trail/d2/s2(同区域多节点意义不大）

【永久自动保活教程】

⚠️实测BTP下线北京时间SG8:03/US8:09

⚠️workflow延迟≈20分钟
（标签页实时显示状态：红色叉号 (X)/失败 (Failure)，工作流运行失败，例如测试未通过、构建错误、部署脚本执行失败等。✔️ 绿色对号成功 。🔵 灰色圆点 (•) 或 ⬤ 橙色圆点，工作流正在排队等待分配运行器，或正在执行中。◼️ 紫色方块已跳过，工作流由于某些条件未满足（例如，触发路径未匹配）而未运行。⚪ 灰色中断图标，已取消 工作流被用户手动取消。❓ 灰色问号，无状态/未运行，工作流最近没有运行记录，或者尚未被触发。）

✅方案一：自建多环境多节点自动重启工作流：

https://github.com/ic6999/sap-alive/actions/workflows/a.yml

方案二：（老罗原创）

1. 在 GitHub 仓库中新建私有库：sap-alive
-新建.github/Workflows/main.yml，复制脚本：
https://github.com/ic6999/sap-alive/blob/main/.github/workflows/main.yml
或：全量混合拉起（北京时间上午8-9点间每5/10/15/20分钟各拉起一次，外加全天每30分钟一次）
https://gist.github.com/uncleluogithub/b19a2ad2e79625351a5b4b907e00ccc5
（Cron 语法字段顺序为：分 时 日 月 周，如*/15 0 * * *：每日UTC 时间 0-1点每15分钟自动更新。免费的Action执行时间并不准时，最多可达30分钟。而BTP通常每天8:05左右下线，可设定8:15和8:50各重启一次）
2. 添加以下 6个Secrets：
-电脑终端运行命令 cf target 和 cf apps 获取参数值：
-在项目settings-secrets and variables-action:分别添加：
CF_API=登录网址
CF_USERNAME=你的邮箱
CF_PASSWORD=你的密码
CF_ORG=org
CF_SPACE=空间名称
CF_APP=项目名称
3. 测试：主控页停止BTP，github手动run workflow，若BTP重新启动即成功。（必须手动运行一次才能激活自动化工作流！
https://github.com/ic6999/sap-alive/actions/workflows/main.yml



【HF保活脚本】

huggingface是全球最大的AI模型托管平台厂商。Hugging Face 的 Spaces 平台为用户提供了一个免费的 CPU 空间，默认是 2 个 vCPU 和 16GB 内存，并且提供 50GB 的临时磁盘空间。但超48小时未活跃即进入休眠状态，重启后必须重新部署。此代码极简，只有一个步骤，只执行一个 curl 命令，用于访问Space首页，是最简单的"保持活跃"的信号。Hugging Face 会将该访问识别为一次有效活动，从而重置Space的休眠倒计时。

部署脚本：
bash <(curl -l -s https://raw.githubusercontent.com/zzzhhh1/free-vps-py/refs/heads/main/test.sh)


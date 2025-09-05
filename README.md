VPN-SAP节点教程
 
（免费90天，每30天重新登录）

SAP官网：https://www.sap.com
临时邮箱：https://mail.cx/zh/
短信接码平台：https://lubansms.com
Cloud Foundry CLi 命令行客户端：https://github.com/cloudfoundry/cli/r...
UUID在线生成网站：https://tooltool.net/zh/uuid
BTP一键启动脚本：https://github.com/xiaolin-007/clash/...  



1.登录美国
cf login -a https://api.cf.us10-001.hana.ondemand...

登录新加坡
cf login -a https://api.cf.ap21.hana.ondemand.com

2.把 APP_NAME 换成你的应用名
cf push iSAP --docker-image ghcr.io/uncleluogithub/mous:latest -m 512M --health-check-type port

3.设置 UUID
cf set-env iSAP UUID 8f0751c5-d28f-4d6f-922a-bbbae1696d6a

4.固化配置并重建
cf restage iSAP

5) 获取二维码/链接
cf logs iSAP --recent

vmess://ewogICJ2IjogIjIiLAogICJwcyI6ICLmsrnnrqHpopHpgZPvvJrogIHnvZflj5Tlj5TvvZxTQVAtVlBO55u06L+e54mIIiwKICAiYWRkIjogImlTQVAuY2ZhcHBzLmFwMjEuaGFuYS5vbmRlbWFuZC5jb20iLAogICJwb3J0IjogIjQ0MyIsCiAgImlkIjogIjhmMDc1MWM1LWQyOGYtNGQ2Zi05MjJhLWJiYmFlMTY5NmQ2YSIsCiAgImFpZCI6ICIwIiwKICAic2N5IjogImF1dG8iLAogICJuZXQiOiAid3MiLAogICJ0eXBlIjogIm5vbmUiLAogICJob3N0IjogImlTQVAuY2ZhcHBzLmFwMjEuaGFuYS5vbmRlbWFuZC5jb20iLAogICJwYXRoIjogIi9sYW9sdW8iLAogICJ0bHMiOiAidGxzIgp9


免费 SAP Cloud Foundry VPN APP保活全攻略｜自动拉起 & 手机一键启动 & 常见问题解答

Uncle LUO老羅叔叔的數字生活指南


🛠️ 实操步骤（简要）
 1. 准备油管小白三宝之一：GitHub账号一枚
 2. Docker 镜像：拉取并配置 SAP CF VPN APP
 3. GitHub Actions：新建 Workflow，写入保活脚本
 • 运行命令 cf target 和 cf apps 确认服务
 • 每日定时拉起，保持 VPN 在线
 4. 一键运行：
 • 在 GitHub Action 页面点击 Run workflow
 • 或用 GitHub 手机 APP 一键执行脚本

✅手机GitHub APP一键拉起步骤：进入📱上的GitHub App主页----》找到最下方一行的“个人资料”点击---〉点击“仓库”---》点击“你刚建立的仓库名（例如：SAP-Keepalive）---〉点击”操作“按钮---》点击SAP CF APP保活---〉点击蓝色字”运行工作流“就可以在手机GitHub App上一键拉取了。

📌 保活代码（以下保活拉起action代码大家自我测试，选择其中一个即可，因为免费的Action有时候时间并不准时，请修改为最适配自己的脚本）

1. 北京时间上午8-9点间每5分钟各拉起一次
https://gist.github.com/uncleluogithu...

2. 北京时间上午8-9点间每10分钟各拉起一次
https://gist.github.com/uncleluogithu...

3. 北京时间上午8-9点间每15分钟各拉起一次
https://gist.github.com/uncleluogithu...

4. 北京时间上午8-9点间每20分钟各拉起一次
https://gist.github.com/uncleluogithu...

5. 全量混合拉起（北京时间上午8-9点间每5/10/15/20分钟各拉起一次，外加全天每30分钟一次）
https://gist.github.com/uncleluogithu...



📌 保活代码使用方法：
 1. 在 GitHub 仓库中新建 .github/workflows/keepalive.yml
 2.在终端分别键入：cf target,cf apps列出参数： 
添加以下 6个Secrets：CF_API、CF_USERNAME、CF_PASSWORD、CF_ORG、CF_SPACE、CF_APP

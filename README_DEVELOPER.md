# 🛠️ 开发者部署：界面、网关与机器人

[项目首页](README.md) · [配置地图](docs/CONFIGURATION.md) · [AI部署手册](README_AI.md)

## 1 · 配套后端

先部署匹配版本的[QQHarnessBot](https://github.com/YZzy2006/QQHarnessBot)，完成DSH/QQ接入和测试白名单。业务逻辑、登录认证、状态、权限、日记生成和操作执行位于机器人进程；本仓库`src/workbench-*.js`是对应模块源码参考，不是独立完整机器人。

本仓库的`server.mjs`只是一个无外部依赖的Node API网关，不服务HTML、不运行模型、不复制聊天状态。不要再启动第二份桥接。配套版本见[compatibility.json](compatibility.json)。

## 2 · 本机账号与Origin

在**机器人目录**创建账号：

```bash
python3 scripts/init-workbench-account.py
```

该入口安全读取密码并经stdin交给Node，只保存加盐哈希。然后在机器人`config.json`合并：

```json
{"workbench":{"origin":"https://workbench.example.com"}}
```

`example.com`是占位域名，改为你自己的HTTPS域名。登录会话是独立HttpOnly Cookie，写操作要求CSRF和同源；不能将原控制台Token传给浏览器。可选独立MySQL/头像OSS见[认证说明](docs/WORKBENCH-AUTH.md)。

## 3 · 安装并启动网关

```bash
git clone https://github.com/YZzy2006/QQHarnessWorkbench.git
cd QQHarnessWorkbench
npm ci
npm test
PORT=3188 npm start
```

网关监听`127.0.0.1:3188`，连接同机机器人`127.0.0.1:3100`；CLI当前上游端口固定3100，自定义上游需调整入口并测试，不能凭空使用不存在的环境变量。Windows PowerShell使用`$env:PORT='3188'; npm start`。

```bash
curl --fail http://127.0.0.1:3188/healthz
```

探活仅确认机器人匿名工作台接口返回预期401；不等于登录、QQ收发或所有业务已验收。

## 4 · 静态站点与HTTPS

将`public/`复制到你自己的站点目录，例如`/var/www/qq-workbench/`。Nginx使用[示例配置](deploy/nginx.conf.example)，自行准备真实域名和证书路径。静态页面从`workbench.html`进入；API只反代`/api/workbench/`到3188，Host为回环网关地址；不要反代整个旧控制台`/api/`。

应用前先备份站点配置、运行`nginx -t`，成功后平滑重载。生产Cookie为Secure，不能在随意HTTP公网地址反复测试登录。

宝塔Node项目可选择本仓库目录、Node≥22.13、启动命令`npm start`、端口3188、普通运行用户。其角色仍是网关；域名站点继续服务静态文件，不能重复创建同域名站点或改动已有博客。示例[systemd网关服务](deploy/qq-workbench.service.example)是不用宝塔时的替代选择，不要两种方式同时启动。

## 5 · 配置与开发

聊天模型在DSH及机器人`dsh`字段；天气、地图、语音、图像等配置在机器人环境与设置，**不是网关的环境**。详见[配置地图](docs/CONFIGURATION.md)。

前端使用原生HTML/CSS/ES Modules，无构建步骤。木框、暖纸、像素边框和本地OFL字体均随源码提供，不依赖外部字体CDN。服务器继续决定权限与真实数据，不用前端隐藏代替隔离。

静态改动无需重启QQ/DSH/机器人；JS/CSS改动更新HTML资源版本。测试用隔离后端，禁止用真实管理员账号、聊天记录或付费模型补测试夹具。

## 6 · 验证与发布

```bash
npm test
npm run check:release
```

检查未登录401、Origin/CSRF、登录退出、范围切换、空记录/错误、手机宽度与滚动保持。接口成功不代表真实QQ客户端人工验收。需要业务集成回归时在配套QQHarnessBot仓库运行`npm run test:workbench`。

发布使用明确的`release-files.json`，`node scripts/stage-release.mjs`先扫描再暂存；账号、state、`.env`、数据库/OSS配置、用户头像和日志永不入库。升级不覆盖机器人状态，回滚静态文件和网关代码即可，不还原旧聊天state。


## 逐步验收与失败定位

| 检查点 | 命令/动作 | 预期结果 | 不符时先检查 |
| --- | --- | --- | --- |
| Node版本 | `node --version` | ≥22.13 | 使用服务实际Node路径，不只看交互终端 |
| 安装完整 | `npm ci` | 按锁文件安装成功 | 网络/registry/原生依赖，不手改lock绕过失败 |
| DSH可独立工作 | 用自己的DSH完成测试 | 模型与工具目录正常 | provider/model是否真实存在、凭据与上游兼容性 |
| QQ网关 | 查已有网关在线状态 | 自己的账号在线，OneBot地址可达 | 登录态、端口、令牌；不要反复重建容器 |
| 机器人 | `npm start`查看脱敏状态 | DSH就绪、QQ在线、控制台启动 | 本机配置、白名单和进程环境 |
| 账号初始化 | `python3 scripts/init-workbench-account.py`（机器人目录） | 只保存加盐哈希，密码不出现在命令行 | Python/Node可执行路径和state目录写权限 |
| 网关探活 | `curl --fail http://127.0.0.1:3188/healthz` | `ok:true`、`backend:ready` | 机器人3100是否就绪，端口/Host是否正确 |
| 未登录保护 | `curl -i https://workbench.example.com/api/workbench/session` | HTTP401 | Nginx是否错反代旧控制台或命中缓存 |
| 静态界面 | 从自己的HTTPS域名打开 | 页面/JS/CSS/本地字体正常 | root是否指向public、资源路径、证书与HTML版本 |
| 登录失败 | 用自己初始化的工作台账号登录 | Cookie登录会话建立 | 独立账号不是QQ/DSH账号；数据库模式是否已明确启用 |
| 写操作403 | 读取会话后按界面操作 | 合法Origin、Cookie、CSRF | `workbench.origin`协议/域名/端口、反代Host与浏览器同源 |
| 退出/改密 | 退出或修改自己的密码 | 原会话失效，需重登 | 是否缓存接口或混用了其他项目Cookie |
| 天气/定位 | 授权测试会话发一次查询 | 有实际来源，缺配置给明确错误 | 专属天气Host、地图服务端Key、供应商权限 |
| 语音/视频 | 在授权会话测试实际媒体 | 成功处理或明确缺失项 | ffmpeg/ffprobe、模型权限、格式、额度与超时 |

`example.com`须替换为自己的域名。上述模型/QQ/媒体动作会产生真实费用或投递，只有获得授权才执行。先通过离线测试和匿名保护检查，不用真实用户聊天作测试数据。

### 数据备份与升级清单

升级前只在本机私有位置备份`config.json`、`.env`、`state/`和自己的人物卡；数据库用户、OSS配置、聊天/日记、上传图片及B站登录态都不入库。逐层识别本次是前端、网关还是机器人改动。不要为了发布源码重新登录QQ、清空记忆或将测试state迁移生产。

### 给AI的完成判据

必须报告“检查了什么、得到什么、没做什么”，分别记录离线回归、匿名401、登录/CSRF、实际QQ投递与供应商测试。无法访问真实账号时明确未验收；不能用接口成功补写“客户端可见”，也不能为了凑通过率跳过失败用例、打印凭据或静默改权限。


### 服务名与部署路径

服务器状态页默认查询`qq-bridge.service`与`dsh-web.service`。机器人服务示例应安装为`qq-bridge.service`，或按你的实际服务名调整状态适配器；服务未知不代表QQ账号已离线。示例路径`/opt/QQHarnessBot`、`/opt/QQHarnessWorkbench`及运行用户都是部署占位，先按自己的目录与权限修改。新建普通服务用户前检查是否已有适合的用户，不重建正在工作的实例。

Python账号初始化工具是可选便利入口；如果机器没有Python3，使用同目录Node账号CLI的安全stdin接口，不把密码加在命令行。源码/主题不包含任何默认可登录密码。


### 单仓与分仓部署

QQHarnessBot与QQHarnessWorkbench均为公开开源仓库，可直接克隆。使用单仓部署时，QQHarnessBot主仓库也包含匹配的界面和网关：在主仓库的`deploy/botwork`目录运行`npm ci && npm start`，Nginx静态root指向主仓库`public/`，首页仍是`workbench.html`。不要为了下载源码索要或共享维护者的GitHub令牌。


### Linux可选媒体依赖与中文结果卡片

Debian/Ubuntu启用实际视频处理和服务端中文PNG结果卡片时，可按需安装：

```bash
sudo apt-get update
sudo apt-get install -y ffmpeg fonts-noto-cjk fontconfig python3
```

其他发行版使用自己的包管理器。视频功能需ffmpeg/ffprobe；后台安全媒体缓存检查与对应Linux回归使用Python3。中文结果卡片需要完整中文字体，可在本机环境设置`QQ_CARD_FONT`指向已授权TTF/TTC字体；网页自带的OFL像素字集仅用于UI，不等于服务端中文字体。缺字体时业务保留文字结果，不会假装PNG已生成。

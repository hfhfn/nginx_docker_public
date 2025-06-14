### 完整nginx配置文件目录

```bash
nginx/
├── conf/                   # 主配置目录（映射到容器内/etc/nginx）
│   ├── nginx.conf          # 主配置文件（入口配置）
│   ├── mime.types          # MIME类型定义文件（官方推荐放这里）
│   ├── conf.d/             # 基础配置片段
│   │   ├── 00-base.conf    # 基础参数（客户端超时/缓冲区等）
│   │   ├── 10-proxy.conf   # 反向代理通用设置
│   │   └── 20-errors.conf  # 统一错误页面配置
│   ├── services/           # 业务服务配置（按功能拆分）
│   │   ├── tts.conf        # 语音服务配置
│   │   └── excel.conf      # Excel服务配置
│   │   └── ...
│   ├── sites-enabled/      # 实际启用的域名配置
│   │   └── ssl.conf        # HTTPS专用配置
│   │   └── dify.hfhfn.xyz.conf  # dify域名代理配置
│   │   └── ...
│   └── templates/          # 动态生成模板
│       ├── location.template  # 服务location块模板
│       └── upstream.template  # 负载均衡模板
│       └── domain.template  # 域名代理模板
├── logs/                   # 日志目录（映射到容器内/var/log/nginx）
│   ├── access.log          # 访问日志
│   └── error.log           # 错误日志
└── html/                   # 静态资源根目录（映射到容器内/usr/share/nginx/html） 暂时没有配在这里     
│   ├── index.html          # 默认首页
│   └── static/             # 静态资源
│       ├── css/
│       └── js/
└── sites-available/  		# location服务的配置文件
└── docker-compose.yaml
└── certs/
```



#### 配置加载顺序

1. `mime.types` （最先加载）
2. `conf.d/00-base.conf` （基础参数）
3. `conf.d/10-proxy.conf` （代理配置）
4. `services/*.conf` （业务服务）
5. `sites-enabled/*.conf` （域名配置）



#### openssl生成证书

```bash
# 通过openssl生成私钥
openssl genrsa -out server.key 2048
# 根据私钥生成证书申请文件csr, 看着填就行了，不想填的可以不填
# Common Name可以输入：*.yourdomain.com，这种方式生成通配符域名证书
openssl req -new -key server.key -out server.csr
# 使用私钥对证书申请进行签名从而生成证书, 这样就生成了有效期为：10年的证书文件，对于自己内网服务使用足够。
openssl x509 -req -in server.csr -out server.crt -signkey server.key -days 3650
```



#### 服务配置采用符号链接，方便启用/禁用站点配置

```powershell
# PowerShell 创建符号链接（需管理员权限）
New-Item -ItemType SymbolicLink -Path "conf\sites-enabled\nginx.hfhfn.xyz.conf" -Target "..\sites-available\nginx.hfhfn.xyz.conf"

sites-available/   # 存放所有可用配置（不自动加载）
sites-enabled/     # 只存放需要启用的配置（通过符号链接指向available）
```



#### 使用envsubst动态生成服务配置

- **Git Bash**	

1. 安装 Git for Windows
2. 在 Git Bash 中直接使用 envsubst 命令

```bash
# 示例：生成Excel服务配置
export SERVICE_PATH="/excel/"
export UPSTREAM_SERVER="http://excel_service:9000"
export SERVICE_NAME="excel"
envsubst < templates/service.template > services/excel.conf
```



- #### **PowerShell 替代方案**

```powershell
# 读取模板并替换变量
(Get-Content .\templates\service.template) `
    -replace '\${SERVICE_PATH}', '/excel/' `
    -replace '\${UPSTREAM_SERVER}', 'http://excel:9000' `
    | Out-File -Encoding utf8 .\services\excel.conf
```


#### 查看nginx所有配置
```bash
docker exec nginx-server nginx -T > nginx_all.conf
```
#### 查看nginx错误日志
```bash
docker exec nginx-server tail -f /var/log/nginx/error.log
```

#### services文件夹中增加了upstream_server文件夹，存放单服务的负载均衡配置，暂未使用，而是采用了location.conf统一配置本地服务。
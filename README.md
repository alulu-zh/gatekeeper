# <img src="https://raw.githubusercontent.com/didi/gatekeeper/master/tmpl/static/images/logo.png"/> GateKeeper [![Build Status](https://www.travis-ci.org/didi/gatekeeper.svg?branch=master)](https://www.travis-ci.org/didi/gatekeeper) [![Go Report Card](https://goreportcard.com/badge/github.com/didi/gatekeeper)](https://goreportcard.com/report/github.com/didi/gatekeeper) [![Hex.pm](https://img.shields.io/hexpm/l/plug.svg)](https://github.com/didi/gatekeeper/blob/master/LICENSE) [![GoDoc](https://godoc.org/github.com/didi/gatekeeper?status.svg)](https://godoc.org/github.com/didi/gatekeeper)


GateKeeper 是一个 Go 编写的不依赖分布式数据库的 API 网关，使用它可以高效进行服务代理，支持在线化热更新服务配置 以及 纯文件方式服务配置，支持主动探测方式自动剔除故障节点以及手动方式关闭下游节点流量，还可以通过自定义中间件方式灵活拓展其他功能。


- [特性](#%E7%89%B9%E6%80%A7)
- [内容](#%E5%86%85%E5%AE%B9)
    - [快速开始](#%E5%BF%AB%E9%80%9F%E5%BC%80%E5%A7%8B)
    - [并发压测](#%E5%B9%B6%E5%8F%91%E5%8E%8B%E6%B5%8B)
    - [使用手册](#%E4%BD%BF%E7%94%A8%E6%89%8B%E5%86%8C)
      - [在线管理接入](#%E5%9C%A8%E7%BA%BF%E7%AE%A1%E7%90%86%E6%8E%A5%E5%85%A5)
      - [配置文件接入](#%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E6%8E%A5%E5%85%A5)
      - [功能拓展](#%E5%8A%9F%E8%83%BD%E6%8B%93%E5%B1%95)
- [测试](#%E6%B5%8B%E8%AF%95)
- [License](#license)

## 特性
- `http`、`websocket`、`tcp`服务代理
- 自动剔除故障节点
- 手动关闭下游节点流量
- 加权负载轮询
- `URL`地址重写
- 服务限流：支持独立`IP`限流
- 高拓展性：支持自定义 `请求前验证request方法`、`请求后更改response方法`、`tcp中间件`、`http中间件` 等。
- 最少依赖：无需任何额外组件即可运行，`mysql`、`redis` 只做在线管理和统计使用可随时关闭。

## 内容
### 快速开始
---
安装`GateKeeper`之前，需要安装`Go`环境 (`golang`版本>=`1.11`), 如果需要界面管理则需要 `mysql`、`redis`支持。

1. clone 代码到本地

```
git clone git@github.com:didichuxing/gatekeeper.git
```

2. 开启 `go mod` 支持及代理支持

```
export GO111MODULE=on
export GOPROXY=https://goproxy.cn
```

3. 创建 `db` 并导入数据

如果不使用在线服务接入以及统计功能，可以跳过本步。
默认使用：`gatekeeper` 作为数据库名
```
mysql -h localhost -u root -p -e "CREATE DATABASE gatekeeper DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;"
mysql -h localhost -u root -p gatekeeper < install/db.sql --default-character-set=utf8
```

4. 调整 `mysql`、`redis` 配置文件

修改 `./conf/dev/mysql.toml` 和 `./conf/dev/redis.toml` 为自己的环境配置。

如果不使用在线服务接入，删除 `./conf/dev/mysql.toml` 和 `./conf/dev/redis.toml` 即可。

5. 运行代码

```
go run main.go
```
6. 登陆管理后台

`http://127.0.0.1:8081/admin/login`

默认账号密码: `admin` / `123456`

### 并发压测

压测条件：
- 硬件: `Xeon(R) CPU E5-2670 48` `128G`
- 软件：`centos 6.5`、`wrk`、下游服务`golang` 服务器无逻辑
- 压测命令：./wrk -t并发数 -c连接数 -d30s url

并发量 | 链接数 | 压测时间 | QPS | 平均相应 | CPU % | MEM |
---|---|---|---|---|---|---
100 | 1000 | 30s | 26420.73	 | 36.35ms	 | 2560.1 | 417m |
200 | 1000 | 30s	 | 27972.51	 | 35.45ms	 | 2541 | 417m |
200 | 2000 | 30s	 | 29871.59	 | 65.96ms	 | 2630.5	 | 358m |
200 | 3000 | 30s	 | 29626.71	 | 105.81ms		 | 2703.1	 | 385m |
200 | 4000 | 30s	 | 30049.34		 | 131.92ms	 | 2710.9	 | 410m |
200 | 5000 | 30s	 | 30650.13		 | 161.50ms	 | 2808.0	 | 417m |
200 | 10000 | 30s	 | 29697.14		 | 170.28ms	 | 2780.3	 | 476m |
300 | 1000 | 30s	 | 27407.33		 | 33.10ms	 | 2528.6 | 417m |
400 | 1000 | 30s	 | 26790.57		 | 30.19ms	 | 2401.3 | 417m |
500 | 1000 | 30s	 | 27812.94		 | 38.94ms	 | 2476.1 | 417m |


### 使用手册

- 集群配置及部署

    集群架构如图
    ![image](http://img-hxy021.didistatic.com/static/itstool_public/do1_CMRHhgPPJCHI4xNcGdmD)
        
    - 结合架构图对每个步骤说明如下：
        1. 用户通过接入层连接到 `GateKeeper` 实例中。
        2. 每个 `GateKeeper` 实例，针对每个服务模块，单独进行服务探测。
        3. 在线服务管理时，配置数据先保存到 `GateKeeper` 配置 `DB` 中，然后再通过调用配置更新接口（ `/reload` ），更新所有实例机器配置。
       
    - 接入层一般选用 `nginx`、`Haproxy`、`LVS`等
    
    以`nginx`作为接入层为例，可根据网络需求确认是否暴露管理地址(`/admin`)。        
  ```
        upstream gatekeeper { 
              server 10.90.80.16:8081; 
              server 10.90.80.17:8081; 
        }
        server {
            listen       8007;
            root         /home/webroot/official-website-api/;
            location ^~ /gatekeeper{
                    proxy_pass http://gatekeeper;
                    proxy_set_header Host $host;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            }
        }
  ```

    - 配置集群
        
        通过修改 `./conf/dev/base.toml` 中的 `cluster` 节点完成集群配置。
        
        - `cluster_ip`
        
        表示实际对外的访问地址，如果是域名填写域名。
        
        - `cluster_addr`
        
        `http` 服务需要监听的端口
        
        - `cluster_list`
        
        集群子机器 `ip` ,多 `ip` 以逗号分隔
        
    - 集群配置同步
        
        通过访问集群子机器的 `/reload` 接口确保所有代理机器配置统一。


#### 在线管理接入

- 管理登陆

    登陆账号密码在 `./conf/dev/admin.toml` 中配置，使用以下地址登陆。
    
    > http://127.0.0.1:8081/admin/login

- 服务管理
    
    - 服务列表
    ![image](http://img-hxy021.didistatic.com/static/itstool_public/do1_MkkNkz67B5yw9uJRmDXZ)
        - 服务地址，如：`http://10.90.164.31:8081/gatekeeper/test_http`
        
        这里的 `10.90.164.31:8081` 表示集群地址，可以在 `./conf/dev/base.toml` 中设置
        
        - `QPS` 集群当前`QPS`
        - `QPD` 集群当天总流量
        - `NODE` 当前可用节点数/总节点数
        
    - 新增、修改 `http`服务
        - 访问前缀设置，如：`/gatekeeper/test_http`
        
        这里的 `/gatekeeper` 是整个http对外路由的公共前缀。你可以在 `./conf/dev/base.toml`中更改。
        - 探活地址，如：`/ping`，`url`重写对该地址无影响
        
        表示需要探测的目标服务器除去主机信息后的地址。若目标主机为: `10.90.164.31:8072`，则真实地址为：`http://10.90.164.31:8072/ping` ，需要保证该地址访问可以正常返回200状态。
        - 重写规则：如：`^/gatekeeper/test_http(.*) $1`
        
        如果访问网关的地址是：`http://127.0.0.1:8081/gatekeeper/test_http/ping`, 则访目标的地址是： `http://10.90.164.31:8072/ping`, 如果不重写则访问目标的地址是：  `http://10.90.164.31:8072/gatekeeper/test_http/ping`, 创建时如未填写则自动填写 `^访问前缀(.*) $1`
        
        - 客户端`IP`限流
        
        表示单台网关实例当前服务最大允许`QPS`，0表示不限流。
    
    - 添加`tcp`服务：功能设置同`http`
    
    - 流量控制：可在针对某台目标机器进行流量关闭

- 租户管理
    
    - 租户列表
    ![image](http://img-hxy021.didistatic.com/static/itstool_public/do1_yKCclTNndRCA3awqly2L)
    
        - 使用租户信息访问下游服务
        
        基于 `app_id` 和 `secret` 可以计算出签名 `sign` (为简化操作，我们这里直接使用 `secret` 作为了签名)，然后直接使用get参数传入就可以访问下游服务了。 如：
        `http://127.0.0.1:8081/gatekeeper/test_http/ping?app_id=test_app&sign=62fda0f2212eaffd90dbf04136768c5f`
        
        - 租户鉴权
        
        参考下文中的 定义请求前验证 `request` 方法
        `service.AuthAppToken`

        - 新增、修改租户
        
        接口列表，如：`/gatekeeper/test_http`。表示通过对应租户`id`，所有能请求到的接口的前缀。这样就可以起到限制某租户访问对应服务的功能了。目前租户只对`http`接口起作用。
        - 日请求总量
        
        表示目前租户，最多允许请求的总量
        
        - `Qps`限流
        
        表示当前租户最大允许`QPS`，0表示无限制
        
#### 配置文件接入
- 纯配置文件接入

    首先确保配置文件中不存在 `mysql_map.toml`、`redis_map.toml`, 否则配置会被重启覆盖。
    
    - 服务配置文件
    
    参照以下`demo`，编辑 `./conf/dev/module.toml`
    ```
    # http服务示例，时间单位ms
    [[module]]
      [module.base]
        load_type = "http" #服务类型
        name = "test_http" #服务标识
        service_name = "test_http" #服务名称
    
      [[module.match_rule]]
        type = "url_prefix" #匹配类型
        rule = "/gatekeeper/test_http" #访问前缀
        url_rewrite = "^/gatekeeper/test_http(.*) $1" #重写规则
        
      [module.load_balance]
        check_method = "httpchk" #探测类型
        check_url = "/ping" #探测地址
        check_timeout = 2000 #探测超时时间
        check_interval = 5000 #探测频率
        type = "round-robin" #轮询类型
        ip_list = "10.90.164.31:8072,10.90.163.51:8072,10.90.163.52:8072,10.90.165.32:8072" #目标服务ip
        weight_list = "50,50,50,80" #目标权重
        forbid_list = "10.90.165.32:8072" #禁用目标ip
        proxy_connect_timeout = 10001 #连接目标服务器超时时间
        proxy_header_timeout = 10002 #获取header头超时
        max_idle_conn = 200 #连接最大空闲时间
        idle_conn_timeout = 10004 #最大空闲连接数
        
      [module.access_control]
        black_list = "" #ip黑名单
        white_list = "" #ip白名单
        white_host_name = "" #host白名单
        client_flow_limit = 0 #客户端IP限流
        open = 1 #访问权限（黑名单与白名单）控制是否打开 1为打开 0为关闭
    
    #tcp配置示例，时间单位ms
    [[module]]
      [module.base]
        load_type = "tcp" #服务类型
        name = "test_tcp" #服务标识
        service_name = "test_tcp" #服务名称
        frontend_addr = ":8900" #监听端口
    
      [module.load_balance]
        check_method = "tcpchk" #探测类型
        check_timeout = 2000 #探测超时时间
        check_interval = 5000 #探测频率
        type = "round-robin" #负载类型
        ip_list = "127.0.0.1:8018" #目标ip列表
        weight_list = "50" #目标权重列表
        forbid_list = "" #禁用ip列表
        proxy_connect_timeout = 10001 #连接超时时间
        
      [module.access_control]
        black_list = "" #黑名单
        white_list = "" #白名单
        white_host_name = "" #host白名单
        client_flow_limit = 0 #客户端ip限流
        open = 1 #访问权限（黑名单与白名单）控制是否打开 1为打开 0为关闭
    ```
    
    - 热加载服务配置
    
    ```
    sh ./reload.sh 8081
    ```
    
#### 功能拓展
- 定义请求前验证 `request` 方法 比如：租户权限验证方法
```
//AuthAppToken app的签名校验
func AuthAppToken(m *dao.GatewayModule, req *http.Request, res http.ResponseWriter) (bool,error) {
	ctx:=public.NewContext(res,req)
	//验证签名
	if err:=AuthAppSign(ctx);err!=nil {
		return false,err
	}
	//限速等操作
	if err := AfterAuthLimit(ctx); err != nil {
		return false,err
	}
	//todo 可以在这里加入sso跳转逻辑
	//ctx.Redirect("/sso/login",301)
	return true,nil
}
```
在 `router.HttpServerRun()` 运行之前调用注册函数

```
service.RegisterBeforeRequestAuthFunc(service.AuthAppToken)
```

- 定义请求后修改 `response` 方法 比如：过滤返回中的城市数据函数

```
//定义数据修改方法
func FilterCityData(filterURLs []string) func(m *dao.GatewayModule, req *http.Request, res *http.Response) error{
	return func(m *dao.GatewayModule, req *http.Request, res *http.Response) error {
		//获取原始请求地址
		v:=req.Context().Value("request_url")
		requestURL,ok := v.(string)
		if !ok{
			requestURL = req.URL.Path
		}

		//获取请求内容
		payload, err := ioutil.ReadAll(res.Body)
		if err!=nil{
			return err
		}

		//验证是否匹配
		for _,matchURL:=range filterURLs{
			if matchURL==requestURL {
				//过滤规则
				filterData, err := filterJsonTreeByKey(string(payload),"data.list", "city_id", []string{"12"},)
				if err!=nil{
					return err
				}
				payload = []byte(filterData)
				break
			}
		}

		//重写请求内容
		res.Body = ioutil.NopCloser(bytes.NewBuffer(payload))
		res.ContentLength = int64(len(payload))
		res.Header.Set("Content-Length", strconv.FormatInt(int64(len(payload)), 10))
		return nil
	}
}
```
在 `router.HttpServerRun()` 运行之前调用注册函数

```
//注册内容更改函数
service.RegisterModifyResponseFunc(service.FilterCityData([]string{"/gatekeeper/tester_filter/goods_list"}))
```

- 注册http中间件

    参照： `./middleware/http_limit.go`

- 注册tcp中间件

    参照： `./middleware/tcp_limit.go`


## 测试
测试套件基于`goconvey`，所以需要确认安装了`goconvey`
- 安装 `goconvey`

```
go get github.com/smartystreets/goconvey
```
- 运行测试用例

```
cd tester
sh bootstrap.sh
```

License
------------
`gatekeeper` is licensed under [Apache License](LICENSE).

非常感谢以下项目对开源做出的贡献：

- [tcpproxy](https://github.com/google/tcpproxy)
- [AdminLTE](https://github.com/ColorlibHQ/AdminLTE)
- [golang_common](https://github.com/e421083458/golang_common)

[nginx.conf 配置及基本优化](#nginxconfigoptimize)


``` bash
# nginx 重启
sudo /etc/init.d/nginx restart
```

# 一、主配置段

```
1、正常运行必备的配置
#运行用户和组，组身份可以省略
user nginx nginx;

#指定nginx守护进程的pid文件
pid path/to/nginx.pid;

#指定所有worker进程所能打开的最大文件句柄数
worker_rlimit_nofile 100000;

2、性能优化相关的配置
#worker进程的个数，通常应该略少于CPU物理核心数，也可以使用auto自动获取
worker_processes auto;

#CPU的亲缘性绑定(同样是无法避免CPU的上下文的切换的)
#优点：提升缓存的命中率
#context switch：会产生CPU不必要的消耗
work_cpu_affinity  00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;

#计时器解析度(请求到达nginx,nginx相应用户请求后，要获取系统时间并记录日志，高并发的时候可能每秒钟获取很多很多次)
#降低此值，可以减少gettimeofday()系统调用的次数
timer_resolution 100ms;

#指明worker进程的nice值：数字越小,优先级越高
#nice值范围：-20,19
#对应的优先级：100，139
worker_priority number;
```
# 二、事件相关的配置
events {
    #master调度用户请求至个worker进程时使用的负载均衡锁:on表示能让多个worker轮流地、序列化的响应新请求
    accept_mutex {off|on}
    
    #延迟等待时间，默认为500ms
    accept_mutex_delay time;
    
    #accept_mutex用到的锁文件路径
    lock_file file;
    
    #指明使用的时间模型：建议让Nginx自行选择
    use [epoll|rtsig|select|poll];
    
    #单个worker进程打开的最大并发连接数，worker_processes*worker_connections
    worker_connections 2048;
    
    #告诉nginx收到一个新链接通知后接受尽可能多的链接
    multi_accept on;    
}

# 三、用于调试、定位问题
```
#是否以守护进程方式运行nginx；调试时应该设置为off
daemon {on|off}

#是否以master/worker模型来运行；调试时可以设置为off
master_process {on|off}

#error_log 位置 级别,若要使用debug，需要在编译nginx时使用--with-debug选项
error_log file | stderr | syslog:server=address[,parameter=value] | memory:size [debug|info|notice|warn|error|crit|alert|emerg];

总结：常需要调整的参数:worker_processes, worker_connections,work_cpu_affinity,worker_priority
新改动配置生效方式：
nginx -s reload其他参数stop，quit，reopen也可以使用nginx -h查看到
```

# 四、nginx作为web服务器使用的配置
```
http {}:由ngx_http_core_module模块所引入
配置框架：
http {
    upstream {
        ...
    }
    server {
        location URL {
            root "/path/to/somedir"
            ...
        }#类似于httpd中的<Location>,用于定义URL与本地文件系统的映射关系
        location URL {
            if ... {
                ...
            }
        }
    }#每个server类似于httpd中的一个<VirtualHost>
    server {
        ...
    }
}
注意：与http相关的额指令仅能够防止与http、server、location、upstream、if上下文，但有些指令仅应用于这5种上下文的某些种。

http {
    #打开或关闭错误页面中的nginx版本号
    server_tokens on;
    #!server_tag on;
    #!server_info on;
    
    #优化磁盘IO设置，指定nginx是否调用sendfile函数来输出文件，普通应用设为on，下载等磁盘IO高的应用，可设为off
    sendfile on;
    #设置nginx在一个数据包里发送所有头文件，而不是一个接一个的发送
    tcp_nopush on;
    #设置nginx不要缓存数据，而是一段一段的发送，
    
    #长连接的超时时长，默认为75s
    keepalive_timeout 30;
    #在一个长连接所能够允许请求的最大资源数
    keepalive_requests 20;
    #为制定类型的User Agent禁用长连接
    keepalive_disable [msie6|safari|none];
    #是否对长连接使用TCP_NODELAY选项,不将多个小文件合并传输
    tcp_nodelay on;
    #读取http请求报文首部的超时时长
    client_header_timeout #;
    #读取http请求报文body部分的超时时长
    client_body_timeout #;
    #发送响应报文的超时时长
    send_timeout #;
    
    #设置用户保存各种key的共享内存的参数，5m指的是5兆
    limit_conn_zone $binary_remote_addr zone=addr:5m;
    #为给定的key设置最大的连接数，这里的key是addr，设定的值是100，就是说允许每一个IP地址最多同时打开100个连接
    limit_conn addr 100;

    #include指在当前文件中包含另一个文件内容
    include mime.types;
    #设置文件使用默认的mine-type
    default_type text/html;
    #设置默认字符集
    charset UTF-8;

    #设置nginx采用gzip压缩的形式发送数据，减少发送数据量，但会增加请求处理时间及CPU处理时间，需要权衡
    gzip on;
    #加vary给代理服务器使用，针对有的浏览器支持压缩，有个不支持，根据客户端的HTTP头来判断是否需要压缩
    gzip_vary on;
    #nginx在压缩资源之前，先查找是否有预先gzip处理过的资源
    #!gzip_static on;
    #为指定的客户端禁用gzip功能
    gzip_disable "MSIE[1-6]\.";
    #允许或禁止压缩基于请求和相应的响应流，any代表压缩所有请求
    gzip_proxied any;
    #设置对数据启用压缩的最少字节数，如果请求小于10240字节则不压缩，会影响请求速度
    gzip_min_length 10240;
    #设置数据压缩等级，1-9之间，9最慢压缩比最大
    gzip_comp_level 2;
    #设置需要压缩的数据格式
    gzip_types text/plain text/css text/xml text/javascript  application/json application/x-javascript application/xml application/xml+rss; 

    #开发缓存的同时也指定了缓存文件的最大数量，20s如果文件没有请求则删除缓存
    open_file_cache max=100000 inactive=20s;
    
    #指多长时间检查一次缓存的有效信息
    open_file_cache_valid 60s;
    
    #文件缓存最小的访问次数，只有访问超过5次的才会被缓存
    open_file_cache_min_uses 5;
    
    #当搜索一个文件时是否缓存错误信息
    open_file_cache_errors on;

    #允许客户端请求的最大单文件字节数
    client_max_body_size 8m;
    
    #冲区代理缓冲用户端请求的最大字节数
    client_header_buffer_size 32k;
    
    #引用/etc/nginx/vhosts下的所有配置文件，如果主机名众多的情况下可以每个主机名建立一个文件，以方便管理
    include /etc/nginx/vhosts/*;
}
```

# 五、虚拟主机设定模块
```
#负载均衡服务器列表(本人通常把负载均衡类别配置在相应的虚拟主机的配置文件中)
upstream fansik {
    #后端服务器访问规则
    ip_hash;
    #weight参数表示权重值，权值越高被分配到的几率越大
    server 192.168.1.101:8081 weight=5;
    server 192.168.1.102:8081 max_fails=3 fail_timeout=10s;
}
server {
    #监听80端口
    listen 80;
    #定义主机名，主机名可以有多个，名称还可以使用正则表达式(~)或通配符
    #(1)先做精确匹配检查
    #(2)左侧通配符匹配检查：*.fansik.com
    #(3)右侧通配符匹配检查：mail.*
    #(4)正则表达式匹配检查：如~^.*\.fansik\.com$
    #(5)detault_server
    server_name fansik.fansik.com;
    #设定本虚拟主机的访问日志
    access_log logs/fansik.fansik.com.access.log;

    location [=|~|~*|^~] uri {...}
    功能：允许根据用户请求的URI来匹配定义的个location，匹配到时，此请求将被相应的location配置块中的配置所处理
    =：表示精确匹配检查
    ~：正则表达式模式匹配检查，区分字符大小写
    ~*：正则表达式模式匹配检查，不区分字符大小写
    ^~：URI的前半部分匹配，不支持正则表达式
    !~：开头表示区分大小写的不匹配的正则
    !~*：开头表示不区分大小写的不匹配的正则
    /：通用匹配，任何请求都会被匹配到
    location / {
        #定义服务器的默认网站根目录位置
        root html;
        #定义首页索引文件的名称
        index index.html index.htm;
        #引用反向代理的配置，配置文件目录根据编译参数而定
        #如果编译时加入了--conf-path=/etc/nginx/nginx.conf指定了配置文件的路径那么就把proxy.conf放在/etc/nginx/目录下
        #如果没有制定配置文件路径那么就把proxy.conf配置放到nginx的conf目录下
        include proxy.conf;    
        #定义后端负载服务器组
        proxy_pass http://fansik;
    }
    alias path和root path的区别；
    location /images/ {
        root "/data/images"
    }
    http://fansik.fansik.com/images/a.jpg <-- /data/images/images/a.jpg
    location /images/ {
        alias "/data/images/"
    }
    http://fansik.fansik.com/images/a.jpg <-- /data/images/a.jpg
    

    #定义错误提示页面
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root html;
    }

    #设定查看Nginx状态的地址
    #只能定义在location中
    #htpasswd -c -m /etc/nginx/.htpasswd fansik(-c 参数第一次创建时使用)
    location /Status {
        stub_status on;
        allow all;
        #access_log off;
        #allow 192.168.1.0/24;
        #deny all;
        #auth_basic "Status";
        #auth_basic_user_file /etc/nginx/.htpasswd;
    }
    status结果实例说明：
    Active connections: 1 (当前所有处于打开状态的连接数)
    server accepts handled requests
    174(已经接受进来的连接) 174(已经处理过的连接) 492(处理的请求，在保持连接模式下，请求数可能会多于连接数量)
    Reading: 0 Writing: 1 Waiting: 0 
    Reading:正处于接受请求状态的连接数
    Writing:请求接受完成，正处于处理请求或发送相应的过程中的连接数
    Waiting:保持连接模式，且处于活动状态的连接数
    
    #基于IP的访问控制
    allow IP/Netmask
    deny IP/Netmask
    location ~ /\.ht {
        deny all;
    }
}
```

# 六、反向代理的配置(反向代理的配置通常放在单独的配置文件中proxy.conf，通过include引用)
```
proxy_redirect off;
#后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#nginx跟后端服务器连接超时时间(代理连接超时)
proxy_connect_timeout 60;
#连接成功后，后端服务器响应时间(代理接收超时)
proxy_read_timeout 120;
#后端服务器数据回传时间(代理发送超时)
proxy_send_timeout 20;
#设置代理服务器（nginx）保存用户头信息的缓冲区大小
proxy_buffer_size 32k;
#proxy_buffers缓冲区，网页平均在32k以下的设置
proxy_buffers 4 128k;
#高负荷下缓冲大小（proxy_buffers*2）
proxy_busy_buffers_size 256k;
#设定缓存文件夹大小，大于这个值，将从upstream服务器传
proxy_temp_file_write_size 256k;
#1G内存缓冲空间，3天不用删除，最大磁盘缓冲空间2G
proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=cache_one:1024m inactive=3d max_size=2g;
```

# 七、https服务的配置
```
server {
        listen       443 ssl;
        server_name  test.fansik.cn;
        ssl_certificate      100doc.cn.crt;
        ssl_certificate_key  100doc.cn.key;     
        ssl_session_cache    shared:SSL:1m;
        ssl_protocols SSLv2 SSLv3 TLSv1;
        ssl_session_timeout  5m;        
        ssl_ciphers ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
        ssl_prefer_server_ciphers  on;       
        location / {
                root /data/app
                index  index.html index.htm;
        }
}
```

# 八、url地址重写
```
rewrite regex replacment flag
例如：rewrite ^/images/(.*\.jpg)$ /imgs/$1 break;#$1是前面括号中的内容哦
http://fansik.fansik.com/images/a/1.jpg --> http://fansik.fansik.com/imgs/a/1.jpg
flag:
    last:一旦此rewrite规则重写完成后，不再被后面其他的rewrite规则进行处理，
    而是由User Agent重新对重写后的URL再一次发起请求，并从头开始执行类似的过程。
    break：一旦此rewrite规则重写完成之后，由User Agent对新的URL重新发起请求，
    且不会被当前location内的任何rewrite规则过检查
    redirect：以302响应码(临时重定向)返回新的URL
    permanent：以301响应码(永久重定向)返回新的URL
```
# 九、if判断
```
语法：if (condition) {...}
应用环境：server，location
condition：
(1)变量名：
变量值为空串，或者以"0"开始，则为false，其他的均为true
(2)以变量为操作数构成的比较表达式
可以使用=，!=类似的比较操作符进行测试
(3)正则表达式的模式匹配操作
~：区分大小写的模式匹配检查
~*：不区分大小写的模式匹配检查
!~和!~*：对上面两种测试取反
(4)测试路径为文件可能性：-f ,~-f
(5)测试制定路径为目录的可能性：-d,!-d
(6)测试文件存在性：-e,!-e
(7)检查文件是否有执行权限：-x,!-x
例如：
if($http_user_agent ~* MSIE){
    rewrite ^(.*)$ /msie/$1 break;
}
```
# 十、防盗链
```
location ~* \.(jpg|gif|jpeg|png)$ {
    valid_referer none blocked www.fansik.com;
    if ($invalid_referer) {
        rewrite ^/ http://www.fansik.com/403.html;
    }
}
```







<h2 id="nginxconfigoptimize">nginx.conf 配置及基本优化</h2>
一：常用功能优化：

1：网络连接的优化：

　　只能在events模块设置，用于防止在同一一个时刻只有一个请求的情况下，出现多个睡眠进程会被唤醒但只能有一个进程可获得请求的尴尬，如果不优化，在多进程的nginx会影响以部分性能。

events {
accept_mutex on; #优化同一时刻只有一个请求而避免多个睡眠进程被唤醒的设置，on为防止被同时唤醒，默认为off，因此nginx刚安装完以后要进行适当的优化。
}
2.设置是否允许同时接受多个网络连接：

　　只能在events模块设置，Nginx服务器的每个工作进程可以同时接受多个新的网络连接，但是需要在配置文件中配置，此指令默认为关闭，即默认为一个工作进程只能一次接受一个新的网络连接，打开后几个同时接受多个，配置语法如下：

events {
accept_mutex on;
multi_accept on; #打开同时接受多个新网络连接请求的功能。
}
3.隐藏ngxin版本号：

　　当前使用的nginx可能会有未知的漏洞，如果被黑客使用将会造成无法估量的损失，但是我们可以将nginx的版本隐藏，如下：

server_tokens off; #在http 模块当中配置
4.：选择事件驱动模型：

　　Nginx支持众多的事件驱动，比如select、poll、epoll，只能设置在events模块中设置：

events {
accept_mutex on;
multi_accept on;
use epoll; #使用epoll事件驱动，因为epoll的性能相比其他事件驱动要好很多
}
5：配置单个工作进程的最大连接数：

　　通过worker_connections number；进行设置，numebr为整数，number的值不能大于操作系统能打开的最大的文件句柄数，使用ulimit -n可以查看当前操作系统支持的最大文件句柄数，默认为为1024.

复制代码
events {
    worker_connections  102400; #设置单个工作进程最大连接数102400
    accept_mutex on;
    multi_accept on;
    use epoll;
}
复制代码
6：定义MIME-Type：

　　在浏览器当中可以显示的内容有HTML/GIF/XML/Flash等内容，浏览器为取得这些资源需要使用MIME Type，即MIME是网络资源的媒体类型，Nginx作为Web服务器必须要能够识别全都请求的资源类型，在nginx.conf文件中引用了一个第三方文件，使用include导入：

include mime.types;
default_type application/octet-stream;
7：自定义访问日志：

　　访问日志是记录客户端即用户的具体请求内容信息，全局配置模块中的error_log是记录nginx服务器运行时的日志保存路径和记录日志的level，因此有着本质的区别，而且Nginx的错误日志一般只有一个，但是访问日志可以在不同server中定义多个，定义一个日志需要使用access_log指定日志的保存路径，使用log_format指定日志的格式，格式中定义要保存的具体日志内容：

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
8：将日志定义为json格式：

　　在使用日志分析工具如ELK对访问日志做统计的时候，就需要将日志格式定义为json格式，以便于取相应字段的key做统计，完整的定义如下：

复制代码
   log_format logstash_json '{"@timestamp":"$time_iso8601",'
        '"host":"$server_addr",'
        '"clientip":"$remote_addr",'
        '"size":$body_bytes_sent,'
        '"responsetime":$request_time,'
        '"upstreamtime":"$upstream_response_time",'
        '"upstreamhost":"$upstream_addr",'
        '"http_host":"$host",'
        '"url":"$uri",'
        '"domain":"$host",'
        '"xff":"$http_x_forwarded_for",'
        '"referer":"$http_referer",'
        '"agent":"$http_user_agent",'
        '"status":"$status"}';
server {
　　　　listen 8090;
　　　　server_name samsung.chinacloudapp.cn;
　　　　access_log /var/log/nginx/samsung1.access.log logstash_json;
　　　　location / {
　　　　　　root html;
　　　　　　index index1.html index.htm;
　　　　}
　　　　error_page 500 502 503 504 /50x.html;
　　　　location = /50x.html {
　　　　root html;
　　　　　}
　　}

    access_log  /var/log/nginx/json.access.log  logstash_json;  #定义日志路径为/var/log/nginx/json.access.log,并引用在主配置文件nginx.conf中定义的json日志格式
复制代码
json格式的日志内如下：

复制代码
{"@timestamp":"2016-04-25T13:16:29+08:00","host":"192.168.0.202","clientip":"106.120.73.171","size":0,"responsetime":0.000,"upstreamtime":"-","upstreamhost":"-","http_host":"samsung.chinacloudapp.cn","url":"/index1.html","domain":"samsung.chinacloudapp.cn","xff":"-","referer":"-","agent":"Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.75 Safari/537.36","status":"304"}
复制代码
9：配置允许sendfile方式传输文件：

　　是由后端程序负责把源文件打包加密生成目标文件，然后程序读取目标文件返回给浏览器；这种做法有个致命的缺陷就是占用大量后端程序资源，如果遇到一些访客下载速度巨慢，就会造成大量资源被长期占用得不到释放（如后端程序占用的CPU/内存/进程等），很快后端程序就会因为没有资源可用而无法正常提供服务。通常表现就是 nginx报502错误，而sendfile打开后配合location可以实现有nginx检测文件使用存在，如果存在就有nginx直接提供静态文件的浏览服务，因此可以提升服务器性能.

　　可以配置在http、server或者location模块，配置如下：

    sendfile        on;
    sendfile_max_chunk 512k;   #Nginxg工作进程每次调用sendfile()传输的数据最大不能超出这个值，默认值为0表示无限制，可以设置在http/server/location模块中。
10：配置nginx工作进程最大打开文件数：

　　可以设置为linux系统最大打开的文件数量一致，在全局模块配置

worker_rlimit_nofile 65535;
11：会话保持时间：

　　用户和服务器建立连接后客户端分配keep-alive链接超时时间，服务器将在这个超时时间过后关闭链接，我们将它设置低些可以让ngnix持续工作的时间更长，1.8.1默认为65秒，一般不超过120秒。

 keepalive_timeout  65 60;  #后面的60为发送给客户端应答报文头部中显示的超时时间设置为60s：如不设置客户端将不显示超时时间。
　　Keep-Alive:timeout=60  #浏览器收到的服务器返回的报文

如果设置为0表示关闭会话保持功能，将如下显示：
　　Connection:close  #浏览器收到的服务器返回的报文
12配置网络监听：

　　使用命令listen，可以配置监听IP+端口，端口或监听unix socket:

listen       8090;   #监听本机的IPV4和IPV6的8090端口，等于listen *:8000
listen       192.168.0.1:8090; #监听指定地址的8090端口
listen     Unix:/www/file  #监听unix socket
 

 二：server部分主要配置：

1、基于域名和IP的虚拟主机

 server_name  localhost www.a.com; ＃多个域名用空格间隔即可
 server_name  192.168.0.2;  #IP是本机的网卡IP地址
2、location 模块正则匹配配置：

　　在没有使用正则表达式的时候，nginx会先在server中的多个location选取匹配度最高的一个uri，uri是用户请求的字符串，即域名后面的web文件路径，然后使用该location模块中的正则url和字符串，如果匹配成功就结束搜索，并使用此location处理此请求。

　　location 正则匹配的语法：

复制代码
=  #用于标准uri前，需要请求字串与uri完全匹配，如果匹配成功就停止向下匹配并立即处理请求。
~  #区分大小写
~*  #不区分大写
!~  #区分大小写不匹配
!~* #不区分大小写不匹配 
^  #匹配以什么开头
$  #匹配以什么结尾
\  #转义字符。可以转. * ?等
*  #代表任意长度的任意字符

-f和!-f #用来判断是否存在文件
-d和!-d #用来判断是否存在目录
-e和!-e #用来判断是否存在文件或目录
-x和!-x #用来判断文件是否可执行
复制代码
 3、常见http状态码：

复制代码
200 #请求成功，即服务器返回成功
301 #永久重定向
302 #临时重定向
403 #禁止访问，一般是服务器权限拒绝
400 #错误请求，请求中有语法问题，或不能满足请求。  
403 #服务器权限问题导致无法显示
404 #服务器找不到用户请求的页面
500 #服务器内部错误，大部分是服务器的设置或内部程序出现问题
501 #没有将正在访问的网站设置为浏览器所请求的内容
502 #网关问题，是代理服务器请求后端服务器时，后端服务器不可用或没有完成 相应网关服务器，这通常是反向代理服务器下面的节点出问题导致的。
503 ＃服务当前不可用，可能是服务器超载或停机导致的，或者是反向代理服务器后面没有可以提供服务的节点。
504 #网关超时，一般是网关代理服务器请求后端服务器时，后端服务器没有在指定的时间内完成处理请求，多数是服务器过载导致没有在特定的时间内返回数据给前端代理服务器。
505 #该网站不支持浏览器用于请求网页的ＨＴＴＰ协议版本（最为常见的是ＨＴＴＰ/1.1）
复制代码
 4.在server部分使用location配置一个web界面：

要求：在html/localtion/myweb 里面有个index.html文件里面写了myweb，当访问nginx 服务器的/myweb的时候要显示此html文件的内容：

复制代码
    server {
        listen       8090;
        server_name  samsung.chinacloudapp.cn;
        access_log  /var/log/nginx/samsung1.access.log   logstash_json;
        location / {
            root   html;
            index  index1.html index.htm;
        }
        
    location ~/myweb { #区分大小写，即访问Myweb是不行的
        root html/localtion;  #定义myweb所在的路径，即在浏览器访问myweb的时候，实际是访问的html/localtion/myweb目录里面的web内容
        index   index.html; #默认首页文件类型
    }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
复制代码
验证如下：



 

三：sysctl.conf针对IPv4内核的7个参数的配置优化：

1、net.core.netdev_max_backlog  #每个网络接口的处理速率比内核处理包的速度快的时候，允许发送队列的最大数目。

[root@Server1 nginx]# sysctl -a | grep max_backlog
net.core.netdev_max_backlog = 1000  这里默认是1000，可以设置的大一些，比如：
net.core.netdev_max_backlog = 102400
2、net.core.somaxconn: #用于调节系统同时发起的TCP连接数，默认值一般为128，在客户端存在高并发请求的时候，128就变得比较小了，可能会导致链接超时或者重传问题。

net.core.somaxconn = 128  #默认为128，高并发的情况时候要设置大一些，比如：
net.core.somaxconn = 102400
3、net.ipv4.tcp_max_orphans:设置系统中做多允许多少TCP套接字不被关联到任何一个用户文件句柄上，如果超出这个值，没有与用户文件句柄关联的TCP套接字将立即被复位，同时给出警告信息，这个值是简单防止DDOS（Denial of service）的攻击，在内存比较充足的时候可以设置大一些：

net.ipv4.tcp_max_orphans = 32768 #默认为32768，可以改该打一些：
net.ipv4.tcp_max_orphans = 102400
4、net.ipv4.tcp_max_syn_backlog #用于记录尚未收到客户度确认消息的连接请求的最大值，一般要设置大一些：

net.ipv4.tcp_max_syn_backlog = 256  #默认为256，设置大一些如下：
net.ipv4.tcp_max_syn_backlog =  102400
5、net.ipv4.tcp_timestamps #用于设置时间戳，可以避免序列号的卷绕，有时候会出现数据包用之前的序列号的情况，此值默认为1表示不允许序列号的数据包，对于Nginx服务器来说，要改为0禁用对于TCP时间戳的支持，这样TCP协议会让内核接受这种数据包，从而避免网络异常，如下：

net.ipv4.tcp_timestamps = 1 #默认为1，改为0，如下：
net.ipv4.tcp_timestamps = 0
6、net.ipv4.tcp_synack_retries #用于设置内核放弃TCP连接之前向客户端发生SYN+ACK包的数量，网络连接建立需要三次握手，客户端首先向服务器发生一个连接请求，服务器收到后由内核回复一个SYN+ACK的报文，这个值不能设置过多，会影响服务器的性能，还会引起syn攻击：

net.ipv4.tcp_synack_retries = 5 #默认为5，可以改为1避免syn攻击
net.ipv4.tcp_synack_retries = 1
7、net.ipv4.tcp_syn_retries  #与上一个功能类似，设置为1即可：

net.ipv4.tcp_syn_retries = 5 #默认为5，可以改为1
net.ipv4.tcp_syn_retries = 1
 

四：配置文件中针对CPU的2个优化参数：

1、woker_precess #设置Nginx 启动多少个工作进程的数量
2、woker_cpu_affinit #固定Nginx 工作进程所运行的CPU核心
 

五：配置文件中与网络相关的4个指令：

复制代码
1、keepalived_timeout 60 50； #设置Nginx服务器与客户端保持连接的时间是60秒，到60秒后服务器与客户端断开连接，50s是使用Keep-Alive消息头与部分浏览器如 chrome等的连接事件，到50秒后浏览器主动与服务器断开连接。
　　keepalived_timeout 60 50;
2、sendtime_out  10s #Http核心模块指令，指定了发送给客户端应答后的超时时间，Timeout是指没有进入完整established状态，只完成了两次握手，如果超过这个时间客户端没有任何响应，nginx将关闭与客户端的连接。
　　sendtime_out 10s;
3、client_header_timeout  #用于指定来自客户端请求头的headerbuffer大小，对于大多数请求，1kb的缓冲区大小已经足够，如果自定义了消息头部或有更大的cookie，可以增加缓冲区大小。
　　client_header_timeout 4k;
4、multi_accept  #设置是否允许，Nginx在已经得到一个新连接的通知时，接收尽可能更多的连接。
    multi_accept on;
复制代码
 

六：配置文件中与驱动模型相关的8个指令： 

1、use； #用于指定Nginx 使用的事件驱动模型

2、woker_process； #指定Nginx启动的工作进程的数量

3、woker_connections  65535； #指定Nginx 每个工作进程的最大连接数，woker_connections  *  woker_process即为Nginx的最大连接数量。

4、woker_rlimit_sigpending 65535  #Nginx每个进程的事件信号队列的上限长度，如果超出长度，Nginx则使用poll模型处理客户的请求。

5、devpoll_changes 和 devpoll_events #用于设置Nginx 在/dev/poll 模型下Nginx服务器可以与内核之间传递事件的数量，前一个设置传递给内核的事件数量，后一个设置从内核读取的事件数量，默认为512。

6、kqueue_changes 和 kqueue_events #设置在kqueue模型下Nginx服务器可以与内核之间传递事件的数量，前一个设置传递给内核的事件数量，后一个设置从内核读取的事件数量，默认为512。

7、epoll_events #设置在epoll驱动模式下Nginx 服务器可以与内核之间传递事件的数量，默认为512。

8、rtsig_signo  #设置Nginx在rtsig 模式使用的两个信号中的第一个，第二个信号是在第一个信号的编号上加1.

9、rtsig_overflow #这些参数指定如何处理rtsig队列溢出。当溢出发生在nginx清空rtsig队列时，它们将连续调用poll()和 rtsig.poll()来处理未完成的事件，直到rtsig被排空以防止新的溢出，当溢出处理完毕，nginx再次启用rtsig模式，rtsig_overflow_events specifies指定经过poll()的事件数，默认为16，rtsig_overflow_test指定poll()处理多少事件后nginx将排空rtsig队列，默认值为32，rtsig_overflow_threshold只能运行在Linux 2.4.x内核下，在排空rtsig队列前nginx检查内核以确定队列是怎样被填满的。默认值为1/10，“rtsig_overflow_threshold 3”意为1/3。











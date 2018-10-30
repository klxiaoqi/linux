
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
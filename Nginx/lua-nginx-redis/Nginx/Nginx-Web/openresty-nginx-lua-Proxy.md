### Openresty-Lua动态修改upstream后端服务
+ nginx.conf 配置文件
    ``` 
    worker_processes  1;
    pid        logs/nginx.pid;
    events {
        worker_connections  1024;
    }

    http {
        include       mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
        sendfile        on;

        keepalive_timeout  65;
             
        lua_shared_dict upstreams 1m;  # 声明一个ngx多进程全局共享内存区域，_G 作为基于shm的Lua字典的存储空间ngx.shared.<name>   
        upstream default_upstream {                        # 配置后端服务器组
                    server 127.0.0.1:8081;
                    server 127.0.0.1:8082;
        }
            
        upstream lua_upstream {                        # 配置后端服务器组
                    server 127.0.0.1:8084;
                    server 127.0.0.1:8083;
        }

        server {
            listen       80;
            server_name  localhost;

            access_log  logs/80.access.log  main;
            error_log   logs/80.error.log error;

            location = /_switch_upstream {
                 content_by_lua_block{
                        local ups = ngx.req.get_uri_args()["upstream"]
                        if ups == nil or ups == "" then
                            ngx.say("upstream is nil 1")
                            return nil
                        end
                        local host = ngx.var.http_host
                        local upstreams = ngx.shared.upstreams
                        local ups_src = upstreams:get(host)
                        ngx.say("Current upstream is :",ups_src)
                        ngx.log(ngx.WARN, host, " change upstream from ", ups_src, " to ", ups)
                        local succ, err, forcible = upstreams:set(host,  ups)
                        ngx.say(host, " change upstream from ", ups_src, " to ", ups)
                }
            }
            
            location / {
                set_by_lua_block $my_upstream {
                    local ups = ngx.shared.upstreams:get(ngx.var.http_host)
                    if ups ~= nil then
                        ngx.log(ngx.ERR, "get [", ups,"] from ngx.shared")
                        return ups
                    end
                    return "default_upstream"
                }

                proxy_next_upstream off;
                proxy_set_header    X-Real-IP           $remote_addr;
                proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
                proxy_set_header    Host                $host;
                proxy_http_version  1.1;
                proxy_set_header    Connection  "";
                proxy_pass          http://$my_upstream ;
            }
        }   

        server {
            listen       8081;
            server_name  localhost;

            location / {
                root   html81;
                index  index.html index.htm;
            }
        }

        server {
            listen       8082;
            server_name  localhost;

            location / {
                root   html82;
                index  index.html index.htm;
            }
        }

        server {
            listen       8083;
            server_name  localhost;

            location / {
                root   html83;
                index  index.html index.htm;
            }
        }

        server {
            listen       8084;
            server_name  localhost;

            location / {
                root   html84;
                index  index.html index.htm;
            }
        }
    }
    ```
+ 4个端口分别对应4个html 根目录
    + html81/index.html 内容 `server name 8081`
    + html82/index.html 内容 `server name 8082`
    + html83/index.html 内容 `server name 8083`
    + html84/index.html 内容 `server name 8084`
+ 如何切换后端upstream
    + `default_upstream` 切换到 `lua_upstream`
        ``` 
        root@tinywan:# curl http://127.0.0.1/_switch_upstream?upstream=lua_upstream
        127.0.0.1 change upstream from default_upstream to lua_upstream    
        ```
    + `lua_upstream` 切换（还原`default_upstream`）到 `default_upstream`
        ``` 
        root@tinywan:# curl http://127.0.0.1/_switch_upstream?upstream=default_upstream
        127.0.0.1 change upstream from lua_upstream to default_upstream    
        ```
+ 一个收集upstream_response_time的平均数据的例子
    ``` 
    lua_shared_dict log_dict 5M  # 声明一个ngx多进程全局共享内存区域
    
    server{
        location / {
            proxy_pass http;//mybackend   # 代理模块
            log_by_lua '
                local log_dict = ngx.shared.log_dict
                local upstream_time = tonumber(ngx.var.upstream_response_time)
                local sum = log_dict:get("upstream_time-sum") or 0
                sum = sum + upstream_time
                log_dict:set("upsteam_time-sum", sum)
                local newval, err = log_dict:incr("upstream_time-nb", 1)
                if not newval and err == "not found" then
                    log_dict:add("upstream_time-nb", 0)
                    log_dict:incr("upstream_time-nb", 1)
                end
            '
        }
        location = /status {
            content_by_lua '
                local log_dict = ngx.shared.log_dict
                local sum = log_dict:get("upstream_time-sum")
                local nb = log_dict:get("upstream_time-nb")
    
                if nb and sum then
                    ngx.say("average upstream response time:  ", sum/nb, " (", nb, " reqs)")
                else
                    ngx.say("no data yet")
                end
            '
        }
    }
    ```
+  动态修改upstream后端服务和upstream_response_time的平均数据 合并后的代码
   ``` 
    #user  nobody;
    worker_processes  1;
    pid        logs/nginx.pid;
    events {
        worker_connections  1024;
    }

    http {
        include       mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
        sendfile        on;

        keepalive_timeout  65;
        
    lua_shared_dict upstreams 1m;  # ngx多进程全局共享内存，保存upstream值   
    upstream default_upstream {                        # 配置后端服务器组
                server 127.0.0.1:8081;
                server 127.0.0.1:8082;
    }
        
    upstream lua_upstream {                        # 配置后端服务器组
                server 127.0.0.1:8084;
                server 127.0.0.1:8083;
    }

    
    server {
        listen       80;
        server_name  localhost;

        access_log  logs/80.access.log  main;
        error_log   logs/80.error.log error;

        location = /_switch_upstream {
                default_type 'text/html';
                content_by_lua_block{
                        local ups = ngx.req.get_uri_args()["upstream"]
                        if ups == nil or ups == "" then
                            ngx.say("upstream is nil 1")
                return nil
                        end
                        local host = ngx.var.http_host
                local upstreams = ngx.shared.upstreams
                        local ups_src = upstreams:get(host)
                ngx.say("Current upstream is :",ups_src)
                        ngx.log(ngx.WARN, host, " change upstream from ", ups_src, " to ", ups)
                        local succ, err, forcible = upstreams:set(host,  ups)
                        ngx.say(host, " change upstream from ", ups_src, " to ", ups)
                }
        }
        
        #location ~ (^/api/|^/p/|^/m/|^/oauthapi/) {
        location / {
            set_by_lua_block $my_upstream {
            local ups = ngx.shared.upstreams:get(ngx.var.http_host)
            if ups ~= nil then
                ngx.log(ngx.ERR, "get [", ups,"] from ngx.shared")
                return ups
            end
            return "default_upstream"
            }

            proxy_next_upstream off;
            proxy_set_header    X-Real-IP           $remote_addr;
            proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
            proxy_set_header    Host                $host;
            proxy_http_version  1.1;
            proxy_set_header    Connection  "";
            proxy_pass          http://$my_upstream ;

            log_by_lua_block {
                local log_dict = ngx.shared.upstreams
                    local upstream_time = tonumber(ngx.var.upstream_response_time)

                    local sum = log_dict:get("upstream_time-sum") or 0
                    sum = sum + upstream_time
                    log_dict:set("upstream_time-sum", sum)

                    local newval, err = log_dict:incr("upstream_time-nb", 1)
                    if not newval and err == "not found" then
                        log_dict:add("upstream_time-nb", 0)
                        log_dict:incr("upstream_time-nb", 1)
                    end
            }
        }

        location = /status {
            default_type 'text/html';  -- 通过浏览器可访问
            content_by_lua_block {
                local log_dict = ngx.shared.upstreams
                local sum = log_dict:get("upstream_time-sum")
                local nb = log_dict:get("upstream_time-nb")

                if nb and sum then
                    ngx.say("average upstream response time: ", sum / nb,
                            " (", nb, " reqs)")
                else
                    ngx.say("no data yet")
                end
            }
        }

    }   

    server {
            listen       8099;
            server_name  localhost;

            #charset koi8-r;

            access_log  logs/80.access.log  main;
        error_log   logs/80.error.log error;
            location / {
            #proxy_pass http://live_node;
            }

            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
                root   html;
            }
        }

        server {
            listen       8081;
            server_name  localhost;

            location / {
                root   html81;
                index  index.html index.htm;
            }
        }

        server {
            listen       8082;
            server_name  localhost;

            location / {
                root   html82;
                index  index.html index.htm;
            }
        }

        server {
            listen       8083;
            server_name  localhost;

            location / {
                root   html83;
                index  index.html index.htm;
            }
        }
    
    server {
            listen       8084;
            server_name  localhost;

            location / {
                root   html84;
                index  index.html index.htm;
            }
        }
    }
   ```
+ 测试结果：
    ``` 
    root@tinywan:/opt/openresty/nginx/conf# curl http://127.0.0.1/status
    average upstream response time: 0.0003953488372093 (129 reqs)
    ```
+ 浏览器测试，记得加上这个`default_type 'text/html';`,在浏览器地址栏直接输入即可访问
    `http://127.0.0.1/_switch_upstream?upstream=default_upstream`
   
+ HELP:[http://chattool.sinaapp.com/?p=2372](http://chattool.sinaapp.com/?p=2372)
# docker-compose语法

## 示例
```sh
version: "3"  # 指定docker-compose语法版本
services:    # 从以下定义服务配置列表
  server_name:   # 可将server_name替换为自定义的名字，如mysql/php都可以
    container_name: container_name  # 指定实例化后的容器名，可将container_name替换为自定义名
    image: xxx:latest # 指定使用的镜像名及标签
    build:  # 如果没有现成的镜像，需要自己构建使用这个选项
      context: /xxx/xxx/Dockerfile  # 指定构建镜像文件的路径
      dockerfile: ....     # 指定Dockerfile文件名，上一条指定，这一条就不要了
    ports:
      - "00:00"  # 容器内的映射端口，本地端口:容器内端口
      - "00:00"  # 可指定多个
    volumes:
      - test1:/xx/xx  # 这里使用managed volume的方法，将容器内的目录映射到物理机，方便管理
      - test2:/xx/xx  # 前者是volumes目录下的名字，后者是容器内目录
      - test3:/xx/xx  # 在文件的最后还要使用volumes指定这几个tests
    volumes_from:  # 指定卷容器
       - volume_container_name  # 卷容器名
    restarts: always  # 设置无论遇到什么错，重启容器
    depends_on:       # 用来解决依赖关系，如这个服务的启动，必须在哪个服务启动之后
      - server_name   # 这个是名字其他服务在这个文件中的server_name
      - server_name1  # 按照先后顺序启动
    links:  # 与depend_on相对应，上面控制容器启动，这个控制容器连接
      - mysql  # 值可以是- 服务名，比较复杂，可以在该服务中使用links中mysql代替这个mysql的ip
    networks: # 加入指定的网络，与之前的添加网卡名类似
      - my_net  # bridge类型的网卡名
      - myapp_net # 如果没有网卡会被创建,建议使用时先创建号，在指定
    environment: # 定义变量，类似dockerfile中的ENV
      - TZ=Asia/Shanghai  # 这里设置容器的时区为亚洲上海，也就解决了容器通过compose编排启动的 时区问题！！！！解决了容器的时区问题！！！
      变量值: 变量名   # 这些变量将会被直接写到镜像中的/etc/profile
    command: [                        #使用 command 可以覆盖容器启动后默认执行的命令
            '--character-set-server=utf8mb4',            #设置数据库表的数据集
            '--collation-server=utf8mb4_unicode_ci',    #设置数据库表的数据集
            '--default-time-zone=+8:00'                    #设置mysql数据库的 时区问题！！！！ 而不是设置容器的时区问题！！！！
    ]
  server_name2:  # 开始第二个容器
    server_name:
      stdin_open: true # 类似于docker run -d
      tty: true  # 类似于docker run -t
volumes:   # 以上每个服务中挂载映射的目录都在这里写入一次,也叫作声明volume
  test1:
  test2:
  test3:
networks:  # 如果要指定ip网段，还是创建好在使用即可，声明networks
  my_net:
    driver: bridge  # 指定网卡类型
  myapp_net:
    driver: bridge 
```
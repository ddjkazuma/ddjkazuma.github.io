### Dockerfile里面CMD, ENTRYPOINT和RUN的区别

* cmd选项和entrypoint选项都可以设置docker启动时的默认命令

* run命令一般用来执行额外的操作，比如更新附带版本

* cmd命令可以在执行docker run的时候被重写，如果docker run没有附加参数，那么就是执行cmd里面的命令，如果指定了具体的命令，那么cmd将会被忽略，而entrypoint则不行

* php镜像添加supervisor的问题

  * php-fpm的镜像构建的容器默认会执行php-fpm命令，在添加了supervisor之后，在docker-compose.yml中设置supervisord命令会重写容器默认命令，因此php-fpm就不能运行了

    
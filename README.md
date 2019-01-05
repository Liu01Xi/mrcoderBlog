# mrcoderBlog

##  代码部署
* git clone https://github.com/FrCoderBlog/mrcoderBlog.git blog
* cd blog 
* composer install
* composer dump-autoload  (加载所有类)
* php artisan key:generate
* 配置.env
* php artisan migrate (建表)
* php artisan db:seed --class=UsersTableSeeder (填充初始密码account:admin@qq.com, pass:admin)
* php artisan storage:link

至此，blog已经可以访问了，后台地址为http://你的域名/mrcoderadmin/,不过搜索功能还不能使用


## sphinx搜索引擎
* 安装
```
yum install make gcc gcc++ gcc-c++ libtool autoconf automake imake mysql-devel libxml2-devel expat-devel
wget http://files.opstool.com/man/coreseek-4.1-beta.tar.gz
tar -xzvf coreseek-4.1-beta.tar.gz
cd coreseek-4.1-beta
cd mmseg-3.2.14
./bootstrap
./configure --prefix=/usr/local/mmseg3
make && make install
cd ..
cd csft-4.1
sh buildconf.sh（这个地方有可能生成不了configure文件，需要修改部分文件，具体内容参考http://blog.csdn.net/jcjc918/article/details/39032689）
（
错误1：
解决办法：在 csft-4.1/configure.ac 文件中，查找：
AC_PROG_RANLIB 
后面加上
AM_PROG_AR 
最终格式为：AC_PROG_RANLIB AM_PROG_AR
再次执行 sh buildconf.sh 

错误2：
'automake --add-missing' can install 'ar-lib'
在命令行执行
#automake --add-missing
再次执行 sh buildconf.sh

错误3
编译的时候出现
sphinxexpr.cpp:1823:43: error: ‘ExprEval’ was not declared in this scope, and no declarations were found by argument-dependent lookup at the point of instantiation [-fpermissive]
   T val = ExprEval ( this->m_pArg, tMatch ); // 'this' fixes gcc braindamage
处理办法：
 #vim /usr/local/src/coreseek-4.1-beta/csft-4.1/src/sphinxexpr.cpp
 1746                  T val = ExprEval ( this->m_pArg, tMatch );
 形式修改为  T val = this->ExprEval ( this->m_pArg, tMatch );
 1777                  T val = ExprEval ( this->m_pArg, tMatch );
 形式修改为  T val = this->ExprEval ( this->m_pArg, tMatch );
 1823                  T val = ExprEval ( this->m_pArg, tMatch );
 形式修改为  T val = this->ExprEval ( this->m_pArg, tMatch );
 错误4
 In file included from sphinxstd.cpp:24:0:
py_layer.h:16:27: fatal error: Python.h: No such file or directory
  #include   <Python.h>  
  这是由于缺少了python环境的devel支持包
  解决办法：yum install python-devel
）

./configure --prefix=/usr/local/coreseek --without-unixodbc --with-mmseg --with-mmseg-includes=/usr/local/mmseg3/include/mmseg/ --with-mmseg-libs=/usr/local/mmseg3/lib/ --with-mysql=/usr   （which mysql_config找到安装位置）
make && make install
```
* 配置
```
mv  /usr/local/coreseek/etc/sphinx.conf /etc/sphinx.conf
vim sphinx.conf:
(简化版的sphinx的配置)
source main_src
{
        type                    = mysql
        sql_host                = localhost 
        sql_user                = root
        sql_pass                = 123456
        sql_db                  = blog
        sql_port                = 3306  # optional, default is 3306
        sql_query_pre           = SET NAMES utf8
        sql_query               = \
                SELECT ar.id,ar.content_html,ar.title,UNIX_TIMESTAMP(ar.created_at) as created_at, ca.name  FROM articles  ar \
                        left join cates ca on ar.cate_id = ca.id;
        sql_attr_uint           = id
        sql_attr_timestamp      = created_at
        sql_ranged_throttle     = 0
        sql_query_info          = SELECT ar.*, ca.name  FROM articles ar left join cates ca\
                        on ar.cate_id = ca.id  WHERE ar.id=$id;
}

index main
{
        source                  = main_src
        path                    = /usr/local/coreseek/var/data/articles
        docinfo                 = extern
        mlock                   = 0
        morphology              = none
        min_word_len            = 1
        charset_type            = zh_cn.utf-8
        html_strip              = 0
        charset_dictpath        = /usr/local/mmseg3/etc/
        ngram_len               = 0
}

indexer
{
        mem_limit               = 32M
}
searchd
{
        listen                  = 9312
        listen                  = 9306:mysql41
        log                     = /usr/local/coreseek/var/log/searchd.log
        query_log               = /usr/local/coreseek/var/log/query.log
        read_timeout            = 5
        client_timeout          = 300
        max_children            = 30
        pid_file                = /usr/local/coreseek/var/log/searchd.pid
        max_matches             = 1000
        seamless_rotate         = 1
        preopen_indexes         = 1
        unlink_old              = 1
        mva_updates_pool        = 1M
        max_packet_size         = 8M
        max_filters             = 256
        max_filter_values       = 4096
        max_batch_queries       = 32
        workers                 = threads # for RT to work
}

建立索引命令
/usr/local/coreseek/bin/indexer -c /etc/sphinx.conf --all --rotate
启动命令
/usr/local/coreseek/bin/searchd -c /etc/sphinx.conf
搜索命令
/usr/local/coreseek/bin/search -c /etc/sphinx.conf 中国
停止命令
/usr/local/coreseek/bin/searchd -c /etc/sphinx.conf --stop

```

* 索引自动化建立
(项目下的crontab子目录已有,根据你实际情况修改)
建立shell脚本，主要用来执行sphinx建立索引，重启服务，同时会记录日志到该目录下的sphinx.log

```
cd crontab
vim indexer.sh:

killall -9 searchd 
/usr/local/coreseek/bin/indexer -c /etc/sphinx.conf --all --rotate && 
/usr/local/coreseek/bin/searchd -c /etc/sphinx.conf
time=`date "+%Y-%m-%d %H:%M:%S "`
echo "${time} sphinx restart success" >> /usr/share/nginx/html/crontab/sphinx.log

```
然后定时任务中每小时执行一次该脚本：

```
crontab -e
0 */1 * * * /usr/share/nginx/html/crontab/indexer.sh
service crond restart

```
配置完毕，搜索功能已经可以使用








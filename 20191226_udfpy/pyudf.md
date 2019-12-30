# 摸坑pyudf<br/>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;事件由来：上周突然出现了较多关于解码转码的需求，这是普通sql无法处理的。这时候就需要使用udf来进行转码，当时写完之后却无法调用，困难重重。于是就求助于晨阳哥来场外援助，一下午问题迎刃而解。<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了用熟悉的方法解决此类问题，
，记录一下python调用udf会出现的问题各种问题以及最终解决方法。



## udf定义


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;UDF函数(User Defined Function)其实就是一个简单的函数，执行过程就是在Hive转换成MapReduce程序后，执行java方法，类似于像MapReduce执行过程中加入一个插件，方便扩展。UDF只能实现一进一出的操作，如果需要实现多进一出，则需要实现UDAF。Hive可以允许用户编写自己定义的函数UDF，来在查询中使用。<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;那么当然目前不仅仅hive支持udf，spark同样有udf方法，并且支持多语言调用。<br/>


* * *

### udf实现

##### 1. 需求描述：
奇迹小说中需要按照所给的token url， 解析出其中加密的uid信息。

##### 2. 生成代码
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当接触类似解码等需求都可以调用第三方包快速实现
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br/>套用hive所要求的格式<br/>
for line in sys.stdin: &nbsp;  # &nbsp;按行读取<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;line = line.split('\t')<br/>
根据按行读取的方式
(默认分隔符tab)读取行信息通过split来判断是第几个字段。再对所需处理的字段进行处理后，print作为输出，print后的字段就作为处理后的结果。<br/>
这里我把秘钥写死了，其实完全可以将秘钥作为参数传入。

```python
# coding:utf-8
import jwt
import sys
# 将上面生成的 jwt 进行解析认证
def jwt_go(token,sk):
    #print(token)
    try:
        result = jwt.decode(token,sk, algorithms=['HS256']) # 解码方法
    except:
        pass
    return result

for line in sys.stdin:
    line = line.split('\t')
    udf_line = line[0].strip()
    sk1 = "c0GHFUERRm3YA9bj0NQTDsfOHlQ7RlFliW5Ln1Z5IWFd7KpsAAoK0QxyM8rTpLls" # 秘钥1
    sk2 = 'qiji-app' # 秘钥2
    try :
        try:
            print(jwt_go(udf_line,sk1)['uid']) #["uid"]
        except:
            print(jwt_go(udf_line, sk2)['uid'])  # ["uid"]
    except:
        print('error_token')
```
##### 3. 上传代码提交至hive调试
建表语句：建立一张测试表放入几条token。
```sql
SELECT 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NzM1MjE4OTksInVpZCI6MjU4MzAwMTIsInhpbWFfdWlkIjoxNTkyNDcyOTh9.1uASgrSet2jWssLbnJWV5Hb8HC1wvN2gVTg_lWL0CN4' id
UNION ALL
SELECT 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NzU2MzkwNzEsInVpZCI6MjgzMjMxNzcsInhpbWFfdWlkIjowfQ.acsNbEwYVmRhq7hzllmBQZbimOrvIWeiiQdYvJJcGo8' id
UNION ALL
SELECT 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NjM1OTA3MDEsInVpZCI6NjI1NjY2OCwieGltYV91aWQiOjE1OTQ5MDU2MH0.8_M_6DSx9mkMMyAUX4uo_ny-I6A1NM11UnT idqyMsW7mo' id
UNION ALL
SELECT 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NjQ2MTg4MDcsInVpZCI6MTk3NjgwMywieGltYV91aWQiOjExNTA5NjEzNH0.qCM2r7lvcLA5D1vR2V9DtJ35lJcOgQlkyjX693fzR8o' id
UNION ALL
SELECT 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NTk3ODE3NDEsInVpZCI6MTU5MTEwMDgsInhpbWFfdWlkIjowfQ.xviLo891Ez6Phrj7oWzbvl4hn-ezqA5N9Tb17ReQGaQ' id
UNION ALL
SELECT 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NzM1NjU0NjcsInVpZCI6MjYxMzQ3NDMsInhpbWFfdWlkIjowfQ.5vEXCg0p-7YWXNujB9wsj72Je6Bl0rg127MnmXhxI8U' id
UNION ALL
SELECT 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NjQ2Njg1NjcsInVpZCI6MjM4ODYzNCwieGltYV91aWQiOjEzMTc5NTk0MX0.5h21jPwpYoZj2DcQfuglcdwPnXzb_UEaM1H idmFm1GqCA' id
UNION ALL
SELECT 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NzMzODMyNzQsInVpZCI6MjU5NTIwOTMsInhpbWFfdWlkIjoxOTg0NDE1NDV9.cItlisbpoKfDYCaJVz8bc2etf-i8P_lmJaB1yulTl2I' id
as result;
save overwrite result as hive.`tmp.tmp_jwt_gyx_20191225`;
```

![a](./image/1_ceshiblack.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;实现将测试数据放入文本中在环境下调试下脚本。并且用临时表去测试函数。<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;结果正常。这就说明我们python脚本语法是正确的。<br/>
add file将本地或者hdfs文件添加至当前hive环境中。transform()为处理字段的方法，这里不局限于py可以使用各种语言。transform作为输出行在经过我们上传的函数处理之后返回单一或者多个结果。as 括号中的内容就是给print出的结果进行别名。

```sql
add file /home/dev/edible/udf/jwt_uid.py;
select transform(id) USING 'python jwt_uid.py'  AS (name) from tmp.tmp_jwt_gyx_20191225;
```
![b](./image/2_ceshihive.jpg)

这里可以看到，具体报错为<br/>An error occurred when trying to close the Operator running your custom script.<br/>
根据查询报错信息得到结论是集群环境没有第三方包引起的。<br/>
大致原因就是如下图所示
![b](./image/4_processon.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
由于using的分发机制。我们提交的主机去分发我们本地的脚本于执行的node上面，在具体执行的node上去调用using内的语句时候报错没有第三方模块导致的。但是，集群环境是我们无法控制的。不可能因为一个解码需求让运维将我们集群所有hive机器的默认python都安装pyjwt包，之后还有可能有MD5解码包，numpy等等。所以我们只有用上传资源来解决这个问题。

##### 4. 创建与上传资源
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;java可以打包为jar后所有依赖都随之移动。那么python则需要将整个资源全部上传，这样既消耗资源并且调用效率也低。这里就可以利用hadoop对于gz文件的可读性，小型python虚拟环境打包为tar.gz减小空间上传资源。<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一个轻量级的虚拟环境打包，conda是我们的不二选择。这里我就直接借用了20.96上的默认conda环境pytest(最小)作为我需要上传的环境。
```
cd ~/.conda/env/pytest
./pip install pyjwt # 为虚拟环境添加依赖包
tar -zcvf pytest.tar.gz ~/.conda/env/pytest # 打包
scp pytest.tar.gz dev@114.80.170.68:~/
scp pytest.tar.gz dev@192.168.17.104:~/指定路径 # 移动
```
进入hive上传打包环境文件，调用函数，成功输出结果。
```sql
add archive /home/dev/edible/pytest.tar.gz;
select transform(id) USING 'pytest.tar.gz/pytest/bin/python2.7 jwt_uid.py'  AS (name) from tmp.tmp_jwt_gyx_20191225;
```

![d](./image/3_ezsuccess.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可喜可贺可喜可贺，可以看到最终udf成功转码。<br/>
那么接下来可以将函数转换一下用相似的方式改到spark上运行。

```python
# coding:utf-8

from pyspark.shell import spark
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType
import jwt

def jwt_go(token,sk):
    #print(token)
    try:
        result = jwt.decode(token,sk, algorithms=['HS256']) # 解码方法
    except:
        pass
    return result

def jwt_sp(cl):
    sk1 = "c0GHFUERRm3YA9bj0NQTDsfOHlQ7RlFliW5Ln1Z5IWFd7KpsAAoK0QxyM8rTpLls" # 秘钥1
    sk2 = "qiji-app" # 秘钥2
    try :
        try:
            result = jwt_go(cl,sk1)['uid']
        except:
            result = jwt_go(cl,sk2)['uid']
    except:
        result = 'error_token'
    return result


print(jwt_sp("eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NzM1MjE4OTksInVpZCI6MjU4MzAwMTIsInhpbWFfdWlkIjoxNTkyNDcyOTh9.1uASgrSet2jWssLbnJWV5Hb8HC1wvN2gVTg_lWL0CN4"))
spark.udf.register('jwt_sp',jwt_sp)
print(spark.sql("""select jwt_sp("eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NzM1MjE4OTksInVpZCI6MjU4MzAwMTIsInhpbWFfdWlkIjoxNTkyNDcyOTh9.1uASgrSet2jWssLbnJWV5Hb8HC1wvN2gVTg_lWL0CN4") x """).collect())
```
使用pyspark 的来注册该函数并用pyspark.shell.spark.sql的方式调用该函数

![e](./image/5_localspark.jpg)

上图为本地测试环境中成功测试的结果，但是，在正式环境中一样需要为我们的分布式集群添加第三方模块。
当然spark提交环境与hive是相似的。

```shell
# 1.使用 archive提交整体环境
spark-submit --archive 'xx.zip' udf.py
# 2.使用 py-files提交模块环境
spark-submit --py-files a.zip,b.zip,c.zip udf.py
# 3.直接在py内部进行sc属性添加
pyFiles_1=["D:/pysparktest/package.zip"] #压缩的包的所在路径
pyFiles_2=["D:/pysparktest/add.py","D:/pysparktest/mult.py","D:/pysparktest/pandas.py"] #第二种
pyspark.SparkContext(pyFiles = pyFiles_1)
# 添加对象属性

```
##### 5. 目的以及总结
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;目前hive不支持非java语言注册永久函数 ，并且add archive不支持上传至hdfs，这就带来了非常大的局限性，但是MaxCompute(ODPS)已经完全支持python注册永久函数，所以不就的将来希望可以直接使用python编写可永久注册udf函数。

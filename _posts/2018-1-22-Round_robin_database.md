# RRD资料整理
rrd的原生命令基本都是写在一行内的，文中所有例子均是为了方便阅读进行的分行。


## **create命令**
ref:https://oss.oetiker.ch/rrdtool/doc/rrdcreate.en.html

    rrdtool create filename
    [--start|-b start time]
    [--step|-s step]
    [--template|-t template-file]
    [--source|-r source-file]
    [--no-overwrite|-O]
    [--daemon|-d address]
    [DS:ds-name[=mapped-ds-name[[source-index]]]:DST:dst arguments]
    [RRA:CF:cf arguments]

`[--start|-b start time]` 指定开始存储数据的时间戳，时间戳前的数据不会被接受。

`[--step|-s step]`指定检测以秒为单位的间隔时间默认300秒。

`[--template|-t template-file]`指定DS，RRA的template，额外的DS,RRA定义可以加到此template中。

`[--source|-r source-file]`可以指定一个或多个rrd源文件

`[--no-overwrite|-O]`不会覆盖现有同名文件。

`[--daemon|-d address]`指定rrdcached daemon address。

`[DS:variable_name:DST:heartbeat:min:max] ` 用于定义`Data Source`。也就是用于存放脚本的结果的变量名(DSN)。每隔一个step, DS的一个新的值就会到来，然后数据库就会被更新，这个值也被叫做 `PDP(Primary Data Point)`，范围由min,max来制定，如果在`heartbeat`时间内没有 收到数据，那么该PDP就会被置为`UNKNOWN`。`DST`即`Data Source Type`。有以下几种:

    COUNTER 与前一个值比较的变化率,只有正值。还有双进度浮点数的DCOUNTER。
    DERIVE 同为变化率,但是允许负值。还有双进度浮点数的DDERIVE。
    GUAGE 值本身,不涉及变化率。
    ABSOLUTE 变化率,默认前一个值为0。
    COMPUTE ？？？应该是自定义的计算方法

`[RRA:CF:xff:step:rows]`是对采集到的数据以某种方式(CF)的归档。`CF` 即 `Consolidation Function` 的缩写。也就是合并数据的方式。有 AVERAGE、MAX、MIN、LAST 四种分别表示对多个PDP 进行取平均、取最大值、取最小值、取当前值四种类型。在这里又有一个新的概念`CDP(consolidated data point)`, 一个CPD是把step个PDP按照指定的CF进行合并得到的一个数值，而这个RRA中包含rows个CDPs。这里的`xff`是指CDP合并时允许出现的UNKNOWN的最大比率。

**例子：**

    rrdtool create target.rrd \     
             --start 1023654125 \     
             --step 300 \      
             DS:mem:GAUGE:600:0:671744 \     
             RRA:AVERAGE:0.5:12:24 \     
             RRA:AVERAGE:0.5:288:31
了解了这些基本概念之后，上面的这个例子就比较容易理解了,首先给这个数据库命名为target.rrd，数据的开始时间是epoch时间1023654125，每隔300s获取一个PDP，然后DS制定了实际被监控的变量及其类型和值域，最后定义了两个RRA，对于第一个RRA, 12(steps)个PDPs以平均(CF)的方式进行合并得到一个CDP，24个这样的CDP组成一个RRA归档。PDP之间的间隔是300s,所以CDP之间的间隔就是12*300，即1小时，24个这样的CDP就是1天，因此我们可以得知，第一个RRA就是mem的一天的监控统计。

## **update命令**  
ref:https://oss.oetiker.ch/rrdtool/doc/rrdupdate.en.html

    rrdtool update car.rrd 1167019500:12345:7.0 1167019800:12357:5.8
    rrdtool update car.rrd 1167020100:12363:5.2 1167020400:12363:5.2
    rrdtool update car.rrd 1167020700:12363:5.2 1167021000:12373:4.2
update后接rrd文件，然后要更新的时间戳：数据，每条update可以更新多个数据，具体个数看各个os对命令长度限制。

## **graph命令**
ref:https://oss.oetiker.ch/rrdtool/doc/rrdgraph.en.html

    rrdtool graph filename
    [option ...] [data definition ...]
    [data calculation ...] [variable definition ...]
    [graph element ...] [print element ...]

## **dump命令**
将一个rrd数据库导出为xml文件     
ref:https://oss.oetiker.ch/rrdtool/doc/rrddump.en.html  

    rrdtool dump filename.rrd [filename.xml]
    [–header|-h {none,xsd,dtd}] [–no-header|-n]
    [–daemon|-d address] [> filename.xml]


## **fetch命令**
从一个rrd数据库根据条件取出数据  
ref:https://oss.oetiker.ch/rrdtool/doc/rrdfetch.en.html

    rrdtool fetch filename CF [–resolution|-r
    resolution] [–start|-s start] [–end|-e end] -e
    [–align-start|-a] [–daemon|-d address]

## **xport命令**
可以从若干个RRD中得到XML或JSON格式的数据。  
ref:https://oss.oetiker.ch/rrdtool/doc/rrdxport.en.html

    rrdtool xport [-s|–start seconds]
    [-e|–end seconds] [-m|–maxrows rows]
    [–step value] [–json] [–enumds]
    [–daemon|-d address] [DEF:vname=rrd:ds-name:CF]
    [CDEF:vname=rpn-expression] [XPORT:vname[:legend]]


## PHP RRDTool

**RRD Functions**  
* `rrd_create` — Creates rrd database file
* `rrd_error` — Gets latest error message.
* `rrd_fetch` — Fetch the data for graph as array.
* `rrd_first` — Gets the timestamp of the first sample from rrd file.
* `rrd_graph` — Creates image from a data.
* `rrd_info` — Gets information about rrd file
* `rrd_last` — Gets unix timestamp of the last sample.
* `rrd_lastupdate` — Gets information about last updated data.
* `rrd_restore` — Restores the RRD file from XML dump.
* `rrd_tune` — Tunes some RRD database file header options.
* `rrd_update` — Updates the RRD database.
* `rrd_version` — Gets information about underlying rrdtool library
* `rrd_xport` — Exports the information about RRD database.
* `rrdc_disconnect` — Close any outstanding connection to rrd caching daemon

**RRDCreator — The RRDCreator class**  
* `RRDCreator::addArchive` — Adds RRA - archive of data values for each data source.  
* `RRDCreator::addDataSource` — Adds data source definition for RRD database.  
* `RRDCreator::__construct` — Creates new RRDCreator instance  
* `RRDCreator::save` — Saves the RRD database to a file  

**RRDGraph — The RRDGraph class**  
* `RRDGraph::__construct` — Creates new RRDGraph instance
* `RRDGraph::save` — Saves the result of query into image
* `RRDGraph::saveVerbose` — Saves the RRD database query into image and returns the verbose information about generated graph.  
* `RRDGraph::setOptions` — Sets the options for rrd graph export  

**RRDUpdater — The RRDUpdater class**  
* `RRDUpdater::__construct` — Creates new RRDUpdater instance  
* `RRDUpdater::update` — Update the RRD database file  
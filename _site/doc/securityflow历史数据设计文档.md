# securityflow历史数据设计文档
## 1. 需求说明
middleware历史数据主要手机event数据并且聚合成middleware和cloudview可用的数据格式，并且存入数据库，以便后续显示。


## 2. 架构
logstash -> middleware -> 历史数据处理模块 -> DB
cloudview -> middleware -> 访问db -> 返回历史数据

## 3. 接口
websocket的request
```
{
  "channel": "flowsLogsSubscribe",
  "params": {
    "showProjectFilter": [],
    "src_ip":"",
    "src_port":"",
    "dst_ip":"",
    "dst_port":"",
    "protocal":"" # icmp,tcp
    "status":"" #accept,reject
  }
}
```
websocket的response
```
{
  "channel": "flowsLogsSubscribe",
    "data": {
        "flowsLogs": [
          {
            "destination_group_uuid": "bytex1",
            "destination_ip": "174.72.211.107",
            "destination_port": 291,
            "projectId":"64ee480747cce0c0748e31cdaca5a29a12e8ec1e",
            "protocol": "ICMP",
            "source_group_uuid": "bytex3",
            "source_ip": "91.204.84.28",
            "source_port": 7955,
            "duration_time":3000,
            "bytes_rate": 128,
            "packet_rate":2,
            "timestamp": 1505370885435
        }
    ]
  }
}
```

## 4. 数据流程
#### 4.1主流程：数据处理架构如下
```flow
st=>start: SecurityFlow(Event)
#end=>end: End
op1=>operation: 根据key从缓存中获取数据
cond1=>condition: 是否获取到数据
opt2=>operation: 翻译neutron port
opt3=>operation: 保存数据到缓存
opt4=>operation: 检查数据总数
cond2=>condition: 总数是否是4条
opt5=>end: 计算flow的流量等数据，并且保存到DB
opt6=>operation: 数据保存到缓存
st->op1->cond1
cond1(no)->opt2->opt3
cond1(yes)->opt4->cond2
cond2(yes)->opt5
cond2(no)->opt6
```

#### 4.2. 副流程：解决数据丢失等情况，会有一个循环的操作在执行
```flow
st=>start: 进程启动
st1=>operation: 获取当前缓存中的数据
end=>end: 结束循环
opt1=>operation: 循环key-value
opt2=>operation: 判断value中数据条数
cond1=>condition: 检查总数是否是奇数
opt3=>operation: 对比value中的time和当前time
cond2=>condition: 判断时间差是否超过10s
opt4=>operation: 结算数据
cond3=>condition: 数据总数是否是3条
opt5=>operation: 认定该条flow是reject
opt6=>operation: 认定该条flow是accept
cond4=>condition: 是否循环结束
endopt=>operation:  休眠15s      
subro=>subroutine: 
st->st1->opt1->opt2->cond1
cond1(yes)->opt3->cond2
cond2(yes)->opt4->cond4
cond4(yes)->endopt(right)->subro(right)->st1
```

## 5. 缓存中数据格式
|protocal                |key                                                                           |value      |
| ----                      | ----                                                                           | ----- |
|tcp/udp                |  `<src_ip>_<src_port>_<dst_ip>_<dst_port> `        |        [ tcp/udp ]   |
|icmp                     | ` <src_ip>_<dst_ip>_<id> `                                     |      [icmp]    |



---
title: 監控工具 Zabbix API 使用記錄
toc: true
date: 2018-11-27 00:36:51
categories: 
- Zabbix
tags: 
- zabbix
- API
thumbnail: /images/zabbix_api.png
---

此篇主要記錄為，在Openstack環境下觀察第三方監控工具Zabbix使用API獲取歷史數據。
<!-- more -->

## Zabbix API 監控架構
![](/images/zabbix_architecture.png)

## 透過 Python zabbix api & CURL

### 1. user.login方法獲取 zabbix server 的認證結果

#### curl 命令：

```bash=
curl -i -X POST -H 'Content-Type:application/json' -d '{"jsonrpc":
"2.0","method":"user.login","params":{"user":"admin","password":"zabbix"},"auth":
null,"id":0}' http://10.111.200.8/zabbix/api_jsonrpc.php
```
#### curl 命令運行結果：
```bash=
{"jsonrpc":"2.0","result":"df432b915671b77035d5e26f85c8bf5c","id":0}
```

or 

#### Python 腳本：
```bash=
$ vim auth.py 

#/usr/bin/env python2.7
#coding=utf-8
import json
import urllib2
# based url and required header
url = "http://10.111.200.8/zabbix/api_jsonrpc.php"
header = {"Content-Type":"application/json"}
# auth user and password
data = json.dumps(
{
   "jsonrpc": "2.0",
   "method": "user.login",
   "params": {
   "user": "admin",
   "password": "zabbix"
},
"id": 0
})
# create request object
request = urllib2.Request(url,data)
for key in header:
   request.add_header(key,header[key])
# auth and get authid
try:
   result = urllib2.urlopen(request)
except URLError as e:
   print "Auth Failed, Please Check Your Name AndPassword:",e.code
else:
   response = json.loads(result.read())
   result.close()
print"Auth Successful. The Auth ID Is:",response['result']

```
#### Python 腳本運行結果：
```bash=
$ python auth.py
Auth Successful. The Auth ID Is: df432b915671b77035d5e26f85c8bf5c
```
---

### 2. hostgroup.get方法獲取所有主機組 ID

#### curl 命令：

```bash=
$ curl -i -X POST -H 'Content-Type:application/json' -d '{"jsonrpc": "2.0","method":"hostgroup.get","params":{"output":["groupid","name"]},"auth":"1d152c36140c245665ec8dc717675fb1","id": 0}' http://10.111.200.8/zabbix/api_jsonrpc.php
```

#### curl 執行結果：
```bash=
{"jsonrpc":"2.0","result":[{"groupid":"7","name":"Ceph Cluster"},{"groupid":"6","name":"Ceph MONs"},{"groupid":"10","name":"Ceph OSDs"},{"groupid":"11","name":"Computes"},{"groupid":"9","name":"Controllers"},{"groupid":"5","name":"Discovered hosts"},{"groupid":"2","name":"Linux servers"},{"groupid":"12","name":"Load Balancers"},{"groupid":"8","name":"ManagedByPuppet"},{"groupid":"1","name":"Templates"}],"id":0}
```

or

#### python 腳本：

```bash=
$ vim get_hostgroup_list.py

#!/usr/bin/env python2.7
#coding=utf-8
import json
import urllib2
# based url and required header
url = "http://10.111.200.8/zabbix/api_jsonrpc.php"
header = {"Content-Type":"application/json"}
# request json
data = json.dumps(
{
   "jsonrpc":"2.0",
   "method":"hostgroup.get",
   "params":{
       "output":["groupid","name"],
   },
   "auth":"df432b915671b77035d5e26f85c8bf5c", # theauth id is what auth script returns, remeber it is string
   "id":1,
})
# create request object
request = urllib2.Request(url,data)
for key in header:
   request.add_header(key,header[key])
# get host list
try:
   result = urllib2.urlopen(request)
except URLError as e:
   if hasattr(e, 'reason'):
       print 'We failed to reach a server.'
       print 'Reason: ', e.reason
   elif hasattr(e, 'code'):
       print 'The server could not fulfill the request.'
       print 'Error code: ', e.code
else:
   response = json.loads(result.read())
   result.close()
   print "Number Of Hosts: ", len(response['result'])
   #print response
   for group in response['result']:
       print "Group ID:",group['groupid'],"\tGroupName:",group['name']
```

#### python 腳本執行結果：

```bash=
$ python get_hostgroup_list.py 
Number Of Hosts:  10
Group ID: 7 	GroupName: Ceph Cluster
Group ID: 6 	GroupName: Ceph MONs
Group ID: 10 	GroupName: Ceph OSDs
Group ID: 11 	GroupName: Computes
Group ID: 9 	GroupName: Controllers
Group ID: 5 	GroupName: Discovered hosts
Group ID: 2 	GroupName: Linux servers
Group ID: 12 	GroupName: Load Balancers
Group ID: 8 	GroupName: ManagedByPuppet
Group ID: 1 	GroupName: Templates
```
---

### 3. host.get方法獲取單個主機組下所有的主機 ID


#### curl 命令：
```bash=
$ curl -i -X POST -H'Content-Type: application/json' -d '{"jsonrpc":"2.0","method":"host.get","params":{"output":["hostid","name"],"groupids":"11"},"auth":"1d152c36140c245665ec8dc717675fb1","id": 0}' http://10.111.200.8/zabbix/api_jsonrpc.php
```

#### curl 命令執行結果：
```bash=
{"jsonrpc":"2.0","result":[
{"hostid":"54","name":"node-1.domain.tld"},
{"hostid":"55","name":"node-3.domain.tld"},
{"hostid":"56","name":"node-2.domain.tld"}],"id":0}
```

or 

#### python 腳本：
```bash=
#!/usr/bin/env python2.7
#coding=utf-8
import json
import urllib2
# based url and required header
url = "http://10.111.200.8/zabbix/api_jsonrpc.php"
header = {"Content-Type":"application/json"}
# request json
data = json.dumps(
{
   "jsonrpc":"2.0",
   "method":"host.get",
   "params":{
       "output":["hostid","name"],
       "groupids":"11",
   },
   "auth":"df432b915671b77035d5e26f85c8bf5c", # theauth id is what auth script returns, remeber it is string
   "id":1,
})
# create request object
request = urllib2.Request(url,data)
for key in header:
   request.add_header(key,header[key])
# get host list
try:
   result = urllib2.urlopen(request)
except URLError as e:
   if hasattr(e, 'reason'):
       print 'We failed to reach a server.'
       print 'Reason: ', e.reason
   elif hasattr(e, 'code'):
       print 'The server could not fulfill the request.'
       print 'Error code: ', e.code
else:
   response = json.loads(result.read())
   result.close()
   print "Number Of Hosts: ", len(response['result'])
   for host in response['result']:
       print "Host ID:",host['hostid'],"HostName:",host['name']
```


#### python 腳本執行結果：
```bash=
$ python get_group_one.py
Number Of Hosts:  3
Host ID: 54 HostName: node-1.domain.tld
Host ID: 55 HostName: node-3.domain.tld
Host ID: 56 HostName: node-2.domain.tld
```


### 4. itemsid.get方法獲取單個主機下所有的監控項 ID
根據標題 3 中獲取到的所有主機 id 與名稱，找到你想要獲取的主機 id ，獲取它下面的所有 items

#### curl 命令：

這邊選擇觀察 `Host ID: 54 HostName: node-1.domain.tld` 
```bash=
$ curl -i -X POST -H 'Content-Type:application/json' -d '{"jsonrpc":"2.0","method":"item.get","params":{"output":"itemids","hostids":"54"},"auth":"df432b915671b77035d5e26f85c8bf5c","id": 0}' http://10.111.200.8/zabbix/api_jsonrpc.php
```

#### curl 命令執行結果：

```bash=
{"jsonrpc":"2.0","result":[{"itemid":"572"},{"itemid":"573"},{"itemid":"574"},{"itemid":"416"},{"itemid":"417"},{"itemid":"418"},{"itemid":"419"},{"itemid":"420"},{"itemid":"421"},{"itemid":"422"},{"itemid":"423"},{"itemid":"458"},{"itemid":"459"},{"itemid":"460"},{"itemid":"3527"},{"itemid":"3541"},{"itemid":"3555"},{"itemid":"3528"},{"itemid":"3542"},{"itemid":"3556"},{"itemid":"3519"},{"itemid":"3533"},{"itemid":"3547"},{"itemid":"3517"},{"itemid":"3531"},{"itemid":"3545"},{"itemid":"3522"},{"itemid":"3536"},{"itemid":"3550"},{"itemid":"3518"},{"itemid":"3532"},{"itemid":"3546"},{"itemid":"3516"},{"itemid":"3530"},{"itemid":"3544"},{"itemid":"3524"},{"itemid":"3538"},{"itemid":"3552"},{"itemid":"3515"},{"itemid":"3529"},{"itemid":"3543"},{"itemid":"3520"},{"itemid":"3534"},{"itemid":"3548"},{"itemid":"3523"},{"itemid":"3537"},{"itemid":"3551"},{"itemid":"3525"},{"itemid":"3539"},{"itemid":"3553"},{"itemid":"3521"},{"itemid":"3535"},{"itemid":"3549"},{"itemid":"3526"},{"itemid":"3540"},{"itemid":"3554"},{"itemid":"4982"},{"itemid":"4986"},{"itemid":"4990"},{"itemid":"4154"},{"itemid":"4159"},{"itemid":"4164"},{"itemid":"4741"},{"itemid":"4745"},{"itemid":"4749"},{"itemid":"4888"},{"itemid":"4892"},{"itemid":"4896"},{"itemid":"4649"},{"itemid":"4653"},{"itemid":"4657"},{"itemid":"4863"},{"itemid":"4867"},{"itemid":"4871"},{"itemid":"4290"},{"itemid":"4294"},{"itemid":"4298"},{"itemid":"4481"},{"itemid":"4485"},{"itemid":"4489"},{"itemid":"4433"},{"itemid":"4437"},{"itemid":"4441"},{"itemid":"4933"},{"itemid":"4937"},{"itemid":"4941"},{"itemid":"4763"},{"itemid":"4767"},{"itemid":"4771"},{"itemid":"4981"},{"itemid":"4985"},{"itemid":"4989"},{"itemid":"4152"},{"itemid":"4157"},{"itemid":"4162"},{"itemid":"4740"},{"itemid":"4744"},{"itemid":"4748"},{"itemid":"4886"},{"itemid":"4890"},{"itemid":"4894"},{"itemid":"4651"},{"itemid":"4655"},{"itemid":"4659"},{"itemid":"4861"},{"itemid":"4865"},{"itemid":"4869"},{"itemid":"4292"},{"itemid":"4296"},{"itemid":"4300"},{"itemid":"4484"},{"itemid":"4488"},{"itemid":"4492"},{"itemid":"4436"},{"itemid":"4440"},{"itemid":"4444"},{"itemid":"4936"},{"itemid":"4940"},{"itemid":"4944"},{"itemid":"4766"},{"itemid":"4770"},{"itemid":"4774"},{"itemid":"4984"},{"itemid":"4988"},{"itemid":"4992"},{"itemid":"4151"},{"itemid":"4156"},{"itemid":"4161"},{"itemid":"4739"},{"itemid":"4743"},{"itemid":"4747"},{"itemid":"4887"},{"itemid":"4891"},{"itemid":"4895"},{"itemid":"4652"},{"itemid":"4656"},{"itemid":"4660"},{"itemid":"4862"},{"itemid":"4866"},{"itemid":"4870"},{"itemid":"4289"},{"itemid":"4293"},{"itemid":"4297"},{"itemid":"4483"},{"itemid":"4487"},{"itemid":"4491"},{"itemid":"4435"},{"itemid":"4439"},{"itemid":"4443"},{"itemid":"4935"},{"itemid":"4939"},{"itemid":"4943"},{"itemid":"4765"},{"itemid":"4769"},{"itemid":"4773"},{"itemid":"4983"},{"itemid":"4987"},{"itemid":"4991"},{"itemid":"4155"},{"itemid":"4160"},{"itemid":"4165"},{"itemid":"4742"},{"itemid":"4746"},{"itemid":"4750"},{"itemid":"4885"},{"itemid":"4889"},{"itemid":"4893"},{"itemid":"4650"},{"itemid":"4654"},{"itemid":"4658"},{"itemid":"4864"},{"itemid":"4868"},{"itemid":"4872"},{"itemid":"4291"},{"itemid":"4295"},{"itemid":"4299"},{"itemid":"4482"},{"itemid":"4486"},{"itemid":"4490"},{"itemid":"4434"},{"itemid":"4438"},{"itemid":"4442"},{"itemid":"4934"},{"itemid":"4938"},{"itemid":"4942"},{"itemid":"4764"},{"itemid":"4768"},{"itemid":"4772"},{"itemid":"4153"},{"itemid":"4158"},{"itemid":"4163"},{"itemid":"3569"},{"itemid":"3583"},{"itemid":"3597"},{"itemid":"3570"},{"itemid":"3584"},{"itemid":"3598"},{"itemid":"3561"},{"itemid":"3575"},{"itemid":"3589"},{"itemid":"3559"},{"itemid":"3573"},{"itemid":"3587"},{"itemid":"3564"},{"itemid":"3578"},{"itemid":"3592"},{"itemid":"3560"},{"itemid":"3574"},{"itemid":"3588"},{"itemid":"3558"},{"itemid":"3572"},{"itemid":"3586"},{"itemid":"3566"},{"itemid":"3580"},{"itemid":"3594"},{"itemid":"3557"},{"itemid":"3571"},{"itemid":"3585"},{"itemid":"3562"},{"itemid":"3576"},{"itemid":"3590"},{"itemid":"3565"},{"itemid":"3579"},{"itemid":"3593"},{"itemid":"3567"},{"itemid":"3581"},{"itemid":"3595"},{"itemid":"3563"},{"itemid":"3577"},{"itemid":"3591"},{"itemid":"3568"},{"itemid":"3582"},{"itemid":"3596"},{"itemid":"4994"},{"itemid":"4998"},{"itemid":"5002"},{"itemid":"4169"},{"itemid":"4174"},{"itemid":"4179"},{"itemid":"4753"},{"itemid":"4757"},{"itemid":"4761"},{"itemid":"4900"},{"itemid":"4904"},{"itemid":"4908"},{"itemid":"4661"},{"itemid":"4665"},{"itemid":"4669"},{"itemid":"4875"},{"itemid":"4879"},{"itemid":"4883"},{"itemid":"4302"},{"itemid":"4306"},{"itemid":"4310"},{"itemid":"4493"},{"itemid":"4497"},{"itemid":"4501"},{"itemid":"4445"},{"itemid":"4449"},{"itemid":"4453"},{"itemid":"4945"},{"itemid":"4949"},{"itemid":"4953"},{"itemid":"4775"},{"itemid":"4779"},{"itemid":"4783"},{"itemid":"4993"},{"itemid":"4997"},{"itemid":"5001"},{"itemid":"4167"},{"itemid":"4172"},{"itemid":"4177"},{"itemid":"4752"},{"itemid":"4756"},{"itemid":"4760"},{"itemid":"4898"},{"itemid":"4902"},{"itemid":"4906"},{"itemid":"4663"},{"itemid":"4667"},{"itemid":"4671"},{"itemid":"4873"},{"itemid":"4877"},{"itemid":"4881"},{"itemid":"4304"},{"itemid":"4308"},{"itemid":"4312"},{"itemid":"4496"},{"itemid":"4500"},{"itemid":"4504"},{"itemid":"4448"},{"itemid":"4452"},{"itemid":"4456"},{"itemid":"4948"},{"itemid":"4952"},{"itemid":"4956"},{"itemid":"4778"},{"itemid":"4782"},{"itemid":"4786"},{"itemid":"4996"},{"itemid":"5000"},{"itemid":"5004"},{"itemid":"4166"},{"itemid":"4171"},{"itemid":"4176"},{"itemid":"4751"},{"itemid":"4755"},{"itemid":"4759"},{"itemid":"4899"},{"itemid":"4903"},{"itemid":"4907"},{"itemid":"4664"},{"itemid":"4668"},{"itemid":"4672"},{"itemid":"4874"},{"itemid":"4878"},{"itemid":"4882"},{"itemid":"4301"},{"itemid":"4305"},{"itemid":"4309"},{"itemid":"4495"},{"itemid":"4499"},{"itemid":"4503"},{"itemid":"4447"},{"itemid":"4451"},{"itemid":"4455"},{"itemid":"4947"},{"itemid":"4951"},{"itemid":"4955"},{"itemid":"4777"},{"itemid":"4781"},{"itemid":"4785"},{"itemid":"4995"},{"itemid":"4999"},{"itemid":"5003"},{"itemid":"4170"},{"itemid":"4175"},{"itemid":"4180"},{"itemid":"4754"},{"itemid":"4758"},{"itemid":"4762"},{"itemid":"4897"},{"itemid":"4901"},{"itemid":"4905"},{"itemid":"4662"},{"itemid":"4666"},{"itemid":"4670"},{"itemid":"4876"},{"itemid":"4880"},{"itemid":"4884"},{"itemid":"4303"},{"itemid":"4307"},{"itemid":"4311"},{"itemid":"4494"},{"itemid":"4498"},{"itemid":"4502"},{"itemid":"4446"},{"itemid":"4450"},{"itemid":"4454"},{"itemid":"4946"},{"itemid":"4950"},{"itemid":"4954"},{"itemid":"4776"},{"itemid":"4780"},{"itemid":"4784"},{"itemid":"4168"},{"itemid":"4173"},{"itemid":"4178"},{"itemid":"461"},{"itemid":"357"},{"itemid":"509"},{"itemid":"462"},{"itemid":"565"},{"itemid":"567"},{"itemid":"356"},{"itemid":"412"},{"itemid":"498"},{"itemid":"499"},{"itemid":"429"},{"itemid":"463"},{"itemid":"464"},{"itemid":"465"},{"itemid":"466"},{"itemid":"467"},{"itemid":"468"},{"itemid":"469"},{"itemid":"470"},{"itemid":"471"},{"itemid":"472"},{"itemid":"473"},{"itemid":"474"},{"itemid":"475"},{"itemid":"476"},{"itemid":"477"},{"itemid":"478"},{"itemid":"479"},{"itemid":"480"},{"itemid":"481"},{"itemid":"482"},{"itemid":"483"},{"itemid":"484"},{"itemid":"485"},{"itemid":"486"},{"itemid":"487"},{"itemid":"488"},{"itemid":"489"},{"itemid":"3599"},{"itemid":"3600"},{"itemid":"3601"},{"itemid":"3602"},{"itemid":"490"},{"itemid":"3603"},{"itemid":"3604"},{"itemid":"3606"},{"itemid":"3605"},{"itemid":"3607"},{"itemid":"3611"},{"itemid":"3615"},{"itemid":"3619"},{"itemid":"3608"},{"itemid":"3612"},{"itemid":"3616"},{"itemid":"3620"},{"itemid":"3610"},{"itemid":"3614"},{"itemid":"3618"},{"itemid":"3622"},{"itemid":"3609"},{"itemid":"3613"},{"itemid":"3617"},{"itemid":"3621"},{"itemid":"491"},{"itemid":"5038"},{"itemid":"492"},{"itemid":"5031"}],"id":0}
```

or 

#### python 腳本：

```bash=
$ vim get_items.py

#!/usr/bin/env python2.7
#coding=utf-8
import json
import urllib2
# based url and required header
url = "http://10.111.200.8/zabbix/api_jsonrpc.php"
header = {"Content-Type":"application/json"}
# request json
data = json.dumps(
{
   "jsonrpc":"2.0",
   "method":"item.get",
   "params":{
       "output":["itemids","key_"],
       "hostids":"55",
   },
   "auth":"df432b915671b77035d5e26f85c8bf5c", # theauth id is what auth script returns, remeber it is string
   "id":1,
})
# create request object
request = urllib2.Request(url,data)
for key in header:
   request.add_header(key,header[key])
# get host list
try:
   result = urllib2.urlopen(request)
except URLError as e:
   if hasattr(e, 'reason'):
       print 'We failed to reach a server.'
       print 'Reason: ', e.reason
   elif hasattr(e, 'code'):
       print 'The server could not fulfill the request.'
       print 'Error code: ', e.code
else:
   response = json.loads(result.read())
   result.close()
   print "Number Of Hosts: ", len(response['result'])
   for host in response['result']:
       print host
       #print "Host ID:",host['hostid'],"HostName:",host['name']
```

#### python 腳本運行結果：
```bash=
$ python get_items.py 
Number Of Hosts:  435
… …
{u'itemid': u'469', u'key_': u'system.cpu.intr'}
{u'itemid': u'470', u'key_': u'system.cpu.load[percpu,avg15]'}
{u'itemid': u'471', u'key_': u'system.cpu.load[percpu,avg1]'}
{u'itemid': u'472', u'key_': u'system.cpu.load[percpu,avg5]'}
{u'itemid': u'473', u'key_': u'system.cpu.switches'}
{u'itemid': u'474', u'key_': u'system.cpu.util[,idle]'}
{u'itemid': u'475', u'key_': u'system.cpu.util[,interrupt]'}
{u'itemid': u'476', u'key_': u'system.cpu.util[,iowait]'}
{u'itemid': u'477', u'key_': u'system.cpu.util[,nice]'}
{u'itemid': u'478', u'key_': u'system.cpu.util[,softirq]'}
{u'itemid': u'479', u'key_': u'system.cpu.util[,steal]'}
{u'itemid': u'480', u'key_': u'system.cpu.util[,system]'}
{u'itemid': u'481', u'key_': u'system.cpu.util[,user]'}
… …
{u'itemid': u'491', u'key_': u'vm.memory.size[available]'}
{u'itemid': u'5038', u'key_': u'vm.memory.size[cached]'}
{u'itemid': u'492', u'key_': u'vm.memory.size[total]'}
{u'itemid': u'5031', u'key_': u'vm.memory.size[used]'}
… …
```
### 5. history.get方法獲取單個監控項的歷史數據



#### curl 命令：

```bash=
$ curl -i -X POST -H 'Content-Type:application/json' -d '{"jsonrpc":"2.0","method":"history.get","params":{"history":3,"itemids":"480","output":"extend","limit":10},"auth":"df432b915671b77035d5e26f85c8bf5c","id": 0}' http://10.111.200.8/zabbix/api_jsonrpc.php
```

> 有些值結果會顯示為0, 暫時還要查看`history.get`獲得歷史數據的參數設定
#### curl 命令運行結果：

```bash=
{"jsonrpc":"2.0","result":[{"itemid":"25154","clock":"1410744134","value":"4840","ns":"375754276"},{"itemid":"25154","clock":"1410744314","value":"5408","ns":"839852515"},{"itemid":"25154","clock":"1410744374","value":"7040","ns":"964558609"},{"itemid":"25154","clock":"1410744554","value":"4072","ns":"943177771"},{"itemid":"25154","clock":"1410744614","value":"8696","ns":"995289716"},{"itemid":"25154","clock":"1410744674","value":"6144","ns":"992462863"},{"itemid":"25154","clock":"1410744734","value":"6472","ns":"152634327"},{"itemid":"25154","clock":"1410744794","value":"4312","ns":"479599424"},{"itemid":"25154","clock":"1410744854","value":"4456","ns":"263314898"},{"itemid":"25154","clock":"1410744914","value":"8656","ns":"840460009"}],"id":0}
```
or 

#### python 腳本：
```bash=
$ get_items_history.py

#!/usr/bin/env python2.7
#coding=utf-8
import json
import urllib2
# based url and required header
url = "http://10.111.200.8/zabbix/api_jsonrpc.php"
header = {"Content-Type":"application/json"}
# request json

itemids_id = input('Pluse input itemids number: ')
print('id ', itemids_id)

data = json.dumps(
{
   "jsonrpc":"2.0",
   "method":"history.get",
   "params":{
       "output":"extend",
       "history":0,
       "itemids": itemids_id,
       "sortfield" : "clock",
       "limit": 10
   },
   "auth":"df432b915671b77035d5e26f85c8bf5c", # theauth id is what auth script returns, remeber it is string
   "id":1,
})
# create request object
request = urllib2.Request(url,data)
for key in header:
   request.add_header(key,header[key])
# get host list
try:
   result = urllib2.urlopen(request)
except URLError as e:
   if hasattr(e, 'reason'):
       print 'We failed to reach a server.'
       print 'Reason: ', e.reason
   elif hasattr(e, 'code'):
       print 'The server could not fulfill the request.'
       print 'Error code: ', e.code
else:
   response = json.loads(result.read())
   result.close()
   print "Number Of Hosts: ", len(response['result'])
   for host in response['result']:
       print host
       #print "Host ID:",host['hostid'],"HostName:",host['name']
```

#### python 腳本執行結果：

這邊觀察`{u'itemid': u'480', u'key_': u'system.cpu.util[,system]'}`狀態

```bash=
$ python get_items_history.py
Pluse input itemids number: 480
('id ', 480)
Number Of Hosts:  10
{u'itemid': u'480', u'ns': u'366027868', u'value': u'2.3541', u'clock': u'1542254040'}
{u'itemid': u'480', u'ns': u'409639017', u'value': u'2.3193', u'clock': u'1542254100'}
{u'itemid': u'480', u'ns': u'476390125', u'value': u'2.2133', u'clock': u'1542254160'}
{u'itemid': u'480', u'ns': u'609209011', u'value': u'2.1803', u'clock': u'1542254220'}
{u'itemid': u'480', u'ns': u'8006955', u'value': u'3.6697', u'clock': u'1542254281'}
{u'itemid': u'480', u'ns': u'676841696', u'value': u'5.0511', u'clock': u'1542254340'}
{u'itemid': u'480', u'ns': u'652330397', u'value': u'3.6032', u'clock': u'1542254400'}
{u'itemid': u'480', u'ns': u'567671572', u'value': u'3.1112', u'clock': u'1542254460'}
{u'itemid': u'480', u'ns': u'553946907', u'value': u'2.5180', u'clock': u'1542254520'}
{u'itemid': u'480', u'ns': u'496168653', u'value': u'2.3969', u'clock': u'1542254580'}

```

### 獲取下2018-11-21到2018-11-21期間的數據

```bash=
$ curl -i -X POST -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","method":"history.get","params":{"history":0,"itemids":["480"],"time_from":"1542844800.0","time_till":"1542758400.0" ,"output":"extend"},"auth": 
"df432b915671b77035d5e26f85c8bf5c","id": 0}' http://10.111.200.8/zabbix/api_jsonrpc.php
```




### #備註參考1:獲取對應監控項一段時間內的歷史數據並格式化輸出
* https://www.yangcs.net/posts/zabbix-api-introduce-and-use/

#### python 腳本:

```bash=
$ vim history_data.py
#!/usr/bin/env python
# encoding: utf-8

"""
Retrieves history data for a given numeric (either int or float) item_id
"""

from zabbix_api import ZabbixAPI
import pprint
from datetime import datetime
import time
server = "http://172.16.241.130/zabbix"
username = "Admin"
password = "zabbix"
zapi = ZabbixAPI(server=server)
zapi.login(username, password)
item_id = "23296"

# Create a time range
time_till = time.mktime(datetime.now().timetuple())
time_from = time_till - 60 * 60 * 24 * 10  # <span id="inline-toc">1.</span> days

# Query item's history (integer) data
history = zapi.history.get({"itemids": item_id, "time_from": time_from, "time_till": time_till, "output": "extend", "limit": "10"})

# If nothing was found, try getting it from history (float) data
if not len(history):
    history = zapi.history.get({"itemids": item_id, "time_from": time_from, "time_till": time_till, "output": "extend", "limit": "10", "history": 0})

for point in history:
        print("{0}: {1}".format(datetime.fromtimestamp(int(point['clock']))
                                .strftime("%x %X"), point['value']))
```
#### python 運行結果:

```bash=
$ python history_data.py
12/29/16 09:40:16: 0.4900
12/29/16 09:41:16: 0.1800
12/29/16 09:42:16: 0.1600
12/29/16 09:43:16: 0.2000
12/29/16 09:44:16: 0.0700
12/29/16 09:45:16: 0.1000
12/29/16 09:46:16: 0.2100
12/29/16 09:47:16: 0.0700
12/29/16 09:48:16: 0.5100
12/29/16 09:49:16: 0.2200
```

### #備註參考2:時間格式化輸出

```bash=
#!/usr/bin/python
# coding=utf-8

import time
import sys

while True:

  choose = input("請選擇轉換時間格式, 輸入1 : 時間轉秒, 輸入2: 秒轉時間: ")

  print ('Your input: ',choose )

  if choose==1 :
    turn_sec = raw_input ('請輸入轉換時間: 2017-04-13 00:00:00: ')
    print ('input: ',turn_sec)
    a = turn_sec
    #輸出: 1492012800.0
    print time.mktime(time.strptime(a,'%Y-%m-%d %H:%M:%S'))
  elif choose == 2:
    turn_date = float(raw_input('請輸入轉換時間: 1492012800.0: '))
    #输出： 2017-04-13 00:00:00
    print ('input: ',turn_date)
    x = time.localtime(turn_date)
    print time.strftime('%Y-%m-%d %H:%M:%S',x)
  else :
    print ("Inpute Error")
```



---

## 引用參考資料
* [[Zabbix API 使用]](https://www.jianshu.com/p/f5282dfc438c)
* [[python調用zabbix api接口實時展示數據]](http://blog.51cto.com/yangrong/1559123)
* [[zabbix API 獲取CPU 信息]](http://blog.51cto.com/swq499809608/1552912)
* [[Zabbix Api 简介和使用]](https://www.yangcs.net/posts/zabbix-api-introduce-and-use/)


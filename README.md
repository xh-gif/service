# service
做的时候用的是VMware虚拟机，虽然用的是容器，但也比较卡，不方便截图，所以就用文字记录了  
docker运行4个容器  
2个diretory做lvs+keepalive，且用Apache做sorry_service  
2个Nginx做后端服务（node1，node2）  
node1用docker run -it --privileged -p8080:80 镜像名  
node1用docker run -it --privileged -p9090:80 镜像名   
注意--privileged要加上，主要是因为lvs用dr模式，后端服务器需抑制ARP，改内核参数，这个privileged是给予容器特权才可以修改  
运行后使用service Nginx start开启服务  
编辑脚本  
```
#！/bin/bash
case $1 in
start)
	echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
	echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
	echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
	echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
	;;
stop)
	echo 0 > /proc/sys/net/ipv4/conf/all/arp_ignore
	echo 0 > /proc/sys/net/ipv4/conf/lo/arp_ignore
	echo 0 > /proc/sys/net/ipv4/conf/all/arp_announce
	echo 0 > /proc/sys/net/ipv4/conf/lo/arp_announce
	;;
esac
```
保存退出并且运行脚本使用命令‘bash 脚本名 start’   
然后cat  /proc/sys/net/ipv4/conf/all/arp_ignore 检查是否变为1，默认是0   
绑定vip地址：  
ifconfig lo:0 172.17.0.88 netmask 255.255.255.255 broadcast 172.17.0.88 up  
route add -host 172.17.0.88  
至此后端配置都完成  
  
调度器同样docker run 运行也是加--privileged  
然后yum install ipvsadm（注意宿主机也需先安装，不然容器安装后会出错，这是内核原因）  
设置vip  
ip addr add 172.17.0.88/32 dev eth0  
测试lvs  
ipvsadm -A -t 172.17.0.88:80 -s rr  
ipvsadm -a -t 172.17.0.88:80 -r 172.17.0.2 -g -w 1  
ipvsadm -a -t 172.17.0.88:80 -r 172.17.0.3 -g -w 2  
然后使用另一个未使用的容器curl http://172.17.0.88:80 疯狂测试  
成功则每次curl都能不报错，且2个后端内容都有显示过   
然后删除调度器刚才的配置  
ip addr del 172.17.0.88/32 dev eth0  
ipvsadm -C  
配置2台调度器  
yum install keepalived  
修改配置  
```
vim /etc/keepalived/keepalived.conf
global_defs{
       	notification_email{
       		root@localhost
       	}
      	 notification_email_from xxxxx@localhost
       	smtp_server 127.0.0.1
       	smtp_connect_timeout 30
      	router_id LVS_DEVEL  
}
vrrp_script chk_mt{
	script "[[ -f /etc/keepalive/down ]] && exit 1 || exit 0" 检查是否有这个文件，表示是否服务被down掉
	interval 1
	weight -20 
}
vrrp_instance V1_1 {
	state MASTER  #另一台就改为BACKUP
	interface eth0
                  virtual_router_id 60
	priority 100 #另一台backup就改低一点90即可
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 45afs4f6f45as #可随便设置，可用openssl命令生成

	}
	virtual_ipaddress{
		172.16.100.88/16 dev eth0 label eth:1
	}
	track_script {
		chk_mt #调用上面的脚本
	}

	#调用写的脚本，在后面会写
	notify_master "/etc/keepalive/notify.sh master"
	notify_backup "/etc/keepalive/notify.sh backup"
	notify_fault "/etc/keepalive/notify.sh fault"
}
#LVS配置
virtual_server 172.16.0.88 80 {
	delay_loop 6
	lb_algo wrr
	lb_kind DR
	nat_mask 255.255.0.0
	protocol TCP
	
	real_server 172.17.0.2 80 {
		weight 1
		HTTP_GET {
			url {
				path /
				status_code 200
			}
			connect_timeout 3
			nb_get_retry 3
			delay_before_retry 3
		}
	}
	real_server 172.17.0.3 80 {
		weight 1
		HTTP_GET {
			url {
				path /
				status_code 200
			}
			connect_timeout 3
			nb_get_retry 3
			delay_before_retry 3
		}
}
}
```
编写脚本  
```
#!/bin/bash
vip = 172.17.0.88
contact='root@localhost'

notify(){
	mailsubject="`hostname` to be $1 : $vip floating"
	mailbody="`date '+%F %H:%M:%S'`:vrrp transition, `homename` changed to be $1"
	echo $mailbody |mail -s "$mailsubject" $contact
}
case "$1" in
	master)
		notify master
		exit 0
		;;
	backup)
		notify backup
		exit 0
		;;
	fault)
		notify fault
		exit 0
		;;
	*)
		echo "Usage :`basename $0 `{master|backup|fault}"
		exit 1
		;;
esac

```
保存退出  
chmod +x notify.sh 给予执行权限  
service keepalived start   
使用mail可以看到脚本的文件  
ipvsadm -L -n检查是否生成相应的lvs信息  
ip addr list检查是否生成相应的vip  
然后可以使用一台未用的容器curl :http://172.17.0.88检查  
在其中一台Nginx关掉服务，然后检查  
 在调度器keepalived.conf中virtual_server 里加sorry_server 127.0.0.1 80表示后端服务器全宕机时会转到这个keepalived的地址  
于是需要安装Apache   
yum install -y httpd  
service httpd start   
这个就是sorry server   
尝试关掉其中一个keepalived   
然后看看ip addr list 里有没有转移到另一个keepalived  
也可看看service keepalived status  
上面很多细节不注意就会出现错误的情况  

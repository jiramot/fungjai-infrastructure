## เมื่อ Server ใหม่มาถึง

​	และแล้วเราก็ได้ server ใหม่สักที ถึงตอนนี้ ฟังใจเราเริ่มมีแยก service ต่างๆออกมามากขึ้น เช่นเดิมที web ที่เราเขียนด้วย Grails เป็นทั้ง frontend และ backend เราก็แยกมันออกมาเป็น 2 ส่วนโดยใช้ model ชุดเดิม แล้วก็เพิ่มบริการ Artist upload เพื่อเป็น Self service สำหรับศิลปินให้อัพโหลดเพลงเข้าสู่ระบบได้เลยโดยที่ไม่ต้องผ่านเรา และเพิ่ม API ที่เขียนด้วย NodeJS (Restify framework) เพื่อเป็น API สำหรับ Mobile Application จากนั้นก็เพิ่ม Nginx เพื่อดึง static file เช่นรูปไปใช้งานทั้งสำหรับ Web Frontend และ Mobile Application ที่ทำเช่นนี้เนื่องจาก ส่วนมากที่เราเจอคือเวปจะตาย แต่ Mobile ไม่ตาย ถ้าแบบเดิม เรา render รูปของเวปผ่าน Controller ของ Grails ผลก็คือ รูปจะหายหมดเลยถ้าเวปตายแค่อย่างเดียว

### ทำไมเราต้องแยก Backend ออกจาก Frontend

​	เรามองว่า Backend ซึ่งเราใช้คนเดียวมันไม่มีความจำเป็นต้อง scale ถ้าเราจะ scale frontend ทำไมเราต้องเอา backend เราไป scale ด้วย เราจึงพยายามแยกออกมาให้เล็กที่สุดเพื่อแยกการ scale นั่นเอง

```
┌───────────────────────────────────────────────────────────────────┐
│ ┌────────┐┌────────┐┌────────┐┌─────────┐┌─────────┐┌───────────┐ │
│ │  Web   ││Backend ││ Artist ││   API   ││Streaming││Static file│ │
│ │        ││        ││ upload ││(Restify)││ Engine  ││  (nginx)  │ │
│ └────────┘└────────┘└────────┘└─────────┘└─────────┘└───────────┘ │
│ ┌───────────────────────────────────────┐┌──────────────────────┐ │
│ │                 MySQL                 ││      GlusterFS       │ │
│ │                                       ││       Storage        │ │
│ └───────────────────────────────────────┘└──────────────────────┘ │
│                        Fungjai services (node)                    │
└───────────────────────────────────────────────────────────────────┘
```

​	อย่างที่บอก เราพยายามทำให้ทุกอย่างออกเป็น 2 node และเพื่อกระจายทำ Load balance ณ ตอนแรก เราเลือกใช้ HAProxy เนื่องจากมัน forward package ได้หลายแบบ รอบรับการทำ Load balance สำหรับ Steaming  Engine หรือแม้แต่ MySQL ก็สามารถทำได้ด้วย 

#### HAProxy ทำ Loadbalance

```
      192.168.0.1        
      ┌──────────┐       
      │ HAProxy  │       
      │   (LB)   │       
      └───┬──┬───┘       
        ┌─┘  └─┐         
        │      │         
 ┌──────▼──┐┌─▼────────┐
 │   Node   ││   Node   │
 │    1     ││    2     │
 └──────────┘└──────────┘
  10.0.0.1     10.0.0.2  
```

### ถ้า HAProxy ตาย พังหมด

​	เราอุตส่าห์พยายามทำให้ทุกอย่างเป็น Cluster เรามีตั่ง 2 เครื่อง แต่ถ้ามี HAProxy ทำ Load balance แค่ตัวเดียว ถ้าเครื่อง Server ตัวใดตัวหนึ่งของเราเกิดพังขึ้นมาก็ตายหมดอยู่ดี เราเลยทำให้ Load balance เราเป็น 2 ตัว

```
         ┌──────────┐                                              
         │   DNS    │                                              
         │          │                                              
         └──┬────┬──┘                                              
            │    │  fungjai.com. 50 IN  A   192.168.1.1            
            │    │  fungjai.com. 50 IN  A   192.168.1.2            
            │    │                                                 
            │    │                                                 
192.168.1.1 │    │ 192.168.1.2                                     
   ┌────────▼─┐ ┌▼─────────┐                                       
   │ HAProxy  │ │ HAProxy  │                                       
   │  (LB 1)  │ │  (LB 2)  │                                       
   └─┬─────┬──┘ └─┬──────┬─┘                                       
     │     │      │      │                                         
     │   ┌─┼──────┘      │                                         
     │   │ └────────┐    │                                         
     │   │          │    │                                         
   ┌─▼───▼──┐  ┌──▼────▼─┐                                       
   │   Node   │ │   Node   │                                       
   │    1     │ │    2     │                                       
   └──────────┘ └──────────┘                                       
```

#### มี 2 Load balance เท่ากับมี 2 public IP

​	อ่าวเชี่ยละ มี 2 public ip เราจะทำยังไงกับมันดี ความรู้กากๆของเราก็เคยจำได้ว่า Google มันมีหลาย IP นี้นา

ลอง dig google ดิ
```
➜  ~ dig google.com +noall +answer

; <<>> DiG 9.8.3-P1 <<>> google.com +noall +answer
;; global options: +cmd
google.com.		46	IN	A	110.164.10.40
google.com.		46	IN	A	110.164.10.29
google.com.		46	IN	A	110.164.10.59
google.com.		46	IN	A	110.164.10.35
google.com.		46	IN	A	110.164.10.34
google.com.		46	IN	A	110.164.10.49
google.com.		46	IN	A	110.164.10.55
google.com.		46	IN	A	110.164.10.44
google.com.		46	IN	A	110.164.10.24
google.com.		46	IN	A	110.164.10.45
google.com.		46	IN	A	110.164.10.25
google.com.		46	IN	A	110.164.10.30
google.com.		46	IN	A	110.164.10.54
google.com.		46	IN	A	110.164.10.39
google.com.		46	IN	A	110.164.10.50
google.com.		46	IN	A	110.164.10.20
```

​	จริงๆตอนแรกเราก็คิดว่า เราอยากได้บริการ DNS ให้เป็น [Round-Robin Load balance](https://www.digitalocean.com/community/tutorials/how-to-configure-dns-round-robin-load-balancing-for-high-availability) แต่ก็ไม่อยากทำเอง พอไปงมๆ DNS Setting ของพวกพวกที่เราซื้อ Domain ลองเซ็ต A record ไป หลายตัว มันก็ได้นะ ลอง dig ดูก็ได้หลาย IP จริงๆ แต่ DNS มันไม่รู้ว่า HAProxy เราตายหรือไม่ตาย มันก็แค่ Round-Robin ไปเรื่อยๆ ก็ไม่น่าจะใช้สิ่งที่เราอยากได้อยู่ดี 

​	ส่ิงที่เราอยากได้คือ อะไรก็ได้ที่จะเป็นตัวกลางสักตัวที่ ถ้า HAProxy ตัวใดตัวหนึ่งตาย ก็ย้ายไปใช้อีกตัวค้นหาไปค้นหามา ถามคนโน้นคนนี้ไปเรื่อย จนเจอ [Tutorial ในเวปของ DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-highly-available-web-servers-with-keepalived-and-floating-ips-on-ubuntu-14-04) เจ้าเก่า เค้าใช้พูดถึง [Keepalived](http://www.keepalived.org/) เราจึงเลือก Keepalived มาแก้ปัญหาในข้างต้น

```
         ┌──────────┐                                              
         │   DNS    │                                              
         │          │                                              
         └──┬────┬──┘                                              
            │    │  fungjai.com. 50 IN  A 192.168.1.3              
            │    │                                                 
            │    │                                                 
192.168.1.1 │    │ 192.168.1.2                                     
192.168.1.3 │    │ 192.168.1.3                                     
   ┌──────────┐ ┌──────────┐         Keepalived                    
   │ HAProxy  │ │ HAProxy  │       for Virtual IP                  
   │  (LB 1)  │ │  (LB 2)  │        192.168.1.3                    
   └─┬─────┬──┘ └─┬──────┬─┘                                       
     │     │      │      │                                         
     │   ┌─┼──────┘      │                                         
     │   │ └────────┐    │                                         
     │   │          │    │                                         
   ┌─▼───▼───┐ ┌──▼────▼─┐                                       
   │   Node   │ │   Node   │                                       
   │    1     │ │    2     │                                       
   └──────────┘ └──────────┘                                       
```

​	เราใช้ Keepalived  มาทำ Virtual IP เข้ามาอีกตัว หมายความว่า เครื่อง HAProxy แต่ละเครื่องจะมี IP 2 ชุด เช่นในรูป LB1 มี IP 192.168.1.1 และ 192.168.1.3 ซึ่ง IP 192.168.1.3 นี้เองเป็น Virtual IP ที่มาจาก Keepalived และเช่นกัน เครื่อง LB2 ก็มี Virtual IP เป็น 192.168.1.3 เหมือนกับเครื่อง LB1

​	จากนั้นเราก็เอา Virtual IP อันนี้ไปใส่ที่ DNS ตามปกติ dig ดูก็จะมองเห็นแค่ IP เดียวที่เป็น Virtual IP ของ Keepalived

​	ความสามารถของ Keepalived คือถ้าเครื่องใดเครื่องหนึ่งตาย มันก็จะเปลี่ยน Virtual IP ไปเป็นอีกเครื่อง ซึ่งเท่ากับว่าถ้า ping 192.168.1.3 ตอนแรกก็จะเข้าไปที่ LB1 แต่ถ้าหากเครื่อง LB1 ตาย เครื่อง LB2 ก็จะเปลี่ยนตัวเองมาเป็น IP 192.168.1.3 เอง แล้ว Request ก็จะไปที่ LB2 เอง

> ลองไปอ่าน Document เพิ่มเติมกันได้ เราเองก็ไม่ได้เก่งมาก ลองอะไรไปเรื่อยอย่างที่บอก เราเองไม่มีคนที่มาจาก Ops จริงๆสักคนเลย อาศัยถามเพื่อนๆและ Google ไปเรื่อยๆ



#### เจอ Bug ของ ESXi 6

​	ตอนแรกเราเลือกที่จะแบ่ง VM ออกเป็นหลายๆเครื่องตามแต่ละ Service เราใช้ VMWare ESXi 6 แต่ตอนนั้นลงแล้วมีปัญหา เจอ BUG ตัวใหญ่ๆของ ESXi6 เราจึง เปลี่ยนมาใช้ ESXi 5.5 แทน ตอนนั้นงมอยู่ 1 week แก้ไม่หาย คือเรา remote ไปที่ ESXi ไม่ได้เลย

> จริงๆแล้ว เราอยากใช้ ESXi ที่มี VCenter แต่มันแพงมากๆ เลยไม่ได้ใช้ ข้อดีของมันคือเราใช้ Vagrant คุมการสร้าง VM ของมันได้เลย (คล้ายๆ Docker machine )

### Ansible + Vagrant

​	จากการ design ทั้งหมดที่เขียนมา จากนั้นเราก็เริ่มเขียน Infrastructure ทั้งหมดด้วย [Ansible](https://www.ansible.com/) แล้วเทสด้วย [Vagrant](https://www.vagrantup.com/) โดยเราเลือกที่จะไม่ใช้ Ansible Galaxy เลยในตอนแรก เพราะว่า เราอยากลองเขียนมันเองโดยทั้งหมด เราเขียนทุก Role เองโดยการ Google ลองหา Tutorial การลองในวิธีปกติ จากนั้นลองมาเขียนเป็น Ansible Role แล้วลองเทสด้วย Vagrant ทีละ Role ไปเรื่อยๆจนหมด ความยากของมันคือการเทส Cluster ด้วยนี่สิ

```
├── README.md
├── Vagrantfile
├── ansible.cfg
├── backend.yml
├── bootstrap.sh
├── bootstrap_ci.sh
├── ci_master.yml
├── db.yml
├── docker
│   ├── mongo
│   └── registry
├── docker.yml
├── fjz.yml
├── gitlab.yml
├── group_vars
│   └── all
├── ha.yml
├── inventory
│   ├── group_vars
│   ├── host
│   └── host_vars
├── key
├── lab.yml
├── metric.yml
├── node.yml
├── openstack.yml
├── playbook.yml
├── production
├── registry.yml
├── roles
│   ├── android
│   ├── ansible
│   ├── apache-php
│   ├── artifactory
│   ├── collectd
│   ├── common
│   ├── dhcp
│   ├── dns-client
│   ├── dns-server
│   ├── docker
│   ├── docker-registry
│   ├── elk
│   ├── ffmpeg
│   ├── fpm
│   ├── git
│   ├── gitlab
│   ├── glusterfs
│   ├── glusterfs-client
│   ├── golang
│   ├── gradle
│   ├── grafana
│   ├── grails
│   ├── ha
│   ├── influxdb
│   ├── java
│   ├── jenkins
│   ├── jetty
│   ├── keepalived
│   ├── mariadb
│   ├── mongodb
│   ├── nginx
│   ├── nginx-compile
│   ├── nginx-static
│   ├── nodejs
│   ├── openssl
│   ├── openstack
│   ├── openvpn
│   ├── pptp
│   ├── python
│   ├── redis
│   ├── statsd
│   └── template_role
└── store.yml
```

​	จากนั้นก็เริ่ม Deploy เข้า Production จริง จุดที่เราต้อง Manual ก็คือ เราต้องใช้ ESXi client เพื่อ remote ไปยัง ESXi Server ของเราเพื่อทำการสร้าง VM ขึ้นมา จากนั้นก็ลง OS แล้ว Fix IP ต่างๆ เมื่อเสร็จเราก็ใช้ Ansible ต่อได้เลย เพื่อให้ Ansible ทำการ remote ไปติดตั่ง Package ตามที่เรากำหนดไว้ในแต่ละเครื่อง

> ถ้ามี VCenter เราก็จะใช้ API ของ VCenter ต่อกับ Vagrant เพื่อให้ Vagrant ไป Provision เครื่องตามค่าต่างๆแทนการ Manual ในข้างต้น เมื่อเสร็จ Vagrant จะทำการ Provision ตาม script vagrant ของเราให้เอง



**ทุกอย่างจบลงอย่างสวยงามจนกระทั่งการมาถึงของ [Exclusive single ของ Polycat Alright](https://www.fungjai.com/artist/Polycat/3706)**



**[<<  การมาของ Mobile App](fungjai-infra-2.md) | [เมื่อ Jenkins ช่วยชีวิต >>](ci.md)** 




[![Analytics](https://ga-beacon.appspot.com/UA-79032210-1/ep3?pixel)](new-server-comming.md)
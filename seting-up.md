# My CNI

### 🔍 สรุปจุดที่พลาดไป (Lesson Learned)

จากการแก้ปัญหาที่ผ่านมา มี **3 จุดหลัก** ที่เป็นอุปสรรคสำคัญ ซึ่งคุณสามารถจดไว้สรุปในรายงานได้ครับ:

1. **การลืม `no ip routing` บน Switch (L2):**
    - **ปัญหา:** Switch (ซึ่งเป็น L3 Image) พยายามทำตัวเป็น Router เอง ทำให้มัน "เมิน" (Ignore) คำสั่ง `ip default-gateway` ที่เราตั้งไว้ ส่งผลให้ Ubuntu2 ปิงมาหา Switch ไม่เจอ เพราะ Switch หาทางส่งข้อมูลกลับข้าม Subnet ไม่ถูก
    - **การแก้ไข:** สั่ง `no ip routing` เพื่อบังคับให้ Switch ทำงานเป็น Layer 2 และยอมใช้ Default Gateway ไปหา Router2
2. **ลืมตั้ง Default Route (`0.0.0.0/0`) ที่ Router 2:**
    - **ปัญหา:** ในตอนแรกเครื่องใน VLAN A/B ปิงออกเน็ต (1.1.1.1) ไม่ได้ เพราะ Router 2 ไม่รู้ว่าถ้าจะไป IP นอกเหนือจากที่รู้จัก (Internet) ต้องส่งไปทางไหน
    - **การแก้ไข:** เพิ่ม `ip route 0.0.0.0 0.0.0.0 192.168.1.1` เพื่อชี้ทางออกไปหา Router 1
3. **การตั้งค่า NAT ที่ Router 1:**
    - **ปัญหา:** ในช่วงแรก Packet วิ่งไปถึง Router 1 แล้ว แต่ไม่มีการแปลง IP (NAT) ทำให้ Packet ออกไปโลกภายนอกไม่ได้ (หรือกลับมาไม่ได้)
    - **การแก้ไข:** ตรวจสอบ `ip nat inside/outside` และเช็ค Access List ให้ครอบคลุมทุก Subnet (`172.61.0.0/16`)

### 2. Checklist สิ่งที่ต้องทำ (Component Checklist)

นี่คือลำดับการ Config เพื่อให้ไม่พลาดครับ:

**Router 1 (Main Gateway & DHCP Server)**

- [ ]  Config IP ขา LAN (เชื่อมไป R2)
- [ ]  Config IP ขา WAN (เชื่อม Cloud) *ส่วนใหญ่รับ DHCP จาก Cloud*
- [ ]  สร้าง DHCP Pool สำหรับ VLAN B (พร้อม Excluded Address เพื่อให้ได้ IP ช่วงที่กำหนด)
- [ ]  Config Static Route ไปหา Network 172.61.x.x (โยนไปหา R2)
- [ ]  Config NAT (เพื่อให้ Ubuntu Ping 1.1.1.1 ได้)

**Router 2 (Router on a Stick & Relay)**

- [ ]  Config IP ขา Uplink (เชื่อมไป R1)
- [ ]  Config Default Route (0.0.0.0/0) ชี้ไป R1
- [ ]  เปิด Interface ขาที่ต่อ Switch (no shut) แต่ไม่ต้องใส่ IP
- [ ]  สร้าง Sub-interface สำหรับ VLAN A (Encapsulation dot1Q 245) + ใส่ IP Gateway
- [ ]  สร้าง Sub-interface สำหรับ VLAN B (Encapsulation dot1Q 199) + ใส่ IP Gateway
- [ ]  ใส่คำสั่ง `ip helper-address` ที่ Sub-interface VLAN B (ชี้ไปหา R1)

**Switch (L2 Configuration)**

- [ ]  สร้าง VLAN 245 และ 199
- [ ]  Config Port ที่ต่อ R2 เป็น Trunk (Allow VLAN 245, 199)
- [ ]  Config Port ที่ต่อ Ubuntu1 เป็น Access VLAN 245
- [ ]  Config Port ที่ต่อ Ubuntu2 เป็น Access VLAN 199
- [ ]  Config Interface VLAN 245 (SVI) เพื่อใส่ IP Management

**Ubuntu 1 & 2**

- [ ]  Ubuntu1: Fix Static IP
- [ ]  Ubuntu2: ตั้งค่าเป็น DHCP Client

### 3. วิธีทำโดยละเอียด (Step-by-Step Configuration)

*(สมมติว่าใช้อุปกรณ์ Cisco IOS สำหรับ Router/Switch และ Linux สำหรับ Ubuntu)*

### **Step 1: Router 1 Configuration**

*หน้าที่: ออกเน็ต, เป็น DHCP Server ให้ Ubuntu2, และ Route กลับมาหา VLAN ต่างๆ*

Bash

`conf t

! 1. ตั้งค่า Interface ไปหา Router 2 (สมมติว่าเป็น e0/1 ตาม Topology จริง หรือ eth_0 ตามตาราง)
! **เช็คสายใน GNS3 ให้ดีว่าพอร์ตไหนต่อ R2**
interface e0/1
 description Link_to_R2
 ip address 192.168.1.1 255.255.255.0
 no shut
 ip nat inside  ! ขานี้คือภายใน
 exit

! 2. ตั้งค่า Interface ออก Cloud (สมมติว่าเป็น e0/0)
interface e0/0
 description Internet_Cloud
 ip address dhcp   ! รับ IP จาก Cloud
 no shut
 ip nat outside ! ขานี้คือภายนอก
 exit

! 3. สร้าง DHCP Pool สำหรับ VLAN B (Ubuntu2)
! โจทย์: Range Host 5-10 (IP .197 ถึง .202)
! เราต้องกัน IP อื่นออก (Exclude)
ip dhcp excluded-address 172.61.199.193 172.61.199.196  ! กัน Gateway ถึง Host 4
ip dhcp excluded-address 172.61.199.203 172.61.199.254  ! กัน Host 11 ถึงสุดท้าย

ip dhcp pool VLAN_B_POOL
 network 172.61.199.192 255.255.255.192
 default-router 172.61.199.193
 dns-server 192.168.1.1  ! โจทย์บอกให้ใช้ DNS คือ R1 (หรือจะใช้ 8.8.8.8 ถ้า R1 ไม่ได้ทำ DNS service)
 ! หมายเหตุ: ถ้า R1 ไม่ได้เปิด Service DNS จริงๆ Ubuntu2 อาจ Ping ชื่อไม่ได้ แต่ Ping IP 1.1.1.1 ได้
 exit

! 4. Static Route กลับไปหา VLAN A และ B (ผ่าน R2)
ip route 172.61.245.128 255.255.255.224 192.168.1.254
ip route 172.61.199.192 255.255.255.192 192.168.1.254

! 5. Config NAT (ให้ออกเน็ตได้)
access-list 1 permit 192.168.1.0 0.0.0.255
access-list 1 permit 172.61.245.128 0.0.0.31
access-list 1 permit 172.61.199.192 0.0.0.63
ip nat inside source list 1 interface e0/0 overload`

### **Step 2: Router 2 Configuration**

*หน้าที่: เป็น Gateway ของ VLANs (Inter-VLAN Routing) และส่งต่อ DHCP Request*

Bash

`conf t

! 1. ตั้งค่า Interface ไปหา Router 1
interface e0/1
 description Link_to_R1
 ip address 192.168.1.254 255.255.255.0
 no shut
 exit

! 2. ตั้งค่า Default Route วิ่งไป R1
ip route 0.0.0.0 0.0.0.0 192.168.1.1

! 3. Router on a Stick (ขาไปหา Switch - สมมติ e0/0)
interface e0/0
 no shut
 exit

! Sub-interface VLAN A
interface e0/0.245
 encapsulation dot1Q 245
 ip address 172.61.245.129 255.255.255.224
 exit

! Sub-interface VLAN B (มี DHCP Relay)
interface e0/0.199
 encapsulation dot1Q 199
 ip address 172.61.199.193 255.255.255.192
 ip helper-address 192.168.1.1  ! ส่งคำขอ DHCP ไปหา R1
 exit`

### **Step 3: Switch Configuration**

*หน้าที่: แยก VLAN และจัดการ Trunk*

Bash

`conf t

! 1. สร้าง VLAN
vlan 245
 name VLAN_A
vlan 199
 name VLAN_B
exit

! 2. Config Trunk ไปหา Router 2 (สมมติ e0/0)
interface e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 245,199
 no shut
 exit

! 3. Access Port - Ubuntu 1 (VLAN A) (สมมติ e0/1)
interface e0/1
 switchport mode access
 switchport access vlan 245
 no shut
 exit

! 4. Access Port - Ubuntu 2 (VLAN B) (สมมติ e0/2)
interface e0/2
 switchport mode access
 switchport access vlan 199
 no shut
 exit

! 5. Management IP (SVI)
interface vlan 245
 ip address 172.61.245.158 255.255.255.224
 no shut
 exit
! ใส่ Gateway ให้ Switch (เผื่อ Ping ข้ามวง)
ip default-gateway 172.61.245.129`

`conf t

! 1. สร้าง VLAN
vlan 245
 name VLAN_A
vlan 199
 name VLAN_B
exit

! 2. Config Trunk ไปหา Router 2 (สมมติ e0/0)
interface e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 245,199
 no shut
 exit

! 3. Access Port - Ubuntu 1 (VLAN A) (สมมติ e0/1)
interface e0/1
 switchport mode access
 switchport access vlan 245
 no shut
 exit

! 4. Access Port - Ubuntu 2 (VLAN B) (สมมติ e0/2)
interface e0/2
 switchport mode access
 switchport access vlan 199
 no shut
 exit

! 5. Management IP (SVI)
interface vlan 245
 ip address 172.61.245.158 255.255.255.224
 no shut
 exit
! ใส่ Gateway ให้ Switch (เผื่อ Ping ข้ามวง)
ip default-gateway 172.61.245.129`

### **Step 4: Ubuntu Configuration**

**Ubuntu 1 (Static IP)***วิธีทำขึ้นอยู่กับ Image ที่ใช้ (ถ้าเป็น Docker ใน GNS3 แก้ที่ไฟล์ config ได้เลย)*
แก้ไขไฟล์ `/etc/network/interfaces` หรือใช้คำสั่ง (ชั่วคราว):

Bash

`# แบบคำสั่ง (หายเมื่อ Reboot)
ip addr add 172.61.245.130/27 dev eth0
ip route add default via 172.61.245.129
echo "nameserver 192.168.1.1" > /etc/resolv.conf`

**Ubuntu 2 (DHCP)***ต้องมั่นใจว่า uncomment config DHCP ในไฟล์ network config แล้ว*

Bash

`# ใน /etc/network/interfaces
auto eth0
iface eth0 inet dhcp`

จากนั้นสั่ง Restart service หรือพิมพ์ `dhclient -v` เพื่อขอดึง IP

### วิธีทำ (แบบ Permit Any)

บน **Router 1**:

Bash

`conf t

! ลบ ACL เดิมออกก่อน (ถ้ามี)
no access-list 1

! สร้าง ACL ใหม่ที่อนุญาตทุกอย่าง
access-list 1 permit any

! ผูก NAT เหมือนเดิม
ip nat inside source list 1 interface e0/1 overload`

# netplan

### 1. Ubuntu 1 (Static IP - VLAN A)

**เงื่อนไข:**

- IP: `172.61.245.130/27`
- Gateway: `172.61.245.129`
- DNS: `192.168.1.1`

YAML

`network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 172.61.245.130/27
      routes:
        - to: default
          via: 172.61.245.129
      nameservers:
        addresses: [192.168.1.1]
      dhcp4: false`

*(หมายเหตุ: ใน Ubuntu รุ่นเก่ามากๆ อาจใช้คำสั่ง `gateway4: 172.61.245.129` แทนบรรทัด `routes` ได้ แต่แบบ `routes` คือมาตรฐานใหม่ครับ)*

---

### 2. Ubuntu 2 (DHCP Client - VLAN B)

**เงื่อนไข:** รับ IP อัตโนมัติจาก DHCP Server

YAML

`network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
      # ถ้าต้องการบังคับไม่ให้รับ DNS จาก DHCP ให้ uncomment บรรทัดล่าง
      # dhcp4-overrides:
      #   use-dns: false`
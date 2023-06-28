# Smart Home on Synology NAS Installation Guide  
ใน Repository นี้จะแนะนำการติดตั้ง Software สำหรับระบบ Smart Home บน Synology NAS ด้วยระบบ Docker Container

## Docker Container คืออะไร
Docker คือแพลตฟอร์ม Software สำหรับการติดตั้ง Application ลงไป Container. คล้ายๆ กับการใช้ VM แต่ว่า Docker จะรันแค่ตัว Application ไม่ใช่ทั้ง OS เหมือน VM

## NAS ที่แนะนำ
ควรใช้ Synology NAS รุ่น [Plus](https://www.synology.com/th-th/products/series/home) ขึ้นไป (CPU ใช้เป็น intel หรือ AMD)

## การติดตั้ง
### สร้าง Shared folder สำหรับเก็บข้อมูล Docker
1. ไปที่ Control Panel > Shared Folder > Create Shared Folder
2. ตั้งชื่อ Folder เป็น `docker` และเลือก Storage pool ที่ต้องการ
3. Config ตัว Permission ของ Drive และ Apply

### ติดตั้งโปรแกรมจาก Package Center
1. Docker
2. Text Editor

### สร้าง Container ใน Docker โดยใช้การตั้งค่าดังนี้

#### Home Assistant
* Port Mapping: HOST
   | Purpose               | Port                  |
   |-----------------------|-----------------------|
   | Home Assistant UI     | 8123                  |

* Volume:
   | Local Folder                         | Mount Path            |
   |--------------------------------------|-----------------------|
   | /`Your NAS Folder`/homeassistant     | /config               |

* การติดตั้ง Integration
  * HACS
     1. ไปที่ Docker Container > Home Assistant Container > Terminal
     2. Create > Launch with command > ใส่ `bash`
     3. พิมพ์ `wget -O - https://get.hacs.xyz | bash -` และ enter เพื่อทำการติดตั้ง
     4. Restart Container
     5. ไปที่ Home Assistant Web > integration > ค้นหาว่า HACS และติดตั้ง
  * NUT UPS - อ่านค่าสถานะ UPS ที่เชื่อมต่อกับ NAS
     1. ไปที่ Control Panel ของ NAS และเลือก Hardware & Power > UPS
     2. เลือก Enable network UPS Server
     3. ใส่ `127.0.0.1` ลงใน Permitted Synology NAS Devices 
     4. Apply
     5. ค้นหา Integration NUT ใน Home Assistant และใส่ host เป็น `localhost` 

#### EMQX MQTT Broker
* Port Mapping: BRIDGE
   | Purpose               | Local Port      | Container Port     |
   |-----------------------|-----------------|--------------------|
   | MQTT/WebSocket        | 1883            | 1883               |
   | Dashboard Management  | 18083           | 18083              |

* Volume: No set

#### Z2M
* Port Mapping: HOST
   | Purpose               | Port                  |
   |-----------------------|-----------------------|
   | Dashboard Management  | 8080                  |

* Volume:
   | Local Folder                         | Mount Path            |
   |--------------------------------------|-----------------------|
   | /`Your NAS Folder`/zigbee2mqtt       | /app/data             |

* แก้ไข Z2M Configuration file ใน  `/Your NAS Folder/zigbee2mqtt/configuration.yaml` 
```
# Home Assistant integration (MQTT discovery)
homeassistant: true

# Web Front End
frontend:
   # Optional, default 8080
   port: 8080

# allow new devices to join
permit_join: true

# MQTT settings
mqtt:
   # MQTT base topic for zigbee2mqtt MQTT messages
   base_topic: zigbee2mqtt
   # MQTT server URL
   server: 'mqtt://127.0.0.1:1883'
   # MQTT server authentication, uncomment if required:
   user: mqtt
   password: mqtt

# Serial settings
serial:
   # Location of SLZB-06
   port: tcp://xxx.xxx.xxx.xxx:6638
   baudrate: 115200
   # Disable green led?
   disable_led: false
   # Set output power to max 20
   advanced:
   transmit_power: 20

```
#### ESPHOME

* Port Mapping: HOST
   | Purpose               | Port                  |
   |-----------------------|-----------------------|
   | Dashboard Management  | 6052                  |

* ตัวอย่าง YAML สำหรับเปิดปิดไฟบนบอร์ด ESP32 (เพิ่มลงไปด้านล่างของไฟล์ yaml)
   ```
   switch:
   -  platform: gpio
      pin: GPIO2
      name: "Onboard LED"
   ```
* สำหรับการติดตั้งครั้งแรก
  1.  กด install > Manual Download > Modern Format.
  2.  บันทึกไฟล์ `.bin` 
  3.  เปิด [ESPHome Web](https://web.esphome.io/)
  4.  เชื่อมต่อบอร์ดเข้ากับ USB ขอบคอมพิวเตอร์, กด CONNECT, และเลือก serial device.
  5.  กด INSTALL, และเลือกไฟล์ `.bin` ที่โหลดมา
  6.  กด INSTALL.
  **หลังการติดตั้งในครั้งแรก บอร์ดจะขึ้น Online ในหน้า Dashboard และในครั้งต่อไป สามารถลง Firmware แบบ OTA ได้**

#### Node-RED
* Port Mapping: BRIDGE
   | Purpose               | Local Port      | Container Port     |
   |-----------------------|-----------------|--------------------|
   | Node-red web          | 1880            | 1880               |


* Volume:
   | Local Folder                         | Mount Path            |
   |--------------------------------------|-----------------------|
   | /`Your NAS Folder`/node-red          | /data                 |

   **Note**: ถ้ารันเจอ error **EACCESS: permission denied** ให้ทำการ ssh ไปเปลี่ยน permission ของ node-red folder โดยใช้ command `sudo chmod -R 777 node-red/`

* การติดตั้งและเชื่อมต่อกับ Home Assitant
   1. ไปที่ Node-RED setting > palette
   2. ติดตั้ง `node-red-contrib-home-assistant-websocket`
   3. ไปที่ Home Assistant > Profile (กดตรงชื่อ username ตรง sidebar) > สร้าง long-lived access token
   4. ใน Node-Red Home Assistant palette, เพิ่ม Home Assistant server base URL เป็น `http://192.168.1.xx:8123` และเพิ่ม Access token.

#### Homebridge

* Port Mapping: HOST

   | Purpose     | Port                  |
   |-------------|-----------------------|
   | Web UI      | 8581                  |

* Volume:
   | Local Folder                         | Mount Path            |
   |--------------------------------------|-----------------------|
   | /`Your NAS Folder`/homebridge        | /homebridge           |

### เพิ่ม Shortcut ไปที่ Sidebar ของ Home Assistant

เพิ่มส่วนนี้ไปที่ด้านล่างของไฟล์ `configuration.yaml` ของ Home Assistant

```
panel_iframe:
   zigbee2mqtt:
      title: "Zigbee2MQTT"
      url: "http://192.168.1.XXX:8080"
      icon: mdi:zigbee

   esphome:
      title: "ESPHome"
      url: "http://192.168.1.XXX:6052"
      icon: mdi:memory

   nodered:
      title: "node-red"
      url: "http://192.168.1.XXX:1880"
      icon: mdi:resistor-nodes

   emqx:
      title: "EMQX"
      url: "http://192.168.1.XXX:18083"
      icon: mdi:lan-connect

   homebridge:
      title: "Homebridge"
      url: "http://192.168.1.XXX:8581"
      icon: mdi:apple
```

### การอัปเดต container
   1. ไปที่ Synology Docker App.
   2. ไปที่ Registry และ Download image เวอร์ชั่นล่าสุด
   3. หยุดการทำงานของ container และไปที่ action > reset.
   **ไฟล์และข้อมูลของ container จะไม่สุญหายหากได้ทำการ mount volume ไว้แล้ว**
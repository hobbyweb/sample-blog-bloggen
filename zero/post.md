
# การนำ PocketBase ไปรันบน Raspberry Pi Zero ด้วย Docker, Nginx และ Cloudflare Tunnel

การนำ PocketBase ไปรันบน Raspberry Pi Zero ผ่าน Docker, Nginx และ Cloudflare Tunnel นั้นเป็นกระบวนการที่สามารถทำได้ เพื่อสร้างระบบ backend ขนาดเล็กสำหรับโปรเจ็กต์ที่ต้องการความสะดวกในการเข้าถึงจากภายนอก ผ่านอินเทอร์เน็ตโดยไม่ต้องเปิดเผย IP ของเครื่อง Raspberry Pi ของคุณ

## 1. ติดตั้ง Raspberry Pi OS

ก่อนอื่นต้องติดตั้งระบบปฏิบัติการลงใน Raspberry Pi Zero ของคุณ โดยสามารถดาวน์โหลด Raspberry Pi OS (Lite) ที่ไม่มี GUI จากเว็บไซต์ Raspberry Pi แล้วเขียนลงใน SD card ด้วยโปรแกรมเช่น `Raspberry Pi Imager` หรือ `balenaEtcher`

1. ดาวน์โหลด Raspberry Pi OS Lite
2. ใช้ `balenaEtcher` เขียนไฟล์ image ลง SD card
3. เชื่อมต่อ SD card กับ Raspberry Pi Zero และเปิดเครื่อง

## 2. การติดตั้ง Docker บน Raspberry Pi Zero

Docker เป็น platform สำหรับการพัฒนาและ deploy applications ที่ช่วยให้การจัดการบริการต่างๆ ทำได้ง่ายขึ้น โดยใช้ container

1. อัปเดตระบบและติดตั้ง Docker

    ```bash
    sudo apt-get update && sudo apt-get upgrade -y
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    ```

2. ตรวจสอบการติดตั้ง Docker

    ```bash
    sudo docker --version
    ```

3. เพิ่มสิทธิ์ให้ผู้ใช้ `pi` ใช้งาน Docker โดยไม่ต้องใช้ `sudo`

    ```bash
    sudo usermod -aG docker pi
    ```

## 3. การติดตั้งและตั้งค่า PocketBase

PocketBase เป็น Backend-as-a-Service ที่ lightweight สามารถรันได้บนหลายแพลตฟอร์ม รวมถึง Raspberry Pi

1. สร้างไฟล์ Dockerfile สำหรับ PocketBase

    สร้างโฟลเดอร์สำหรับโปรเจ็กต์และสร้างไฟล์ Dockerfile

    ```bash
    mkdir pocketbase-project && cd pocketbase-project
    nano Dockerfile
    ```

    จากนั้นใส่เนื้อหาต่อไปนี้ใน Dockerfile

    ```Dockerfile
    FROM arm32v6/alpine:3.12

    WORKDIR /app

    RUN apk add --no-cache wget

    # Download PocketBase
    RUN wget https://github.com/pocketbase/pocketbase/releases/download/v0.7.9/pocketbase_0.7.9_linux_armv6.zip

    RUN unzip pocketbase_0.7.9_linux_armv6.zip -d /app && rm pocketbase_0.7.9_linux_armv6.zip

    EXPOSE 8090

    CMD ["./pocketbase", "serve", "--http=0.0.0.0:8090"]
    ```

2. สร้าง Docker image และ run container

    ```bash
    sudo docker build -t pocketbase .
    sudo docker run -d -p 8090:8090 pocketbase
    ```

## 4. การตั้งค่า Nginx เป็น reverse proxy

Nginx จะทำหน้าที่เป็น reverse proxy สำหรับ PocketBase เพื่อจัดการการเชื่อมต่อจากอินเทอร์เน็ตไปยังบริการที่รันบน Raspberry Pi

1. ติดตั้ง Nginx

    ```bash
    sudo apt-get install nginx -y
    ```

2. ตั้งค่า Nginx

    สร้างไฟล์การตั้งค่าสำหรับ Nginx

    ```bash
    sudo nano /etc/nginx/sites-available/pocketbase
    ```

    ใส่เนื้อหาดังนี้

    ```nginx
    server {
        listen 80;

        server_name yourdomain.com;

        location / {
            proxy_pass http://localhost:8090;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```

3. สร้าง symbolic link ไปที่ `sites-enabled`

    ```bash
    sudo ln -s /etc/nginx/sites-available/pocketbase /etc/nginx/sites-enabled/
    sudo nginx -t
    sudo systemctl restart nginx
    ```

## 5. ตั้งค่า Cloudflare Tunnel

Cloudflare Tunnel จะช่วยสร้างการเชื่อมต่อที่ปลอดภัยระหว่าง Cloudflare และ Raspberry Pi ของคุณ โดยไม่ต้องเปิดพอร์ตหรือแก้ไขไฟร์วอลล์

1. ติดตั้ง Cloudflare Tunnel

    ```bash
    curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm.tgz | tar zx
    sudo mv cloudflared /usr/local/bin
    ```

2. ตั้งค่า Tunnel

    ```bash
    cloudflared tunnel login
    cloudflared tunnel create mytunnel
    cloudflared tunnel route dns mytunnel yourdomain.com
    ```

3. สร้างไฟล์ config สำหรับ Cloudflare Tunnel

    ```bash
    sudo nano /etc/cloudflared/config.yml
    ```

    ใส่เนื้อหาดังนี้

    ```yaml
    tunnel: mytunnel
    credentials-file: /root/.cloudflared/<TUNNEL_ID>.json

    ingress:
      - hostname: yourdomain.com
        service: http://localhost:80
      - service: http_status:404
    ```

4. เริ่มต้น Cloudflare Tunnel

    ```bash
    sudo cloudflared tunnel run mytunnel
    ```

## 6. ทดสอบการทำงาน

เมื่อทำทุกขั้นตอนเสร็จเรียบร้อย ให้ลองเข้าถึงโดเมนของคุณที่ได้ตั้งค่าไว้ใน Cloudflare Tunnel และตรวจสอบว่า PocketBase ทำงานได้อย่างถูกต้องผ่านทาง reverse proxy ของ Nginx

## สรุป

การตั้งค่า PocketBase บน Raspberry Pi Zero ผ่าน Docker, Nginx และ Cloudflare Tunnel เป็นวิธีการที่ช่วยให้คุณสร้าง backend สำหรับโปรเจ็กต์ส่วนตัวได้ง่ายและปลอดภัย โดยใช้ประโยชน์จากเทคโนโลยีที่มีอยู่เพื่อให้สามารถเข้าถึงจากภายนอกได้โดยไม่ต้องเปิดเผย IP จริงของ Raspberry Pi


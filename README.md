# CI Workflow — คำอธิบาย

ไฟล์ workflow ที่อยู่ที่ `.github/workflows/ci.yml` ถูกตั้งค่าให้รันเมื่อมีการเปลี่ยนแปลงบน branch `main` โดยใช้ทั้ง `push` และ `pull_request` และมี jobs หลัก ๆ ที่รันตามลำดับการขึ้นต่อกัน (dependencies)

## ทำไมต้องใช้ `on: push` และ `on: pull_request`

- `on: push` — จะทำให้ workflow รันเมื่อมีการส่ง (push) commits ขึ้นไปยัง branch ที่กำหนด (ในที่นี้คือ `main`) เช่น เมื่อคนที่มีสิทธิ์ commit ตรงไปยัง `main` หรือเมื่อ branch ถูก merged เข้า `main` โดยตรง
- `on: pull_request` — จะทำให้ workflow รันเมื่อมีการสร้างหรืออัพเดต Pull Request ที่มีเป้าหมายเป็น `main` (เช่น เมื่อมีคนเปิด PR จาก feature branch มายัง `main`). การใช้ `pull_request` มีประโยชน์ในการตรวจโค้ดก่อน merge เพื่อจับข้อผิดพลาดก่อนจะเข้าถึง `main`

การใช้ทั้งสองอย่างร่วมกันช่วยให้:
- ตรวจทั้งกรณีที่มีการ push โดยตรงไปยัง `main` และกรณีที่มีการเปิด/อัพเดต PR
- ป้องกันข้อผิดพลาดก่อน merge (ผ่าน PR) และยังตรวจหลัง merge/หลัง push ได้

## `jobs` และ `steps` ทำหน้าที่อะไร

- jobs: เป็นกลุ่มของงาน (units of work) ที่รันบน runner แยกกันโดยค่าเริ่มต้น (เช่น `ubuntu-latest`) แต่สามารถกำหนดความขึ้นต่อกันด้วย `needs` เพื่อให้ job หนึ่งรันหลังอีก job ที่ต้องเสร็จ
  - แต่ละ job มี `runs-on` เพื่อระบุ environment
- steps: เป็นชุดคำสั่งภายใน job แต่ละขั้นตอนจะรันแบบลำดับ (sequential) ภายใน job เดียวกัน
  - step ประกอบด้วย `uses` (เรียกใช้ action ที่เขียนไว้แล้ว เช่น `actions/checkout@v4`) หรือ `run` (รันคำสั่ง shell เช่น `echo "Build successful"`)

ข้อสรุปสั้น ๆ: jobs = หน่วยงานแยก (สามารถรันพร้อมกันหรือกำหนดลำดับได้), steps = คำสั่งทีละขั้นตอนภายใน job

## อธิบาย jobs ใน workflow นี้

- Checkout Repository
  - ทำหน้าที่ checkout โค้ดจาก repository ลงบน runner เพื่อให้ steps ถัดไปเข้าถึงไฟล์ได้ (ใช้ `actions/checkout`)

- Lint YAML
  - ตรวจ syntax ของไฟล์ YAML ใน repo (ตัวอย่างนี้ใช้ `github/super-linter` และเปิดการตรวจ `VALIDATE_YAML`)
  - หาก YAML มีข้อผิดพลาด job นี้จะล้มเหลวและ jobs ที่ต้องพึ่งพาจะไม่ถูกรัน

- Build Application
  - ตัวอย่างแบบง่าย: รันคำสั่ง `echo "Build successful"` — ในโปรเจกต์จริงจะเป็น `npm build`, `dotnet build`, `mvn package` ฯลฯ

- Notify
  - ตัวอย่างแบบง่าย: รันคำสั่ง `echo "Workflow finished"` — ในการใช้งานจริงอาจเป็นการส่งการแจ้งเตือนไปยัง Slack, Email หรือระบบอื่น ๆ

## วิธีทดสอบ (ตัวอย่าง PowerShell)

สั่ง commit และ push เพื่อทดสอบ trigger `push` (การ push จะทำให้ workflow รันบน GitHub Actions):

```powershell
git add .github/workflows/ci.yml
git commit -m "Add CI workflow"
git push origin main
```

หรือ

- สร้าง branch ใหม่, เปลี่ยนอะไรแล้ว push แล้วเปิด Pull Request มายัง `main` เพื่อทดสอบ trigger `pull_request`.

## หมายเหตุเพิ่มเติม

- หากต้องการลดเวลาหรือความซับซ้อนของ lint ขั้นตอน `Lint YAML` สามารถเปลี่ยนเป็นการรัน `yamllint` แบบเบา ๆ แทน `super-linter` ได้
- หากต้องการให้ job บางอันรันพร้อมกัน ให้เอา `needs` ออก หากต้องการให้ job รันตามลำดับ ให้ใช้ `needs` ระบุชื่อ job ที่ต้องเสร็จก่อน

ถ้าต้องการ ผมสามารถอัพเดต README ให้มีตัวอย่างการตั้งค่า `yamllint` หรือเพิ่มตัวอย่างการแจ้งเตือน (เช่น Slack) ให้ได้ครับ

## Docker Compose — ไฟล์ `docker-compose.yml`

ไฟล์ `docker-compose.yml` ใน repository จะสร้าง 2 services คือ `web` และ `db` เพื่อให้สามารถรันสภาพแวดล้อมแบบพื้นฐานได้ง่าย ๆ

- `web`:
  - image: `nginx:latest`
  - ports: `8080:80` — แม็ปพอร์ต 80 ของคอนเทนเนอร์ไปยังพอร์ต 8080 ของเครื่อง host
  - depends_on: `db` — รอให้ `db` เริ่มก่อน (ไม่รับประกันว่า DB พร้อมรับการเชื่อมต่อแค่เพียงเริ่มต้นแล้ว)

- `db`:
  - image: `mysql:5.7`
  - environment: กำหนดตัวแปรสำหรับ `MYSQL_ROOT_PASSWORD` และ `MYSQL_DATABASE` (ตัวอย่างใช้ `examplePassword` และ `example_db`)
  - volumes: เก็บข้อมูลฐานข้อมูลถาวรด้วย `db_data` mapped ไปที่ `/var/lib/mysql`

ตัวอย่างไฟล์ที่เพิ่ม:

```yaml
version: '3.8'
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    depends_on:
      - db
  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: examplePassword
      MYSQL_DATABASE: example_db
    volumes:
      - db_data:/var/lib/mysql
volumes:
  db_data:
    driver: local
```

### อธิบาย key ต่าง ๆ

- services: คือการกำหนดคอนเทนเนอร์หลายตัวที่ทำงานร่วมกันในแอปเดียวกัน (เช่น web, db)
- ports: ใช้แม็ปพอร์ตจาก host ไปยังคอนเทนเนอร์ (`HOST:CONTAINER`) เพื่อให้บริการสามารถเข้าถึงจากเครื่อง host หรือเครือข่ายภายนอกได้
- volumes: เก็บข้อมูลถาวรหรือแชร์ไฟล์ระหว่าง host และคอนเทนเนอร์ เช่น ฐานข้อมูลจะเก็บข้อมูลใน volume แทนการสูญหายเมื่อคอนเทนเนอร์ถูกลบ
- environment: ตัวแปรสภาพแวดล้อมที่ส่งให้คอนเทนเนอร์ใช้กำหนดค่า (เช่น รหัสผ่าน, ชื่อฐานข้อมูล, คีย์ต่าง ๆ)

### Docker Compose ทำงานร่วมกับ Project ยังไง

- สร้างสภาพแวดล้อมการพัฒนา/ทดสอบที่เหมือนกันสำหรับทุกคนในทีม โดยไม่ต้องติดตั้งบริการต่าง ๆ บนเครื่อง host โดยตรง
- ทำให้ง่ายต่อการรันหลายคอนเทนเนอร์พร้อมกัน (`docker compose up -d`) และจัดการวงจรชีวิตของพวกมัน (`stop`, `down`, `logs`)
- ใน repo นี้คุณสามารถใช้ `docker-compose.yml` เพื่อรัน nginx เป็น reverse proxy หรือ static server และมี MySQL เป็นฐานข้อมูลตัวอย่างสำหรับการพัฒนา

### วิธีรัน (ตัวอย่าง PowerShell)

```powershell
# รัน backend และ database แบบ background
docker compose up -d

# ดู logs
docker compose logs -f

# หยุดและลบคอนเทนเนอร์ (แต่เก็บ volume)
docker compose down
```

หมายเหตุ: ให้แก้ `MYSQL_ROOT_PASSWORD` ใน `docker-compose.yml` เป็นรหัสผ่านที่ปลอดภัยก่อนใช้งานจริง ไม่ควรเก็บรหัสผ่านจริงใน repository — ใช้ตัวแปรแวดล้อมจากระบบหรือตัวจัดการความลับเมื่อเป็น production

## Kubernetes manifests (โฟลเดอร์ `k8s/`)

ไฟล์ที่เพิ่มใน `k8s/`:

- `deployment.yml` — สร้าง Deployment ของ Nginx จำนวน 2 replicas
- `service.yml` — สร้าง Service แบบ `NodePort` เพื่อเข้าถึง Web จาก node ภายนอก

### ตัวอย่างไฟล์

`k8s/deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

`k8s/service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

### อธิบายคำสำคัญ

- Deployment: เป็น object ที่จัดการการ deploy ของแอปบน Kubernetes — คอนโทรลเลอร์จะดูแลให้มีจำนวน Pod ตามที่กำหนด, ดูแลการอัพเดต, การ rollback, และการสร้าง/ลบ Pod ตามความจำเป็น
- replicas: จำนวนสำเนาของ Pod ที่ต้องการให้รันพร้อมกัน (ในตัวอย่างตั้งเป็น 2)
- selector (matchLabels): กำหนดเงื่อนไขการเลือก Pod ที่ Deployment จะควบคุมหรือ Service จะชี้ไปหา โดยปกติจะเป็น label key/value เช่น `app: nginx`
- template: ชิ้นส่วนที่เป็น blueprint ของ Pod ที่จะถูกสร้าง (รวม metadata.labels และ spec.containers)

### Service และพอร์ต

- Service: เป็น abstraction ที่ให้การเข้าถึงชุดของ Pod (โดยใช้ selector) โดยไม่ต้องรู้ IP ของ Pod แต่ละตัว Service มี IP และพอร์ตคงที่สำหรับการเข้าถึง
- port: พอร์ตที่ Service เปิดให้ภายใน cluster (client ของ Service จะติดต่อผ่านพอร์ตนี้)
- targetPort: พอร์ตภายใน Pod ที่ทราฟฟิคจะถูกส่งไป (เช่น containerPort ที่กำหนดใน Deployment)
- NodePort: ถ้ากำหนด type: NodePort จะเปิดพอร์ตบนทุก Node เพื่อให้เข้าถึง Service ได้จากภายนอกโดยใช้ `NODE_IP:nodePort`

### ความสัมพันธ์ระหว่าง Deployment และ Service

- Deployment สร้างและดูแล Pod (จำนวน replicas)
- Service จะใช้ `selector` เพื่อค้นหา Pod ที่ถูกสร้างโดย Deployment (ผ่าน labels)
- เมื่อ Service ถูกเรียก จะส่งทราฟฟิคไปยัง Pod ที่ตรงกับ selector โดย Kubernetes จะทำ load-balancing ระหว่าง Pod เหล่านั้น

### วิธีทดสอบ (ตัวอย่าง PowerShell กับ kubectl)

```powershell
# สร้าง resources
kubectl apply -f k8s/deployment.yml
kubectl apply -f k8s/service.yml

# ตรวจสอบ Pod
kubectl get pods -l app=nginx

# ตรวจสอบ Service
kubectl get svc nginx-service

# เข้าถึงผ่าน NodePort (สมมติ NODE_IP เป็น IP ของ node)
# http://<NODE_IP>:30080
```

## การทำงานร่วมกันของไฟล์ YAML ทั้งหมด

ไฟล์ YAML ในโปรเจกต์นี้ มีบทบาทต่างกันแต่ทำงานร่วมกันเพื่อรองรับทั้งการพัฒนาแบบโลคอลและการ deploy ไปยัง Kubernetes:

- `.github/workflows/ci.yml` — ควบคุม CI/CD pipeline (lint, validate docker-compose, kubectl dry-run, build, notify). จะรันบน GitHub เมื่อมีการ push/PR ไปยัง `main` ตามที่กำหนด
- `docker-compose.yml` — ใช้สำหรับรันสภาพแวดล้อมพัฒนา/ทดสอบแบบโลคอล (nginx + MySQL) โดยไม่ต้องขึ้น cluster
- `k8s/*.yml` — manifests สำหรับ deploy บน Kubernetes (Deployment + Service)

การรวมกันของไฟล์เหล่านี้ช่วยให้:
- ทีมสามารถรันโปรเจกต์แบบเต็ม (ด้วย Docker Compose) ในเครื่องของแต่ละคน
- CI ตรวจสอบความถูกต้องของ config/manifest ก่อนการ merge
- เมื่อพร้อม จะนำ manifests หรืออิมเมจที่ถูก build ขึ้นไปรันบน Kubernetes ผ่านกระบวนการ CD

## ลำดับขั้นตอนเมื่อ Push โค้ดไปที่ GitHub (Workflow → Docker → Kubernetes)

1. Push/PR: ผู้พัฒนาส่งโค้ดขึ้น GitHub (push หรือเปิด PR ไปยัง `main`)
2. GitHub Actions (CI): `.github/workflows/ci.yml` ทำงานตามลำดับ
  - Lint: ตรวจ syntax ของ YAML และ static checks
  - Validate Docker Compose: ตรวจ `docker-compose.yml` (config sanity)
  - K8s dry-run: รัน `kubectl apply --dry-run=client` เพื่อตรวจ manifest syntax
  - Build: (ที่เป็นไปได้) สร้าง/เทสต์แอป ถ้ามีขั้นตอน build image จะสร้าง Docker image และ push ขึ้น registry
3. Docker: ถ้ามีขั้นตอนการ build image ใน workflow, image จะถูก push ไปยัง container registry (เช่น Docker Hub, GitHub Container Registry)
4. Kubernetes (CD): เมื่อ image พร้อมและ/หรือ manifests ถูกอัพเดต ระบบ CD จะ deploy ไปยัง Kubernetes cluster โดยการรัน `kubectl apply -f k8s/` หรือใช้ tool เช่น ArgoCD/Flux เพื่อทำ automated deployment

หมายเหตุ: ใน repository ปัจจุบัน workflow จะทำ validation/dry-run แต่ยังไม่ได้ build/push image ไปยัง registry หรือตั้งค่า credentials สำหรับ cluster — ถ้าต้องการให้ CI ทำ build+push และ deploy จริง ๆ ผมสามารถเพิ่มขั้นตอนเหล่านั้นและแนะนำวิธีตั้งค่า secrets (เช่น `DOCKER_USERNAME`, `DOCKER_PASSWORD`, `KUBE_CONFIG`) ให้ได้

## Project structure (คำอธิบายไฟล์หลัก)

รายชื่อไฟล์/โฟลเดอร์สำคัญและหน้าที่สั้น ๆ:

- `/.github/workflows/ci.yml` — CI pipeline (lint YAML, validate docker-compose, k8s dry-run, build, notify)
- `/docker-compose.yml` — Compose file สำหรับรัน nginx + mysql ในเครื่องพัฒนา
- `/k8s/deployment.yml` — Kubernetes Deployment สำหรับ nginx (2 replicas)
- `/k8s/service.yml` — Kubernetes Service (NodePort) สำหรับ expose nginx
- `/README.md` — เอกสารโปรเจกต์ (คู่มือนี้)

## Diagram — ความสัมพันธ์ระหว่าง CI/CD Pipeline, Docker Compose และ Kubernetes

ASCII diagram (เรียบง่าย) แสดง flow และความสัมพันธ์:

```
            +----------------------+
            |  Developer Machine   |
            |----------------------|
            | - docker-compose.yml |
            | - k8s/ manifests     |
            +----------+-----------+
                   |
                   | push / PR
                   v
           +-------------------------+
           |  GitHub (Repo)         |
           +-------------------------+
                   |
                   | trigger
                   v
           +-------------------------+
           |  GitHub Actions (CI)   |
           |-------------------------|
           | - lint YAML             |
           | - validate compose      |
           | - k8s dry-run           |
           | - (build & push image)  |
           +----+------------+--------+
              |            |
      optional:   |            | optional:
      run locally |            | deploy to cluster
              v            v
    +----------------------+   +--------------------+
    | Docker Compose (dev) |   | Kubernetes Cluster |
    | - runs services      |   | - run k8s manifests|
    +----------------------+   +--------------------+

Notes:
- Docker Compose is primarily for local/dev runs. CI validates its config.
- Kubernetes is the production-like environment where manifests from `k8s/` are applied.
- CI can build images and push to registry; Kubernetes pulls images from registry to run in cluster.
```

## ถัดไป (ข้อเสนอแนะ)

- หากต้องการให้ CI ทำการ build + push image ผมจะแนะนำขั้นตอนและ secrets ที่ต้องการ (เช่น `CR_PAT`/`DOCKER_USERNAME`, `DOCKER_PASSWORD` หรือใช้ GitHub Packages)
- หากต้องการให้ CI ทำ deploy อัตโนมัติไปยัง cluster จำเป็นต้องตั้งค่า `KUBECONFIG` (หรือใช้ service account/secret) ใน GitHub Secrets — ผมสามารถเพิ่มตัวอย่าง workflow ให้


---
title: บันทึกสิ่งที่ได้เรียนรู้จากการ setup gitlab self host ephimeral runner
draft: true
description: "บันทึกขั้นตอนและปัญหาที่เจอในการทำ Ephemeral Runner"
slug: บันทึกสิ่งที่ได้เรียนรู้จากการ-setup-gitlab-self-host-ephimeral-runner
date: 2026-01-22 00:00:00+0000
image: IMG_20260101_073828.jpg
categories:
    - devlog
tags: ["gitlab", "runner", "docker"]
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# บันทึกสิ่งที่ได้เรียนรู้จากการ setup gitlab self-host ephimeral runner

1. อธิบายทำไมต้องมี self-host gitlab runner, กำลังทำอะไร ช่วยบันทึกสิ่งที่คุณกำลังทำ ณ วันนี้ลงไปใน post ด้วยทางอ้อม

ผมเริ่มมองหาวิธีการทำ self-host runner เนื่องจากกำลังแพลนและดีไซต์ระบบ ทำโปรเจค saas ส่วนตัว การ poc ความเป็นไปได้ของ infastructure ตามที่ดีไซต์ไว้จึงเป็นสิ่งที่จำเป็น ก่อนที่จะลงมือทำจริงๆ<br>
infastructure ของผมนั้นง่ายๆเลย คือ ใช้ hashicorp nomad cluster ในการ deploy แอปต่างๆ (เป็นต้องกล่าวถึงไหม เพราะตอน walkthru setup อาจไม่ได้ deploy (CD) เข้าไปยัง nomad?)<br>
ใช้ gitlab ในการเก็บ source code และรัน ci/cd pipeline<br>
แต่เนื่องจากเคยได้อ่านบทความผ่านตามาว่า gitlab runner ที่เราใช้รัน ci/cd pipeline กันฟรีๆนั่นเป็นแบบ shared runner กล่าวคือ ci/cd pipeline และ source code ของเรา ถูกนำไปรันบน runner ร่วมกันกับของคนอื่นๆ ซึ่งจุดนี้นี่เองที่ทำให้ผมกลับมาฉุกคิดและกังวนถึงเรื่องความปลอดภัยและการรั่วไหลของข้อมูลที่อาจเกิดขึ้นได้<br>
ดังนั้นไหนๆก็กำลังแพลนและ poc อยู่แล้ว ก็ลองสร้าง runner เองเลยดีกว่า.

> runner คือ agent ที่ execute หรือรัน ci/cd jobs ใน pipeline

ผมลองหาวิธีการสร้าง runner แบบส่วนตัว (self-hosted runner) อยู่สักพักก็พบว่า gitlab มี document ที่ค่อนข้างดีเลยทีเดียวในการช่วย setup self-host runner

โดยการสร้าง runner จำเป็นต้องเลือก executor ที่อาจมองได้ว่าเหมือนเป็น runtime ของ runner อีกที<br>
โดย gitlab มี executor ให้เลือกหลากหลายตัวด้วยกัน เช่น Docker, Docker autoscaler, Kubernates, Instance เป็นต้น
> executor คือ รูปแบบการรันและ tool ที่ใช้ในการรัน job ของ ci/cd

1. Docker เป็น executor ตัวหนึ่งที่ใช้ความสามารถของเทคโนโลยี container เข้ามารัน job โดยแต่ละ job สามารถ ใช้ container แยกรันกันไปได้<br>
ข้อดีคือ การ setup และการใช้งานตรงไปตรงมา ไม่ซับซ้อน<br>
แต่ข้อเสียก็คือ เราต้องมี spec เครื่อง vps ที่มากพอในการรัน job ในทุกๆครั้ง ซึ่งบางครั้งหากจัดสรร spec ไม่ดีหรือเพียงพอต่อความต้องการของ runner อาจมีปัญหาในการรัน ci/cd pipeline เกิดขึ้นได้<br>
และในมุมของค่าใช้จ่ายในการมี vps ที่ต้องการใช้ spec เยอะ อาจมีค่าใช้จ่ายค่อนข้างสูงตามมาด้วย.

2. Kubernates, หาก infastructure โปรเจคนี้ของผมดีไซส์ให้ใช้ Kubernates เป็น base อยู่ก่อนแล้ว ผมคงไม่ลังเลที่จะใช้ executor ตัวนี้
ข้อดีคือ การใช้ kubernates pods ในการรัน job<br>
ส่วนข้อเสียคือ ผมไม่ได้ตั้งใจจะใช้ kubernates cluster ในโปรเจคนี้ตั้งแต่แรก จึงจำเป็นต้องข้ามตัวเลือกนี้ไป.

3. Instance เป็น executor ตัวหนึ่งที่ใช้ fleeting plugin (refไปยังคำอธิบายด้านล่าง)เป็นตัวกลางในการติดต่อกับ cloud provider api เพื่อสร้าง instance group (vps) ขึ้นมาใหม่ชั่วคราวในการใช้เป็น ephemical runner<br>
ข้อดีคือ เป็น executor แบบ auto scaling ที่สร้าง vps ขึ้นมาใหม่เพื่อรัน job
ข้อเสียคือ เนื่องจาก vps (ephemical runner) ที่ถูกสร้างขึ้นมาเป็น bare metal เปล่าๆ การ config vps และติดตั้ง dependency ต่างๆให้รองรับสิ่งที่ ci/cd pipeline ต้องการ ค่อนค้างไม่ยืดหยุ่น ยกตัวอย่าง เช่น ใน pipeline ของ microservices ของเรา อาจต้องการใช้ library ตัวเดียวกันแต่คนละ version การ set up ให้ vps (ephemical runner) ใช้งานได้จึงมาตกอยู่ที่การ set up vps ซึ่งค่อนข้างยุ่งยากและไม่ยืดหยุ่น
> ephemical runner คือ runner ย่อย/เล็กๆ ที่ถูกสร้างขึ้นมาชั่วคราวเพื่อรัน job เมื่อรันเสร็จจะถูกลบออกไป<br>

4. Docker AutoScaler, เป็น executor อีกตัวที่ใช้ fleeting plugin ในการทำ auto scaling เป็น executor ที่ประยุกต์ความสามารถของ executor Docker และ Instance เข้าด้วยกัน คือ ใช้ข้อดีของ Docker ในการรันและสะดวกในการ แยก dependency ของแต่ละ job
และยังเป็น auto scaler ที่ใช้ fleeting plugin ในการติดต่อกับ cloud provider เพื่อสร้าง instance group (vps) (ที่มี docker) ใหม่ชั่วคราวขึ้นมาเพื่อรัน job ในระยะเวลาสั้นๆ เสร็จแล้วจึงลบ vps นั้นออกไปเพื่อไม่ให้เปลือง cost และด้านความปลอดภัย.

> fleeting คือ library ที่เป็นตัวกลาง (abstraction layer) ระหว่าง gitlab-runner และ cloud provider เพื่อทำ auto-scaling<br>
โดย fleeting จะไม่ได้ติดต่อกับ cloud provider โดยตรง แต่จะใช้ fleeting-plugin ซึ่งเป็น sub process (หรือจะคิดว่าเป็น api server ก็ได้) ที่ implement interface ของ fleeting เข้ากับ cloud provider APIs อีกที
ซึ่งนี้เป็นเหตุผลว่าทำไม gitlab มี support fleeting plugin ให้แค่ cloud provider ใหญ่เท่านั้น เพราะ artchitecture ที่ออกแบบมาให้เป็นแบบ plug & adapter ที่ gitlab fleeting ไม่ได้สนใจว่า cloud provider ข้างหลังจะเป็นเจ้าไหน แค่ plugin implement interface ของ fleeting ก็เพียงพอ<br>
https://docs.gitlab.com/runner/fleet_scaling/fleeting/
โดยเดิมที gitlab ใช้ executor ที่ชื่อว่า Docker machine ในการทำ auto scaling แต่เนื่องด้วยโปรเจค Docker machine ไม่ได้ถูกพัฒนาและดูแลต่อ ดังนั้น gitlab จึงต้องมองหา solution ใหม่ จึงได้พัฒนา fleeting ซึ่งเป็น library ที่ gitlab เรียกว่า [Next Runner Auto-scaling Architecture](https://handbook.gitlab.com/handbook/engineering/architecture/design-documents/runner_scaling/) เพื่อตอบโจทย์การทำ auto scaling ต่อไป<br>
โดย fleeting plugin คือ library ที่มี interface ที่สามารถปลั๊กกับ cloud provider APIs ได้ 
[gitlab-fleeting-repository](https://gitlab.com/gitlab-org/fleeting/fleeting) 
![gitlab fleeting](gitlab-fleeting.drawio.svg)

executor ที่ดูเหมือนจะตอบโจทย์ความต้องการของผมมากที่สุด คือ Docker Autoscaler
แต่พอลองอ่านรายละเอียดการ setup ผมกลับพบว่า gitlab มี fleeting plugin ให้เฉพาะ cloud provider เจ้าใหญ่ๆเท่านั้น อาทิ เช่น aws, azure, google cloud<br>
โดยส่วนใหญ่ผมจะใช้ Digital Ocean เป็นหลัก สำหรับโปรเจคส่วนตัว และโปรเจคที่มี scale ที่ไม่ใหญ่มาก
ตอนนี้ผมจึงมี 2 ทางเลือกคือ เปลี่ยน cloud provider เพื่อความสะดวกในการ setup ต่อ หรือจะลองหา fleeting plugin ของ digital ocean ดูก่อน
โดย fleeting plugin คือ libery ที่เป็น standard กำหนดโดย gitlab ในการเป็นตัวกลางระหว่าง gitlab และ cloud provider ในการทำ auto scaling หรือ สร้าง ephemical runner (vps) ขึ้นมาใหม่ชั่วคราวในการรัน job
หากลองไล่ดู source code ของ fleeting plugin ของ cloud provider ที่ gitlab มีให้ น่าจะสามารถ เขียน fleeting plugin ขึ้นมาเองได้ หากไม่มีสำหรับ Digital ocean จริงๆ
โดย implement methods ที่ gitlab ต้องการ เข้ากับ api ของ Digital Ocean<br>
แต่โชคดี ที่เจอ ที่มีคนเคยสร้าง fleeting plugin ของ Digital Ocean ไว้ <3

เมื่อหาข้อมูลพอสมควรแล้ว ทีนี้ก็เหลือแค่ลงมือทำ

### 1. เตรียม gitlab reposioty และ vps
รายการที่ต้องใช้
* gitlab repository
* digital ocean account

1.1 ก่อนอื่นเลย สร้าง repository ขึ้นมาเพื่อเก็บ source code และ ci/cd pipeline<br>
1.2 ใส่ tags เพื่อให้เราสามารถกำหนด runner ใน pipeline ได้<br>
1.3 กด create และจะได้ token ให้เราเก็บ token นี้ไว้ให้ดี<br>
1.2 สร้าง runner โดยเลือกที่เมนู runner และ กดสร้าง, สำคัญ! เก็บ token เอาไว้
![create-project-runner](gitlab-create-project-runner.png)


1.2. เตรียม digital ocean token เพื่อให้ runner manager ใช้ติดต่อกับ digital ocean เพื่อสร้าง ephemical runners<br>
![](do-gen-token.png)

1.3. สร้าง vps โดยเลือก แบบ 2gb ซึ่งเพียงพอต่อ การเป็น runner manager<br>
- Region: Singapore (sgp1)
- Size: $4/month (s-1vcpu-512mb-10gb)
- OS: Ubuntu (ubuntu-24-04-x64)


### Config VPS
เมื่อสร้าง vps มาแล้ว เรามา config กัน

1. ssh เข้าไปใน vps และทำการ ติดตั้ง docker เพราะ exucutor ที่ผมจะใช้ คือ Docker autoscaler
ทำตามเพื่อติดตั้ง docker บน ubuntu https://docs.docker.com/engine/install/ubuntu/

2. ติดตั้ง gitlab-runner
```sh
# Download the binary for your system (amd64)
sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

# Give it permission to execute
sudo chmod +x /usr/local/bin/gitlab-runner

# Create a GitLab Runner user
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

# Install and run as a service
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
        sudo gitlab-runner start
```

3. ต่อมา เราจำเป็นต้องมี ไฟล์ binary fleeting-plugin ของ digital ocean เนื่องจากที่ได้อธิบายไปในตอนต้นว่า gitlab runner ไม่ได้มี official fleeting plugin มาให้ ดังนั้น เราต้อง compile เองจาก source code<br>
โดยใช้วิธี [manual install](https://docs.gitlab.com/runner/fleet_scaling/fleeting/#:~:text=version%201.5.1.-,Install%20binary%20manually,-To%20manually%20install)<br>
3.1. ให้ clone git repository https://gitlab.com/bearx3f/fleeting-plugin-digitalocean<br>
3.2. compile source code 
```sh
export PACKAGE_REGISTRY_URL="https://gitlab.com/api/v4/projects/75321582/packages/generic/fleeting-plugin-digitalocean"

curl -sSL https://gitlab.com/bearx3f/fleeting-plugin-digitalocean/-/raw/main/install.sh | bash
```
3.3. สร้างไฟล์ config.toml ที่ path /etc/gitlab-runner/config.toml
```sh
# /etc/gitlab-runner/config.toml
concurrent = 4
check_interval = 3
connection_max_age = "15m0s"
shutdown_timeout = 0
[session_server]
session_timeout = 1800

    [[runners]]
        name = "DigitalOcean-Autoscaler-Runner"
        limit = 10
        url = "https://gitlab.com"
        token = "TOKEN-FROM-GITLAB" # ใส่ token ที่ generate จาก Gitlab repository
        executor = "docker-autoscaler"
        shell = "sh"
        [runners.docker]
            tls_verify = false
            image = "busybox:latest"
            pull_policy = "if-not-present"
            privileged = true
            disable_entrypoint_overwrite = false
            oom_kill_disable = false
            disable_cache = false
            shm_size = 0
            network_mtu = 0
            network_mode = "host"
            volumes = ["/var/run/docker.sock:/var/run/docker.sock"]
        [runners.autoscaler]
            plugin = "/usr/local/bin/fleeting-plugin-digitalocean"

            capacity_per_instance = 2
            max_use_count = 10
            max_instances = 10
        [runners.autoscaler.plugin_config]
            access_token = "DIGITAL-OCEAN-TOKEN" # ใส่ token ที่ generate จาก Digital Ocean
            image = "docker-20-04"
            instance_slug = "s-1vcpu-1gb"
            name = "ephemeral-runner"
            region_slug = "sgp1"
            ssh_private_key_file = "/cache/id_rsa"
            tag = "ci-runners"
        [runners.autoscaler.connector_config]
            protocol = "ssh"
            username = "root"
            keepalive = "10s"
            timeout = "10s"
            use_external_addr = false
        [[runners.autoscaler.policy]]
            idle_count = 0
            scale_factor = 1.0
            idle_time = "5m0s"
            scale_factor_limit = 5
```

3.4. เปลี่ยน owner และเซต permission ให้ไฟล์ config.toml 
```sh
sudo chown root:root /etc/gitlab-runner/config.toml
sudo chmod 600 /etc/gitlab-runner/config.toml
```

3.5. start gitlab-runner
```sh
sudo systemctl start gitlab-runner
sudo systemctl enable gitlab-runner
```

เช็ค gitlab-runner logs มี status บอกว่า เชื่อมต่อกับ Digital Ocean สำเร็จ และ เช็ค status runner ในหน้า gitlab repository มี status ACTIVE แสดงว่า ทั้งฝั่ง gitlab instance, gitlab runner และ digital ocean พร้อม.

### Push code รัน Pipeline
ก่อนอื่น ผมจะสร้าง vps มาอีกหนึ่งตัวเพื่อจำลองเป็นที่ deploy app

1.1. สร้างไฟล์ Dockerfile
```docker
# Use a robust, well-maintained echo image
FROM mendhak/http-https-echo:latest

# The image listens on port 8080 by default
EXPOSE 8080
```

1.1. สร้างไฟล์ gitlab-ci.yml
```sh
stages:
  - docker_publish
  - deploy

default:
  image: docker:20.10.16
  tags:
    - containerized-multi-host

docker_publish_image:
  stage: docker_publish
  script:
    - echo "--- Logging into GitLab Container Registry ---"
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - IMAGE_TAG=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy_to_droplet:
  stage: deploy
  image: alpine:latest
  script:
    - echo "--- Setting up SSH Access ---"
    - apk add --no-cache openssh-client coreutils
    - mkdir -p ~/.ssh
    # Using printf and -di to ensure the key is decoded perfectly
    - printf '%s' "$SSH_PRIVATE_KEY_B64" | base64 -di > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - eval "$(ssh-agent -s)"
    - ssh-add ~/.ssh/id_rsa
    - ssh-keyscan -H $TARGET_VPS_PRIVATE_IP >> ~/.ssh/known_hosts

    - echo "--- Deploying mendhak/http-https-echo to VPS ($TARGET_VPS_PRIVATE_IP) ---"
    - >
      ssh root@$TARGET_VPS_PRIVATE_IP "
        docker login -u ${CI_REGISTRY_USER} -p ${CI_REGISTRY_PASSWORD} ${CI_REGISTRY} &&
        docker pull ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA} &&
        docker stop my-echo-app || true &&
        docker rm my-echo-app || true &&
        # Note: Port 8081 on VPS -> Port 8080 inside the container
        docker run -d --name my-echo-app \
          -p 8081:8080 \
          --restart unless-stopped \
          ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}
      "
  dependencies:
    - docker_publish_image
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

- เช็ค runner logs บนหน้า gitlab repository
- access vps deployed app ว่า app ถูก deploy ขึ้นจริงๆ
- เช็ค หน้า Digital Ocean ว่า Ephemical runner ถูก scale down ลง เหลือเท่ากับจำนวน idle
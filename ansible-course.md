# Ansible 완전 정복: 초급 ~ 고급 Cloud Engineer 강의 + 문제집

> **대상**: 초급자 ~ 고급 Cloud Engineer (AWS / Azure 환경)
> **구성**: 이론 강의 → 샘플 코드 → 실습 문제 → 답안 및 해설

---

# Part 1. 기초 편 (Beginner)

---

## 1장. Ansible이란 무엇인가?

### 1.1 정의

Ansible은 Red Hat이 관리하는 **오픈소스 IT 자동화 도구**다. 서버 구성 관리(Configuration Management), 애플리케이션 배포(Deployment), 오케스트레이션(Orchestration)을 **에이전트 없이(Agentless)** 수행한다. SSH(Linux) 또는 WinRM(Windows)을 통해 원격 노드에 접속하여 작업을 실행하며, 원하는 상태(Desired State)를 YAML로 선언하면 Ansible이 현재 상태를 확인하고 차이가 있을 때만 변경을 적용한다.

### 1.2 왜 Ansible인가? (다른 도구와 비교)

| 항목 | Ansible | Terraform | Chef | Puppet |
|---|---|---|---|---|
| 주요 용도 | 구성 관리 + 배포 | 인프라 프로비저닝 | 구성 관리 | 구성 관리 |
| 언어 | YAML | HCL | Ruby DSL | Puppet DSL |
| 에이전트 | 불필요 (Agentless) | 불필요 | 필요 | 필요 |
| 상태 관리 | Stateless (멱등성) | Stateful (.tfstate) | Stateful | Stateful |
| 학습 곡선 | 낮음 | 중간 | 높음 | 높음 |
| Cloud 통합 | 모듈 기반 | Provider 기반 | Cookbook | Module |

핵심 차별점은 **Agentless + YAML 기반의 낮은 학습 곡선 + 멱등성(Idempotency)** 이다.

### 1.3 핵심 용어 정리

- **Control Node**: Ansible이 설치되어 명령을 실행하는 머신. Linux/macOS만 지원(Windows는 WSL 통해 가능).
- **Managed Node (Host)**: Ansible이 관리하는 대상 서버.
- **Inventory**: 관리 대상 호스트 목록을 정의하는 파일.
- **Module**: Ansible이 원격 노드에서 실행하는 작업 단위 (예: `apt`, `yum`, `copy`, `ec2_instance`).
- **Task**: 하나의 Module 호출.
- **Play**: 특정 호스트 그룹에 대해 실행할 Task들의 집합.
- **Playbook**: 하나 이상의 Play를 포함하는 YAML 파일.
- **Role**: 재사용 가능한 Playbook 구성 단위 (디렉토리 구조).
- **Handler**: 특정 Task의 변경(changed)이 발생했을 때만 실행되는 Task.
- **Facts**: Ansible이 Managed Node에서 자동 수집하는 시스템 정보.
- **Vault**: 민감 정보(비밀번호, 키 등)를 암호화하는 기능.
- **Collection**: Module, Role, Plugin 등을 패키지로 묶어 배포하는 단위.

### 1.4 아키텍처

```
┌─────────────────┐
│  Control Node   │
│  (ansible 설치)  │
│                 │
│  ┌───────────┐  │         SSH / WinRM
│  │ Playbook  │──┼──────────────────────┐
│  │ (YAML)    │  │                      │
│  └───────────┘  │                      ▼
│  ┌───────────┐  │              ┌──────────────┐
│  │ Inventory │──┼──────────────│ Managed Node │
│  │           │  │              │   (Host 1)   │
│  └───────────┘  │              └──────────────┘
│  ┌───────────┐  │              ┌──────────────┐
│  │ Modules   │──┼──────────────│ Managed Node │
│  │           │  │              │   (Host 2)   │
│  └───────────┘  │              └──────────────┘
└─────────────────┘              ┌──────────────┐
                                 │ Managed Node │
                                 │   (Host N)   │
                                 └──────────────┘
```

작동 방식은 다음과 같다. Control Node에서 Playbook을 실행하면, Ansible은 Inventory에서 대상 호스트를 파악하고, 각 Task에 해당하는 Module을 Python 코드로 변환하여 대상 호스트로 전송한다. 대상 호스트에서 해당 코드가 실행되고 결과를 Control Node로 반환한다. 이 모든 과정에서 대상 호스트에는 어떤 에이전트도 설치할 필요가 없다.

---

## 2장. 설치 및 환경 구성

### 2.1 Control Node 설치

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

# RHEL / CentOS / Amazon Linux 2
sudo yum install -y epel-release
sudo yum install -y ansible

# pip 설치 (권장 - 최신 버전)
python3 -m pip install --user ansible

# 버전 확인
ansible --version
```

### 2.2 SSH 키 설정

```bash
# 키 생성
ssh-keygen -t ed25519 -C "ansible-control"

# 대상 호스트에 공개키 배포
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.1.10
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.1.11

# 접속 테스트
ssh user@192.168.1.10 "hostname"
```

### 2.3 ansible.cfg 설정

Ansible은 설정 파일을 다음 순서로 탐색한다: `ANSIBLE_CONFIG` 환경변수 → 현재 디렉토리의 `ansible.cfg` → 홈 디렉토리의 `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`. 프로젝트별로 설정을 관리하려면 프로젝트 루트에 `ansible.cfg`를 생성하는 것이 좋다.

```ini
# ./ansible.cfg
[defaults]
inventory         = ./inventory/hosts.yml
remote_user       = ubuntu
private_key_file  = ~/.ssh/id_ed25519
host_key_checking = False
retry_files_enabled = False
timeout           = 30
forks             = 10
log_path          = ./ansible.log

[privilege_escalation]
become            = True
become_method     = sudo
become_user       = root
become_ask_pass   = False

[ssh_connection]
pipelining        = True
ssh_args           = -o ControlMaster=auto -o ControlPersist=60s
```

- `forks`: 동시에 처리할 호스트 수 (기본값 5, 대규모 환경에서는 20~50 추천)
- `pipelining`: SSH 연결 최적화 (requiretty가 sudoers에서 비활성화 되어있어야 함)

---

## 3장. Inventory (인벤토리)

### 3.1 정적 인벤토리 (INI 형식)

```ini
# inventory/hosts.ini
[webservers]
web1 ansible_host=10.0.1.10 ansible_user=ubuntu
web2 ansible_host=10.0.1.11 ansible_user=ubuntu

[dbservers]
db1  ansible_host=10.0.2.10 ansible_user=ec2-user
db2  ansible_host=10.0.2.11 ansible_user=ec2-user

[loadbalancers]
lb1  ansible_host=10.0.0.10

# 그룹의 그룹 (children)
[production:children]
webservers
dbservers
loadbalancers

# 그룹 변수
[webservers:vars]
http_port=80
app_env=production

[dbservers:vars]
db_port=5432
```

### 3.2 정적 인벤토리 (YAML 형식 - 권장)

```yaml
# inventory/hosts.yml
all:
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 10.0.1.10
          ansible_user: ubuntu
          http_port: 80
        web2:
          ansible_host: 10.0.1.11
          ansible_user: ubuntu
          http_port: 8080
      vars:
        app_env: production
        nginx_version: "1.24"

    dbservers:
      hosts:
        db1:
          ansible_host: 10.0.2.10
          ansible_user: ec2-user
          db_role: primary
        db2:
          ansible_host: 10.0.2.11
          ansible_user: ec2-user
          db_role: replica
      vars:
        db_port: 5432
        db_engine: postgresql

    production:
      children:
        webservers:
        dbservers:
```

### 3.3 호스트 패턴 및 범위

```ini
# 숫자 범위
[webservers]
web[01:50].example.com

# 알파벳 범위
[dbservers]
db-[a:f].example.com
```

### 3.4 Ad-Hoc 명령으로 Inventory 테스트

```bash
# 모든 호스트에 ping (연결 확인)
ansible all -m ping

# 특정 그룹만
ansible webservers -m ping

# 인벤토리 목록 확인
ansible-inventory --list -y
ansible-inventory --graph
```

---

## 4장. Ad-Hoc 명령

Ad-Hoc 명령은 Playbook을 작성하지 않고 단일 명령을 즉시 실행하는 방식이다. 일회성 작업이나 빠른 확인에 적합하다.

```bash
# 기본 문법
ansible <호스트패턴> -m <모듈> -a "<모듈 인자>"

# 시스템 정보 확인
ansible all -m setup -a "filter=ansible_distribution*"

# 패키지 설치
ansible webservers -m apt -a "name=nginx state=present" --become

# 서비스 관리
ansible webservers -m service -a "name=nginx state=started enabled=yes" --become

# 파일 복사
ansible webservers -m copy -a "src=./index.html dest=/var/www/html/index.html owner=www-data mode=0644" --become

# 명령 실행 (command 모듈 - 기본)
ansible all -m command -a "uptime"

# 셸 명령 실행 (파이프, 리다이렉트 가능)
ansible all -m shell -a "df -h | grep /dev/sda1"

# 사용자 생성
ansible all -m user -a "name=deploy state=present shell=/bin/bash" --become

# 파일 삭제
ansible all -m file -a "path=/tmp/test.txt state=absent"
```

---

## 5장. Playbook 기초

### 5.1 첫 번째 Playbook

```yaml
# playbooks/01_first_playbook.yml
---
- name: 웹 서버 기본 구성
  hosts: webservers
  become: yes

  tasks:
    - name: 시스템 패키지 업데이트
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Nginx 설치
      apt:
        name: nginx
        state: present

    - name: Nginx 시작 및 자동 실행 등록
      service:
        name: nginx
        state: started
        enabled: yes

    - name: 기본 index.html 배포
      copy:
        content: |
          <!DOCTYPE html>
          <html>
          <head><title>{{ inventory_hostname }}</title></head>
          <body><h1>Hello from {{ inventory_hostname }}</h1></body>
          </html>
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: "0644"
```

```bash
# 실행
ansible-playbook playbooks/01_first_playbook.yml

# Dry-run (Check mode)
ansible-playbook playbooks/01_first_playbook.yml --check

# 차이점 확인
ansible-playbook playbooks/01_first_playbook.yml --check --diff

# 특정 호스트만 실행
ansible-playbook playbooks/01_first_playbook.yml --limit web1

# 태그로 필터링 실행
ansible-playbook playbooks/01_first_playbook.yml --tags "deploy"
```

### 5.2 멱등성(Idempotency) 이해

멱등성이란 **같은 작업을 여러 번 실행해도 결과가 동일**한 것을 의미한다. Ansible 모듈은 기본적으로 멱등성을 보장하도록 설계되어 있다. 예를 들어 `apt: name=nginx state=present`는 Nginx가 이미 설치되어 있으면 아무 작업도 하지 않고 `ok`를 반환한다. 하지만 `command`나 `shell` 모듈은 멱등성을 보장하지 않으므로, `creates` 또는 `when` 조건과 함께 사용해야 한다.

```yaml
# 멱등하지 않은 예 (매번 실행됨)
- name: 앱 빌드 (비멱등)
  command: /opt/app/build.sh

# 멱등하게 만든 예
- name: 앱 빌드 (멱등)
  command: /opt/app/build.sh
  args:
    creates: /opt/app/dist/app.jar
```

---

## 6장. 변수 (Variables)

### 6.1 변수 정의 위치와 우선순위

Ansible 변수 우선순위 (낮음 → 높음, 총 22단계 중 핵심):

1. Role defaults (`roles/x/defaults/main.yml`)
2. Inventory group_vars
3. Inventory host_vars
4. Playbook group_vars
5. Playbook host_vars
6. Play vars
7. Play vars_files
8. Task vars
9. set_fact / register
10. Role params
11. include_vars
12. Extra vars (`-e` 옵션) ← **가장 높은 우선순위**

### 6.2 변수 사용 예시

```yaml
# group_vars/webservers.yml
---
app_name: mywebapp
app_port: 8080
app_env: production
nginx_worker_processes: auto
nginx_worker_connections: 1024

allowed_origins:
  - "https://example.com"
  - "https://api.example.com"

db_config:
  host: db1.internal
  port: 5432
  name: mywebapp_db
  pool_size: 20
```

```yaml
# host_vars/web1.yml
---
nginx_worker_connections: 2048  # web1만 오버라이드
custom_ssl_cert: /etc/ssl/certs/web1.pem
```

```yaml
# playbooks/02_variables.yml
---
- name: 변수 활용 예시
  hosts: webservers
  become: yes
  vars:
    deploy_version: "2.1.0"
    deploy_timestamp: "{{ ansible_date_time.iso8601 }}"

  vars_files:
    - ../vars/secrets.yml

  tasks:
    - name: 변수 디버깅
      debug:
        msg: |
          호스트: {{ inventory_hostname }}
          앱 이름: {{ app_name }}
          앱 포트: {{ app_port }}
          배포 버전: {{ deploy_version }}
          DB 호스트: {{ db_config.host }}
          DB 포트: {{ db_config['port'] }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}

    - name: 리스트 변수 활용
      debug:
        msg: "허용 Origin: {{ item }}"
      loop: "{{ allowed_origins }}"

    - name: 변수 등록 (register)
      command: cat /etc/hostname
      register: hostname_result

    - name: 등록된 변수 사용
      debug:
        msg: "호스트네임: {{ hostname_result.stdout }}"

    - name: 조건부 변수 설정 (set_fact)
      set_fact:
        max_memory: "{{ '4096M' if ansible_memtotal_mb > 8000 else '2048M' }}"

    - name: set_fact 결과 확인
      debug:
        msg: "최대 메모리: {{ max_memory }}"
```

### 6.3 특수 변수 (Magic Variables)

```yaml
# 자주 사용하는 특수 변수
- debug:
    msg: |
      inventory_hostname: {{ inventory_hostname }}
      ansible_hostname: {{ ansible_hostname }}
      group_names: {{ group_names }}
      groups['webservers']: {{ groups['webservers'] }}
      hostvars['web2'].ansible_host: {{ hostvars['web2'].ansible_host }}
      play_hosts: {{ play_hosts }}
      ansible_play_batch: {{ ansible_play_batch }}
```

---

## 7장. 조건문, 반복문, 핸들러

### 7.1 조건문 (when)

```yaml
---
- name: 조건문 예시
  hosts: all
  become: yes
  tasks:
    # OS별 분기
    - name: Ubuntu에 패키지 설치
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - curl
        - wget
        - unzip
      when: ansible_os_family == "Debian"

    - name: CentOS에 패키지 설치
      yum:
        name: "{{ item }}"
        state: present
      loop:
        - curl
        - wget
        - unzip
      when: ansible_os_family == "RedHat"

    # 복합 조건
    - name: 프로덕션 Ubuntu 서버에만 적용
      debug:
        msg: "프로덕션 Ubuntu 서버입니다"
      when:
        - app_env == "production"
        - ansible_distribution == "Ubuntu"
        - ansible_distribution_major_version | int >= 20

    # 변수 존재 여부
    - name: 변수가 정의된 경우만 실행
      debug:
        msg: "SSL 인증서: {{ custom_ssl_cert }}"
      when: custom_ssl_cert is defined

    # register 결과 기반 조건
    - name: 설정 파일 존재 확인
      stat:
        path: /etc/myapp/config.yml
      register: config_file

    - name: 설정 파일이 없으면 기본값 배포
      template:
        src: default_config.yml.j2
        dest: /etc/myapp/config.yml
      when: not config_file.stat.exists
```

### 7.2 반복문 (loop)

```yaml
---
- name: 반복문 예시
  hosts: webservers
  become: yes
  tasks:
    # 기본 loop
    - name: 여러 사용자 생성
      user:
        name: "{{ item }}"
        state: present
        shell: /bin/bash
        groups: sudo
      loop:
        - alice
        - bob
        - charlie

    # 딕셔너리 loop
    - name: 여러 사용자 (상세 정보 포함)
      user:
        name: "{{ item.name }}"
        uid: "{{ item.uid }}"
        groups: "{{ item.groups }}"
        shell: "{{ item.shell | default('/bin/bash') }}"
      loop:
        - { name: alice, uid: 1001, groups: "sudo,docker" }
        - { name: bob,   uid: 1002, groups: "docker" }
        - { name: charlie, uid: 1003, groups: "sudo" }

    # with_dict (딕셔너리 반복)
    - name: 환경 변수 설정
      lineinfile:
        path: /etc/environment
        regexp: "^{{ item.key }}="
        line: "{{ item.key }}={{ item.value }}"
      loop: "{{ lookup('dict', env_vars) }}"
      vars:
        env_vars:
          APP_ENV: production
          APP_PORT: "8080"
          LOG_LEVEL: info

    # loop + when 조합
    - name: 특정 조건의 패키지만 설치
      apt:
        name: "{{ item.name }}"
        state: present
      loop:
        - { name: nginx, install: true }
        - { name: apache2, install: false }
        - { name: php-fpm, install: true }
      when: item.install

    # loop_control로 제어
    - name: 파일 배포 (인덱스 포함)
      copy:
        src: "configs/{{ item }}"
        dest: "/etc/myapp/{{ item }}"
      loop:
        - app.conf
        - db.conf
        - cache.conf
      loop_control:
        index_var: idx
        label: "{{ item }} ({{ idx + 1 }}/3)"
        pause: 1  # 각 반복 사이 1초 대기
```

### 7.3 핸들러 (Handlers)

핸들러는 `notify`로 호출되며, Play 내 모든 Task가 완료된 후에 실행된다. 여러 Task에서 같은 Handler를 notify해도 **한 번만** 실행된다.

```yaml
---
- name: 핸들러 예시
  hosts: webservers
  become: yes

  handlers:
    - name: Nginx 재시작
      service:
        name: nginx
        state: restarted

    - name: Nginx 설정 검증
      command: nginx -t
      listen: "nginx config changed"

    - name: Nginx 리로드
      service:
        name: nginx
        state: reloaded
      listen: "nginx config changed"

  tasks:
    - name: Nginx 메인 설정 배포
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: "0644"
      notify: "nginx config changed"

    - name: 사이트 설정 배포
      template:
        src: site.conf.j2
        dest: /etc/nginx/sites-available/myapp.conf
      notify: "nginx config changed"

    # 즉시 핸들러 실행이 필요한 경우
    - name: 핸들러 즉시 실행
      meta: flush_handlers

    - name: 이후 작업 (위 핸들러 완료 후 실행)
      uri:
        url: "http://localhost/health"
        status_code: 200
```

---

## 8장. 템플릿 (Jinja2 Templates)

### 8.1 Jinja2 기본 문법

```
{{ variable }}          → 변수 출력
{% if condition %}      → 제어문
{# comment #}          → 주석
{{ var | filter }}      → 필터
```

### 8.2 Nginx 설정 템플릿 예시

```jinja2
{# templates/nginx.conf.j2 #}
user www-data;
worker_processes {{ nginx_worker_processes | default('auto') }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;

    # Logging
    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log {{ 'debug' if app_env == 'development' else 'warn' }};

    # Virtual Hosts
{% for server in virtual_hosts %}
    server {
        listen {{ server.port | default(80) }};
        server_name {{ server.domain }};
        root {{ server.root | default('/var/www/html') }};

{% if server.ssl | default(false) %}
        listen 443 ssl;
        ssl_certificate     {{ server.ssl_cert }};
        ssl_certificate_key {{ server.ssl_key }};
{% endif %}

{% if server.locations is defined %}
{% for location in server.locations %}
        location {{ location.path }} {
            proxy_pass {{ location.backend }};
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
{% if location.websocket | default(false) %}
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
{% endif %}
        }
{% endfor %}
{% endif %}
    }
{% endfor %}
}
```

### 8.3 주요 Jinja2 필터

```yaml
tasks:
  - name: 필터 예시
    debug:
      msg: |
        대문자: {{ app_name | upper }}
        소문자: {{ app_name | lower }}
        기본값: {{ undefined_var | default('fallback') }}
        필수값: {{ required_var | mandatory }}
        리스트 조인: {{ allowed_origins | join(', ') }}
        JSON: {{ db_config | to_json }}
        YAML: {{ db_config | to_nice_yaml }}
        해시: {{ 'password' | password_hash('sha512') }}
        정규식: {{ url | regex_replace('^https?://', '') }}
        경로: {{ '/etc/nginx/nginx.conf' | basename }}
        디렉토리: {{ '/etc/nginx/nginx.conf' | dirname }}
        IP주소: {{ ansible_default_ipv4.address | ipaddr }}
        Base64: {{ 'hello' | b64encode }}
        숫자변환: {{ '8080' | int }}
        min/max: {{ [1, 5, 3, 9, 2] | max }}
        고유값: {{ [1, 1, 2, 3, 3] | unique | list }}
        combine: {{ dict1 | combine(dict2) }}
```

---

# Part 2. 중급 편 (Intermediate)

---

## 9장. Roles (역할)

### 9.1 Role 디렉토리 구조

```
roles/
└── nginx/
    ├── defaults/        # 기본 변수 (가장 낮은 우선순위)
    │   └── main.yml
    ├── vars/            # Role 변수 (높은 우선순위)
    │   └── main.yml
    ├── tasks/           # Task 목록
    │   ├── main.yml
    │   ├── install.yml
    │   └── configure.yml
    ├── handlers/        # 핸들러
    │   └── main.yml
    ├── templates/       # Jinja2 템플릿
    │   ├── nginx.conf.j2
    │   └── site.conf.j2
    ├── files/           # 정적 파일
    │   └── ssl/
    ├── meta/            # 의존성 및 메타데이터
    │   └── main.yml
    └── README.md
```

### 9.2 Role 예시: Nginx Role

```yaml
# roles/nginx/defaults/main.yml
---
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65
nginx_client_max_body_size: 10m
nginx_access_log: /var/log/nginx/access.log
nginx_error_log: /var/log/nginx/error.log
nginx_sites: []
nginx_remove_default_site: true
```

```yaml
# roles/nginx/tasks/main.yml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- import_tasks: install.yml
- import_tasks: configure.yml

- name: Ensure nginx is started
  service:
    name: nginx
    state: started
    enabled: yes
```

```yaml
# roles/nginx/tasks/install.yml
---
- name: Install nginx (Debian)
  apt:
    name: nginx
    state: present
    update_cache: yes
    cache_valid_time: 3600
  when: ansible_os_family == "Debian"

- name: Install nginx (RedHat)
  yum:
    name: nginx
    state: present
  when: ansible_os_family == "RedHat"
```

```yaml
# roles/nginx/tasks/configure.yml
---
- name: Deploy nginx.conf
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "0644"
    validate: "nginx -t -c %s"
  notify: Reload nginx

- name: Remove default site
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  when: nginx_remove_default_site
  notify: Reload nginx

- name: Deploy site configurations
  template:
    src: site.conf.j2
    dest: "/etc/nginx/sites-available/{{ item.name }}.conf"
    owner: root
    mode: "0644"
  loop: "{{ nginx_sites }}"
  notify: Reload nginx

- name: Enable sites
  file:
    src: "/etc/nginx/sites-available/{{ item.name }}.conf"
    dest: "/etc/nginx/sites-enabled/{{ item.name }}.conf"
    state: link
  loop: "{{ nginx_sites }}"
  notify: Reload nginx
```

```yaml
# roles/nginx/handlers/main.yml
---
- name: Reload nginx
  service:
    name: nginx
    state: reloaded

- name: Restart nginx
  service:
    name: nginx
    state: restarted
```

```yaml
# roles/nginx/meta/main.yml
---
galaxy_info:
  author: CloudOps Team
  description: Nginx installation and configuration
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: Ubuntu
      versions: [focal, jammy]
    - name: EL
      versions: [8, 9]

dependencies:
  - role: common
  - role: firewall
    vars:
      firewall_allowed_ports:
        - 80/tcp
        - 443/tcp
```

### 9.3 Role 사용법

```yaml
# playbooks/03_use_roles.yml
---
- name: 웹 서버 구성
  hosts: webservers
  become: yes
  roles:
    - common
    - role: nginx
      vars:
        nginx_sites:
          - name: myapp
            domain: myapp.example.com
            port: 80
            backend: "http://127.0.0.1:8080"
    - role: certbot
      when: app_env == "production"

# Role을 Task 레벨에서 동적으로 호출
- name: 조건부 Role 적용
  hosts: webservers
  become: yes
  tasks:
    - name: 기본 설정
      include_role:
        name: common

    - name: 환경별 Role 적용
      include_role:
        name: "{{ app_env }}_config"
```

### 9.4 Role 생성 명령

```bash
# 표준 구조의 Role 스캐폴딩
ansible-galaxy role init roles/myapp

# Ansible Galaxy에서 Role 설치
ansible-galaxy role install geerlingguy.docker
ansible-galaxy role install -r requirements.yml
```

```yaml
# requirements.yml
---
roles:
  - name: geerlingguy.docker
    version: "6.1.0"
  - name: geerlingguy.certbot
    version: "5.0.0"

collections:
  - name: amazon.aws
    version: ">=6.0.0"
  - name: azure.azcollection
    version: ">=1.15.0"
```

---

## 10장. Ansible Vault (비밀 관리)

### 10.1 기본 사용법

```bash
# 암호화 파일 생성
ansible-vault create vars/secrets.yml

# 기존 파일 암호화
ansible-vault encrypt vars/secrets.yml

# 복호화
ansible-vault decrypt vars/secrets.yml

# 내용 보기
ansible-vault view vars/secrets.yml

# 편집
ansible-vault edit vars/secrets.yml

# 비밀번호 변경
ansible-vault rekey vars/secrets.yml

# 비밀번호 파일로 실행
ansible-playbook site.yml --vault-password-file=~/.vault_pass
```

### 10.2 변수 단위 암호화 (Inline Vault)

```bash
# 개별 변수 값 암호화
ansible-vault encrypt_string 'SuperSecretP@ssw0rd!' --name 'db_password'
```

```yaml
# vars/secrets.yml
---
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  36613332626663326431653062303265643332...

api_key: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  31346432653563363930613939663161306139...

aws_secret_key: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  62323836323439653266326538366231353734...
```

### 10.3 Multi-Vault (여러 비밀번호)

```bash
# Vault ID를 사용한 다중 비밀번호
ansible-vault encrypt --vault-id dev@prompt vars/dev_secrets.yml
ansible-vault encrypt --vault-id prod@~/.vault_prod vars/prod_secrets.yml

# 실행 시 여러 Vault ID 전달
ansible-playbook site.yml \
  --vault-id dev@prompt \
  --vault-id prod@~/.vault_prod
```

---

## 11장. 에러 처리 및 디버깅

### 11.1 에러 처리

```yaml
---
- name: 에러 처리 패턴
  hosts: webservers
  become: yes
  tasks:
    # 실패 무시
    - name: 실패해도 계속 진행
      command: /opt/scripts/optional_cleanup.sh
      ignore_errors: yes

    # 실패 조건 커스터마이즈
    - name: API 헬스체크
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
        return_content: yes
      register: health_check
      failed_when:
        - health_check.status != 200
        - "'healthy' not in health_check.content"

    # changed 조건 커스터마이즈
    - name: DB 마이그레이션 실행
      command: /opt/app/migrate.sh
      register: migration_result
      changed_when: "'Applied' in migration_result.stdout"

    # Block / Rescue / Always (try-catch-finally)
    - name: 배포 프로세스
      block:
        - name: 신규 버전 배포
          copy:
            src: app-v2.jar
            dest: /opt/app/app.jar

        - name: 애플리케이션 재시작
          service:
            name: myapp
            state: restarted

        - name: 헬스체크
          uri:
            url: "http://localhost:{{ app_port }}/health"
            status_code: 200
          retries: 5
          delay: 10

      rescue:
        - name: 롤백 - 이전 버전 복원
          copy:
            src: /opt/app/backup/app.jar
            dest: /opt/app/app.jar

        - name: 롤백 - 서비스 재시작
          service:
            name: myapp
            state: restarted

        - name: 슬랙 알림 전송
          uri:
            url: "{{ slack_webhook }}"
            method: POST
            body_format: json
            body:
              text: "⚠️ 배포 실패! {{ inventory_hostname }}에서 롤백 수행됨"

      always:
        - name: 배포 로그 기록
          lineinfile:
            path: /var/log/deploy.log
            line: "{{ ansible_date_time.iso8601 }} - Deploy {{ 'FAILED' if ansible_failed_task is defined else 'SUCCESS' }}"
            create: yes
```

### 11.2 디버깅 기법

```bash
# 상세 출력
ansible-playbook site.yml -v      # 기본 상세
ansible-playbook site.yml -vv     # 더 상세
ansible-playbook site.yml -vvv    # 연결 정보 포함
ansible-playbook site.yml -vvvv   # 최대 상세 (SSH 디버깅)

# 단계별 실행
ansible-playbook site.yml --step

# 특정 Task부터 시작
ansible-playbook site.yml --start-at-task="Nginx 설치"
```

```yaml
# Playbook 내 디버깅
tasks:
  - name: 변수 확인
    debug:
      var: my_variable

  - name: 메시지 출력
    debug:
      msg: "현재 값: {{ my_variable }}, 타입: {{ my_variable | type_debug }}"

  - name: 디버거 중단점
    debug:
      msg: "여기서 중단"
    debugger: on_failed  # always, never, on_failed, on_unreachable, on_skipped
```

---

## 12장. 고급 Playbook 기법

### 12.1 include와 import의 차이

```yaml
# import_tasks: 정적 포함 (파싱 시점에 로드)
# - 태그, when 조건이 개별 Task에 적용됨
# - loop 사용 불가
- import_tasks: setup.yml

# include_tasks: 동적 포함 (실행 시점에 로드)
# - 태그, when 조건이 include 자체에만 적용됨
# - loop 사용 가능
- include_tasks: "setup_{{ ansible_os_family | lower }}.yml"
- include_tasks: deploy.yml
  loop: "{{ services }}"
```

### 12.2 태그 (Tags)

```yaml
---
- name: 태그 활용
  hosts: webservers
  become: yes
  tasks:
    - name: 패키지 설치
      apt:
        name: nginx
      tags: [install, nginx]

    - name: 설정 배포
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags: [configure, nginx]

    - name: 코드 배포
      copy:
        src: app/
        dest: /var/www/app/
      tags: [deploy]

    - name: 서비스 시작
      service:
        name: nginx
        state: started
      tags: [always]  # 항상 실행
```

```bash
# 태그별 실행
ansible-playbook site.yml --tags "deploy"
ansible-playbook site.yml --tags "install,configure"
ansible-playbook site.yml --skip-tags "deploy"
ansible-playbook site.yml --list-tags
```

### 12.3 Delegation & Serial

```yaml
---
- name: Rolling Update (무중단 배포)
  hosts: webservers
  become: yes
  serial: "25%"          # 전체 호스트의 25%씩 순차 처리
  max_fail_percentage: 10 # 10% 이상 실패 시 중단

  pre_tasks:
    - name: LB에서 제거
      uri:
        url: "http://{{ lb_host }}/api/servers/{{ inventory_hostname }}"
        method: DELETE
      delegate_to: localhost

  tasks:
    - name: 애플리케이션 업데이트
      copy:
        src: app-new.jar
        dest: /opt/app/app.jar
      notify: Restart app

  post_tasks:
    - name: 헬스체크 대기
      uri:
        url: "http://{{ inventory_hostname }}:{{ app_port }}/health"
        status_code: 200
      retries: 10
      delay: 5

    - name: LB에 재등록
      uri:
        url: "http://{{ lb_host }}/api/servers"
        method: POST
        body_format: json
        body:
          server: "{{ inventory_hostname }}"
      delegate_to: localhost

  handlers:
    - name: Restart app
      service:
        name: myapp
        state: restarted
```

### 12.4 비동기 작업 (Async)

```yaml
tasks:
  # 장시간 작업을 비동기로 실행
  - name: 대용량 데이터 백업 (비동기)
    command: /opt/scripts/full_backup.sh
    async: 3600    # 최대 1시간
    poll: 0        # 대기하지 않음
    register: backup_job

  - name: 다른 작업 수행
    apt:
      name: htop
      state: present

  # 비동기 작업 완료 대기
  - name: 백업 완료 대기
    async_status:
      jid: "{{ backup_job.ansible_job_id }}"
    register: backup_result
    until: backup_result.finished
    retries: 60
    delay: 60
```

---

# Part 3. 고급 편 (Advanced)

---

## 13장. Dynamic Inventory

### 13.1 AWS Dynamic Inventory

```yaml
# inventory/aws_ec2.yml
---
plugin: amazon.aws.aws_ec2

regions:
  - ap-northeast-2
  - us-east-1

filters:
  tag:Environment:
    - production
    - staging
  instance-state-name: running

keyed_groups:
  - key: tags.Environment
    prefix: env
    separator: "_"
  - key: tags.Role
    prefix: role
    separator: "_"
  - key: placement.availability_zone
    prefix: az
  - key: instance_type
    prefix: type

hostnames:
  - tag:Name
  - private-ip-address

compose:
  ansible_host: private_ip_address
  ansible_user: "'ubuntu' if image_id.startswith('ami-ubuntu') else 'ec2-user'"
  instance_name: tags.Name | default('unnamed')
```

```bash
# AWS 자격 증명 설정
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="ap-northeast-2"

# 또는 AWS CLI 프로파일 사용
export AWS_PROFILE=production

# 동적 인벤토리 테스트
ansible-inventory -i inventory/aws_ec2.yml --list
ansible-inventory -i inventory/aws_ec2.yml --graph

# 동적 인벤토리 기반 실행
ansible -i inventory/aws_ec2.yml env_production -m ping
```

### 13.2 Azure Dynamic Inventory

```yaml
# inventory/azure_rm.yml
---
plugin: azure.azcollection.azure_rm

auth_source: auto  # env, cli, msi, auto

include_vm_resource_groups:
  - production-rg
  - staging-rg

keyed_groups:
  - key: tags.Environment | default('untagged')
    prefix: env
  - key: tags.Role | default('untagged')
    prefix: role
  - key: location
    prefix: region

hostnames:
  - tag:Name
  - default  # VM 이름

compose:
  ansible_host: private_ipv4_addresses[0] | default(public_ipv4_addresses[0], true)
  ansible_user: "'azureuser'"

conditional_groups:
  linux: os_profile.linux_configuration is defined
  windows: os_profile.windows_configuration is defined

exclude_host_filters:
  - powerstate != 'running'
```

```bash
# Azure 자격 증명
export AZURE_SUBSCRIPTION_ID="xxxx-xxxx-xxxx"
export AZURE_CLIENT_ID="xxxx-xxxx-xxxx"
export AZURE_SECRET="xxxx-xxxx-xxxx"
export AZURE_TENANT="xxxx-xxxx-xxxx"

# 또는 Azure CLI 로그인
az login

ansible-inventory -i inventory/azure_rm.yml --graph
```

---

## 14장. AWS 인프라 자동화

### 14.1 필수 컬렉션 설치

```bash
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install community.aws
```

### 14.2 VPC + EC2 + ALB 완전 자동화

```yaml
# playbooks/aws_infra.yml
---
- name: AWS 인프라 프로비저닝
  hosts: localhost
  connection: local
  gather_facts: no

  vars:
    aws_region: ap-northeast-2
    project_name: mywebapp
    environment: production
    vpc_cidr: "10.0.0.0/16"
    public_subnets:
      - { cidr: "10.0.1.0/24", az: "ap-northeast-2a" }
      - { cidr: "10.0.2.0/24", az: "ap-northeast-2c" }
    private_subnets:
      - { cidr: "10.0.10.0/24", az: "ap-northeast-2a" }
      - { cidr: "10.0.11.0/24", az: "ap-northeast-2c" }
    ec2_instance_type: t3.medium
    ec2_ami: ami-0c9c942bd7bf113a2  # Ubuntu 22.04
    ec2_key_name: my-keypair
    ec2_instance_count: 2

  tasks:
    # ─── VPC ───
    - name: VPC 생성
      amazon.aws.ec2_vpc_net:
        name: "{{ project_name }}-vpc"
        cidr_block: "{{ vpc_cidr }}"
        region: "{{ aws_region }}"
        dns_support: yes
        dns_hostnames: yes
        tags:
          Project: "{{ project_name }}"
          Environment: "{{ environment }}"
      register: vpc

    # ─── Internet Gateway ───
    - name: Internet Gateway 생성
      amazon.aws.ec2_vpc_igw:
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        state: present
        tags:
          Name: "{{ project_name }}-igw"
      register: igw

    # ─── Public Subnets ───
    - name: Public 서브넷 생성
      amazon.aws.ec2_vpc_subnet:
        vpc_id: "{{ vpc.vpc.id }}"
        cidr: "{{ item.cidr }}"
        az: "{{ item.az }}"
        region: "{{ aws_region }}"
        map_public: yes
        tags:
          Name: "{{ project_name }}-public-{{ item.az }}"
          Tier: public
      loop: "{{ public_subnets }}"
      register: public_subnet_results

    # ─── Private Subnets ───
    - name: Private 서브넷 생성
      amazon.aws.ec2_vpc_subnet:
        vpc_id: "{{ vpc.vpc.id }}"
        cidr: "{{ item.cidr }}"
        az: "{{ item.az }}"
        region: "{{ aws_region }}"
        tags:
          Name: "{{ project_name }}-private-{{ item.az }}"
          Tier: private
      loop: "{{ private_subnets }}"
      register: private_subnet_results

    # ─── NAT Gateway ───
    - name: EIP for NAT Gateway
      amazon.aws.ec2_eip:
        region: "{{ aws_region }}"
        in_vpc: yes
        tags:
          Name: "{{ project_name }}-nat-eip"
      register: nat_eip

    - name: NAT Gateway 생성
      amazon.aws.ec2_vpc_nat_gateway:
        subnet_id: "{{ public_subnet_results.results[0].subnet.id }}"
        eip_address: "{{ nat_eip.public_ip }}"
        region: "{{ aws_region }}"
        wait: yes
        tags:
          Name: "{{ project_name }}-nat"
      register: nat_gw

    # ─── Route Tables ───
    - name: Public 라우트 테이블
      amazon.aws.ec2_vpc_route_table:
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        subnets: "{{ public_subnet_results.results | map(attribute='subnet.id') | list }}"
        routes:
          - dest: 0.0.0.0/0
            gateway_id: "{{ igw.gateway_id }}"
        tags:
          Name: "{{ project_name }}-public-rt"

    - name: Private 라우트 테이블
      amazon.aws.ec2_vpc_route_table:
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        subnets: "{{ private_subnet_results.results | map(attribute='subnet.id') | list }}"
        routes:
          - dest: 0.0.0.0/0
            nat_gateway_id: "{{ nat_gw.nat_gateway_id }}"
        tags:
          Name: "{{ project_name }}-private-rt"

    # ─── Security Groups ───
    - name: ALB 보안 그룹
      amazon.aws.ec2_security_group:
        name: "{{ project_name }}-alb-sg"
        description: ALB Security Group
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            ports: [80, 443]
            cidr_ip: 0.0.0.0/0
            rule_desc: Allow HTTP/HTTPS
        tags:
          Name: "{{ project_name }}-alb-sg"
      register: alb_sg

    - name: EC2 보안 그룹
      amazon.aws.ec2_security_group:
        name: "{{ project_name }}-ec2-sg"
        description: EC2 Security Group
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            ports: [8080]
            group_id: "{{ alb_sg.group_id }}"
            rule_desc: Allow from ALB
          - proto: tcp
            ports: [22]
            cidr_ip: 10.0.0.0/16
            rule_desc: SSH from VPC
        tags:
          Name: "{{ project_name }}-ec2-sg"
      register: ec2_sg

    # ─── EC2 Instances ───
    - name: EC2 인스턴스 생성
      amazon.aws.ec2_instance:
        name: "{{ project_name }}-web-{{ item }}"
        key_name: "{{ ec2_key_name }}"
        instance_type: "{{ ec2_instance_type }}"
        image_id: "{{ ec2_ami }}"
        region: "{{ aws_region }}"
        vpc_subnet_id: "{{ private_subnet_results.results[item % 2].subnet.id }}"
        security_groups:
          - "{{ ec2_sg.group_id }}"
        wait: yes
        volumes:
          - device_name: /dev/sda1
            ebs:
              volume_size: 30
              volume_type: gp3
              delete_on_termination: true
        user_data: |
          #!/bin/bash
          apt-get update && apt-get install -y python3
        tags:
          Project: "{{ project_name }}"
          Environment: "{{ environment }}"
          Role: webserver
      loop: "{{ range(ec2_instance_count) | list }}"
      register: ec2_instances

    # ─── ALB ───
    - name: Target Group 생성
      community.aws.elb_target_group:
        name: "{{ project_name }}-tg"
        protocol: HTTP
        port: 8080
        vpc_id: "{{ vpc.vpc.id }}"
        region: "{{ aws_region }}"
        health_check_protocol: HTTP
        health_check_path: /health
        health_check_interval: 30
        healthy_threshold_count: 3
        unhealthy_threshold_count: 3
        targets: "{{ ec2_instances.results | map(attribute='instances') | flatten | map(attribute='instance_id') | map('community.general.dict_kv', 'Id') | list }}"
        state: present
      register: target_group

    - name: ALB 생성
      community.aws.elb_application_lb:
        name: "{{ project_name }}-alb"
        region: "{{ aws_region }}"
        scheme: internet-facing
        security_groups:
          - "{{ alb_sg.group_id }}"
        subnets: "{{ public_subnet_results.results | map(attribute='subnet.id') | list }}"
        listeners:
          - Protocol: HTTP
            Port: 80
            DefaultActions:
              - Type: forward
                TargetGroupArn: "{{ target_group.target_group_arn }}"
        state: present
        tags:
          Project: "{{ project_name }}"
      register: alb

    - name: 인프라 요약 출력
      debug:
        msg: |
          ===== 인프라 프로비저닝 완료 =====
          VPC ID: {{ vpc.vpc.id }}
          ALB DNS: {{ alb.dns_name }}
          EC2 인스턴스: {{ ec2_instances.results | map(attribute='instances') | flatten | map(attribute='instance_id') | list }}
```

### 14.3 S3 + CloudFront 정적 웹사이트

```yaml
# playbooks/aws_static_site.yml
---
- name: S3 + CloudFront 정적 사이트 배포
  hosts: localhost
  connection: local
  gather_facts: no

  vars:
    bucket_name: mywebapp-static-assets
    aws_region: ap-northeast-2
    domain_name: static.example.com

  tasks:
    - name: S3 버킷 생성
      amazon.aws.s3_bucket:
        name: "{{ bucket_name }}"
        region: "{{ aws_region }}"
        state: present
        public_access:
          block_public_acls: true
          ignore_public_acls: true
          block_public_policy: true
          restrict_public_buckets: true
        versioning: yes
        tags:
          Project: mywebapp

    - name: 정적 파일 업로드
      amazon.aws.s3_object:
        bucket: "{{ bucket_name }}"
        object: "{{ item }}"
        src: "dist/{{ item }}"
        mode: put
        content_type: "{{ item | regex_search('\\.(html|css|js|json|png|jpg|svg)$') | ternary(
          {'html': 'text/html', 'css': 'text/css', 'js': 'application/javascript',
           'json': 'application/json', 'png': 'image/png', 'jpg': 'image/jpeg',
           'svg': 'image/svg+xml'}[item | regex_search('\\w+$')],
          'application/octet-stream') }}"
      loop:
        - index.html
        - styles/main.css
        - scripts/app.js

    - name: CloudFront OAC 정책으로 배포
      community.aws.cloudfront_distribution:
        state: present
        origins:
          - id: "S3-{{ bucket_name }}"
            domain_name: "{{ bucket_name }}.s3.{{ aws_region }}.amazonaws.com"
            s3_origin_config:
              origin_access_identity: ""
        default_cache_behavior:
          target_origin_id: "S3-{{ bucket_name }}"
          viewer_protocol_policy: redirect-to-https
          allowed_methods: [GET, HEAD]
          cached_methods: [GET, HEAD]
          compress: true
          forwarded_values:
            query_string: false
        default_root_object: index.html
        enabled: true
        comment: "{{ domain_name }} static site"
      register: cf_distribution

    - name: CloudFront 배포 정보
      debug:
        msg: "CloudFront Domain: {{ cf_distribution.domain_name }}"
```

---

## 15장. Azure 인프라 자동화

### 15.1 필수 컬렉션 설치

```bash
ansible-galaxy collection install azure.azcollection
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt
```

### 15.2 VNet + VM + Load Balancer 완전 자동화

```yaml
# playbooks/azure_infra.yml
---
- name: Azure 인프라 프로비저닝
  hosts: localhost
  connection: local
  gather_facts: no

  vars:
    resource_group: mywebapp-rg
    location: koreacentral
    project_name: mywebapp
    vnet_cidr: "10.0.0.0/16"
    subnet_web_cidr: "10.0.1.0/24"
    subnet_db_cidr: "10.0.2.0/24"
    vm_size: Standard_B2s
    vm_count: 2
    admin_username: azureuser

  tasks:
    # ─── Resource Group ───
    - name: 리소스 그룹 생성
      azure.azcollection.azure_rm_resourcegroup:
        name: "{{ resource_group }}"
        location: "{{ location }}"
        tags:
          Project: "{{ project_name }}"
          Environment: production

    # ─── VNet ───
    - name: Virtual Network 생성
      azure.azcollection.azure_rm_virtualnetwork:
        resource_group: "{{ resource_group }}"
        name: "{{ project_name }}-vnet"
        address_prefixes: "{{ vnet_cidr }}"
        tags:
          Project: "{{ project_name }}"

    # ─── Subnets ───
    - name: Web 서브넷 생성
      azure.azcollection.azure_rm_subnet:
        resource_group: "{{ resource_group }}"
        virtual_network: "{{ project_name }}-vnet"
        name: "{{ project_name }}-web-subnet"
        address_prefix: "{{ subnet_web_cidr }}"
      register: web_subnet

    - name: DB 서브넷 생성
      azure.azcollection.azure_rm_subnet:
        resource_group: "{{ resource_group }}"
        virtual_network: "{{ project_name }}-vnet"
        name: "{{ project_name }}-db-subnet"
        address_prefix: "{{ subnet_db_cidr }}"

    # ─── NSG ───
    - name: Web NSG 생성
      azure.azcollection.azure_rm_securitygroup:
        resource_group: "{{ resource_group }}"
        name: "{{ project_name }}-web-nsg"
        rules:
          - name: AllowHTTP
            protocol: Tcp
            destination_port_range: [80, 443]
            access: Allow
            priority: 100
            direction: Inbound
            source_address_prefix: "*"
          - name: AllowSSH
            protocol: Tcp
            destination_port_range: 22
            access: Allow
            priority: 110
            direction: Inbound
            source_address_prefix: "10.0.0.0/16"
      register: web_nsg

    # ─── Public IP ───
    - name: Public IP 생성 (LB용)
      azure.azcollection.azure_rm_publicipaddress:
        resource_group: "{{ resource_group }}"
        name: "{{ project_name }}-lb-pip"
        allocation_method: Static
        sku: Standard
      register: lb_pip

    # ─── Load Balancer ───
    - name: Load Balancer 생성
      azure.azcollection.azure_rm_loadbalancer:
        resource_group: "{{ resource_group }}"
        name: "{{ project_name }}-lb"
        sku: Standard
        frontend_ip_configurations:
          - name: frontend
            public_ip_address: "{{ project_name }}-lb-pip"
        backend_address_pools:
          - name: webpool
        probes:
          - name: healthprobe
            protocol: Http
            port: 80
            request_path: /health
            interval: 15
            fail_count: 3
        load_balancing_rules:
          - name: http-rule
            frontend_ip_configuration: frontend
            backend_address_pool: webpool
            probe: healthprobe
            protocol: Tcp
            frontend_port: 80
            backend_port: 80
      register: lb

    # ─── Availability Set ───
    - name: Availability Set 생성
      azure.azcollection.azure_rm_availabilityset:
        resource_group: "{{ resource_group }}"
        name: "{{ project_name }}-avset"
        platform_fault_domain_count: 2
        platform_update_domain_count: 5
        sku: Aligned

    # ─── NIC + VM ───
    - name: NIC 생성
      azure.azcollection.azure_rm_networkinterface:
        resource_group: "{{ resource_group }}"
        name: "{{ project_name }}-nic-{{ item }}"
        virtual_network: "{{ project_name }}-vnet"
        subnet: "{{ project_name }}-web-subnet"
        security_group: "{{ project_name }}-web-nsg"
        ip_configurations:
          - name: default
            primary: yes
            load_balancer_backend_address_pools:
              - name: webpool
                load_balancer: "{{ project_name }}-lb"
      loop: "{{ range(vm_count) | list }}"

    - name: VM 생성
      azure.azcollection.azure_rm_virtualmachine:
        resource_group: "{{ resource_group }}"
        name: "{{ project_name }}-vm-{{ item }}"
        vm_size: "{{ vm_size }}"
        admin_username: "{{ admin_username }}"
        ssh_password_enabled: false
        ssh_public_keys:
          - path: "/home/{{ admin_username }}/.ssh/authorized_keys"
            key_data: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
        network_interfaces:
          - "{{ project_name }}-nic-{{ item }}"
        availability_set: "{{ project_name }}-avset"
        image:
          offer: 0001-com-ubuntu-server-jammy
          publisher: Canonical
          sku: "22_04-lts-gen2"
          version: latest
        managed_disk_type: Premium_LRS
        os_disk_size_gb: 30
        tags:
          Project: "{{ project_name }}"
          Role: webserver
      loop: "{{ range(vm_count) | list }}"
      async: 600
      poll: 0
      register: vm_jobs

    - name: VM 생성 완료 대기
      async_status:
        jid: "{{ item.ansible_job_id }}"
      loop: "{{ vm_jobs.results }}"
      register: vm_results
      until: vm_results.finished
      retries: 30
      delay: 20

    - name: 결과 출력
      debug:
        msg: |
          ===== Azure 인프라 완료 =====
          Resource Group: {{ resource_group }}
          LB Public IP: {{ lb_pip.state.ip_address }}
          VM 수: {{ vm_count }}
```

### 15.3 Azure AKS 클러스터 배포

```yaml
# playbooks/azure_aks.yml
---
- name: AKS 클러스터 배포
  hosts: localhost
  connection: local
  gather_facts: no

  vars:
    resource_group: aks-rg
    location: koreacentral
    cluster_name: myapp-aks
    k8s_version: "1.28"
    node_count: 3
    node_vm_size: Standard_D4s_v3

  tasks:
    - name: 리소스 그룹 생성
      azure.azcollection.azure_rm_resourcegroup:
        name: "{{ resource_group }}"
        location: "{{ location }}"

    - name: AKS 클러스터 생성
      azure.azcollection.azure_rm_aks:
        name: "{{ cluster_name }}"
        resource_group: "{{ resource_group }}"
        location: "{{ location }}"
        kubernetes_version: "{{ k8s_version }}"
        dns_prefix: "{{ cluster_name }}"
        linux_profile:
          admin_username: azureuser
          ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
        agent_pool_profiles:
          - name: nodepool1
            count: "{{ node_count }}"
            vm_size: "{{ node_vm_size }}"
            os_disk_size_gb: 100
            mode: System
            type: VirtualMachineScaleSets
            enable_auto_scaling: true
            min_count: 2
            max_count: 5
        network_profile:
          network_plugin: azure
          network_policy: calico
          load_balancer_sku: standard
        addon:
          monitoring:
            enabled: true
        tags:
          Project: myapp
          Environment: production
      register: aks_cluster

    - name: kubeconfig 가져오기
      azure.azcollection.azure_rm_aks_info:
        name: "{{ cluster_name }}"
        resource_group: "{{ resource_group }}"
        show_kubeconfig: user
      register: aks_info

    - name: kubeconfig 저장
      copy:
        content: "{{ aks_info.aks[0].kube_config }}"
        dest: "~/.kube/config-{{ cluster_name }}"
        mode: "0600"
```

---

## 16장. Ansible Collections 심화

### 16.1 Collection 구조

```
collections/
└── ansible_collections/
    └── mycompany/
        └── cloud_ops/
            ├── galaxy.yml
            ├── plugins/
            │   ├── modules/
            │   │   └── deploy_app.py
            │   ├── inventory/
            │   │   └── custom_cmdb.py
            │   ├── filter/
            │   │   └── custom_filters.py
            │   └── callback/
            │       └── slack_notify.py
            ├── roles/
            │   ├── webserver/
            │   └── database/
            └── playbooks/
                └── full_deploy.yml
```

### 16.2 Custom Module 작성

```python
#!/usr/bin/python
# plugins/modules/deploy_app.py

from ansible.module_utils.basic import AnsibleModule
import os
import subprocess

DOCUMENTATION = r'''
---
module: deploy_app
short_description: Deploy application with health check
description:
  - Downloads application artifact, deploys it, and validates health
options:
  artifact_url:
    description: URL of the application artifact
    required: true
    type: str
  deploy_path:
    description: Deployment target directory
    required: true
    type: str
  health_url:
    description: Health check endpoint
    required: false
    type: str
  health_retries:
    description: Number of health check retries
    required: false
    type: int
    default: 5
'''

def main():
    module = AnsibleModule(
        argument_spec=dict(
            artifact_url=dict(type='str', required=True),
            deploy_path=dict(type='str', required=True),
            health_url=dict(type='str', required=False),
            health_retries=dict(type='int', default=5),
        ),
        supports_check_mode=True,
    )

    artifact_url = module.params['artifact_url']
    deploy_path = module.params['deploy_path']
    health_url = module.params['health_url']

    if module.check_mode:
        module.exit_json(changed=True, msg="Would deploy from {}".format(artifact_url))

    # 배포 로직
    try:
        os.makedirs(deploy_path, exist_ok=True)
        rc, stdout, stderr = module.run_command(
            ['curl', '-fsSL', '-o', os.path.join(deploy_path, 'app.jar'), artifact_url]
        )
        if rc != 0:
            module.fail_json(msg="Download failed: {}".format(stderr))

        module.exit_json(
            changed=True,
            msg="Deployed successfully",
            artifact_url=artifact_url,
            deploy_path=deploy_path,
        )
    except Exception as e:
        module.fail_json(msg=str(e))

if __name__ == '__main__':
    main()
```

### 16.3 Custom Filter Plugin

```python
# plugins/filter/custom_filters.py

class FilterModule(object):
    def filters(self):
        return {
            'to_tag_list': self.to_tag_list,
            'subnet_ids': self.subnet_ids,
        }

    def to_tag_list(self, tags_dict):
        """Dict를 AWS 태그 리스트 형식으로 변환"""
        return [{"Key": k, "Value": v} for k, v in tags_dict.items()]

    def subnet_ids(self, subnet_results):
        """서브넷 결과에서 ID만 추출"""
        return [s['subnet']['id'] for s in subnet_results]
```

---

## 17장. Ansible AWX / Tower / AAP

### 17.1 AWX (오픈소스 Tower) Docker 설치

```bash
# AWX Operator로 설치 (K8s 환경)
git clone https://github.com/ansible/awx-operator.git
cd awx-operator
git checkout tags/2.7.0
make deploy

# AWX 인스턴스 생성
cat <<EOF | kubectl apply -f -
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: NodePort
  projects_persistence: true
  projects_storage_size: 8Gi
EOF
```

### 17.2 AWX 활용 시나리오

AWX/Tower/AAP는 다음과 같은 기능을 제공한다. 첫째, 웹 기반 UI로 Playbook 실행 및 모니터링. 둘째, RBAC(역할 기반 접근 제어)로 팀별 권한 관리. 셋째, Credential 관리로 SSH 키, 클라우드 인증 정보를 중앙화. 넷째, Job 스케줄링으로 정기적인 자동화 작업 설정. 다섯째, Workflow로 여러 Playbook을 연결하여 복잡한 파이프라인 구성. 여섯째, API를 통해 CI/CD 파이프라인과 통합.

---

## 18장. CI/CD 파이프라인 통합

### 18.1 GitHub Actions + Ansible

```yaml
# .github/workflows/deploy.yml
name: Deploy with Ansible

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  ANSIBLE_HOST_KEY_CHECKING: "False"

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install Dependencies
        run: |
          pip install ansible boto3 botocore
          ansible-galaxy install -r requirements.yml

      - name: Configure SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519

      - name: Run Ansible Playbook
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASSWORD }}
        run: |
          echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
          ansible-playbook playbooks/deploy.yml \
            -i inventory/aws_ec2.yml \
            --vault-password-file .vault_pass \
            -e "deploy_version=${{ github.sha }}"
          rm -f .vault_pass
```

### 18.2 프로젝트 디렉토리 표준 구조

```
ansible-project/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── production/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   ├── webservers.yml
│   │   │   └── dbservers.yml
│   │   └── host_vars/
│   │       └── web1.yml
│   ├── staging/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── all.yml
│   └── aws_ec2.yml          # 동적 인벤토리
├── playbooks/
│   ├── site.yml              # 마스터 Playbook
│   ├── webservers.yml
│   ├── dbservers.yml
│   ├── deploy.yml
│   └── rollback.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── app/
│   └── postgresql/
├── vars/
│   ├── secrets.yml           # Vault 암호화
│   └── global.yml
├── templates/
├── files/
├── filter_plugins/
├── library/                  # Custom Modules
└── .github/
    └── workflows/
        └── deploy.yml
```

```yaml
# playbooks/site.yml (마스터 Playbook)
---
- name: 공통 설정
  hosts: all
  become: yes
  roles:
    - common

- name: 웹 서버 구성
  hosts: webservers
  become: yes
  roles:
    - nginx
    - app

- name: DB 서버 구성
  hosts: dbservers
  become: yes
  roles:
    - postgresql
```

---

## 19장. 성능 최적화

### 19.1 ansible.cfg 최적화

```ini
[defaults]
forks = 20
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600
callback_whitelist = timer, profile_tasks
stdout_callback = yaml
internal_poll_interval = 0.001

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=600s -o PreferredAuthentications=publickey
```

### 19.2 Playbook 최적화 기법

```yaml
---
- name: 최적화된 Playbook
  hosts: webservers
  become: yes
  gather_facts: no  # Facts가 불필요하면 비활성화

  # 필요한 Facts만 수집
  # gather_facts: yes
  # gather_subset:
  #   - network
  #   - hardware

  pre_tasks:
    - name: 최소한의 Facts만 수집
      setup:
        gather_subset:
          - "!all"
          - network
      when: ansible_facts == {}

  tasks:
    # 패키지를 개별이 아닌 한 번에 설치
    - name: 패키지 일괄 설치 (비효율)
      apt:
        name: "{{ item }}"
      loop:
        - nginx
        - curl
        - htop

    # 이것이 더 효율적
    - name: 패키지 일괄 설치 (효율)
      apt:
        name:
          - nginx
          - curl
          - htop
        state: present
        update_cache: yes
        cache_valid_time: 3600

    # free 전략으로 병렬 실행
  strategy: free  # 기본은 linear (모든 호스트가 동일 Task 완료 후 다음으로)
```

### 19.3 Mitogen (속도 향상 플러그인)

Mitogen은 Ansible의 SSH 실행을 최적화하여 2~7배 성능 향상을 제공하는 strategy 플러그인이다. 작동 원리는 SSH 연결을 통해 Python 인터프리터를 직접 실행하여, 임시 파일 복사를 최소화하는 것이다.

```ini
# ansible.cfg
[defaults]
strategy_plugins = /path/to/mitogen/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
```

---

# Part 4. 문제집 (총 30문제)

---

## 초급 문제 (1~10번)

### 문제 1.
Ansible의 특징으로 **올바르지 않은** 것은?

A) Agentless 아키텍처를 사용한다
B) YAML 형식으로 Playbook을 작성한다
C) 대상 호스트에 Ansible Agent 설치가 필요하다
D) SSH를 통해 원격 호스트에 접속한다

**정답: C**

**해설:** Ansible의 핵심 차별점은 Agentless 아키텍처다. 대상 호스트에 별도의 에이전트를 설치할 필요 없이 SSH(Linux) 또는 WinRM(Windows)을 통해 원격으로 작업을 수행한다. Chef나 Puppet과 달리 대상 호스트에는 Python만 설치되어 있으면 된다.

---

### 문제 2.
다음 인벤토리에서 `ansible dbservers -m ping`을 실행하면 ping을 수행하는 호스트는?

```ini
[webservers]
web1 ansible_host=10.0.1.10
web2 ansible_host=10.0.1.11

[dbservers]
db1 ansible_host=10.0.2.10

[production:children]
webservers
dbservers
```

A) web1, web2, db1
B) db1만
C) web1, web2만
D) 모든 호스트

**정답: B**

**해설:** 호스트 패턴 `dbservers`는 해당 그룹의 호스트만 지정한다. `production:children` 은 `webservers`와 `dbservers`를 하위 그룹으로 포함하는 상위 그룹이지만, 명시적으로 `dbservers`만 지정했으므로 `db1`만 대상이 된다. `production` 그룹을 지정해야 3개 호스트 모두가 대상이 된다.

---

### 문제 3.
Ansible에서 **멱등성(Idempotency)**의 정의로 올바른 것은?

A) 한 번에 여러 호스트에 동시 실행하는 능력
B) 동일한 작업을 여러 번 실행해도 결과가 동일한 성질
C) 암호화된 변수를 사용하는 기능
D) 롤백이 가능한 배포 방식

**정답: B**

**해설:** 멱등성은 동일한 작업을 1번 실행하든 100번 실행하든 최종 결과가 동일함을 보장하는 성질이다. 예를 들어 `apt: name=nginx state=present`는 Nginx가 이미 설치되어 있으면 변경 없이 `ok`를 반환한다. 이 성질 덕분에 Ansible Playbook을 안전하게 반복 실행할 수 있다.

---

### 문제 4.
다음 Task에서 `notify`가 트리거하는 핸들러는 언제 실행되는가?

```yaml
tasks:
  - name: Nginx 설정 배포
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart Nginx

  - name: 앱 코드 배포
    copy:
      src: app/
      dest: /var/www/app/
```

A) template Task 직후 즉시
B) copy Task 직후 즉시
C) 모든 Task가 완료된 후
D) Playbook 전체가 끝난 후

**정답: C**

**해설:** Handler는 해당 Play 내의 모든 Task가 완료된 후에 실행된다. Task 직후가 아니다. 만약 즉시 실행이 필요하면 `meta: flush_handlers`를 사용한다. 또한 여러 Task에서 같은 Handler를 notify해도 한 번만 실행된다.

---

### 문제 5.
Ansible 변수의 우선순위가 **가장 높은** 것은?

A) Role defaults
B) Inventory group_vars
C) Playbook vars
D) Extra vars (-e 옵션)

**정답: D**

**해설:** Ansible은 22단계의 변수 우선순위를 가지며, `extra vars`(-e 옵션)가 가장 높은 우선순위를 가진다. Role defaults가 가장 낮고, 그 위로 inventory 변수, playbook 변수, task 변수 순으로 올라간다. 따라서 `-e "version=2.0"` 으로 전달한 값은 어떤 곳에서 정의한 값이든 덮어쓴다.

---

### 문제 6.
다음 중 Ansible Ad-Hoc 명령의 올바른 문법은?

A) `ansible -m ping all`
B) `ansible all -m ping`
C) `ansible ping -m all`
D) `ansible-playbook all -m ping`

**정답: B**

**해설:** Ad-Hoc 명령의 기본 문법은 `ansible <호스트패턴> -m <모듈명> -a "<인자>"`이다. 호스트 패턴이 먼저 오고, `-m`으로 모듈을 지정한다. `ansible-playbook`은 Playbook 실행 명령이므로 Ad-Hoc으로 사용하지 않는다.

---

### 문제 7.
다음 Playbook에서 빈칸에 들어갈 알맞은 키워드는?

```yaml
- name: Ubuntu에만 실행
  apt:
    name: nginx
    state: present
  ______: ansible_os_family == "Debian"
```

A) if
B) condition
C) when
D) only_if

**정답: C**

**해설:** Ansible에서 조건부 실행은 `when` 키워드를 사용한다. `when` 절은 Jinja2 표현식을 평가하여 True이면 Task를 실행하고, False이면 건너뛴다. `if`는 Jinja2 템플릿 내에서 사용하는 구문이고, `condition`이나 `only_if`는 Ansible에 존재하지 않는 키워드다.

---

### 문제 8.
`ansible.cfg` 설정 파일의 탐색 순서로 올바른 것은?

A) `/etc/ansible/ansible.cfg` → `~/.ansible.cfg` → `./ansible.cfg` → `ANSIBLE_CONFIG`
B) `ANSIBLE_CONFIG` → `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`
C) `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg` → `ANSIBLE_CONFIG`
D) `ANSIBLE_CONFIG` → `/etc/ansible/ansible.cfg` → `~/.ansible.cfg` → `./ansible.cfg`

**정답: B**

**해설:** Ansible은 설정 파일을 다음 순서로 탐색하여 가장 먼저 발견된 파일을 사용한다: `ANSIBLE_CONFIG` 환경변수 → 현재 디렉토리 `./ansible.cfg` → 홈 디렉토리 `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`. 프로젝트별 설정은 프로젝트 루트에 `ansible.cfg`를 두는 것이 일반적이다.

---

### 문제 9.
다음 Jinja2 표현식의 결과로 올바른 것은?

```yaml
vars:
  servers:
    - web1
    - web2
    - web3
tasks:
  - debug:
      msg: "{{ servers | join(', ') }}"
```

A) `["web1", "web2", "web3"]`
B) `web1, web2, web3`
C) `web1 web2 web3`
D) 에러 발생

**정답: B**

**해설:** Jinja2의 `join` 필터는 리스트의 요소를 지정된 구분자로 연결하여 하나의 문자열로 만든다. `', '`를 구분자로 사용했으므로 결과는 `web1, web2, web3`이다.

---

### 문제 10.
Ansible Playbook을 실제 실행하지 않고 변경될 내용만 확인하는 명령은?

A) `ansible-playbook site.yml --dry-run`
B) `ansible-playbook site.yml --check --diff`
C) `ansible-playbook site.yml --test`
D) `ansible-playbook site.yml --preview`

**정답: B**

**해설:** `--check` 옵션은 Dry-run 모드로, 실제 변경을 가하지 않고 변경 여부만 확인한다. `--diff`를 함께 사용하면 파일 변경의 구체적인 차이점(diff)을 보여준다. 단, 모든 모듈이 check mode를 지원하는 것은 아니며, `command`나 `shell` 모듈은 check mode에서 skip된다.

---

## 중급 문제 (11~20번)

### 문제 11.
다음 Role 디렉토리 구조에서, 변수 우선순위가 **가장 낮은** 곳은?

```
roles/nginx/
├── defaults/main.yml    # (A)
├── vars/main.yml        # (B)
├── tasks/main.yml
└── meta/main.yml
```

A) defaults/main.yml
B) vars/main.yml
C) 둘 다 동일
D) tasks에서 정의한 변수

**정답: A**

**해설:** `defaults/main.yml`은 Ansible 전체 변수 우선순위 체계에서 가장 낮다. 이 파일은 Role의 "기본값"을 제공하는 용도로, 사용자가 어디서든 쉽게 덮어쓸 수 있도록 의도적으로 낮은 우선순위를 가진다. 반면 `vars/main.yml`은 훨씬 높은 우선순위를 가져, Role 내에서 변경되지 않아야 할 값을 정의할 때 사용한다.

---

### 문제 12.
`import_tasks`와 `include_tasks`의 차이로 올바른 것은?

A) `import_tasks`는 동적, `include_tasks`는 정적
B) `import_tasks`는 정적(파싱 시점), `include_tasks`는 동적(실행 시점)
C) 둘은 완전히 동일한 기능
D) `include_tasks`만 loop 사용 가능, `import_tasks`도 loop 사용 가능

**정답: B**

**해설:** `import_tasks`는 Playbook 파싱 시점에 정적으로 포함되어, 태그와 when 조건이 포함된 개별 Task에 각각 적용된다. `include_tasks`는 실행 시점에 동적으로 포함되어, 변수에 따른 파일명 결정이나 loop 사용이 가능하다. `import_tasks`에서는 loop를 사용할 수 없다.

---

### 문제 13.
Ansible Vault로 개별 변수를 암호화하는 올바른 명령은?

A) `ansible-vault encrypt vars/secrets.yml`
B) `ansible-vault encrypt_string 'MySecret' --name 'db_password'`
C) `ansible-vault create --variable db_password`
D) `ansible-vault encrypt --inline 'MySecret'`

**정답: B**

**해설:** `encrypt_string`은 파일 전체가 아닌 개별 문자열을 암호화하는 명령이다. `--name` 옵션으로 변수 이름을 지정하면, YAML에 직접 붙여넣을 수 있는 `!vault` 형식의 암호화된 값을 출력한다. 이 방식을 사용하면 비밀이 아닌 변수와 비밀 변수를 같은 파일에 혼합할 수 있어 관리가 편리하다.

---

### 문제 14.
다음 코드의 `block/rescue/always` 구조에서 rescue 블록이 실행되는 조건은?

```yaml
- block:
    - name: 배포
      copy: ...
    - name: 헬스체크
      uri: ...
  rescue:
    - name: 롤백
      copy: ...
  always:
    - name: 로그 기록
      lineinfile: ...
```

A) block 내 모든 Task 성공 시
B) block 내 하나 이상의 Task 실패 시
C) always 블록 실행 전 항상
D) Playbook 전체 종료 시

**정답: B**

**해설:** `block/rescue/always`는 프로그래밍의 try/catch/finally와 유사하다. `block` 내의 Task 중 하나라도 실패하면 `rescue` 블록이 실행된다. `always` 블록은 block의 성공/실패와 관계없이 항상 실행된다. 이 패턴은 배포 후 롤백 시나리오에서 매우 유용하다.

---

### 문제 15.
Rolling Update 시 `serial: "25%"`의 의미는?

A) 전체 작업의 25%만 실행
B) 전체 호스트의 25%씩 순차적으로 처리
C) 25%의 확률로 실행
D) 25초 간격으로 실행

**정답: B**

**해설:** `serial`은 한 번에 처리할 호스트의 수 또는 비율을 지정한다. `serial: "25%"`는 전체 호스트를 4개의 배치로 나누어 순차적으로 처리한다. 예를 들어 20대의 서버가 있다면 5대씩 4번에 나누어 처리한다. 이는 무중단 배포(Rolling Update)에서 서비스 가용성을 유지하면서 점진적으로 업데이트하는 데 핵심적인 기법이다.

---

### 문제 16.
AWS Dynamic Inventory에서 `keyed_groups`의 역할은?

```yaml
keyed_groups:
  - key: tags.Environment
    prefix: env
    separator: "_"
```

A) EC2 인스턴스에 태그를 추가한다
B) 태그 값을 기반으로 자동으로 호스트 그룹을 생성한다
C) 특정 태그가 있는 인스턴스만 필터링한다
D) 태그를 변수로 매핑한다

**정답: B**

**해설:** `keyed_groups`는 인스턴스의 속성(태그, AZ, 인스턴스 타입 등)을 기반으로 자동으로 인벤토리 그룹을 생성한다. 위 예시에서 `Environment=production` 태그가 있는 인스턴스는 자동으로 `env_production` 그룹에 포함된다. 이를 통해 수동으로 인벤토리를 관리하지 않고도 클라우드 인프라의 변경에 자동으로 대응할 수 있다.

---

### 문제 17.
다음 코드에서 `delegate_to: localhost`의 효과는?

```yaml
- name: LB에서 서버 제거
  uri:
    url: "http://{{ lb_host }}/api/servers/{{ inventory_hostname }}"
    method: DELETE
  delegate_to: localhost
```

A) Task를 localhost에서 실행하되, 변수는 현재 호스트의 것을 사용
B) Task를 현재 대상 호스트에서 실행
C) localhost의 변수를 사용하여 실행
D) 에러 발생 (delegate_to는 유효하지 않음)

**정답: A**

**해설:** `delegate_to`는 Task의 실행 위치만 변경하고, `inventory_hostname` 등의 변수는 여전히 원래 대상 호스트의 것을 사용한다. 이 패턴은 LB 관리, 모니터링 알림 전송, DNS 업데이트 등 "현재 호스트에 대한 작업이지만 다른 서버에서 실행해야 하는" 경우에 사용한다.

---

### 문제 18.
Ansible에서 비동기 작업을 실행할 때, `async: 3600`과 `poll: 0`을 함께 사용하면?

A) 3600초 동안 매 0초마다 상태 확인
B) 최대 3600초 동안 실행하되, 결과를 기다리지 않고 다음 Task로 진행
C) 3600초 후에 자동으로 작업 취소
D) 에러 발생

**정답: B**

**해설:** `async`는 작업의 최대 실행 시간을, `poll`은 상태 확인 간격을 지정한다. `poll: 0`은 "Fire and Forget" 모드로, 작업을 실행만 하고 결과를 기다리지 않는다. 이후 `async_status` 모듈로 작업 완료를 확인할 수 있다. 장시간 실행되는 백업, 마이그레이션 등에서 다른 작업과 병렬로 진행할 때 유용하다.

---

### 문제 19.
다음 Jinja2 템플릿의 출력 결과는? (변수: `app_env: "production"`)

```jinja2
error_log /var/log/nginx/error.log {{ 'debug' if app_env == 'development' else 'warn' }};
```

A) `error_log /var/log/nginx/error.log debug;`
B) `error_log /var/log/nginx/error.log warn;`
C) `error_log /var/log/nginx/error.log production;`
D) 문법 에러

**정답: B**

**해설:** Jinja2의 인라인 조건식(ternary) 형식이다. `app_env`가 `development`가 아니므로 else 절의 `warn`이 출력된다. 이 패턴은 환경별로 다른 설정값을 적용할 때 템플릿 내에서 자주 사용된다.

---

### 문제 20.
`ansible-playbook site.yml --tags "deploy"` 실행 시, 다음 중 실행되는 Task는?

```yaml
tasks:
  - name: 패키지 설치
    apt: name=nginx
    tags: [install]

  - name: 설정 배포
    template: ...
    tags: [configure]

  - name: 코드 배포
    copy: ...
    tags: [deploy]

  - name: 서비스 시작
    service: ...
    tags: [always]
```

A) "코드 배포"만
B) "코드 배포"와 "서비스 시작"
C) 모든 Task
D) 아무것도 실행되지 않음

**정답: B**

**해설:** `--tags "deploy"`는 `deploy` 태그가 있는 Task만 실행한다. 하지만 `always` 태그는 특수 태그로, `--tags` 옵션과 관계없이 항상 실행된다. 따라서 "코드 배포"(deploy 태그)와 "서비스 시작"(always 태그)이 실행된다.

---

## 고급 문제 (21~30번)

### 문제 21.
다음 AWS 인프라 Playbook에서, EC2 보안 그룹이 ALB로부터의 트래픽만 허용하도록 하는 올바른 설정은?

A)
```yaml
rules:
  - proto: tcp
    ports: [8080]
    cidr_ip: 0.0.0.0/0
```

B)
```yaml
rules:
  - proto: tcp
    ports: [8080]
    group_id: "{{ alb_sg.group_id }}"
```

C)
```yaml
rules:
  - proto: tcp
    ports: [8080]
    cidr_ip: "{{ alb_ip }}"
```

D)
```yaml
rules:
  - proto: tcp
    ports: [8080]
    group_name: alb-sg
```

**정답: B**

**해설:** AWS 보안 그룹에서 다른 보안 그룹을 소스로 참조하면, 해당 보안 그룹에 속한 리소스로부터의 트래픽만 허용한다. `group_id`를 사용하면 ALB 보안 그룹에 속한 ALB로부터의 트래픽만 EC2로 전달된다. IP 기반(cidr_ip)보다 보안 그룹 기반 참조가 ALB→EC2 아키텍처에서 더 안전하고 유지보수가 쉽다.

---

### 문제 22.
Azure Dynamic Inventory에서 `compose` 섹션의 목적은?

```yaml
compose:
  ansible_host: private_ipv4_addresses[0] | default(public_ipv4_addresses[0], true)
  ansible_user: "'azureuser'"
```

A) Azure 리소스를 생성한다
B) 인벤토리 호스트 변수를 동적으로 구성한다
C) Azure 태그를 업데이트한다
D) VM의 네트워크 구성을 변경한다

**정답: B**

**해설:** `compose` 섹션은 동적 인벤토리 플러그인이 각 호스트에 대해 변수를 계산하여 할당하는 곳이다. 위 예시에서는 `ansible_host`를 private IP 우선, 없으면 public IP로 설정하고, `ansible_user`를 `azureuser`로 고정한다. 이를 통해 클라우드 인스턴스의 속성을 Ansible 연결 변수에 자동으로 매핑할 수 있다.

---

### 문제 23.
Ansible에서 Custom Module을 작성할 때, `supports_check_mode=True`를 설정하는 이유는?

A) 모듈의 실행 속도를 향상시키기 위해
B) `--check` 옵션으로 실행 시 실제 변경 없이 결과를 예측할 수 있도록
C) 모듈의 에러 처리를 활성화하기 위해
D) 암호화된 변수를 지원하기 위해

**정답: B**

**해설:** `supports_check_mode=True`로 설정하면 Ansible이 `--check` 모드로 실행될 때 해당 모듈이 건너뛰지 않고 실행된다. 모듈 내부에서 `module.check_mode`를 확인하여 실제 변경 대신 예상 결과만 반환하도록 로직을 구현해야 한다. 이를 통해 Dry-run 시에도 정확한 변경 예측이 가능하다.

---

### 문제 24.
다음 중 Ansible 성능 최적화에 **가장 효과적인** 조합은?

A) `forks=5`, `gathering=implicit`, `pipelining=False`
B) `forks=20`, `gathering=smart`, `pipelining=True`, `fact_caching=jsonfile`
C) `forks=1`, `gathering=explicit`, `pipelining=True`
D) `forks=50`, `gathering=implicit`, `pipelining=False`

**정답: B**

**해설:** `forks=20`은 동시 처리 호스트 수를 늘리고, `gathering=smart`는 Facts를 캐시에서 재사용하며, `pipelining=True`는 SSH 연결을 최적화하고, `fact_caching=jsonfile`은 Facts를 파일에 캐시하여 중복 수집을 방지한다. 이 조합이 성능 최적화에 가장 효과적이다. `forks`만 높여도 `pipelining`이 비활성화되면 SSH 오버헤드가 크다.

---

### 문제 25.
다음 Rolling Update 전략에서 `max_fail_percentage: 10`의 동작은?

```yaml
- hosts: webservers  # 20대
  serial: 5
  max_fail_percentage: 10
```

A) 각 배치에서 10% (0.5대 → 1대) 이상 실패하면 전체 중단
B) 전체 20대 중 2대 이상 실패하면 전체 중단
C) 10% 속도로 실행
D) 실패한 호스트를 10%까지만 재시도

**정답: A**

**해설:** `max_fail_percentage`는 각 `serial` 배치 단위로 평가된다. `serial: 5`이므로 5대씩 처리하며, 각 배치에서 10%(0.5대, 올림하여 1대) 이상 실패하면 나머지 배치 실행을 중단한다. 이는 무중단 배포에서 문제가 발생할 때 전체 서비스로 확산되는 것을 방지하는 안전장치다.

---

### 문제 26.
Ansible Collection에서 `galaxy.yml`의 `namespace`와 `name`이 `mycompany.cloud_ops`일 때, 이 Collection의 모듈을 Playbook에서 호출하는 올바른 방법은?

A) `module: cloud_ops.deploy_app`
B) `module: mycompany.cloud_ops.deploy_app`
C) `mycompany.cloud_ops.deploy_app:`
D) B와 C 모두 가능

**정답: D**

**해설:** Ansible Collection의 모듈은 FQCN(Fully Qualified Collection Name) 형식인 `namespace.collection.module`로 호출한다. `mycompany.cloud_ops.deploy_app`이 FQCN이며, YAML에서는 `module:` 키 형태와 직접 키로 사용하는 형태 모두 가능하다. Ansible 2.10+에서는 FQCN 사용이 권장된다.

---

### 문제 27.
GitHub Actions에서 Ansible을 실행할 때, Vault 비밀번호를 안전하게 전달하는 올바른 방법은?

A) Playbook에 비밀번호를 하드코딩
B) GitHub Secrets에 저장하고 환경 변수로 파일에 기록 후 `--vault-password-file` 사용
C) 공개 리포지토리에 `.vault_pass` 파일 커밋
D) Vault를 사용하지 않고 평문으로 변수 저장

**정답: B**

**해설:** CI/CD 파이프라인에서 Vault 비밀번호는 GitHub Secrets, AWS SSM Parameter Store, HashiCorp Vault 등 시크릿 관리 서비스에 저장해야 한다. 실행 시 환경 변수에서 꺼내 임시 파일에 쓰고, `--vault-password-file`로 전달한 뒤, 실행 후 즉시 삭제한다. 비밀번호를 코드에 하드코딩하거나 리포지토리에 커밋하는 것은 보안 위반이다.

---

### 문제 28.
다음 Azure Playbook에서 VM 생성을 병렬로 처리하기 위해 사용한 기법의 이름과 핵심 매개변수 조합은?

```yaml
- name: VM 생성
  azure.azcollection.azure_rm_virtualmachine:
    name: "myapp-vm-{{ item }}"
    ...
  loop: "{{ range(3) | list }}"
  async: 600
  poll: 0
  register: vm_jobs

- name: VM 생성 완료 대기
  async_status:
    jid: "{{ item.ansible_job_id }}"
  loop: "{{ vm_jobs.results }}"
  register: vm_results
  until: vm_results.finished
  retries: 30
  delay: 20
```

A) Delegation 패턴 + delegate_to
B) Fire-and-Forget 비동기 패턴 + async/poll/async_status
C) Strategy free 패턴 + strategy: free
D) Fork 병렬화 + forks 설정

**정답: B**

**해설:** `async`와 `poll: 0`의 조합은 "Fire-and-Forget" 비동기 패턴이다. 각 loop 반복에서 VM 생성을 시작만 하고 결과를 기다리지 않는다(poll: 0). `register`로 각 작업의 Job ID를 저장하고, 이후 `async_status`와 `until`을 사용하여 모든 VM의 생성 완료를 한꺼번에 확인한다. 이를 통해 3개의 VM이 순차가 아닌 동시에 생성되어 전체 소요 시간이 대폭 줄어든다.

---

### 문제 29.
다음 시나리오에서 가장 적절한 Ansible 전략(strategy)은?

> 100대의 웹 서버에 설정 파일을 배포한다. 각 서버는 독립적이며, 한 서버의 작업 완료를 다른 서버가 기다릴 필요가 없다. 최대한 빠르게 완료해야 한다.

A) `strategy: linear` (기본값)
B) `strategy: free`
C) `strategy: debug`
D) `serial: 1`

**정답: B**

**해설:** `strategy: free`는 각 호스트가 독립적으로 Task를 진행하여, 빠른 호스트가 느린 호스트를 기다리지 않는다. `linear`(기본값)은 모든 호스트가 현재 Task를 완료해야 다음 Task로 넘어가므로, 가장 느린 호스트가 병목이 된다. 100대의 독립적인 서버에 동일한 설정을 최대한 빨리 배포하려면 `free` 전략이 최적이다.

---

### 문제 30.
다음 요구사항을 만족하는 Ansible 아키텍처를 설계하시오. (서술형)

> **요구사항:**
> - AWS와 Azure 멀티 클라우드 환경
> - 프로덕션/스테이징 환경 분리
> - CI/CD 파이프라인 통합
> - 비밀 관리
> - 100+ 서버 관리

**모범 답안:**

```
project-root/
├── ansible.cfg                     # 최적화 설정 (forks=20, pipelining=True 등)
├── requirements.yml                # Collections: amazon.aws, azure.azcollection
│
├── inventory/
│   ├── production/
│   │   ├── aws_ec2.yml            # AWS 동적 인벤토리
│   │   ├── azure_rm.yml           # Azure 동적 인벤토리
│   │   └── group_vars/
│   │       ├── all.yml
│   │       ├── all/vault.yml      # Vault 암호화된 프로덕션 비밀
│   │       ├── aws_hosts.yml
│   │       └── azure_hosts.yml
│   └── staging/
│       ├── aws_ec2.yml
│       ├── azure_rm.yml
│       └── group_vars/
│           ├── all.yml
│           └── all/vault.yml
│
├── roles/
│   ├── common/                    # OS 기본 설정, 보안 하드닝
│   ├── webserver/                 # Nginx + 앱 배포
│   ├── database/                  # PostgreSQL/MySQL
│   ├── monitoring/                # Prometheus + Grafana 에이전트
│   └── cloud_init/               # 클라우드별 초기화
│
├── playbooks/
│   ├── site.yml                   # 마스터 Playbook
│   ├── infra_provision.yml        # 인프라 프로비저닝
│   ├── deploy.yml                 # 앱 배포 (Rolling Update)
│   └── rollback.yml               # 롤백
│
├── vars/
│   └── versions.yml               # 앱/의존성 버전 관리
│
├── filter_plugins/                # 커스텀 필터
├── library/                       # 커스텀 모듈
│
└── .github/workflows/
    ├── deploy-staging.yml         # PR → staging 자동 배포
    └── deploy-production.yml      # main 머지 → production 배포
```

**핵심 설계 포인트:**

1. **동적 인벤토리**: AWS와 Azure 모두 동적 인벤토리 플러그인을 사용하여 인스턴스 변경에 자동 대응한다. 태그 기반 `keyed_groups`로 환경별, 역할별 그룹을 자동 생성한다.

2. **환경 분리**: `inventory/production/`과 `inventory/staging/` 디렉토리로 환경별 변수와 인벤토리를 완전히 분리한다. 실행 시 `-i inventory/production/`으로 환경을 지정한다.

3. **비밀 관리**: Ansible Vault + Multi-Vault ID로 환경별 비밀을 분리 암호화한다. CI/CD에서는 GitHub Secrets에서 Vault 비밀번호를 가져온다.

4. **성능 최적화**: `forks=20`, `pipelining=True`, `gathering=smart`, `fact_caching=jsonfile`을 설정하고, 앱 배포 시 `strategy: free`, 인프라 작업 시 `serial` + `max_fail_percentage`를 사용한다.

5. **CI/CD**: GitHub Actions에서 staging은 PR 시 자동 배포, production은 main 브랜치 머지 후 승인 기반 배포를 수행한다. `--check --diff`로 사전 검증 후 실제 배포를 진행한다.

---

# 부록: 자주 사용하는 Ansible 모듈 레퍼런스

| 카테고리 | 모듈 | 용도 |
|---|---|---|
| 시스템 | `apt`, `yum`, `dnf` | 패키지 관리 |
| 시스템 | `service`, `systemd` | 서비스 관리 |
| 시스템 | `user`, `group` | 사용자/그룹 관리 |
| 파일 | `copy`, `template` | 파일 배포 |
| 파일 | `file` | 파일/디렉토리 생성, 권한 |
| 파일 | `lineinfile`, `blockinfile` | 파일 내용 편집 |
| 파일 | `unarchive` | 압축 해제 |
| 네트워크 | `uri` | HTTP 요청 |
| 네트워크 | `get_url` | 파일 다운로드 |
| 실행 | `command`, `shell` | 명령 실행 |
| 실행 | `script` | 스크립트 실행 |
| AWS | `amazon.aws.ec2_instance` | EC2 인스턴스 |
| AWS | `amazon.aws.ec2_vpc_net` | VPC |
| AWS | `amazon.aws.s3_bucket` | S3 버킷 |
| AWS | `amazon.aws.ec2_security_group` | 보안 그룹 |
| AWS | `community.aws.elb_application_lb` | ALB |
| AWS | `amazon.aws.rds_instance` | RDS |
| Azure | `azure.azcollection.azure_rm_virtualmachine` | VM |
| Azure | `azure.azcollection.azure_rm_virtualnetwork` | VNet |
| Azure | `azure.azcollection.azure_rm_aks` | AKS 클러스터 |
| Azure | `azure.azcollection.azure_rm_loadbalancer` | Load Balancer |
| K8s | `kubernetes.core.k8s` | K8s 리소스 관리 |
| Docker | `community.docker.docker_container` | 컨테이너 관리 |

---

> **학습 로드맵 요약**
>
> **Week 1-2 (초급)**: 설치 → 인벤토리 → Ad-Hoc → 기본 Playbook → 변수 → 조건문/반복문
>
> **Week 3-4 (중급)**: 템플릿 → Roles → Vault → 에러 처리 → 태그/include/import
>
> **Week 5-6 (고급)**: Dynamic Inventory → AWS/Azure 모듈 → Collections → Custom Module → CI/CD 통합 → 성능 최적화
>
> **Week 7+ (실무)**: AWX/AAP 운영 → 멀티 클라우드 아키텍처 → 대규모 환경 운영

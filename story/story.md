# Роль neovim\_ppa: Ansible + Molecula

#ansible #molecula

## Введение

Цель: иметь ansible роль, которая бы устанавливала репозиторий PPA для стабильной версии пакета Neovim. Версия Neovim в стандартном репозитории слишком старая, использовать snap не хочу, оставляя причину за скобками. 

Настоящий документ не является обучающим пособием и предназначен для фиксации основных моментов, связанных с разработкой  конкретной ansible role. 

Разработка роли рассматривалась как часть процесса ознакомления с ansible и molecula. Степень полезности и полноты рассматриваемой роли не обсуждается, конструктивные замечания приветствуются.

Многие приемы были почерпнуты из интернета, по возможности я укажу источник "вдохновения". 

В настоящем документе не будет рассказано, что такое docker, ansible, molecula и как это установить. Если вы не состоянии разобраться в документации или, на худой конец, посмотреть youtube, этот текст не для вас. 

Исходный сетап.

```sh
➜  ~> lsb_release -a 
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.1 LTS
Release:        22.04
Codename:       jammy
➜  ~> docker version --format='{{.Server.Version}}' 
20.10.21
➜  ~> ansible --version | head -1
ansible [core 2.13.7]
➜  ~> molecule --version 
molecule 4.0.4 using python 3.10 
    ansible:2.13.7
    delegated:4.0.4 from molecule
    docker:2.1.0 from molecule_docker requiring collections: community.docker>=3.0.2 ansible.posix>=1.4.0
    vagrant:1.0.0 from molecule_vagrant
```

## Роль neovim\_ppa. Часть 1

Театр начинается с вешалки, а роль начинается с генерации файловой структуры. Поскольку предполагается для тестирования роли использоваться molecula, с ее помощью и генерируем файловую структуру. 

```sh
molecule init role coeng.neovim_ppa --drive-name=docker
```

Мы указываем сразу, что будем использовать docker.

В итоге получает заготовку:

```sh
➜  neovim_ppa> tree 
.
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── molecule
│   └── default
│       ├── converge.yml
│       ├── molecule.yml
│       └── verify.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml

10 directories, 11 files
```

Для чего в файловой структуре роли нужны те или иные файлы - см. документацию. Я также не буду останавливаться на заполнении файла meta/mail.yml

В результате выполнения роли в Ubuntu должен быть установлен репозиторий neovim ppa (см. https://launchpad.net/~neovim-ppa/+archive/ubuntu/stable ). Делов на одну задачу: добавить репозиторий  *ppa:neovim-ppa/stable*.  Пишем соответствующий фрагмент в tasks/main.yml

```yml
- name: Add neovim stable repository from PPA and install its signing key on Ubuntu target
  ansible.builtin.apt_repository:
    repo: ppa:neovim-ppa/stable
    state: present
    update_cache: true
```

Роль почти готова. Для того, чтобы сразу отсечь несовместимые хосты, добавим проверку отдельной задачей, которую вынесем в отдельный файл tasks/assets.yml

```yml
---
- name: test if operating eviroment suitable for role
  ansible.builtin.assert:
    that:
      - ansible_distribution == "Ubuntu" and ansible_distribution_major_version|int > 17
    quiet: yes

```

Разрешаем выполнять роль только для Ubuntu от версии bionic, поскольку для более старых версий репозитория PPA не предусмотрено.

Задачу проверки импортируем в самом начале tasks/main.yml

```yml
- name: import assert.yml
  ansible.builtin.import_tasks: assert.yml
  run_once: true
  delegate_to: localhost
```

готова, осталось проверить. Для этого воспользуемся molecule.  

## Molecule. Часть 2

Проверять будем в контейнерах docker, в качестве базового образа берем ubuntu:jammy. Смотрим в основной файл сценария по умолчанию molecule/default/molecule.yml

```yml
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: instance
    image: quay.io/centos/centos:stream8
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible

```

Меняем образ,переименовываем instance  и добавляем линтеры. Линтеры это хорошо. Линтеры это полезно. Как установить приведенные линтеры - разберетесь.

```yml
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: docker-ubuntu-jammy
    image: ubuntu:jammy
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
lint: |
  set -e
  yamllint .
  ansible-lint .
```

И немедленно создаем контейнер.

```sh
molecule create
```

И проверяем, что контейнер у нас запущен.

```sh
➜  neovim_ppa> molecule list
INFO     Running default > list
                      ╷             ╷                  ╷               ╷         ╷            
  Instance Name       │ Driver Name │ Provisioner Name │ Scenario Name │ Created │ Converged  
╶─────────────────────┼─────────────┼──────────────────┼───────────────┼─────────┼───────────╴
  docker-ubuntu-jammy │ docker      │ ansible          │ default       │ true    │ false      
                      ╵             ╵                  ╵               ╵         ╵            

➜  neovim_ppa> docker ps  | grep docker-ubuntu-jammy 
706f11543b15   ubuntu:jammy               "bash -c 'while true…"   About a minute ago   Up About a minute                                                   docker-ubuntu-jammy

```

И даже в контейнер можно зайти

```sh
➜  neovim_ppa> molecule login 
INFO     Running default > login
root@docker-ubuntu-jammy:/# 
```

Накидываем нашу роль:

```sh
molecule converge
```

И немедленно узнаем, что образ то голый. Т.е. там ничего нужного для ansible нет. А именно, самого python

```sh
TASK [Gathering Facts] *********************************************************
fatal: [docker-ubuntu-jammy]: FAILED! => {"ansible_facts": {}, "changed": false, "failed_modules": {"ansible.legacy.setup": {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "failed": true, "module_stderr": "/bin/sh: 1: /usr/bin/python: not found\n", "module_stdout": "", "msg": "The module failed to execute correctly, you probably need to set the interpreter.\nSee stdout/stderr for the exact error", "rc": 127}}, "msg": "The following modules failed to execute: ansible.legacy.setup\n"}
```

Далее у нас варианты:
1. Найти в пабликах подходящий образ, который содержит необходимые пакеты
2. Приготовить заранее свой образ
3. Приготовить Dockerfile, что бы образ делался molecule на ходу.
Поcкольку предполагается тестирование этой роли, да и последующих ролей на различных образах, то пойдем путем №3. У пути №3 есть два способа
1. Наделать минимальные Dockerfile под разные системы и выбирать их через шаблонизатор из репозитория. Например, как у [Eric Anderson ](https://github.com/ericsysmin) (См. его репозиторй https://github.com/ericsysmin/ansible-molecule-dockerfiles)
2. Сделать шаблон собственно Dockerfile и настраивать его исходя из параметров контейнера.

Недостаток многих Dockerfile в репозитории в том, что приходит вносить правки по мере стабилизации одновременно в несколько Dockerfile, в то время как один большой шаблон становится все сложнее  и сложнее, чем больше семейств и версий мы пытаемся учесть.  Я пошел по пути множественности Dockerfile под каждое семейство и версию, взяв за основу вышеприведенный репозиторий  [Eric Anderson ](https://github.com/ericsysmin).  Поскольку в нем отсутствовал Dockerfile для ubuntu:jammy, я его склонировал к себе и добавил необходимые файлы (см. https://github.com/docsvt/ansible-molecule-dockerfiles)
Переписываем нашем в сценарий molecule/default/molecule.yml секцию platforms (Не забываем перед этим выполнить molecule destroy):

```yml
platforms:
  - name: nvim_ppa_ubuntu-jammy
    dockerfile: ../resources/Dockerfile.j2
    image: ubuntu-jammy
    privileged: true
    command: sleep infinity
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro

```

И добавляем в файл molecule/resourses/Dockerfile.j2

```
{{ lookup('url', 'https://raw.githubusercontent.com/docsvt/ansible-molecule-dockerfiles/main/' ~ item.image ~ '/Dockerfile', split_lines=False) }}
```

Шаблон при исполнении заменяется содержимым файла из репозитория на основании значения атрибута image в записи platforms

TODO: зачем в описании платформы указывается подключение томов - /sys/fs/cgroup:/sys/fs/cgroup:ro

Содержимое собственно Dockerfile на примере ubuntu-jammy:

```dockerfile
FROM ubuntu:22.04
ENV LANG C.UTF-8
ENV LC_ALL C.UTF-8

RUN apt-get update \
  && apt-get install -y --no-install-recommends \
   apt-utils \
   locales \
   python3-setuptools \
   python3-pip \
   python3-cryptography \
   software-properties-common \
   rsyslog systemd systemd-cron sudo iproute2 \
  && rm -Rf /var/lib/apt/lists/* \
  && rm -Rf /usr/share/doc && rm -Rf /usr/share/man \
  && apt-get clean
RUN sed -i 's/^\($ModLoad imklog\)/#\1/' /etc/rsyslog.conf

# Fix potential UTF-8 errors with ansible-test.
RUN locale-gen en_US.UTF-8

# Remove unnecessary getty and udev targets that result in high CPU usage when using
# multiple containers with Molecule (https://github.com/ansible/molecule/issues/1104)
RUN rm -f /lib/systemd/system/systemd*udev* \
  && rm -f /lib/systemd/system/getty.target

VOLUME ["/sys/fs/cgroup", "/tmp", "/run"]
CMD ["/lib/systemd/systemd"]
```

И снова проводим цикл create/converge для molecula. И снова фиаско

```sh
TASK [coeng.neovim_ppa : Add neovim stable repository from PPA and install its signing key on Ubuntu target] ***
fatal: [nvim_ppa_ubuntu-jammy]: FAILED! => {"changed": false, "cmd": "apt-key adv --recv-keys --no-tty --keyserver hkp://keyserver.ubuntu.com:80 9DBB0BE9366964F134855E2255F96FCF8231B6DD", "msg": "Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).\ngpg: error running '/usr/bin/dirmngr': probably not installed\ngpg: failed to start the dirmngr '/usr/bin/dirmngr': Configuration error\ngpg: connecting dirmngr at '/tmp/apt-key-gpghome.rQymQgXosj/S.dirmngr' failed: Configuration error\ngpg: keyserver receive failed: No dirmngr", "rc": 2, "stderr": "Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).\ngpg: error running '/usr/bin/dirmngr': probably not installed\ngpg: failed to start the dirmngr '/usr/bin/dirmngr': Configuration error\ngpg: connecting dirmngr at '/tmp/apt-key-gpghome.rQymQgXosj/S.dirmngr' failed: Configuration error\ngpg: keyserver receive failed: No dirmngr\n", "stderr_lines": ["Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).", "gpg: error running '/usr/bin/dirmngr': probably not installed", "gpg: failed to start the dirmngr '/usr/bin/dirmngr': Configuration error", "gpg: connecting dirmngr at '/tmp/apt-key-gpghome.rQymQgXosj/S.dirmngr' failed: Configuration error", "gpg: keyserver receive failed: No dirmngr"], "stdout": "Executing: /tmp/apt-key-gpghome.rQymQgXosj/gpg.1.sh --recv-keys --no-tty --keyserver hkp://keyserver.ubuntu.com:80 9DBB0BE9366964F134855E2255F96FCF8231B6DD\n", "stdout_lines": ["Executing: /tmp/apt-key-gpghome.rQymQgXosj/gpg.1.sh --recv-keys --no-tty --keyserver hkp://keyserver.ubuntu.com:80 9DBB0BE9366964F134855E2255F96FCF8231B6DD"]}
```

Проблемы:
1. В образе отсутствует dirmgr, необходимый для установки ключа репозитория. 
2. В ubuntu/jammy метод управления ключами репозиториев через apt-key считается deprecated. Предлагается размещать ключи в каталоге /etc/apt/trusted.gpg.d 

## Роль neovim\_ppa. Часть 3

Для установки пакетов в контейнер используется дополнительная задача. Можно было необходимые пакеты добавить и в образ, но а) чем проще образ, тем проще его использоваться для других ролей, б) роль может быть применена на хост, на котором может и не оказаться dirmgr

Поскольку мы предполагаем применять на разных версиях Ubuntu, может так случится, что состав недостающих пакетов может меняться от версии к версии. Поэтому, перечень пакетов будет передавать через переменную, а саму переменную загружать из файла, выбираемого на основании значения переменных *ansible_distribution* и *ansible_distribution_major_version*. Выбор файла с переменными производится от частного общего директивой *with_first_found*

```yml
- name: Gather os specific variables
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution }}-{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_distribution }}.yml"
  tags: vars

- name: Add prerequisites packages
  ansible.builtin.apt: 
    name: "{{ packages }}"
    update_cache: true
    state: present

```

Файл с переменными var/Ubuntu.yml

```yml
---
packages:
  - dirmngr
```

После очередного выполнения *molecule converge* можно зайти в контейнер и убедиться, что репозиторий установлен:

```sh
➜  neovim_ppa> docker exec -ti nvim_ppa_ubuntu-jammy ls -ls /etc/apt/sources.list.d                                
total 4
4 -rw-r--r-- 1 root root 65 Dec  8 18:21 ppa_neovim_ppa_stable_jammy.list
➜  neovim_ppa> docker exec -ti nvim_ppa_ubuntu-jammy cat  /etc/apt/sources.list.d/ppa_neovim_ppa_stable_jammy.list 
deb http://ppa.launchpad.net/neovim-ppa/stable/ubuntu jammy main

```

## Molecule. Часть 4

Приступим к верификации сделанного. Верификация выполняется playbook molecule/default/molecule/verify.yml

```yml
---
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
  - name: Search neovim PPA repository in OE
    ansible.builtin.shell: |
      find /etc/apt/ -name *.list | \
      xargs cat | \
      grep  -q '^[[:space:]]*deb[[:blank:]+].\+neovim-ppa.\+'
    ignore_errors: true
    register: grep_neovim_ppa
  - name: Check if neovim ppa exist in OE
    ansible.builtin.assert:
      that:
        grep_neovim_ppa.rc == 0

```

Ищем в перечне действующих репозиториев, репозиторий с паттерном neovim-ppa. Если находим - успех. 

Запускаем *molecule verify* и смотрим за результатом

```log
TASK [Check if neovim ppa exist in OE] *****************************************
ok: [nvim_ppa_ubuntu-jammy] => {
    "changed": false,
    "msg": "All assertions passed"
}
```

И теперь allzuzammen: *molecule test*, который по умолчанию последовательно запустит 

* dependency
* lint
* cleanup
* destroy
* syntax
* create
* prepare
* converge
* idempotence
* side_effect
* verify
* destroy

## Molecule. Часть 5

Расширим область тестирования на Ubuntu Focal  и Bionic.  Для каждой версии сделаем отдельный сценарий. Поскольку сценарии отличаются только образом, то вынесем общие playbook converge.yml  и verify.yml  в каталог molecula/resources/playbooks и модифицируем файл сценария следующим образом:

```yml
provisioner:
  name: ansible
  log: true
  playbooks:
    converge: ../resources/playbooks/converge.yml
    verify: ../resources/playbooks/verify.yml
```

Сделаем копии каталога molecule/default в

* docker-ubuntu-focal
* docker-ubuntu-bionic

Во вновь созданных файлах сценариях поменяем соответственно имена платформ и образы. После всех манипуляций *molecule list* выдает

```txt
INFO     Running default > list
INFO     Running docker-ubuntu-bionic > list
INFO     Running docker-ubuntu-focal > list
                 ╷             ╷                ╷               ╷         ╷            
                 │             │ Provisioner    │               │         │            
  Instance Name  │ Driver Name │ Name           │ Scenario Name │ Created │ Converged  
╶────────────────┼─────────────┼────────────────┼───────────────┼─────────┼───────────╴
  nvim_ppa_ubun… │ docker      │ ansible        │ default       │ false   │ false      
  nvim_ppa_ubun… │ docker      │ ansible        │ docker-ubunt… │ false   │ false      
  nvim_ppa_ubun… │ docker      │ ansible        │ docker-ubunt… │ false   │ false      
                 ╵             ╵                ╵               ╵         ╵          
```

Далее запускаем цикл проверок с указанием имени сценария, например

```sh
molecule create -s docker-ubuntu-focal
molecule converge -s docker-ubuntu-focal
molecule verify -s docker-ubuntu-focal
```

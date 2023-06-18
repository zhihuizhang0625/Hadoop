---


---

<h1 id="ha模式hadoop部署">HA模式Hadoop部署</h1>
<h1 id="gce-vm配置">GCE VM配置</h1>
<ul>
<li>需要在google cloud engine上开5台ubuntu VM, 一台作为中控机配置ansible, 其他四台受控机在运行过程中也需要全部开启。</li>
</ul>
<h1 id="中控机配置">中控机配置</h1>
<h2 id="安装依赖">安装依赖</h2>
<ul>
<li>安装 pip（版本为 Python 2.7）</li>
</ul>
<pre><code>curl https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip.py 
python get-pip.py --user 
pip -V
</code></pre>
<ul>
<li>安装依赖库</li>
</ul>
<pre><code>sudo apt install -y gcc glibc-devel zlib-devel rpm-build openssl-devel
sudo apt install -y python-devel python-yaml python-jinja2 python2-jmespath
</code></pre>
<ul>
<li>编译安装</li>
</ul>
<p>而 Python2 仅支持 2.9 系列，因此无法通过 yum 进行安装</p>
<pre><code>wget https://releases.ansible.com/ansible/ansible-2.9.27.tar.gz tar -xf ansible-2.9.27.tar.gz 
pushd ansible-2.9.27/ 
python setup.py build 
sudo python setup.py install 
popd 
ansible --version
</code></pre>
<h2 id="ssh-key设置">SSH Key设置</h2>
<ul>
<li>在主控机生成密钥</li>
</ul>
<pre><code>ssh-keygen -t rsa -b 3072 cat ~/.ssh/id_rsa.pub
ssh-copy-id -i ~/.ssh/id_rsa.pub clarazhang0625@10.128.0.17
</code></pre>
<p>copy id到所有受控机</p>
<ul>
<li>受控机访问授权</li>
</ul>
<pre><code>cat &lt;&lt;EOF &gt;&gt; ~/.ssh/authorized_keys 
&gt;ssh-rsa XXX 
&gt;EOF
</code></pre>
<p>这里的ssh-rsa后面要加username @ip地址<br>
e.g. ssh-rsa <a href="mailto:clarazhang0625@10.128.0.17">clarazhang0625@10.128.0.17</a></p>
<ul>
<li>禁用受控机 SSH 登陆询问</li>
</ul>
<pre><code>vim /etc/ssh/ssh_config 
# 在 Host * 后加上 
Host * 
	StrictHostKeyChecking no
</code></pre>
<h2 id="设置文件">设置文件</h2>
<p>在～上设置ansible.cfg  book  conf  hosts  repo</p>
<h4 id="hosts">hosts:</h4>
<pre><code>[newborn]
10.128.0.17
10.128.0.18
10.128.0.19
10.128.0.20

[nodes]
10.128.0.17 hostname='my.hadoop1 my.zk1'
10.128.0.18 hostname='my.hadoop2 my.zk2'
10.128.0.19 hostname='my.hadoop3 my.zk3'
10.128.0.20 hostname='my.hadoop4'

[zk_nodes]
my.zk1 ansible_host=10.128.0.17 myid=1
my.zk2 ansible_host=10.128.0.18 myid=2
my.zk3 ansible_host=10.128.0.19 myid=3

[hadoop_nodes]
my.hadoop[1:4]

[namenodes]
my.hadoop1 id=nn1 rpc_port=8020 http_port=9870
my.hadoop2 id=nn2 rpc_port=8020 http_port=9870

[datanodes]
my.hadoop[1:4]

[journalnodes]
my.hadoop1 journal_port=8485
my.hadoop2 journal_port=8485
my.hadoop3 journal_port=8485

[resourcemanagers]
my.hadoop3 id=rm1 peer_port=8032 tracker_port=8031 scheduler_port=8030 web_port=8088
my.hadoop4 id=rm2 peer_port=8032 tracker_port=8031 scheduler_port=8030 web_port=8088

[all:vars]
ansible_user=clarazhang0625
deploy_dir=/opt
data_dir=/data
</code></pre>
<p>ansible_user需要设置为自己的username<br>
nodes ip地址需要设置为受控机地址</p>
<h4 id="ansible.cfg">ansible.cfg:</h4>
<pre><code>[defaults]
inventory      = ./hosts
host_key_checking = False
</code></pre>
<h4 id="book--conf-repo文件夹">book,  conf, repo文件夹</h4>
<pre><code> mkdir -p book 
 mkdir -p conf  
 mkdir -p repo 
</code></pre>
<p>进入~/book，配置config-hadoop.yml  config-zk.yml  install-hadoop.yml  sync-host.yml  vars.yml</p>
<h4 id="config-hadoop.yml">config-hadoop.yml</h4>
<pre><code>---
- name: Change Hadoop Config
  hosts: hadoop_nodes
  gather_facts: no
  any_errors_fatal: true

  vars:
    template_dir: ../conf/hadoop
    hadoop_home: '{{ deploy_dir }}/hadoop'
    hadoop_conf_dir: '{{ hadoop_home }}/etc/hadoop'
    hadoop_data_dir: '{{ data_dir }}/hadoop'

  tasks:

    - name: Include common vars
      include_vars: file=vars.yml

    - name: Create data directory
      become: true
      file:
        state: directory
        path: '{{ hadoop_data_dir }}'
        owner: '{{ ansible_user }}'
        group: '{{ ansible_user }}'
        mode: 0775
        recurse: yes

    - name: Sync hadoop config
      template:
        src: '{{ template_dir }}/{{ item }}'
        dest: '{{ hadoop_conf_dir }}/{{ item }}'
      with_items: 
        - core-site.xml
        - hdfs-site.xml
        - mapred-site.xml
        - yarn-site.xml
        - workers

    - name: Config hadoop env
      blockinfile:
        dest: '{{ hadoop_conf_dir }}/hadoop-env.sh'
        marker: "# {mark} ANSIBLE MANAGED HADOOP ENV"
        block: |
          export HADOOP_PID_DIR={{ hadoop_home }}/pid
          export HADOOP_LOG_DIR={{ hadoop_data_dir }}/logs

          JVM_OPTS="-XX:+AlwaysPreTouch"
          export HDFS_JOURNALNODE_OPTS="-Xmx1G $JVM_OPTS $HDFS_JOURNALNODE_OPTS"
          export HDFS_NAMENODE_OPTS="-Xmx4G $JVM_OPTS $HDFS_NAMENODE_OPTS"
          export HDFS_DATANODE_OPTS="-Xmx8G $JVM_OPTS $HDFS_DATANODE_OPTS"

    - name: Config yarn env
      blockinfile:
        dest: '{{ hadoop_conf_dir }}/yarn-env.sh'
        marker: "# {mark} ANSIBLE MANAGED YARN ENV"
        block: |
          JVM_OPTS=""
          export YARN_RESOURCEMANAGER_OPTS="$JVM_OPTS $YARN_RESOURCEMANAGER_OPTS"
          export YARN_NODEMANAGER_OPTS="$JVM_OPTS $YARN_NODEMANAGER_OPTS"
</code></pre>
<h4 id="config-zk.yml">config-zk.yml</h4>
<pre><code>---
- name: Change Zk Config
  hosts: zk_nodes
  gather_facts: no
  any_errors_fatal: true

  vars:
    template_dir: ../conf/zk
    zk_home: '{{ deploy_dir }}/zookeeper'
    zk_data_dir: '{{ zk_home }}/status/data'
    zk_data_log_dir: '{{ zk_home }}/status/logs'

  tasks:

    - name: Create data directory
      file:
        state: directory
        path: '{{ item }}'
        recurse: yes
      with_items: 
        - '{{ zk_data_dir }}'
        - '{{ zk_data_log_dir }}'

    - name: Init zookeeper myid
      template:
        src: '{{ template_dir }}/myid'
        dest: '{{ zk_data_dir }}'

    - name: Update zookeeper env
      become: true
      blockinfile:
        dest: '{{ zk_home }}/bin/zkEnv.sh'
        marker: "# {mark} ANSIBLE MANAGED ZK ENV"
        block: |
          export SERVER_JVMFLAGS="-Xmx1G -XX:+UseShenandoahGC -XX:+AlwaysPreTouch -Djute.maxbuffer=8388608"
      notify:
        - Restart zookeeper service

    - name: Update zookeeper config
      template:
        src: '{{ template_dir }}/zoo.cfg'
        dest: '{{ zk_home }}/conf'
      notify:
        - Restart zookeeper service

    - name: Restart zookeeper service
      shell:
        cmd: '{{ zk_home }}/bin/zkServer.sh restart'

</code></pre>
<h4 id="install-hadoop.yml">install-hadoop.yml</h4>
<pre><code>---
- name: Install Hadoop Package
  hosts: newborn
  gather_facts: no
  any_errors_fatal: true

  vars:
    local_repo: '../repo/hadoop'
    remote_repo: '~/repo/hadoop'
    package_info:
      - {src: 'OpenJDK17U-jdk_x64_linux_hotspot_17.0.2_8.tar.gz', dst: 'java/jdk-17.0.2+8', home: 'jdk17'}
      - {src: 'OpenJDK8U-jdk_x64_linux_hotspot_8u372b07.tar.gz', dst: 'java/jdk8u372-b07', home: 'jdk8'}
      - {src: 'apache-zookeeper-3.6.4-bin.tar.gz', dst: 'apache/zookeeper-3.6.4', home: 'zookeeper'}
      - {src: 'hadoop-3.2.3.tar.gz', dst: 'apache/hadoop-3.2.3', home: 'hadoop'}

  tasks:

    - name: test connectivity
      ping:

    - name: prepare directory
      become: true # become root
      file:
        state: directory
        path: '{{ deploy_dir }}/{{ item.dst }}'
        owner: '{{ ansible_user }}'
        group: '{{ ansible_user }}'
        mode: 0775
        recurse: yes
      with_items: '{{ package_info }}'

    - name: create link
      become: true # become root
      file:
        state: link
        src: '{{ deploy_dir }}/{{ item.dst }}'
        dest: '{{ deploy_dir }}/{{ item.home }}'
        owner: '{{ ansible_user }}'
        group: '{{ ansible_user }}'
      with_items: '{{ package_info }}'

    - name: install package
      unarchive:
        src: '{{ remote_repo }}/{{ item.src }}'
        dest: '{{ deploy_dir }}/{{ item.dst }}'
        remote_src: yes
        extra_opts:
          - --strip-components=1
      with_items: '{{ package_info }}'

    - name: config /etc/profile
      become: true
      blockinfile:  
        dest: '/etc/profile'
        marker: "# {mark} ANSIBLE MANAGED PROFILE"
        block: |
          export JAVA_HOME={{ deploy_dir }}/jdk8
          export HADOOP_HOME={{ deploy_dir }}/hadoop
          export HBASE_HOME={{ deploy_dir }}/hbase
          export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$HBASE_HOME/bin:$PATH

    - name: config zkEnv.sh
      lineinfile:
        path: '{{ deploy_dir }}/zookeeper/bin/zkEnv.sh'
        line: 'JAVA_HOME={{ deploy_dir }}/jdk17'
        insertafter: '^#\!\/usr\/bin'
        firstmatch: yes

    - name: config hadoop-env.sh
      blockinfile:
        dest: '{{ deploy_dir }}/hadoop/etc/hadoop/hadoop-env.sh'
        marker: "# {mark} ANSIBLE MANAGED DEFAULT HADOOP ENV"
        block: |
          export JAVA_HOME={{ deploy_dir }}/jdk8

    - name: add epel-release repo
      shell: 'sudo apt-get install -y software-properties-common &amp;&amp; sudo add-apt-repository -y ppa:git-core/ppa'

    - name: install native libary
      shell: 'sudo apt-get install -y libsnappy-dev liblz4-dev libzstd-dev'

    - name: check hadoop native
      shell: '{{ deploy_dir }}/hadoop/bin/hadoop checknative -a'
      register: hadoop_checknative
      failed_when: false
      changed_when: false
      ignore_errors: yes
      environment:
        JAVA_HOME: '{{ deploy_dir }}/jdk8'

    - name: hadoop native status
      debug:
        msg: "{{ hadoop_checknative.stdout_lines }}"

    - name: native compresssion status
      shell: '{{ deploy_dir }}/hadoop/bin/hadoop checknative -a'
      register: hadoop_checknative
      failed_when: false
      changed_when: false
      ignore_errors: yes
      environment:
        JAVA_HOME: '{{ deploy_dir }}/jdk8'
</code></pre>
<h4 id="sync-host.yml">sync-host.yml</h4>
<pre><code>---
- name: Config Hostname &amp; SSH Keys
  hosts: nodes  
  connection: local
  gather_facts: no
  any_errors_fatal: true

  vars:
    hostnames: |
      {% for h in groups['nodes'] if hostvars[h].hostname is defined %}{{h}} {{ hostvars[h].hostname }}
      {% endfor %}

  tasks:

    - name: test connectivity
      ping:
      connection: ssh

    - name: change local hostname 
      become: true
      blockinfile:  
        dest: '/etc/hosts'
        marker: "# {mark} ANSIBLE MANAGED HOSTNAME"
        block: '{{ hostnames }}'
      run_once: true

    - name: sync remote hostname 
      become: true
      blockinfile:  
        dest: '/etc/hosts'
        marker: "# {mark} ANSIBLE MANAGED HOSTNAME"
        block: '{{ hostnames }}'
      connection: ssh

    - name: fetch exist status
      stat:
        path: '~/.ssh/id_rsa'
      register: ssh_key_path
      connection: ssh

    - name: generate ssh key
      openssh_keypair:
        path: '~/.ssh/id_rsa'
        comment: '{{ ansible_user }}@{{ inventory_hostname }}'
        type: rsa
        size: 2048
        state: present
        force: no
      connection: ssh
      when: not ssh_key_path.stat.exists

    - name: collect ssh key
      command: ssh {{ansible_user}}@{{ansible_host|default(inventory_hostname)}} 'cat ~/.ssh/id_rsa.pub'
      register: host_keys  # cache data in hostvars[hostname].host_keys
      changed_when: false

    - name: create temp file
      tempfile:
        state: file
        suffix: _keys
      register: temp_ssh_keys
      changed_when: false
      run_once: true

    - name: save ssh key ({{temp_ssh_keys.path}})
      blockinfile:  
        dest: "{{temp_ssh_keys.path}}"  
        block: |  
          {% for h in groups['nodes'] if hostvars[h].host_keys is defined %}  
          {{ hostvars[h].host_keys.stdout }}  
          {% endfor %}  
      changed_when: false
      run_once: true

    - name: deploy ssh key
      vars:
        ssh_keys: "{{ lookup('file', temp_ssh_keys.path).split('\n') | select('match', '^ssh') | join('\n') }}"
      authorized_key:
        user: "{{ ansible_user }}"
        key: "{{ ssh_keys }}"
        state: present
      connection: ssh
</code></pre>
<h4 id="vars.yml">vars.yml</h4>
<pre><code>hdfs_name: my-hdfs
yarn_name: my-yarn
</code></pre>
<p>在~/conf配置hadoop  zk文件夹</p>
<pre><code>mkdir -p hadoop 
mkdir -p zk
</code></pre>
<p>在~/conf/hadoop配置 core-site.xml  hdfs-site.xml  mapred-site.xml  workers  yarn-site.xml</p>
<h4 id="core-site.xml">core-site.xml</h4>
<pre><code>&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;?xml-stylesheet type="text/xsl" href="configuration.xsl"?&gt;
&lt;configuration&gt;
  &lt;!-- 指定 NameNode 地址 (使用集群名称替代) --&gt;
  &lt;property&gt;
    &lt;name&gt;fs.defaultFS&lt;/name&gt;
    &lt;value&gt;hdfs://{{ hdfs_name }}&lt;/value&gt;
  &lt;/property&gt;
  &lt;!-- 指定数据存储目录 --&gt;
  &lt;property&gt;
    &lt;name&gt;hadoop.tmp.dir&lt;/name&gt;
    &lt;value&gt;{{ hadoop_data_dir }}&lt;/value&gt;
  &lt;/property&gt;
  &lt;!-- 指定 Web 用户权限（默认用户 dr.who 无法上传文件） --&gt;
  &lt;property&gt;
     &lt;name&gt;hadoop.http.staticuser.user&lt;/name&gt;
     &lt;value&gt;{{ ansible_user }}&lt;/value&gt;
  &lt;/property&gt;
  &lt;!-- 指定 DFSZKFailoverController 所需的 ZK --&gt;
  &lt;property&gt;
    &lt;name&gt;ha.zookeeper.quorum&lt;/name&gt;
    &lt;value&gt;{{ groups['zk_nodes'] | map('regex_replace','^(.+)$','\\1:2181') | join(',') }}&lt;/value&gt;
  &lt;/property&gt;
&lt;/configuration&gt;
</code></pre>
<h4 id="hdfs-site.xml">hdfs-site.xml</h4>
<pre><code>&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;?xml-stylesheet type="text/xsl" href="configuration.xsl"?&gt;
&lt;configuration&gt;
 &lt;!-- NameNode 数据存储目录 --&gt;
 &lt;property&gt;
   &lt;name&gt;dfs.namenode.name.dir&lt;/name&gt;
   &lt;value&gt;file://${hadoop.tmp.dir}/name&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- DataNode 数据存储目录 --&gt;
 &lt;property&gt;
   &lt;name&gt;dfs.datanode.data.dir&lt;/name&gt;
   &lt;value&gt;file://${hadoop.tmp.dir}/data&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- JournalNode 数据存储目录（绝对路径，不能带 file://） --&gt;
 &lt;property&gt;
   &lt;name&gt;dfs.journalnode.edits.dir&lt;/name&gt;
   &lt;value&gt;${hadoop.tmp.dir}/journal&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- HDFS 集群名称 --&gt;
 &lt;property&gt;
   &lt;name&gt;dfs.nameservices&lt;/name&gt;
   &lt;value&gt;{{ hdfs_name }}&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- 集群 NameNode 节点列表 --&gt;
 &lt;property&gt;
   &lt;name&gt;dfs.ha.namenodes.{{hdfs_name}}&lt;/name&gt;
   &lt;value&gt;{{ groups['namenodes'] | map('extract', hostvars) | map(attribute='id') | join(',') }}&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- NameNode RPC 地址 --&gt;
 {% for host in groups['namenodes'] %}
 &lt;property&gt;
   &lt;name&gt;dfs.namenode.rpc-address.{{hdfs_name}}.{{hostvars[host]['id']}}&lt;/name&gt;
   &lt;value&gt;{{host}}:{{hostvars[host]['rpc_port']}}&lt;/value&gt;
 &lt;/property&gt;
 {% endfor %}
 &lt;!-- NameNode HTTP 地址 --&gt;
 {% for host in groups['namenodes'] %}
 &lt;property&gt;
   &lt;name&gt;dfs.namenode.http-address.{{hdfs_name}}.{{hostvars[host]['id']}}&lt;/name&gt;
    &lt;value&gt;{{host}}:{{hostvars[host]['http_port']}}&lt;/value&gt;
 &lt;/property&gt;
 {% endfor %}
 &lt;!-- NameNode 元数据在 JournalNode 上的存放位置 --&gt;
 &lt;property&gt;
   &lt;name&gt;dfs.namenode.shared.edits.dir&lt;/name&gt;
   &lt;value&gt;qjournal://{{groups['journalnodes'] | zip( groups['journalnodes']|map('extract', hostvars)|map(attribute='journal_port') )| map('join', ':') | join(';') }}/{{hdfs_name}}&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- fail-over 代理类 (client 通过 proxy 来确定 Active NameNode) --&gt;
 &lt;property&gt;
   &lt;name&gt;dfs.client.failover.proxy.provider.my-hdfs&lt;/name&gt;
   &lt;value&gt;org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- 隔离机制 (保证只存在唯一的 Active NameNode) --&gt;
 &lt;property&gt;
   &lt;name&gt;dfs.ha.fencing.methods&lt;/name&gt;
   &lt;value&gt;sshfence&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- SSH 隔离机制依赖的登录秘钥 --&gt;
 &lt;property&gt;
   &lt;name&gt;dfs.ha.fencing.ssh.private-key-files&lt;/name&gt;
   &lt;value&gt;/home/{{ ansible_user }}/.ssh/id_rsa&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- 启用自动故障转移 --&gt;
 &lt;property&gt;
    &lt;name&gt;dfs.ha.automatic-failover.enabled&lt;/name&gt;
   &lt;value&gt;true&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- NameNode 工作线程数量 --&gt;
 &lt;property&gt;
   &lt;name&gt;dfs.namenode.handler.count&lt;/name&gt;
   &lt;value&gt;21&lt;/value&gt;
 &lt;/property&gt;
&lt;/configuration&gt;
</code></pre>
<h4 id="mapred-site.xml">mapred-site.xml</h4>
<pre><code>&lt;?xml version="1.0"?&gt;
&lt;?xml-stylesheet type="text/xsl" href="configuration.xsl"?&gt;
&lt;configuration&gt;
 &lt;!-- MapReduce 运行在 YARN 上 --&gt;
 &lt;property&gt;
   &lt;name&gt;mapreduce.framework.name&lt;/name&gt;
   &lt;value&gt;yarn&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- MapReduce Classpath --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.app.mapreduce.am.env&lt;/name&gt;
   &lt;value&gt;HADOOP_MAPRED_HOME=${HADOOP_HOME}&lt;/value&gt;
 &lt;/property&gt;
 &lt;property&gt;
   &lt;name&gt;mapreduce.map.env&lt;/name&gt;
   &lt;value&gt;HADOOP_MAPRED_HOME=${HADOOP_HOME}&lt;/value&gt;
 &lt;/property&gt;
 &lt;property&gt;
   &lt;name&gt;mapreduce.reduce.env&lt;/name&gt;
   &lt;value&gt;HADOOP_MAPRED_HOME=${HADOOP_HOME}&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- MapReduce JVM 参数（不允许换行） --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.app.mapreduce.am.command-opts&lt;/name&gt;
   &lt;value&gt;-Xmx1024m --add-opens java.base/java.lang=ALL-UNNAMED&lt;/value&gt;
 &lt;/property&gt;
 &lt;property&gt;
   &lt;name&gt;mapred.child.java.opts&lt;/name&gt;
   &lt;value&gt;--add-opens java.base/java.lang=ALL-UNNAMED -verbose:gc -Xloggc:/tmp/@taskid@.gc&lt;/value&gt;
 &lt;/property&gt;
&lt;/configuration&gt;
</code></pre>
<h4 id="workers">workers</h4>
<pre><code>{% for host in groups['datanodes'] %}
{{ host }}
{% endfor %}
</code></pre>
<h4 id="yarn-site.xml">yarn-site.xml</h4>
<pre><code>&lt;?xml version="1.0"?&gt;
&lt;configuration&gt;
 &lt;!-- 启用 ResourceManager HA --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.resourcemanager.ha.enabled&lt;/name&gt;
   &lt;value&gt;true&lt;/value&gt;
 &lt;/property&gt;  
 &lt;!-- YARN 集群名称 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.resourcemanager.cluster-id&lt;/name&gt;
   &lt;value&gt;{{yarn_name}}&lt;/value&gt;
 &lt;/property&gt;  
&lt;!-- ResourceManager 节点列表 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.resourcemanager.ha.rm-ids&lt;/name&gt;
   &lt;value&gt;{{ groups['resourcemanagers'] | map('extract', hostvars) | map(attribute='id') | join(',') }}&lt;/value&gt;
 &lt;/property&gt;  
 &lt;!-- ResourceManager 地址 --&gt;
 {% for host in groups['resourcemanagers'] %}
 &lt;property&gt;
   &lt;name&gt;yarn.resourcemanager.hostname.{{hostvars[host]['id']}}&lt;/name&gt;
   &lt;value&gt;{{host}}&lt;/value&gt;
 &lt;/property&gt;
 {% endfor %}
 &lt;!-- ResourceManager 内部通信地址 --&gt;
 {% for host in groups['resourcemanagers'] %}
 &lt;property&gt;
     &lt;name&gt;yarn.resourcemanager.address.{{hostvars[host]['id']}}&lt;/name&gt;
     &lt;value&gt;{{host}}:{{hostvars[host]['peer_port']}}&lt;/value&gt;
 &lt;/property&gt;
 {% endfor %}
 &lt;!-- NM 访问 ResourceManager 地址 --&gt;
 {% for host in groups['resourcemanagers'] %}
 &lt;property&gt;
     &lt;name&gt;yarn.resourcemanager.resource-tracker.{{hostvars[host]['id']}}&lt;/name&gt;
     &lt;value&gt;{{host}}:{{hostvars[host]['tracker_port']}}&lt;/value&gt;
 &lt;/property&gt;
 {% endfor %}
 &lt;!-- AM 向 ResourceManager 申请资源地址 --&gt;
 {% for host in groups['resourcemanagers'] %}
 &lt;property&gt;
     &lt;name&gt;yarn.resourcemanager.scheduler.address.{{hostvars[host]['id']}}&lt;/name&gt;
     &lt;value&gt;{{host}}:{{hostvars[host]['scheduler_port']}}&lt;/value&gt;
 &lt;/property&gt;
 {% endfor %}
 &lt;!-- ResourceManager Web 入口 --&gt;
 {% for host in groups['resourcemanagers'] %}
 &lt;property&gt;
     &lt;name&gt;yarn.resourcemanager.webapp.address.{{hostvars[host]['id']}}&lt;/name&gt;
     &lt;value&gt;{{host}}:{{hostvars[host]['web_port']}}&lt;/value&gt;
 &lt;/property&gt;
 {% endfor %}
 &lt;!-- 启用自动故障转移 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.resourcemanager.recovery.enabled&lt;/name&gt;
   &lt;value&gt;true&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- 指定 Zookeeper 列表 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.resourcemanager.zk-address&lt;/name&gt;
   &lt;value&gt;{{ groups['zk_nodes'] | map('regex_replace','^(.+)$','\\1:2181') | join(',') }}&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- 将状态信息存储在 Zookeeper 集群--&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.resourcemanager.store.class&lt;/name&gt;
   &lt;value&gt;org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- 减少 ResourceManager 处理 Client 请求的线程--&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.resourcemanager.scheduler.client.thread-count&lt;/name&gt;
   &lt;value&gt;10&lt;/value&gt;
 &lt;/property&gt;  
 &lt;!-- 禁止 NodeManager 自适应硬件配置（非独占节点）--&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.nodemanager.resource.detect-hardware-capbilities&lt;/name&gt;
   &lt;value&gt;false&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- NodeManager 给容器分配的 CPU 核数--&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.nodemanager.resource.cpu-vcores&lt;/name&gt;
   &lt;value&gt;4&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- NodeManager 使用物理核计算 CPU 数量（可选）--&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.nodemanager.resource.count-logical-processors-as-cores&lt;/name&gt;
   &lt;value&gt;false&lt;/value&gt;
 &lt;/property&gt;  
 &lt;!-- 减少 NodeManager 使用内存--&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.nodemanager.resource.memory-mb&lt;/name&gt;
   &lt;value&gt;4096&lt;/value&gt;
 &lt;/property&gt;  
 &lt;!-- 容器内存下限 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.scheduler.minimum-allocation-mb&lt;/name&gt;
   &lt;value&gt;1024&lt;/value&gt;
 &lt;/property&gt;  
 &lt;!-- 容器内存上限 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.scheduler.maximum-allocation-mb&lt;/name&gt;
   &lt;value&gt;2048&lt;/value&gt;
 &lt;/property&gt;  
 &lt;!-- 容器CPU下限 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.scheduler.minimum-allocation-vcores&lt;/name&gt;
   &lt;value&gt;1&lt;/value&gt;
 &lt;/property&gt;  
 &lt;!-- 容器CPU上限 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.scheduler.maximum-allocation-vcores&lt;/name&gt;
   &lt;value&gt;2&lt;/value&gt;
 &lt;/property&gt;  
 &lt;!-- 容器CPU上限 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.scheduler.maximum-allocation-vcores&lt;/name&gt;
   &lt;value&gt;2&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- 关闭虚拟内存检查 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.nodemanager.vmem-check-enabled&lt;/name&gt;
   &lt;value&gt;false&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- 设置虚拟内存和物理内存的比例 --&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.nodemanager.vmem-pmem-ratio&lt;/name&gt;
   &lt;value&gt;2.1&lt;/value&gt;
 &lt;/property&gt;
 &lt;!-- NodeManager 在 MR 过程中使用 Shuffle（可选）--&gt;
 &lt;property&gt;
   &lt;name&gt;yarn.nodemanager.aux-services&lt;/name&gt;
   &lt;value&gt;mapreduce_shuffle&lt;/value&gt;
 &lt;/property&gt;  
&lt;/configuration&gt;
</code></pre>
<p>-在 ~/conf/zk配置 myid  zoo.cfg</p>
<h4 id="myid">myid</h4>
<pre><code>{{ myid }}
</code></pre>
<h4 id="zoo.cfg">zoo.cfg</h4>
<pre><code># ZK 与客户端间的心跳间隔，单位 mills
tickTime=2000
# Leader 与 Follower 间建立连接的超时时间，单位为 tick
initLimit=30
# Leader 与 Follower 间通信的超时时间，单位为 tick
syncLimit=5
# 快照目录
dataDir={{ zk_data_dir }}
# WAL目录，最好为其指定一个独立的空闲设备（建议使用 SSD）
dataLogDir={{ zk_data_log_dir }}
# 使用默认通信端口
clientPort=2181
# 增加最大连接数
maxClientCnxns=2000
# 开启 Prometheus 监控
metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
metricsProvider.httpHost={{ ansible_host | default(inventory_hostname) }}
metricsProvider.httpPort=7000
metricsProvider.exportJvmInfo=true
# 配置集群信息
# server.{myid}={server-address}:{rpc-port}:{election-port}
{% for host in groups['zk_nodes'] %}
server.{{ hostvars[host]['myid'] }}={{ hostvars[host]['ansible_host'] }}:2888:3888
{% endfor %}
</code></pre>
<ul>
<li>在~/repo配置hadoop文件夹</li>
</ul>
<pre><code>mkdir -p hadoop
</code></pre>
<ul>
<li>在~/repo/hadoop配置<br>
hadoop-3.2.3.tar.gz<br>
hadoop-3.2.3<br>
OpenJDK17U-jdk_x64_linux_hotspot_17.0.2_8.tar.gz<br>
jdk-17.0.2+8<br>
apache-zookeeper-3.6.4-bin.tar.gz<br>
apache-zookeeper-3.6.4-bin<br>
OpenJDK8U-jdk_x64_linux_hotspot_8u372b07.tar.gz<br>
jdk8u372-b07</li>
</ul>
<p>在～/repo/hadoop底下安装以下的所有包，下载版本可以有略微不同</p>
<pre><code>sudo wget https://dlcdn.apache.org/zookeeper/zookeeper-3.6.4/apache-zookeeper-3.6.4-bin.tar.gz
sudo tar -xzf apache-zookeeper-3.6.4-bin.tar.gz

sudo wget https://downloads.apache.org/hadoop/common/hadoop-3.2.3/hadoop-3.2.3.tar.gz
sudo tar xzf hadoop-3.2.3.tar.gz

sudo wget https://github.com/adoptium/temurin17-binaries/releases/download/jdk-17.0.2%2B8/OpenJDK17U-jdk_x64_linux_hotspot_17.0.2_8.tar.gz
sudo tar xzf OpenJDK17U-jdk_x64_linux_hotspot_17.0.2_8.tar.gz

sudo wget http://www.cs.tohoku-gakuin.ac.jp/pub/Tools/OpenJDK/JDK8-HotSpot/OpenJDK8U-jdk_x64_linux_hotspot_8u372b07.tar.gz
sudo tar xzf OpenJDK8U-jdk_x64_linux_hotspot_8u372b07.tar.gz

</code></pre>
<h2 id="运行ansible">运行Ansible</h2>
<p>在～运行以下commands:</p>
<pre><code>ansible-playbook book/sync-host.yml
ansible-playbook book/install-hadoop.yml
ansible-playbook book/config-zk.yml
ansible-playbook book/config-hadoop.yml
</code></pre>
<p>然后需要建立/opt/hadoop/pid directory并修改权限</p>
<pre><code>sudo chmod 777 /opt/hadoop/pid
</code></pre>
<pre><code>ansible journalnodes -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; nohup hdfs --daemon start journalnode"'
ansible journalnodes -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; jps | grep JournalNode"'

telnet my.hadoop2 8485
netstat -tuln | grep 8485
sudo ufw allow 8485
sudo iptables -A INPUT -p tcp --dport 8485 -j ACCEPT
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; hdfs namenode -format"'
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; nohup hdfs --daemon start namenode"'
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; jps | grep NameNode"'

ansible 'namenodes[1:]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; hdfs namenode -bootstrapStandby"'
ansible 'namenodes[1:]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; nohup hdfs --daemon start namenode"'
ansible 'namenodes[1:]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; jps | grep NameNode"'

ansible datanodes -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; nohup hdfs --daemon start datanode"'
ansible datanodes -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; jps | grep DataNode"'

ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; hdfs haadmin -getServiceState nn1"'
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; hdfs haadmin -getServiceState nn2"'

ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; hdfs zkfc -formatZK"'

ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; export HDFS_NAMENODE_USER=clarazhang0625 &amp;&amp; export HDFS_DATANODE_USER=clarazhang0625 &amp;&amp; export HDFS_JOURNALNODE_USER=clarazhang0625 &amp;&amp; export HDFS_ZKFC_USER=clarazhang0625 &amp;&amp; stop-dfs.sh"'
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; export HDFS_NAMENODE_USER=clarazhang0625 &amp;&amp; export HDFS_DATANODE_USER=clarazhang0625 &amp;&amp; export HDFS_JOURNALNODE_USER=clarazhang0625 &amp;&amp; export HDFS_ZKFC_USER=clarazhang0625 &amp;&amp; start-dfs.sh"'
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; jps | grep FailoverController"'

ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; hdfs haadmin -getServiceState nn1"'
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; hdfs haadmin -getServiceState nn2"'

ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; start-yarn.sh"'
ansible 'hadoop_nodes' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; jps | grep Manager"'

ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; yarn rmadmin -getServiceState rm1"'
ansible 'namenodes[0]' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; yarn rmadmin -getServiceState rm2"'
</code></pre>
<p>最后运行jps查看Hadoop运行状态</p>
<pre><code>ansible 'hadoop_nodes' -m shell -a 'sudo -- bash -c ". /etc/profile &amp;&amp; jps"'
</code></pre>


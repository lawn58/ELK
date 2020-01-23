# ELK

Elasticsearch+kibana+logstash+filebeat

Разворачиваем Vagrant-стенд с помощью команды

  vagrant up

Подключаемся по ssh к машине elk:

  vagrant ssh elk

  ШАГ 1. Отключаем SELINUХ

Отключаем SELINUХ, для этого открываем файл 

  vim /etc/sysconfig/selinux

И меняем на

  SELINUX=disabled

После чего перезагружаем машину, для применения настроек SELINUX

  reboot

Проверяем состояние SELINUX

  getenforce

И убеждаемся что disabled

  ШАГ 2. Устанавливаем Java
  
    
Устанавливаем java
    
    yum install java
    
проверяем версию java
    
    java -version
    
    
  ШАГ 3. Устанавливаем и конфигурирем Elasticsearch
  
  Импортируем ключ
    
    rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
  
  Скачиваем пакет Elasticsearch
    
    wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.1.1.rpm
  
  Устанавливаем пакет Elasticsearch
    
    rpm -ivh elasticsearch-5.1.1.rpm
  
  Изменяем файл:
  
    vim etc/elasticsearch/elasticsearch.yml
    
  Раскоментируем три строчки в файле:
  
    bootstrap.memory_lock: true
    network.host: localhost
    http.port: 9200
    
  Сохраняем и выходим из файла.
  
  Перезагружаем systemd, добавляем Elasticsearch в автозагрузку и стартуем сервис:
  
    systemctl daemon-reload
    systemctl enable elasticsearch
    systemctl start elasticsearch
    
   Проверяем что Elasticsearch работает:
   
    systemctl status elasticsearch
    
   И "слушает" порт:
   
    ss -tulnp | grep 9200
    
   ШАГ 4. Устанавливаем и конфигурируем Kibana и nginx
   
   Скачиваем пакет Kibana и устанавливаем его:
   
    wget https://artifacts.elastic.co/downloads/kibana/kibana-5.1.1-x86_64.rpm
    rpm -ivh kibana-5.1.1-x86_64.rpm
    
   Редактируем файл и раскоментируем следующие строки:
   
    vim /etc/kibana/kibana.yml
    
    server.port: 5601
    server.host: "0.0.0.0"
    elasticsearch.url: "http://localhost:9200" 
   
   Сохраняем и выходим из файла
   
   Добавляем Kibana в автозагрузку и стартуем ее
   
    systemctl enable kibana
    systemctl start kibana
    
   Проверяем что сервис запустился и слушает порты:
   
    systemctl status kibana
    ss -tulnp | grep 5601
    
   Открываем другой терминал и подключаемся к машине Web:
   
    vagrant ssh web
   
   Устанавливаем nginx:
   
    yum -y install epel-release
    yum -y install nginx httpd-tools
    
   Добавляем его в автозагрузку и запускаем:
   
    systemctl enable nginx
    systemctl start nginx
    
   ШАГ 4. Устанавливаем и конфигурируем Logstash
   
   Переходим обратно на машину elk
   Скачиваем пакет Logstash и устанавливаем его:
   
    wget https://artifacts.elastic.co/downloads/logstash/logstash-5.1.1.rpm
    rpm -ivh logstash-5.1.1.rpm
    
   Создаем новый файл и добавляем в него следующее содержимое:
   
    vim /etc/logstash/conf.d/filebeat-input.conf
    
    input {
  beats {
    port => 5443
    ssl => true
    ssl_certificate => "/etc/pki/tls/certs/logstash-forwarder.crt"
    ssl_key => "/etc/pki/tls/private/logstash-forwarder.key"
  }
}

   Сохраняем файл и выходим из него
   
   Создаем файл syslog-filter.conf и добавляем в него следующее содержимое:
   
    vim /etc/logstash/conf.d/syslog-filter.conf
    
    filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}


  Сохраняем файл и выходим
    
  Содаем файл output-elasticsearch.conf и добавляем в него следующее содержимое:
  
  vim /etc/logstash/conf.d/output-elasticsearch.conf
  
  output {
  elasticsearch { hosts => ["localhost:9200"]
    hosts => "localhost:9200"
    manage_template => false
    index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
    document_type => "%{[@metadata][type]}"
  }
}

  Сохраняем файл и выходим
  Добавляем logstash в автозагрузку и стартуем его:
  
    systemctl enable logstash
    systemctl start logstash
    
   Проверяем что он запустился и прослушивает порт 5443
   
    systemctl status logstash
    ss -tulnp | grep 5443
    
   ШАГ 6. Устанавливаем и конфигурируем Filebeat на клиенте CentOS
   
   Переходим обратно на машину web и добавляем ключ
   
    rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
    
   Скачиваем пакет Filebeat и устанавливаем его:
   
    wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.1.1-x86_64.rpm
    rpm -ivh filebeat-5.1.1-x86_64.rpm
    
   Редактируем файл
   
    vim /etc/filebeat/filebeat.yml
    
   Добавляем логи:
    
    paths:
    - /var/log/messages
    - /var/log/secure
    - /var/log/audit/audit.log
    - /var/log/nginx/error.log

   Комментируем в этом же файле строки:
   
    #output.elasticsearch:
    # Array of hosts to connect to.
    #  hosts: ["localhost:9200"]
    
   Раскоментируем строки logstash и добавим свои:
   
    output.logstash:
    # The Logstash hosts
    hosts: ["ELK_IP:5443"]
    bulk_max_size: 1024
    template.name: "filebeat"
    template.path: "filebeat.template.json"
    template.overwrite: false
    
   Вместо ELK_IP ставим IP ELK-машины
   
   Сохраняем файл
   Добавляем filebeat в автозагрузку и стартуем его:
   
     systemctl enable filebeat
     systemctl start filebeat
     
   Проверяем работу:
   
      systemctl status filebeat
      
   ШАГ 7. Тестируем ELK
   
   Заходим с браузера на IP_ELK:5601
   Записываем в поле "Index name or pattern":  filebeat-*
   И нажимаем кнопку Create
   
   После чего нажимаем в меню Discover и смотрим логи идущие с машины web.
   
   
   
   
   https://www.howtoforge.com/tutorial/how-to-install-elastic-stack-on-centos-7/?__cf_chl_captcha_tk__=7f71277d032bd8cf886d223cd8f56253e8d601df-1579787901-0-Af2uoCeRWq0eF-EqTXD7uHAMj2T0tSQ9kvw_LguW_YWpdZhCLrjOKAAxUKV4NHxwlw-RxedVoeZ-0m6HWhFgeYSBKW5gKtAuGE64LA-ItzB5lHFV3F90egLFHNME0C9jQQKoMKAu_DnpxmcN3x9e1YQ4VwlCjFyhFcUDiDhjHG-X6duY3EtQCKRlyJnFL4Wa47fV-8ZVlPcV_ZPzmHDDpF4DaU9hW2tnz4jtW0jSbB0YvcjSoTEDTCA9y09y9yfNovNK4KAgrSG5Sn-W7pbAD_evAJZMFZTmXl57rrno6KQdNeU540CrG58eB9MqkSBV6fh1CmJLRZU1w0b2YiAN4Y0fXZm6Te73b_olgo_v2C-QvU-DfIcjLA6uDCnRnTxljg

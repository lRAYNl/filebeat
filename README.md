- Для работы ELK-стека, нужно установить агента сборщика (если будете использовать не filebeat, то пропускаем). Первое, пробрасываем ключ (бывают случаи, что нет доступа к этим директориям, поэтому, можно использовать зеркала, которые есть в свободном доступе, для скачивания пакета)
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
    echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
    sudo apt-get update
    sudo apt install filebeat=9.2.2 -y

Настройка агента по сбору логов:
-
- Для того, чтобы успешно собирать и отправлять логи на logstash, требуется установить filebeat и сбросить конфигурационный файл из репозитория (если используете другой сборщик, можете пропустить этот пункт). Если вы установили filebeat ранее, можете проводить его настройку
- Удаляем (или можем перенести) конфигурационный файл из стандартной директории и переносим из скачанного репозитория
    rm /etc/filebeat/filebeat.yml
    mv elk-cluster/filebeat.yml

- Меняем данные на нужные
    nano /etc/filebeat/filebeat.yml

- Создаем папку с сертификатами
    sudo mkdir /etc/filebeat/certs #Затем перекидываем logstash.crt
    cp ~/elk-cluster/certs/logstash.crt /etc/filebeat/certs

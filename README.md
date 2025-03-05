# kafka Tutorial

L'objectif de ce tuturial est de vous permettre de faire une prise en main du streaming ingestion avec Kafka.

## Installation

Nous allons installer kafa dans notre mini-cluster hadoop vagrant en suivant les étapes suivant.

- Téléchargement

```shell
# Download the latest version 
wget https://dlcdn.apache.org/kafka/3.9.0/kafka_2.13-3.9.0.tgz
sudo cp kafka_2.13-3.9.0.tgz /opt
cd /opt
sudo tar -xzf kafka_2.13-3.9.0.tgz
sudo ln -s kafka_2.13-3.9.0 kafka
```

- Ajouter le chemin kafka dans PATH

```shell
cd ~
vi .bash_profile
# Add the following lines before the line export PATH 
export KAFKA_HOME=/opt/kafka
PATH=$PATH:$KAFKA_HOME/bin
# save .bash_profile
# source the .bash_profile with the following command :
source .bash_profile
```

## Demarrer kafka

- Démarrer [Zookeeper](https://zookeeper.apache.org/)

```shell
# Open terminal 1
# Start the ZooKeeper service
zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties
```

- Démarrer kafka server

```shell
#  Open terminal 2
# Start the Kafka broker service
kafka-server-start.sh $KAFKA_HOME/config/server.properties
```

- Premier pas

```shell
# Open terminal 3

# List the existing topics
kafka-topics.sh --list --bootstrap-server localhost:9092

# Create new topic
kafka-topics.sh --create --topic my-first-topic --bootstrap-server localhost:9092

# Describe the details of the topic
kafka-topics.sh --describe --topic my-first-topic --bootstrap-server localhost:9092

# Delete a topic
kafka-topics.sh --delete --topic my-first-topic --bootstrap-server localhost:9092


```

## Labs

### Lab 1 : Premier pas avec Kafka

- **Création de topic**  
Creer un topic avec replication factor 2 et un nombre de partitions égal à 3 sur trois brokers. Le nombre de replication peut etre adapté en fonction du nombre de brokers.

```shell
# Create topic
kafka-topics.sh --create \
    --bootstrap-server broker1.int.com:9092,broker2.int.com:9092,broker3.int.com:9092 \
    --replication-factor 1 \
    --partitions 3 \
    --topic `whoami`_test

# Describe topic
kafka-topics.sh --describe \
    --bootstrap-server broker1.int.com:9092,broker2.int.com:9092,broker3.int.com:9092 \
    --topic `whoami`_test

## List the existing topics
kafka-topics.sh --list \
    --bootstrap-server broker1.int.com:9092,broker2.int.com:9092,broker3.int.com:9092
```

**Note** : Comme ici nous avons un seul broker donc nous allons utiliser la cmd suivante avec une replication-factor égale à **1**.

```shell
# Create topic in multinode cluster
kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 3 \
    --topic `whoami`__test

kafka-topics.sh --describe \
    --bootstrap-server localhost:9092 \
    --topic `whoami`__test

# List the existing topics
kafka-topics.sh --list --bootstrap-server localhost:9092
```

- **Producer/Consumer : Generation de Streams**  
Le produce génére des messages(event) dans un topic et le consumer souscrit pour consommer les messages.
  - **Producer**
  
  ```shell
  # Open terminal 3
  kafka-console-producer.sh --topic `whoami`__test --bootstrap-server localhost:9092

    >Hello,
    >Mon premier message 
  ```

  - Consumer
  
  ```shell
  # Open terminal 4
  kafka-console-consumer.sh --topic `whoami`_test --from-beginning --bootstrap-server localhost:9092

  >Hello,
  >Mon premier message
  
  ```

### Lab 2 : Ingestion de Web Server Logs

- Il faut d'abord télécharger le dossier **gen_logs**
- Copier le dossier dans /opt/
- Ouvrir le fichier **.bash_profile** et ajouter le chemin **/opt/gen_logs/** dans la variable **PATH**.
- Ajouter les droits d'execution aux scripts  **start_logs.sh**, **tail_logs.sh** et **stop_logs.sh** qui se trouve dans le repertoire **gen_logs**.

- Executer les scripts ci-dessus et vérifier le fichier **access.log** est généré dans le repertoire **/opt/gen_logs/logs** avec les commandes ci-dessous.

```shell
head /opt/gen_logs/logs/access.log
cat /opt/gen_logs/logs/access.log | wc -l

tail -F /opt/gen_logs/logs/access.log
```

- Création d'un nouveau topic nommé **vagrant_retail**
- Génération des logs retail par le producer:

```shell
tail_logs.sh|kafka-console-producer.sh \
    --bootstrap-server localhost:9092\
    --topic `whoami`_retail
```

- Consommation des logs par le consumer:

```shell
kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic `whoami`_retail \
    --from-beginning
```

- Arreter la génération des logs avec la commande **stop_logs.sh**.

### Lab 3 : Kafka Connect

- **Définir Kafka Connect pour produire des messages**

  Nous allons préparer les fichiers de configs de Kafka Connect afin que les données puissent être produites dans le topic Kafka en utilisant Kafka Connect.
  
    - Vérifier l'existence du topic Kafka
  
    ```bash
    kafka-topics.sh \
      --zookeeper m01.itversity.com:2181,m02.itversity.com:2181,w01.itversity.com:2181 \
      --describe \
      --topic `whoami`_retail
     ```
  
  - Créer un dossier **/home/vagrant/kafka_connect/retail_logs_produce**
  
    ```bash
    mkdir -p /home/vagrant/kafka_connect/retail_logs_produce
    cd /home/vagrant/kafka_connect/retail_logs_produce
    ```
  
  - Copiez les fichiers de configuration depuis le dossier de **/opt/kafka/config/** de Kafka :
  
    ```bash
    cp /opt/kafka/config/connect-standalone.properties \
        /home/vagrant/kafka_connect/retail_logs_produce/retail_logs_standalone.properties
    cp /opt/kafka/config/connect-file-source.properties \
        /home/vagrant/kafka_connect/retail_logs_produce/retail_logs_file_source.properties
    ```
  
  - Mettre à jour le fichier **retail_logs_standalone.properties**   
  
    Assurez-vous de mettre à jour le serveur bootstrap avec **localhost:9092**.
  
    ```bash
    bootstrap.servers=localhost:9092
    ```  
    
    Si vous utiliser un cluster multi-nœuds, vous pouvez ajouter les brokers :
    
    ```bash
    bootstrap.servers=broker1.int.com:9092,broker2.int.com:9092
    ```
    Modifer les key/value converters
    
    ```bash
    key.converter=org.apache.kafka.connect.storage.StringConverter
    value.converter=org.apache.kafka.connect.storage.StringConverter
    
    key.converter.schemas.enable=true
    value.converter.schemas.enable=true
    ```
    
    Ajouter le nom du fichier de stockage des offsets :
  
    ```bash
    offset.storage.file.filename=/home/vagrant/kafka_connect/retail_logs_produce/retail.offsets
    ```
    
    Intervalle de vidage des offsets (utile pour les tests et le débogage) :
    ```bash
    offset.flush.interval.ms=10000
    ```
    
    Ajouter le numéro de port pour l'API REST de Kafka Connect : Assurez-vous de choisir un numéro de port à 5 chiffres inférieur à 65535.
  
    ```bash
    rest.port=18083
    ```
    
    - Mettre à jour **retail_logs_file_source.properties**
    Définissez l'emplacement du fichier et le nom du topic Kafka
    
    ```bash
    name=retail-file-source
    connector.class=FileStreamSource
    tasks.max=1
    file=/opt/gen_logs/logs/access.log
    topic=vagrant_retail
    ```
    
- Validation de l'ingestion

  - Assurez-vous que le fichier **/opt/gen_logs/logs/access.log** n'est trop volumineux avant de le réinitialiser avec les commandes ci-dessous:

  ```bash
  stop_logs.sh
  cat /dev/null > /opt/gen_logs/logs/access.log
  start_logs.sh
  tail_logs.sh
  ```
  
  - Ouvrir un Terminal et exécutez **connect-standalone.sh** pour produire les données du fichier log en tant que messages dans le topic Kafka en passant comme parametres les deux fichiers de configs :
  
  ```bash
  /opt/kafka/bin/connect-standalone.sh \
      retail_logs_standalone.properties \
      retail_logs_file_source.properties
  ```
  
  - Exécutant le script kafka-console-consumer.sh dans une autre session de terminal pour valider l'ingestion:
  
  ```bash
  /opt/kafka/bin/kafka-console-consumer.sh \
      --bootstrap-server localhost:9092 \
      --topic `whoami`_retail
  ```

### Lab 4 : Kafka Streams

Il faut suivre le tutorial présenté dans la section [Kafka Streams](https://kafka.apache.org/documentation/streams/). Le code source est disponible dans le repertoire streams.examples



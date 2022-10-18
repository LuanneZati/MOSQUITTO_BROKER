# Pesquisa MQTT Broker Mosquitto – AWS

**Recursos utilizados na aplicação:**

1. Placa CPU EMBFLEX ESP32;
2. Máquina virtual EC2 - aws;
3. Mqtt Broker instalado na EC2;
4. Técnica de link de broker IoT com DB;

### Apresentação da aplicação

O objetivo dessa pesquisa é desenvolver uma aplicação de publish e subscibe de um sensor de temperatura implementado na placa CPU EMBFLEX ESP32, onde o sensor publica sua temperatura em um MQTT Broker, nesse caso utilizaremos o Mosquitto, e um script se inscreve, captura esses dados e os envia para um banco de dados. Para essa implementação em cloud, deve-se primeiro ter uma conta em um cloud web servisse, nesse caso foi utilizado a AWS.

**Etapas da aplicação**

1. Criar uma máquina virtual EC2 ubuntu na AWS;
2. Configurar essa instância;
3. Installar Eclipse Mosquitto;
4. Gerar chaves e certificados de acesso;
5. Desenvolver um script em python para fazer a conexão de acesso;

## Criando a instância EC2

-Crie uma IAM role para acesso ao AWSIotConfigAcess;
-Clique em 'Launch Instances' dentro do EC2 e escolha seu SO;
![Alt](./images/ec2.png)
-Selecione o tipo da instância;
![Alt](./images/ec2-1.png)
-Crie uma key pair;
![Alt](./images/ec2-4.png)
-Selecione a IAM Role que foi criada anteriormente;
![Alt](./images/ec2-2.png)
-Faça a configuração do security group, isso da permissão para entradas externas na sua maquina virtual;
![Alt](./images/ec2-3.png)

Instância criada!

## Configuração da instância e instalação do Broker

Faça o acesso a essa maquina atrav´s do SSH, como foi configurado no socurity groups. A key pair que foi criada, deverá estar instalada na máquina que vai acessar a instância criada.

- Entre no seu terminar, acesse a pasta que foi baixada a key pair e digite o código de conexão;
  ![Alt](./images/ec2-5.png)

Em seguida, dentro da sua máquina virtual digite os seguintes comenados:

```js
sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa
sudo apt-get update

sudo apt-get install mosquitto
sudo apt-get install mosquitto-clients
sudo apt install awscli
```

Configuração:

```js
aws configure
```

Configuração AWS, deixe AWS Acess Key ID, AWS Secret Acess Key e Default output format em branco.
Preencha sua região em Default Region name

Criando a IAM policy:

```js
aws iot create-policy --policy-name bridgeMQTT --policy-document '{"Version": "2012-10-17","Statement": [{"Effect": "Allow","Action": "iot:*","Resource": "*"}]}'
```

Mude sua pasta e baixe a AmazonRootCA1:

```js
cd /etc/mosquitto/certs/
sudo wget https://www.amazontrust.com/repository/AmazonRootCA1.pem -O rootCA.pem
```

Crie suas chaves e certificados:

```js
sudo aws iot create-keys-and-certificate --set-as-active --certificate-pem-outfile cert.crt --private-key-outfile private.key --public-key-outfile public.key --region us-east-1
```

Copie sua ARN do certificado (arn:aws:iot:us-east-1:0123456789:cert/xyzxy) e substitua na linha abaixo:

```js
aws iot attach-principal-policy --policy-name bridgeMQTT --principal <certificate ARN>
```

Por fim, fornaça as permissões para a chave e o certificado:

```js
sudo chmod 644 private.key
sudo chmod 644 cert.crt
```

## Script em python para conexão com o Mosquitto

Primeiramente, deve-se instalar as bibliotecas python para conexão com o seu banco de preferência.
Nesse caso, 2 exemplos para PostgreSQL (o que foi usado) e MariaDB (Mysql).
_Observação: desenvolvido em windows_

MariaDB:

```js
pip install mysqlclient
pip install mysql-connector-python
pip install pymysql
```

PostgreSQL:

```js
pip install psycopg2
pip install pygresql
pip install pandas
```

Faça a instalação também da biblioteca AWS:

```js
pip install awscli
python setup.py install
git clone https://github.com/aws/aws-iot-device-sdk-python-v2.git
python -m pip install ./aws-iot-device-sdk-python-v2
```

**Começando o srcript**
Crie um arquivo chamado 'main.py'

Coloque as bibliotecas:
_Se for MariaDB:_

```js
from tkinter import TOP
from awscrt import io, mqtt, auth, http
from awsiot import mqtt_connection_builder
import time as t
import json
# Bibliotecas MySql
import mysql.connector
from mysql.connector import Error
from mysql.connector import errorcode

import time
from datetime import datetime
```

_Se for PostgreSQL_

```js
from tkinter import TOP
from awscrt import io, mqtt, auth, http
from awsiot import mqtt_connection_builder
import time as t
import json
# Bibliotecas PostgreSQL
import requests
import json
import pandas as pd
import psycopg2

import time
from datetime import datetime
```

Variáveis para acesso ao Broker:

```js
ENDPOINT = 'code-ats.iot.region.amazonaws.com'
CLIENT_ID = 'unique_name'
PATH_TO_CERTIFICATE = 'cert.crt'
PATH_TO_PRIVATE_KEY = 'private.pem.key'
PATH_TO_AMAZON_ROOT_CA_1 = 'AmazonRootCA1.pem'

TOPIC = 'esp32/pub'
```

_Observação: cert.crt, private.key e AmazonRootCA1 precisam estar instaladas na mesma pasta do script_

Conexão:

```js
event_loop_group = io.EventLoopGroup(1)
host_resolver = io.DefaultHostResolver(event_loop_group)
client_bootstrap = io.ClientBootstrap(event_loop_group, host_resolver)
mqtt_connection = mqtt_connection_builder.mtls_from_path(
    endpoint=ENDPOINT,
    cert_filepath=PATH_TO_CERTIFICATE,
    pri_key_filepath=PATH_TO_PRIVATE_KEY,
    client_bootstrap=client_bootstrap,
    ca_filepath=PATH_TO_AMAZON_ROOT_CA_1,
    client_id=CLIENT_ID,
    clean_session=False,
    keep_alive_secs=6
)
print("Connecting to {} with client ID '{}'...".format(
    ENDPOINT, CLIENT_ID))
# Make the connect() call
connect_future = mqtt_connection.connect()
# Future.result() waits until a result is available
connect_future.result()
print("Python -> AWS Connected!")
```

Agora, crie um outro script para subscribe no tópico, com o nome de 'subscribe.py'.

Começe importando as bibliotecas:
_Se for MariaDB:_

```js
from decimal import Decimal
import json
from unicodedata import decimal
from main import TOPIC, CLIENT_ID, mqtt_connection
from awscrt import mqtt
import time
import json
# Bibliotecas MySql
import mysql.connector
from mysql.connector import Error
from mysql.connector import errorcode
```

_Se for PostgreSQL:_

```js
from decimal import Decimal
import json
from unicodedata import decimal
from main import TOPIC, CLIENT_ID, mqtt_connection
from awscrt import mqtt
import time
import json
# Bibliotecas PostgreSql
import requests
import json
import pandas as pd
import psycopg2
```

Faça um loop para inserir as dados no banco:

_PostgreSQL:_

```js
while True:
    def on_message_received(topic, payload, dup, qos, retain, **kwargs):
        obj = json.loads(payload)
        print(obj)
        temp = (obj['Temperature'])
        print(temp)
        dh = (obj['Data Hora'])
        print(dh)
        print("Received message from topic '{}': {} - {}".format(TOPIC, temp, dh))
        try:
            connection = psycopg2.connect(host='ip da instancia mqtt',
                         database='mqtt',
                         user='postgres',
                         password='password')

            postgresql_insert_query = f"INSERT INTO temperature(temp, dh) VALUES ({temp}, '{dh}')"

            cursor = connection.cursor()
            cursor.execute(postgresql_insert_query)
            connection.commit()
            print(cursor.rowcount, "Data entered successfully")
            cursor.close()
            connection.close()
            print("Closed connection")
        except (Exception, psycopg2.DatabaseError) as error:
            print("Database failure".format(error))
        time.sleep(5)
    # subscribe to topic
    subscribe_future, packet_id = mqtt_connection.subscribe(
        TOPIC,
        qos=mqtt.QoS.AT_LEAST_ONCE,
        callback=on_message_received)
    print(f'Subscribed to {TOPIC}')

    # result() waits until a result is available
    subscribe_result = subscribe_future.result()
    time.sleep(5)

```

_MariaDB:_

```js
while True:
    def on_message_received(topic, payload, dup, qos, retain, **kwargs):
        obj = json.loads(payload)
        print(obj)
        temp = (obj['Temperature'])
        print(temp)
        dt = (obj['Data Hora'])
        print(dt)
        print("Received message from topic '{}': {} - {}".format(TOPIC, temp, dt))
        try:
            connection = mysql.connector.connect(host='ip da instância mqtt',
                                                 database='mqtt',
                                                 user='root',
                                                 password='')
            mySql_insert_query = f"INSERT INTO temperature(temperature, data_hora) VALUES ({temp}, '{dt}')"
            cursor = connection.cursor()
            cursor.execute(mySql_insert_query)
            connection.commit()
            print(cursor.rowcount, "Data entered successfully")
            cursor.close()
        except mysql.connector.Error as error:
            print("Database failure".format(error))
        finally:
            if (connection.is_connected()):
                connection.close()
                print("Closed connection")
        time.sleep(10)
    # subscribe to topic
    subscribe_future, packet_id = mqtt_connection.subscribe(
        TOPIC,
        qos=mqtt.QoS.AT_LEAST_ONCE,
        callback=on_message_received)
    print(f'Subscribed to {TOPIC}')

    # result() waits until a result is available
    subscribe_result = subscribe_future.result()
    time.sleep(10)

```

Esse script acima se inscreve no tópico esp32/pub, captura a mensagem recebida em JSON, separa a temperatura
da data hora e transforma em string.
Tenta-se a conexão com o banco, se conectar, o código manda esses dados diretamente para o banco de dados.

## Conclusão da pesquisa MQTT Broker

**Sobre o MOSQUITTO Broker**
Escrito em C, o Mosquitto certamente está entre as principais opções para um corretor MQTT. É leve e escalável. Alguns contras, no entanto, seriam a falta de suporte para clustering e uso de CPU multi-thread. Suporta MQTT 3.xe MQTT 5.

Nessa pesquisa foi utilizado o MQTT Broker Eclipse Mosquitto, script em Python, e bancos de dados alternativos.
Na fase de testes finais, a aplicação foi bem sucedida.

## Referências bibliográficas

[https://aws.amazon.com/pt/blogs/iot/how-to-bridge-mosquitto-mqtt-broker-to-aws-iot/](https://aws.amazon.com/pt/blogs/iot/how-to-bridge-mosquitto-mqtt-broker-to-aws-iot/)  
[https://dev.to/aws-builders/aws-iot-pubsub-over-mqtt-1oig](https://dev.to/aws-builders/aws-iot-pubsub-over-mqtt-1oig)  
[https://www.teracomsystems.com/blog/top-5-mqtt-brokers-2/](https://www.teracomsystems.com/blog/top-5-mqtt-brokers-2/)

---
hide:
  - navigation
---

# Extração de dados e Carga automática para nuvem (AWS S3)

Atividade proposta durante a participação na trilha de Engenharia de Dados do Programa de Formação em Dados Encantech, realizado durante os meses de Março, Abril e Maio de 2022, em uma parceria entre as Lojas Renner S.A. e a CESAR School.

Consiste na construção de uma pipeline com estágios de extração de dados e de carregamento automático na nuvem (bucket do AWS S3).

[acesso aos arquivos](https://github.com/peuvitor/aws-s3-database-dump)

## Overview

A ideia é simular o armazenamento na nuvem de backups de um banco de dados em funcionamento.

1.  O usuário realiza o dump no banco de dados quando achar mais pertinente (ex: horário em que ninguém esteja acessando o banco de dados);
    
2.  Em paralelo, um serviço verifica periodicamente a existência do arquivo resultante do dump:
    
    *   Se houver arquivo, este é carregado em um bucket do AWS S3;
        
    *   Se não houver arquivo, registra-se a tentativa no arquivo de log e nada mais é executado.
        

Resumo do funcionamento:

![](https://github.com/peuvitor/aws-s3-database-dump/blob/main/images/pipeline.png?raw=true)

### Pontos de atenção

*   Arquivo resultante do dump no banco de dados: existe uma pasta compartilhada entre o docker responsável pelo dump no banco de dados e o docker responsável pelo envio deste arquivo para a nuvem. Os seguintes cenários são possíveis:
    
    1.  Dump realizado e pasta vazia: armazena o arquivo na pasta;
        
    2.  Dump realizado e pasta cheia: sobrescreve o arquivo antigo;
        
    3.  Pasta com arquivo e tentativa de envio para nuvem: envia o arquivo e o exclui da pasta compartilhada;
        
    4.  Pasta sem arquivo e tentativa de envio para nuvem: nada é executado.
        
*   Armazenamento na nuvem: pastas divididas por data.
    
    *   Mais de um dump na mesma data: adiciona-se um índice no final do nome do arquivo.
*   Periodicidade de envio para nuvem: editar o arquivo de agendamento crontab, vinculado ao Dockerfile. Pensando apenas na implementação deste projeto, definiu-se um período de 1 minuto.
    

### Arquivo de log

`<data>,<hora>,<mensagem>`

<mensagem> pode ser:

*   _database dump completed_ - dump no banco de dados finalizado;
    
*   _there is no file to upload_ - tentativa de envio do arquivo de dump para nuvem, porém a pasta encontra-se vazia;
    
*   _file successfully uploaded_ - arquivo de dump enviado para o respectivo bucket do AWS S3;
    
*   erro apresentado ao tentar enviar arquivo para a nuvem (retorno da cláusula except).
    

Exemplo:

![](https://github.com/peuvitor/aws-s3-database-dump/blob/main/images/log-file-example.PNG?raw=true)

### Segurança - Credenciais da AWS

Para usar a AWS com o boto3 (o AWS SDK para Python) é necessário indicar as chaves de acesso do seu usuário ('Access key ID' e 'Secret access key'). Por segurança, estas chaves não devem ser compartilhadas com ninguém.

Além disso, outra informação levada em conta para esta seção é que o nome do bucket escolhido no projeto é único (cada nome de bucket deve ser exclusivo em todas as contas da AWS em todas as regiões da AWS em uma partição).

A abordagem escolhida foi a de registrar essas três informações (Access key ID, Secret access key e nome do bucket) em um único arquivo fora do repositório deste projeto. Em uma tentativa de execução, se faz necessário a criação deste arquivo com suas próprias credenciais.

## Scripts

### Estrutura

```
project
│   README.md
│   docker-compose.yml    
│ 
└───db_data
│   │   FACULDADE.sql
│   
└───dockerfiles
│   └───mysql-client
│       │   Dockerfile
│   └───python
│       │   Dockerfile
│       │   crontab
│   
└───logs
│   │   2022-06-02.txt
│     
└───scripts
    │   database_dump.sh
    │   send-to-s3.py
```

### Dockerfiles

1. MySQL Client

    ```dockerfile
    FROM ubuntu:focal

    RUN apt-get update && apt-get install -y mysql-client

    ENV TZ=America/Sao_Paulo

    RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone && \
        apt-get update && apt-get install -y tzdata && dpkg-reconfigure -f noninteractive tzdata

    CMD tail -f /dev/null
    ```

2. Python

    ```dockerfile
    FROM python:3.8-alpine3.14

    COPY ./crontab /var/spool/cron/crontabs/root

    RUN pip install boto3

    ENV TZ=America/Sao_Paulo

    RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

    CMD crond -l 2 -f && tail -f /var/log/cron.log
    ```

    crontab:

    ```
    *	*	*	*	*	python /bin/send-to-s3.py
    ```

### Docker compose

```yaml
version: '3.1'

services:
  database:
    image: mariadb
    container_name: database
    hostname: database
    environment:
      - MARIADB_ROOT_PASSWORD=password
      - MARIADB_DATABASE=FACULDADE
    volumes:
      - ./db_data/FACULDADE.sql:/docker-entrypoint-initdb.d/FACULDADE.sql

  dump:
    image: mysql-client
    container_name: dump
    hostname: dump
    environment:
      - MARIADB_ROOT_PASSWORD=password
    depends_on:
      - database
    volumes:
      - ./dumps/:/dumps/
      - ./scripts/database_dump.sh:/database_dump.sh
      - ./logs/:/logs/
    command:
      - ./database_dump.sh

  send-to-s3:
    image: python-s3
    container_name: send-to-s3
    hostname: send-to-s3
    depends_on:
      - dump
    volumes:
      - ./dumps/:/dumps/
      - ../credentials.txt:/credentials.txt
      - ./scripts/send-to-s3.py:/bin/send-to-s3.py
      - ./logs/:/logs/
```

### Extração

Script em bash que realiza o dump no banco de dados, salva o arquivo resultante em uma pasta compartilhada e registra operação no arquivo de log.

```bash
#!/bin/bash

date=$(date +'%F')
dateAndTime=$(date +'%F,%H:%M:%S')

mysqldump -hdatabase -uuser -ppassword FACULDADE > dumps/datadump.sql

echo "${dateAndTime},database dump completed" >> logs/"${date}".txt
```

### Carga

1. Quando o script é chamado, antes de executar qualquer ação, é verificado se existe algum arquivo a ser enviado.

    ```python
    if __name__ == '__main__':
    
        DUMP_FILE = '/dumps/datadump.sql'

        current_date = datetime.today().strftime("%Y-%m-%d")
        current_hour = datetime.today().strftime("%H:%M:%S")
        log_file = f"../logs/{current_date}.txt"
        
        if os.path.exists(DUMP_FILE):
            main()

        else:
            with open(log_file, 'a+') as file:
                file.write(f"{current_date},{current_hour},there is no file to upload\n")
    ```

2. Existindo o arquivo, abre-se uma conexão com a AWS e este é devidamente carregado em um bucket da AWS S3.

    ```python
    def main():

        KEY_ID, ACCESS_KEY, BUCKET_NAME = get_credentials('../credentials.txt')
        
        s3 = boto3.client('s3', 
                        region_name='us-east-1',
                        aws_access_key_id=KEY_ID,
                        aws_secret_access_key=ACCESS_KEY)
        
        send_dump_file = AWSobjectWrapper(s3)

        send_dump_file.create_bucket_if_necessary(BUCKET_NAME)     

        send_dump_file.upload_dump_file(DUMP_FILE)
    ```

3. Abaixo segue o que é a classe AWSobjectWrapper e seus métodos.

    ```python
    class AWSobjectWrapper:

        def __init__(self, s3_object):
            self.object = s3_object

        def create_bucket_if_necessary(self, s3_bucket):
            self.bucket_name = s3_bucket
            
            existing_buckets = [bucket['Name'] for bucket in self.object.list_buckets()['Buckets']]
        
            if self.bucket_name not in existing_buckets: 
                self.object.create_bucket(Bucket=self.bucket_name)
        
        def count_files_in_folder(self):
            self.files_per_day = 0
            
            existing_files = self.object.list_objects(Bucket=self.bucket_name)

            if 'Contents' in existing_files:
                self.files_per_day = sum(1 for content in existing_files.get('Contents') \
                                                if current_date in content.get('Key'))

        def message_log_file(self, message):
            with open(log_file, 'a+') as file:
                file.write(f"{current_date},{current_hour},{message}\n")
        
        def upload_dump_file(self, source_file):
            self.count_files_in_folder()

            try:
                self.object.upload_file(
                        Filename=source_file,
                        Bucket=self.bucket_name,
                        Key=f'{current_date}/datadump{self.files_per_day + 1}.sql'
                        )

            except Exception as e:
                self.message_log_file(e)

            else:
                self.message_log_file('file successfully uploaded')
                os.remove(source_file)
    ```
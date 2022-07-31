---
hide:
  - navigation
---

# Extração de dados e Envio automático para nuvem (AWS S3)

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
*   Periodicidade de envio para nuvem: editar o arquivo de agendamento [crontab](https://github.com/peuvitor/aws-s3-database-dump/blob/main/dockerfiles/python/crontab), vinculado ao Dockerfile. Pensando apenas na implementação deste projeto, definiu-se um período de 1 minuto.
    

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

## [TODO] Colocar trechos de código e explicar
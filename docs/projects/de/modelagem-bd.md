---
hide:
  - navigation
---

# Modelagem de banco de dados e execução em docker

Atividade proposta durante a participação na trilha de Engenharia de Dados do Programa de Formação em Dados Encantech, realizado durante os meses de Março, Abril e Maio de 2022, em uma parceria entre as Lojas Renner S.A. e a CESAR School.

Consiste na modelagem de um banco de dados, da criação de um gerador de massa e da execução em um docker container.

[acesso aos arquivos](https://github.com/peuvitor/modelagem-banco-de-dados-e-docker)

## Mini mundo

Uma faculdade contratou você como engenheiro de dados e pediu seu apoio para a construção de um modelo de banco de dados para gerenciar seus funcionários, divididos entre administrativos e professores, permitindo também alocar um professor por matéria por semestre, permitindo assim a rotação entre os professores durante os semestres.

O sistema também deve registrar os dados de matrícula dos alunos, o ano letivo e suas matérias que serão cursadas por semestre, vinculando aos professores que irão ministrar aulas nesse período.

Os dados referentes aos alunos devem conter sua filiação, sua data de nascimento, endereço completo, telefone para contato, e-mail e seu histórico de notas por matérias.

## Modelagem

O modelo desenvolvido visa garantir que os requisitos do mini mundo sejam respeitados, mas vai além, considerando outras relações existentes quando se fala de uma faculdade. Vale considerar que a normalização do banco de dados também foi levada em conta.

*   Modelo conceitual:
    
![](https://github.com/peuvitor/modelagem-banco-de-dados-e-docker/blob/main/modelagem-banco-de-dados/modelo-relacional.jpg?raw=true)

*   Modelo lógico:
    
![](https://github.com/peuvitor/modelagem-banco-de-dados-e-docker/blob/main/modelagem-banco-de-dados/modelo-logico.png?raw=true)

*   [Modelo físico](https://github.com/peuvitor/modelagem-banco-de-dados-e-docker/blob/main/scripts_sql/DDL_FACULDADE.sql): gerado a partir do MySQL Workbench.
    

## Gerador de massa

O modelo criado conta com um total de 26 tabelas. Para gerar um script capaz de popular todo o banco de dados, a estratégia foi popular as tabelas seguindo a ordem abaixo, pois como há cruzamentos de informações, é necessário garantir que não haverá problemas de dependência.

1.  tabelas de domínio (exemplos: CIDADE, DEPARTAMENTO);
    
2.  tabelas que dependem apenas das tabelas de domínio (exemplo: PESSOA);
    
3.  tabelas de especializações (exemplos: ALUNO, PROFESSOR);
    
4.  tabelas de relacionamentos (exemplo: AULA).

Para tal, os arquivos de suporte foram: [lista de nomes](https://github.com/peuvitor/modelagem-banco-de-dados-e-docker/blob/main/gerador-de-massa/nomes.txt) (30 nomes), [lista de sobrenomes](https://github.com/peuvitor/modelagem-banco-de-dados-e-docker/blob/main/gerador-de-massa/sobrenomes.txt) (434 sobrenomes) e [lista de CEPs válidos](https://github.com/peuvitor/modelagem-banco-de-dados-e-docker/blob/main/gerador-de-massa/ceps.txt) (732.763 CEPs, contendo rua, número, bairro, cidade, estado). A partir desses arquivos, diversas combinações foram geradas, permitindo a população do banco completo sem replicar registros.

As tabelas de domínio que precisam de mais informações foram definidas manualmente:

- STATUS_ALUNO: 

    ["Cursando", "Trancado", "Formado", "Jubilado"]

- STATUS_FUNCIONARIO: 

    ["Ativo", "Desligado", "Férias", "Afastado"]

- TITULACAO: 

    ["Graduado", "Especialista", "Mestre", "Doutor"]

- SETOR: 

    ["Secretariado", "Coordenação", "Biblioteca", "Facilities"]

- TURNO: 

    ["Matutino", "Vespertino", "Noturno", "Integral"]

- TIPO_AVALIACAO: 

    ["Prova 1", "Prova 2", "Prova 3", "Prova de Recuperação", "Prova de Final"]

- DEPARTAMENTO: 

    ["DADOS", "EXATAS", "BIOLOGICAS", "HUMANAS E SOCIAIS", "ARTES", "LINGUAS"]

- CURSO_GRADUACAO: 

    ["Engenharia de Dados", "Ciência de Dados", "Análise de Dados", "Engenharias de Machine Learning", "Estatística", "Matemática", "Física", "Química", "Biologia", "Medicina", "Enfermagem", "Fisioterapia", "Filosofia", "Administração", "Sociologia", "Pedagogia", "Artes Cênicas", "Música", "Turismo", "Artes Visuais", "Português", "Linguas Estrangeiras", "Linguas Clássicas", "Filologia"]

O [script desenvolvido em python](https://github.com/peuvitor/modelagem-banco-de-dados-e-docker/blob/main/gerador-de-massa/GERADOR-MASSA.ipynb) cria o DML em SQL para cada uma das tabelas, abaixo um exemplo do que foi feito para a tabela STATUS_FUNCIONARIO:

```python
with open("INSERTS_FACULDADE_DOMINIOS/STATUS_FUNCIONARIO.sql", "w", encoding="utf-8") as f:
    ID_STATUS_LIST = [1,2,3,4]
    DESCRICAO_STATUS_LIST = ["Ativo", "Desligado", "Férias", "Afastado"]
    for i in range(len(ID_STATUS_LIST)):
        ID_STATUS = ID_STATUS_LIST[i]
        DESCRICAO_STATUS = DESCRICAO_STATUS_LIST[i]
        sql = f'INSERT INTO STATUS_FUNCIONARIO (ID_STATUS, DESCRICAO_STATUS) VALUES ({ID_STATUS}, "{DESCRICAO_STATUS}")'
        f.write(sql + ";" + "\n")
```

No final, as instruções de inserção foram divididas em dois arquivos: [popular tabelas de domínio](https://github.com/peuvitor/modelagem-banco-de-dados-e-docker/blob/main/scripts_sql/DML_TABELAS_DOMINIO.SQL) e [popuplar tabelas fato](https://github.com/peuvitor/modelagem-banco-de-dados-e-docker/blob/main/scripts_sql/DML_MASSA.SQL).

## Docker

Agora com os scripts de DDL e DML em mãos, o banco de dados será executado um docker a partir de uma imagem do SGBD MariaDB.

```yaml
version: '3.1'

services:
  database:
    image: mariadb
    container_name: mariadb-bd
    environment:
      - MARIADB_ROOT_PASSWORD=password
```

Após subir o container, basta executar os scripts de DDL e DML na ordem abaixo. Com isso, chegamos em uma simulação de um banco de dados em operação.

1. definição do banco de dados: scripts_sql/DDL_FACULDADE.sql

2. popular tabelas de domínio: scripts_sql/DML_TABELAS_DOMINIO.sql

3. popular tabelas fato: scripts_sql/DML_MASSA.sql
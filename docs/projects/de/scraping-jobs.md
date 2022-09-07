---
hide:
  - navigation
---

# Raspagem de dados do LinkedIn

Requisitos para rodar a aplicação: arquivo com credenciais para acessar o Google Sheets e ter o Docker instalado.

[acesso aos arquivos](https://github.com/peuvitor/linkedin-jobs-scraper)

## Overview

Periodicamente extrair do LinkedIn as vagas abertas de **Data Engineer** no **Brasil** e salvar os detalhes de cada uma delas em uma planilha do Google Sheets.

1.  O script em shell extrai periodicamente as vagas abertas e salva em uma pasta compartilhada um arquivo .txt com o link de cada uma delas;
    
2.  O script em python:
    
    *   periodicamente (não sincronizado com o script em shell) verifica os arquivos presentes na pasta compartilhada;
        
    *   para cada arquivo encontrado, varre todos os links existentes;
        
    *   em cada link de vaga de emprego extrai-se: o título da vaga, a empresa que está ofertando, o local de trabalho e a descrição;
        
    *   todas essas informações são salvas em uma planilha do Google Sheets, onde cada aba corresponde a um arquivo;
        
    *   após finalizar a carga na planilha, o arquivo com os links das vagas é movido para uma subpasta da pasta compartilhada, de modo a manter o registro de todas as extrações realizadas e salvas.
        

Resumo do funcionamento:

![](https://github.com/peuvitor/linkedin-jobs-scraper/blob/main/images/overview.png?raw=true)

## Mais detalhes

### Automação da aplicação

*   a aplicação é executada com um cron job e, para efeito de demonstração, foram escolhidos intervalos de 2 e 11 minutos para os scripts em shell e em python, respectivamente;
    
*   existência de uma aba de consolidação, que guarda um resumo dos dados de todas as outras abas existentes.
    

### Planilha final

Abaixo é a captura da tela assim que a planilha é aberta.

![](https://github.com/peuvitor/linkedin-jobs-scraper/blob/main/images/planilha-geral-1.PNG?raw=true)

Agora, um exemplo do conteúdo presente nas abas referentes a cada arquivo da pasta compartilhada. É interessante pontuar que o resultado obtido ao realizar a coleta de dados na pesquisa de vagas no LinkedIn conta sempre com 25 vagas de emprego. Quando você está conectado a uma conta do LinkedIn, o conteúdo é apresentado em diversas páginas de 25 vagas, o que permitiria a iteração via código ao mudar um parâmetro presente no URL da pesquisa do LinkedIn.

Porém, da forma que está sendo feita a aplicação, não é realizado nenhum login antes da coleta de dados. Por conta disso, a página do LinkedIn é mostrada de maneira diferente, não por páginas contendo 25 vagas de emprego, mas sim com uma rolagem (scroll) infinita que apresenta mais 25 novas vagas assim que carregada. Dessa forma, para conseguir extrair mais de 25 vagas por vez, se faz necessário o acréscimo de um algoritmo que interaja com a página, o que, considerando o objetivo deste projeto, não foi realizado aqui.

![](https://github.com/peuvitor/linkedin-jobs-scraper/blob/main/images/planilha-exemplo-2.PNG?raw=true)

Por fim, é possível observar todas as abas existentes na planilha. É interessante observar a questão da periodicidade de execução dos scripts em shell e em python e também checar como os arquivos estão organizados na pasta "job-links/".

O script em shell (extrair links das vagas) executa diversas vezes antes do script em python (extrair informações de cada link e salvar na planilha), por isso, as abas presentes na planilha correspondem aos arquivos presentes na pasta "job-links/sent/". Ainda há outros arquivos na pasta "job-links/", mas a aplicação foi parada antes do script em python executar novamente.

![](https://github.com/peuvitor/linkedin-jobs-scraper/blob/main/images/planilha-abas-3.png?raw=true)

### Filtragem do conteúdo HTML

Página de vagas de emprego do LinkedIn de acordo com os filtros de título e local de trabalho;

![](https://github.com/peuvitor/linkedin-jobs-scraper/blob/main/images/linkedin-all-jobs-1.JPG?raw=true)

No resultado da coleta de dados da página que lista todas as vagas abertas, é possível observar que cada uma das 25 vagas são identificadas por um ID, acessível em "data-entity-urn";

![](https://github.com/peuvitor/linkedin-jobs-scraper/blob/main/images/html-all-jobs-2.JPG?raw=true)

Após extrair os IDs das vagas mostradas no momento do acesso, é possível acessar cada uma delas a partir do URL apresentado na imagem. Por conta disso, o arquivo de texto criado conta com o URL completo para cada uma das 25 vagas, onde será possível extrair: o título da vaga, a empresa que está ofertando, o local de trabalho e a descrição;

![](https://github.com/peuvitor/linkedin-jobs-scraper/blob/main/images/linkedin-job-3.JPG?raw=true)

Investigando o resultado da coleta de dados da página específica de uma vaga de emprego, são observadas as seguintes relações com o que se deseja extrair. Com isso, é implementada a extração de informações a partir dessas classes, que depois são armazenadas e enviadas para a planilha do Google Sheets.

![](https://github.com/peuvitor/linkedin-jobs-scraper/blob/main/images/html-job-4.JPG?raw=true)

## Scripts

### Estrutura

*   arquivo "docker-compose.yml": para funcionamento da aplicação, basta subir o container presente neste arquivo. Nele constam a criação da imagem e o mapeamento dos volumes;
    
*   pasta "data": armazena o arquivo JSON com as credenciais que autorizam o acesso à planilha;
    
*   pasta "docker-image": armazena o arquivo do Dockerfile para criação da imagem e o arquivo crontab que agenda a execução dos scripts;
    
*   pasta "scripts": armazena os scripts em shell e em python;
    
*   pasta "job-links": armazena os arquivos resultantes das coletas de dados do LinkedIn.

### Dockerfile

```dockerfile
FROM python:3.8-alpine3.14

COPY ./crontab /var/spool/cron/crontabs/root

RUN apk --no-cache add curl grep sed dos2unix

RUN dos2unix /var/spool/cron/crontabs/root

RUN pip install gspread oauth2client beautifulsoup4

CMD crond -l 2 -f && tail -f /var/log/cron.log
```

crontab:

```
*/11	*	*	*	*	python /bin/etl-script.py
*/2	    *	*	*	*	sh /bin/get-job-links.sh
```

### Docker compose

```yaml
version: '3.1'

services:
  google-api:
    build: ./docker-image
    image: python
    container_name: google-api
    volumes:
      - ./data/client_secrets.json:/client_secrets.json
      - ./job-links/:/job-links
      - ./scripts/etl-script.py:/bin/etl-script.py
      - ./scripts/get-job-links.sh:/bin/get-job-links.sh
```

### Extrair links das vagas

```bash
#!/bin/sh

JOB_LINK="https\:\/\/www.linkedin.com\/jobs\/view\/"

JOB_TITLE="data%20engineer"
LOCATION="brazil"
URL="https://www.linkedin.com/jobs/search?keywords=${JOB_TITLE}&location=${LOCATION}"

dateAndTime=$(date -u +'%F %H-%M-%S %Z')
OUTPUT_FILE="../job-links/$dateAndTime.txt"

curl "$URL" | grep data-entity-urn  | sed "s/^.*urn:li:jobPosting:/$JOB_LINK/" | sed "s/\" data-search-id.*$//" >> "$OUTPUT_FILE"
```

[Resultados da execução do script](https://github.com/peuvitor/linkedin-jobs-scraper/tree/main/job-links)

### Extrair informações de cada link e salvar na planilha

1. Função principal que lê os arquivos que contêm os links das vagas de emprego, extrai o conteúdo de cada uma delas e salva em uma planilha do Google Sheets.

    ```python
    def main():
        CREDENTIALS_FILE = "../client_secrets.json"
        SPREADSHEET_KEY = "1KSsI8nJFyp_vYyaKFBZFJ-w-wZqou85dAdIj9E6a3zI"
        JOB_LINKS_PATH = "../job-links/"
        SAVED_JOB_LINKS_PATH = JOB_LINKS_PATH + "sent/"

        spreadsheet = Spreadsheet(CREDENTIALS_FILE, SPREADSHEET_KEY)

        files_with_job_links = [f 
                                for f in os.listdir(JOB_LINKS_PATH) 
                                if os.path.isfile(os.path.join(JOB_LINKS_PATH, f))]
        
        for urls_file in files_with_job_links:
            job_data = []
            
            for url in file_lines_into_list(JOB_LINKS_PATH+urls_file):
                job_data.append(get_url_content_as_dict(url))

            data_keys = sorted(get_all_keys_as_list(job_data))

            # nome da aba é o carimbo data e hora da extração dos links das vagas 
            # = nome do arquivo sem a sua extensão
            worksheet_name = urls_file.split('.')[0]

            spreadsheet.insert_values(job_data, data_keys, worksheet_name)

            spreadsheet.consolidation()

            if not os.path.exists(SAVED_JOB_LINKS_PATH): 
                os.makedirs(SAVED_JOB_LINKS_PATH)

            os.replace(JOB_LINKS_PATH+urls_file, SAVED_JOB_LINKS_PATH+urls_file)
    ```

2. Funções utilizadas para ler os arquivos que contêm os links das vagas, extrair e organizar o conteúdo html das páginas.

    ```python
    def file_lines_into_list(file):
        with open(file, 'r') as f:
            lines = f.read().splitlines() 
        return lines

    def filter_html_content(html_content, filter):
        try:
            result = html_content.find(True, {"class": filter}).text.strip()
        except:
            result = ""
        return result

    def get_url_content_as_dict(url):
        content = {}
        page = requests.get(url)
        soup = BeautifulSoup(page.content, "html.parser")

        content["url"] = url
        content['job title'] = filter_html_content(soup, "sub-nav-cta__header")
        content['company'] = filter_html_content(soup, "sub-nav-cta__optional-url")
        content['location'] = filter_html_content(soup, "sub-nav-cta__meta-text")
        content['description'] = filter_html_content(soup, "show-more-less-html__markup show-more-less-html__markup--clamp-after-5")
        
        return content

    def get_all_keys_as_list(list_of_dictionaries):
        # retorna todas as chaves presentes em uma lista de dicionários
        all_keys = list( set().union( *(dict.keys() for dict in list_of_dictionaries) ) )
        return all_keys
    ```

3. Abaixo segue o que é a classe Spreadsheet e seus métodos.

    ```python
    class Spreadsheet():

    SCOPE = ['https://spreadsheets.google.com/feeds', 'https://www.googleapis.com/auth/drive']
    CONSOLIDATION_WORKSHEET = "DATA CONSOLIDATION"
    HEADER_INDEX = 1
    HEADER_FIRST_CELL = CONSOLIDATION_WORKSHEET+"!A1"

    def __init__(self, credentials, spreadsheet_key):
        # a inicialização consiste em se conectar com a referente planilha e deixá-la disponível para trabalho
        self.spreadsheet_key = spreadsheet_key
        self.creds = ServiceAccountCredentials.from_json_keyfile_name(credentials, Spreadsheet.SCOPE)
        self.client = gspread.authorize(self.creds)
        self.spreadsheet = self.client.open_by_key(self.spreadsheet_key)

    def _create_worksheet(self, worksheet_name):
        return self.spreadsheet.add_worksheet(title=worksheet_name, rows=1000, cols=20)

    def list_worksheets(self):
        worksheet_objs = self.spreadsheet.worksheets()
        worksheets_list = [worksheet.title for worksheet in worksheet_objs]
        return worksheets_list

    def _arrange_values_for_insertion(self, data, column_header):
        list_values_to_insert = []
        # mapeamento cabeçalho e chave
        header_to_key = { i : i for i in column_header }
        
        # organizar os dados de acordo com o cabeçalho
        # caso um registro não tenha determinada chave, o valor referente ficará em branco
        for instance in data:
            arrange_key_to_header = []
            for column in column_header:
                try:
                    arrange_key_to_header.append( instance[ header_to_key[column] ] )
                except:
                    arrange_key_to_header.append("")
            list_values_to_insert.append(arrange_key_to_header)

        return list_values_to_insert
    
    def insert_values(self, data, column_header, worksheet_name):
        self.data = data
        self.last_header = column_header

        if worksheet_name not in self.list_worksheets(): 
            self._create_worksheet(worksheet_name)
        self.current_worksheet = self.spreadsheet.worksheet(worksheet_name)
        
        # insere cabeçalho
        self.current_worksheet.insert_row(self.last_header, Spreadsheet.HEADER_INDEX)

        # anexa os dados alinhando conteúdo e cabeçalho
        list_values_to_insert = self._arrange_values_for_insertion(self.data, self.last_header)
        self.spreadsheet.values_append(
                                    worksheet_name, 
                                    {'valueInputOption': 'USER_ENTERED'}, 
                                    {'values': list_values_to_insert})

    def consolidation(self):
        if Spreadsheet.CONSOLIDATION_WORKSHEET not in self.list_worksheets(): 
            self._create_worksheet(Spreadsheet.CONSOLIDATION_WORKSHEET)
        self.data_consolidation_worksheet = self.spreadsheet.worksheet(Spreadsheet.CONSOLIDATION_WORKSHEET)
        
        # insere cabeçalho considerando o já existente na aba DATA CONSOLIDATION,
        # além de possíveis novas colunas da última aba criada
        self.current_header = self.data_consolidation_worksheet.row_values(Spreadsheet.HEADER_INDEX)
        self.current_header.extend([i for i in self.last_header if i not in self.current_header])
        self.spreadsheet.values_update(
                                    Spreadsheet.HEADER_FIRST_CELL, 
                                    {'valueInputOption': 'USER_ENTERED'}, 
                                    {'values': [self.current_header]})

        # lista com todos os urls presentes na aba DATA CONSOLIDATION
        key = "url"
        key_cell = self.data_consolidation_worksheet.find(key)
        values_in_data_consolidation = self.data_consolidation_worksheet.col_values(key_cell.col)[1:]

        # urls da última aba criada que ainda não constam na aba "DATA CONSOLIDATION"
        new_rows = [instance 
                    for instance in self.data 
                    if instance.get(key) not in values_in_data_consolidation]

        # anexa os dados alinhando conteúdo e cabeçalho
        list_values_to_insert = self._arrange_values_for_insertion(new_rows, self.current_header)
        self.spreadsheet.values_append(
                                    Spreadsheet.CONSOLIDATION_WORKSHEET, 
                                    {'valueInputOption': 'USER_ENTERED'}, 
                                    {'values': list_values_to_insert})
    ```
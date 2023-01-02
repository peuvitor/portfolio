---
hide:
  - navigation
---

# Desenvolvimento de aplicativo no MATLAB para projetos com controladores repetitivos

O aplicativo foi construído no Trabalho de Conclusão de Curso de Engenharia de Controle e Automação, da Universidade Federal de Pernambuco. O controlador repetitivo utilizado como base do aplicativo é proposto na tese de doutorado de [Rafael Cavalcanti Neto](https://scholar.google.com/citations?hl=pt-BR&user=BPnQFyUAAAAJ). 

No [repositório](https://github.com/peuvitor/repetitive-controller-designer) é possível ter acesso aos scripts desenvolvidos. O aplicativo também está [disponível no site oficial da MathWorks](https://www.mathworks.com/matlabcentral/fileexchange/74759-repetitive-controller-designer).

## Compatibilidade entre versões

Criado com a versão R2016a, compatível com qualquer versão

## Funcionamento

### Janela inicial

Nesta etapa define-se a planta a ser controlada. Em (1), é ilustrado o diagrama de blocos de um sistema genérico que utiliza o controlador repetitivo. O primeiro passo para o usuário é definir o ganho em (2) e qual a planta a ser trabalhada. 

Para a planta, duas abordagens são possíveis: digitar manualmente os parâmetros da planta (3) ou escolher uma variável já definida no workspace do MATLAB (4). Apenas após o botão OK (5) ser pressionado, os dois botões presentes em (6) são habilitados, permitindo ao usuário a análise de estabilidade do sistema (botão System Stability) ou o dimenionamento de um filtro ideal (botão Filter Design). Apertar o botão em (7) abre uma página onde é possível encontrar informações a respeito da criação da ferramenta e um pequeno resumo de seu funcionamento.

![image](https://drive.google.com/uc?export=view&id=1ndzOjbVY1jx10e3BlGFHJnYwjrhIRClH)

### Análise de estabilidade

Apertar o botão 'System Stability' na janela inicial possibilita a observação do impacto da alteração dos parâmetros do controlador na estabilidade do sistema. A estabilidade é garantida quando o diagrama de Nyquist da planta escolhida está completamente contido no domínio de estabilidade do controlador.

São dadas duas possibilidades ao usuário para observar a influência desses dois parâmetros (1). É possível ou variar o valor dos parâmetros com a ajuda de sliders ou definir livremente valores reais. O impacto dessas variações é imediatamente observado no gráfico ao lado (3), onde, além do domínio de estabilidade em verde, também é mostrado o diagrama de Nyquist da planta em cinza e, em vermelho, a frequência limite até onde o diagrama de Nyquist pertence ao domínio de estabilidade do sistema. Esta frequência é mostrada em (2), além de também mostrar se, com tal configuração, o sistema é estável ou não.

Caso alguma configuração seja útil ao usuário, em (4) é possível fazer uma captura de tal gráfico e escolher o local onde se deseja salvar a imagem gerada. São também disponibilizados alguns recursos do próprio MATLAB para que o usuário possa analisar o gráfico gerado (5), como aplicar ou remover zoom, ver os valores em determinados pontos das curvas e realizar translação do gráfico. Ainda, em (6), é possível ajustar os parâmetros dos sliders (passo e limites máximo e mínimo) e das caixas de textos disponíveis para escolha dos valores (resolução para arredondamento), e também configurar os limites mínimos e máximos dos eixos do gráfico.

![image](https://drive.google.com/uc?export=view&id=1rfaeBZOQ9vwvFshEwh_v82JlrRM9bMVQ)

![image](https://drive.google.com/uc?export=view&id=1VI5S8Iod3gUluMHvCAEbrt6m1BYeFpPd)

![image](https://drive.google.com/uc?export=view&id=1De9EwUqogvhw2NSvAqCnTqUfJyNAMXU0)

![image](https://drive.google.com/uc?export=view&id=1tsjMQ4OfWA5O4yD4I9Pkz46ytxHQjlab)

### Dimensionamento do filtro

Apertar o botão 'Filter Design' na janela inicial possibilita a execução de um algoritmo que retorna um filtro ideal para o sistema, de acordo com os parâmetros escolhidos inicialmente. A curva deste filtro indica uma magnitude máxima para uma dada frequência. Assim, ao exportar os dados gerados, o usuário pode projetar qualquer outro filtro que respeite esses limites.

Para executar o algoritmo (1), é requerido a definição de alguns valores. A curva resultante do algoritmo é apresentada em (2) e se refere ao comportamento do filtro ideal a ser utilizado em tal aplicação. Em (3), são mostradas a ordem e a frequência de corte correspondentes. Por fim, ainda é possível extrair os pontos que geraram o gráfico, além dos valores de ordem e frequência de corte do filtro, ao clicar no botão 'Export data to workspace...' (4).

![image](https://drive.google.com/uc?export=view&id=16KP4EsKe-fhoDDQpidC2C54JmkTxYNaK)

## Desenvolvimento

A ferramenta engloba uma grande quantidade de arquivos, os quais possuem várias linhas de código, referentes à configuração dos objetos presentes nas figuras e também aos algoritmos desenvolvidos. Anteriormente todos os arquivos já foram disponibilizados e abaixo segue como a ferramenta está estruturada. Por isso, adiante são apresentados apenas alguns tópicos que são pontos chaves da aplicação e necessitaram de uma atenção especial.

### Estrutura

```
code_files/
│   
└───main: primeira janela, onde é definida a planta a ser controlada
│    └───about: janela que contêm informações sobre a ferramenta
│
└───controlador: janela da análise de estabilidade em função dos parâmetros do controlador
│   └───config_plot_control: janela de configuração dos eixos do gráfico
│   │
│   └───config_slider: janela de configuração dos parâmetros dos sliders
│   │
│   └───controlador_salvar: rotina que salva uma captura do gráfico
│   │
│   └───plotar: rotina de criação do gráfico (domínio de estabilidade em conjunto com diagrama de Nyquist)
│   
└───filtro: janela do dimensionamento de um filtro ideal
│   
└───inequacao: implementação da inequação referente ao controlador repetitivo
```

### Compartilhamento de dados entre arquivos

Sabendo da estrutura da ferramenta, por vezes uma informação adquirida em uma seção é necessária em uma outra. Por exemplo, a planta que se deseja controlar é definida logo na primeira janela do aplicativo, mas é necessária em todas as outras. A estratégia utilizada para compartilhar dados foi a de associar dados a um componente específico usando a função 'setappdata', onde mais tarde é possível acessá-los usando a função 'getappdata'.

```matlab
% Em main.m é criada uma estrutura que armazenará os dados da planta escolhida
    data = struct('ganho', 0, 'num', [], 'den', [], 'ts', 0);
% Associa-se a estrutura criada à figura que compõe a janela principal (fig_main),  
% onde 'PlantData' é o nome identificador escolhido
    setappdata(handles.fig_main, 'PlantData', data);
%%%%%
% Ainda em main.m, após a definição da planta, os valores de 'ganho', 'num', 'den' e 'ts' definidos 
% pelo usuário são salvos na estrutura criada
%%%%%
% Primeiro recupera-se os dados salvos no objeto fig_main com nome 'PlantData'
    PlantData = getappdata(handles.fig_main, 'PlantData');
% Salva por cima os dados que foram obtidos através do usuário
    PlantData.ganho = ganho;
    PlantData.num = num;
    PlantData.den = den;
    PlantData.ts = ts;
% Armazena o conteúdo no objeto fig_main com nome 'PlantData'
    setappdata(handles.fig_main, 'PlantData', PlantData);
%%%%%
% Agora, os parâmetros da planta são acessíveis em qualquer arquivo, desde que as propriedades da figura 
% que compõem a janela principal (fig_main) sejam carregadas. Segue script utilizado em filtro.m, onde é 
% necessário conhecer os dados da planta para executar o seu algoritmo
%%%%%
% Localiza-se objeto de nome 'fig_main' e recupera-se os seus dados
    GUI = findobj(allchild(groot), 'flat', 'Tag', 'fig_main');
    handlesGUI = guidata(GUI);
% Salva os parâmetros da planta em outras variáveis para que se possa usar ao longo do algoritmo
    PlantData = getappdata(handlesGUI.fig_main, 'PlantData');
    ganho = PlantData.ganho;
    num = PlantData.num;
    den = PlantData.den;
    ts = PlantData.ts;
```

### Gráfico de estabilidade

O gráfico de estabilidade é a composição do domínio de estabilidade do controlador com o diagrama de Nyquist da planta, além de apresentar até que frequência o diagrama está contido no domínio de estabilidade.

```matlab
% Em controlador.m são obtidos os valores dos parâmetros 'a' e 'Q', seja por meio dos sliders ou por meio 
% das caixas de texto. Com isso, a função 'plotar' é chamada para gerar o gráfico do diagrama de Nyquist 
% em conjunto com a região de estabilidade do controlador

function plotar(a_value, q_value, handles, salvar)

% Localiza-se objeto de nome 'fig_main' e recupera-se os seus dados
    GUI1 = findobj(allchild(groot), 'flat', 'Tag', 'fig_main');
    handlesGUI1 = guidata(GUI1);
    
% Dados enviados quando a função foi chamada
    a = a_value; 
    q = q_value; 
    
% Recupera dados referentes ao diagrama de Nyquist a partir do compartilhamento de dados entre arquivos
    NyquistData = getappdata(handlesGUI1.fig_main, 'NyquistData');
    re = NyquistData.real;
    im = NyquistData.imag;
    freq = NyquistData.freq;
    
% Definição dos limites do gráfico a partir das configurações definidas em config_plot_control.m
    lim_x = get(handles.plot_estabilidade, 'XLim');
    lim_y = get(handles.plot_estabilidade, 'YLim');
    r = min(lim_x(1), lim_y(1)):0.01:max(lim_x(2), lim_y(2));
    [X, Y] = meshgrid(r);
    
% Definição da região de estabilidade
    a = real(a);
    q = abs(q)^2;
    cond = inequacao(a, q, X, Y);
    
% Rotina para descobrir até onde o diagrama de Nyquist pertence ao domínio de estabilidade. 
% No final, os vetores 're_stable', 'im_stable' e 'last_w' contêm este limite
    re_stable = [re(1)];
    im_stable = [im(1)];
    last_w = [freq(1)];
    for i = 2:1:length(freq)
        cond = inequacao(a, q, re(i), im(i));
        if cond
            re_stable(i) = re(i);
            im_stable(i) = im(i);
            last_w(i) = freq(i);
        else   
            break
        end    
    end
%%%%%
% O restante da função se resume em utilizar o que foi adquirido acima para criar o gráfico
%%%%%
end % end function
```

### Algoritmo de dimensionamento do filtro ideal

```matlab
% Localiza-se objeto de nome 'fig_main' e recupera os seus dados
    GUI1 = findobj(allchild(groot), 'flat', 'Tag', 'fig_main');
    handlesGUI1 = guidata(GUI1);
    
% Recupera dados referentes à planta a partir do compartilhamento de dados entre arquivos    
    PlantData = getappdata(handlesGUI1.fig_main, 'PlantData');
    ganho = PlantData.ganho;
    num = PlantData.num;
    den = PlantData.den;
    ts = PlantData.ts;
% Obtêm pontos do diagrama de Nyquist da função de transferência da planta    
    ft = ganho*tf(num, den, ts);  
    [re, im] = nyquist(ft, 2*pi*freq);
% Variáveis para executar o algoritmo
    len = length(freq);      % Comprimento do vetor frequência
    q = q_init*ones(1, len); % O vetor referente ao parâmetro Q(s) é inicializado como todos os valores 
                             % igual ao escolhido pelo usuário ('q_init')
    flag = 1;                %
    mag_corte1 = 0;          % Variáveis 
    mag_corte2 = 0;          % de
    f_c1 = 10;               % suporte
    f_c2 = 0;                % 

% Laço responsável pelo algoritmo de dimensionamento do filtro    
    for i = 1:1:len
        aux = abs(q(i))^2;
        cond = inequacao(a, aux, re(i), im(i));
        % Caso, para os atuais valores de frequência, 'a' e 'Q', a inequação não seja válida, o valor de 'Q' 
        % é diminuído, no passo escolhido, até que a mesma se torne válida. Só assim, dando prosseguimento 
        % ao algoritmo
        while(~cond)
            q(i:end) = q(i) - q_step;
            aux = abs(q(i))^2;
            cond = inequacao(a, aux, re(i), im(i));
        end
        % Obtêm-se os pontos pelas redondezas da frequência de corte (-3 dB)
        if (flag == 0 && mag2db(q(i)) < -3)
            mag_corte1 = mag2db(q(i-1));
            mag_corte2 = mag2db(q(i));
            f_c1 = freq(i-1);
            f_c2 = freq(i);
            flag = 1;
        end
        % Se entrar na condição anterior com i=1 vai dar problema no i-1. Tratamento utilizado para 
        % o caso em que o primeiro ponto testado já é menor do que -3dB
        if (i==1)
            flag=0; 
        end
    end % end for

% Cálculo do coeficiente angular (decaimento)
    coef_angular = (mag_corte1-mag_corte2)/log10(f_c1/f_c2);
    % O coeficiente angular será igual a zero apenas no caso em que o primeiro ponto testado já é menor 
    % do que -3dB. Já que 'flag' sempre foi igual a 1, as variáveis ainda terão os seus valores iniciais. 
    % Nesse caso, é necessário aumentar o intervalo da frequência para que o algoritmo possa ser executado
    if coef_angular == 0
        errordlg('You must increase the frequency range.','Invalid Input','modal');
        return    
    else
        % Calcula-se a frequência de corte do filtro
        (*$\color{codegreen}Coef. Angular = \frac{x_1 + 3dB}{log(f_1 - Freq.Corte)}$*) 
        freq_corte = 10^(log10(f_c1) - (mag_corte1+3)/coef_angular);
        % A ordem do filtro é encontrada considerando que:
        % Primeira ordem: decaimento de -20dB/década
        % Segunda ordem: decaimento de -40dB/década
        % Terceira ordem: decaimento de -60dB/década
        % ...
        % Por fim, arredonda-se pra cima o valor encontrado e garante-se que o mesmo seja múltiplo de 2.
        filtro_ordem = ceil(coef_angular/-20);
        if mod(filtro_ordem, 2) ~= 0
            filtro_ordem = filtro_ordem + 1;
        end
%%%%%
% O restante do código se resume em utilizar o que foi adquirido para criar o gráfico
%%%%%
    end % end if
```

#### Fluxograma correspondente ao algoritmo

![image](https://drive.google.com/uc?export=view&id=1dmUxKuHSDZMCvmhAw8C760EKo954kxMf)

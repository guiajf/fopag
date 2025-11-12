# Análise da folha de pagamento do município de Três Corações

Alguns anos atrás, utilizei [jq](https://rpubs.com/guilhermeferreirajf/pmtc) para analisar a folha de pagamento da Prefeitura Municipal de Três Corações,
município situado às margens da Fernão Dias, no Sul de Minas. Agora, reproduzimos a mesma rotina em Python.

### Coleta de dados

Para download dos dados, criamos uma função para baixar o arquivo XML, disponível no 
[Portal da Transparência](https://trescoracoes-mg.portaltp.com.br/api/transparencia.asmx/json_servidores),
salvo no formato JSON depois de processado. Poderá ser fornecida uma lista de períodos a serem baixados,
de acordo com o interesse do usuário, ao final do notebbok *baixar_fopag.ipynb*:

```Python
import requests
import re
from datetime import datetime
import time
import os
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def criar_sessao_segura():
    """Cria uma sessão com retry e verificação SSL"""
    session = requests.Session()
    
    # Configura política de retry (3 tentativas com backoff)
    retry = Retry(
        total=3,
        backoff_factor=1,
        status_forcelist=[500, 502, 503, 504]
    )
    
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    
    return session

def baixar_dados_transparencia(periodos, delay_segundos=2):
    base_url = "https://trescoracoes-mg.portaltp.com.br/api/transparencia.asmx/json_servidores"
    
    # Criar sessão segura
    session = criar_sessao_segura()
    
    for i, periodo in enumerate(periodos):
        try:
            # Parse da data no formato mm/aaaa
            data = datetime.strptime(periodo, '%m/%Y')
            mes = data.month
            ano = data.year
            
            # Baixar o arquivo XML com verificação SSL
            params = {'ano': ano, 'mes': mes}
            print(f"Baixando dados para {mes}/{ano}...")
            response = session.get(base_url, params=params, verify=True)  # verify=True é o padrão
            
            # Verificar se a requisição foi bem-sucedida
            response.raise_for_status()
            
            # Nome do arquivo no formato fptc_mmAANO.xml
            nome_arquivo_xml = f'fptc_{mes:02d}{ano}.xml'
            nome_arquivo_json = f'fptc_{mes:02d}{ano}.json'
            
            # Salvar o XML
            with open(nome_arquivo_xml, 'wb') as f:
                f.write(response.content)
            
            # Processar para remover tags e salvar como JSON
            with open(nome_arquivo_xml, 'r', encoding='utf-8') as xml_file:
                content = xml_file.read()
            
            clean_content = re.sub('<[^>]*>', '', content)
            
            with open(nome_arquivo_json, 'w', encoding='utf-8') as json_file:
                json_file.write(clean_content)

            # Remover o arquivo XML após processamento
            os.remove(nome_arquivo_xml)
                
            print(f"Arquivos para {mes:02d}/{ano} processados com sucesso!")
            
            # Adicionar delay entre requisições (exceto após o último item)
            if i < len(periodos) - 1:
                print(f"Aguardando {delay_segundos} segundos...")
                time.sleep(delay_segundos)
            
        except requests.exceptions.SSLError as e:
            print(f"Erro de SSL ao processar {periodo}: {str(e)}")
            print("Tentando com verificação SSL desativada (modo inseguro)...")
            # Fallback para verify=False se o SSL falhar
            try:
                response = session.get(base_url, params=params, verify=False)
                response.raise_for_status()
                # ... resto do processamento ...
            except Exception as fallback_e:
                print(f"Falha mesmo com verify=False: {str(fallback_e)}")
                
        except Exception as e:
            print(f"Erro ao processar {periodo}: {str(e)}")

# Lista de períodos a serem baixados
periodos = [#'11/2022', 
            #'12/2024', 
            #'06/2025',
            #'07/2025',
            '08/2025']

# Chamar a função com delay de 3 segundos entre requisições
baixar_dados_transparencia(periodos, delay_segundos=3)
```

### Dashboard

Criamos um dashboard básico para visualização de algumas métricas e estatísticas baseadas nos dados referentes 
ao período estudado. O notebbok *fopag_dashboard* contém as funções para uma análise simplificada, que pode 
ser expandida, de acordo com as necessidades do usuário. Os principais indicadores analisados são:

*Visão Geral*, *Distribuição Salarial*, *Situação Funcional*, *Natureza do Vínculo*, *Lotação por Secretaria*, *Gasto por Centro de Custo*, *Cargos de Comando*, *Análise de Professores* e *Supersalários*.

O código **jq** convertido para **Python** ficou assim: 

´´Ṕython
import re
import json
import pandas as pd
import numpy as np
import plotly.express as px
import plotly.graph_objects as go
from plotly.subplots import make_subplots
import dash
from dash import dcc, html, Input, Output, callback

# Configurações para formato brasileiro
pd.options.display.float_format = '{:.2f}'.format

# Funções de formatação
def formatar_br(numero):
    return f"{numero:,.0f}".replace(",", ".")

def formatar_moeda(numero):
    return f"R$ {numero:,.2f}".replace(",", "X").replace(".", ",").replace("X", ".")

# Carregar os dados
arquivo = 'fptc_072025.json'

match = re.search(r'fptc_(\d{2})(\d{4})\.json', arquivo)
if match:
    mes_num = match.group(1)
    ano = match.group(2)
    
    # Converte número do mês para nome
    meses = ['Janeiro', 'Fevereiro', 'Março', 'Abril', 'Maio', 'Junho', 
             'Julho', 'Agosto', 'Setembro', 'Outubro', 'Novembro', 'Dezembro']
    mes_nome = meses[int(mes_num) - 1]
    
    periodo = f"{mes_nome} de {ano}"
else:
    periodo = "Período Desconhecido"

# Carrega os dados
with open(arquivo, 'r', encoding='utf-8') as f:
    data = json.load(f)


df = pd.DataFrame(data)

# Pré-processamento
df['valor_rem05'] = pd.to_numeric(df['valor_rem05'])
df_salarios_positivos = df[df['valor_rem05'] > 0]
professores = df[df['cargo'].str.contains('PROF', case=False, na=False)]

# Inicialização do Dash
app = dash.Dash(__name__)
server = app.server  # Importante para Render

# Meta tags para responsividade
app.index_string = '''
<!DOCTYPE html>
<html>
    <head>
        {%metas%}
        <title>{%title%}</title>
        {%favicon%}
        {%css%}
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            /* Estilos CSS para melhor responsividade */
            .js-plotly-plot .plotly .main-svg {
                width: 100% !important;
            }
            .plot-container {
                width: 100% !important;
            }
        </style>
    </head>
    <body>
        {%app_entry%}
        <footer>
            {%config%}
            {%scripts%}
            {%renderer%}
        </footer>
    </body>
</html>
'''

# Layout do dashboard
app.layout = html.Div([
    # Container principal com responsividade
    html.Div([
        html.Div([
            html.H1("Folha de Pagamento", style={'margin': '0', 'fontSize': 'clamp(1.8rem, 2vw, 1.2rem)'}),
            html.H2("PM Três Corações", style={'margin': '0', 'fontSize': 'clamp(1.4rem, 1.8vw, 1.1rem)'}),
        ], style={'textAlign': 'center', 'marginBottom': '20px', 'padding': '0 10px'}),
        
        html.Div([
            dcc.Dropdown(
                id='analise-dropdown',
                options=[
                    {'label': 'Visão Geral', 'value': 'Visão Geral'},
                    {'label': 'Distribuição Salarial', 'value': 'Distribuição Salarial'},
                    {'label': 'Situação Funcional', 'value': 'Situação Funcional'},
                    {'label': 'Natureza do Vínculo', 'value': 'Natureza do Vínculo'},
                    {'label': 'Lotação por Secretaria', 'value': 'Lotação por Secretaria'},
                    {'label': 'Gasto por Centro de Custo', 'value': 'Gasto por Centro de Custo'},
                    {'label': 'Cargos de Comando', 'value': 'Cargos de Comando'},
                    {'label': 'Análise de Professores', 'value': 'Análise de Professores'},
                    {'label': 'Supersalários', 'value': 'Supersalários'}
                ],
                value='Visão Geral',
                style={
                    'width': '100%',
                    'maxWidth': '500px',
                    'margin': '14px auto',
                    'fontSize': '16px'  # Melhor para mobile
                }
            )
        ], style={'textAlign': 'center'}),
        
        html.Div(id='output-graph')
    ], style={
        'maxWidth': '1200px',
        'margin': '0 auto',
        'padding': '15px',
        'fontFamily': 'Arial, sans-serif'
    })
])

def configurar_layout_responsivo(fig, titulo=None):
    """Configura layout responsivo para gráficos Plotly"""
    if titulo:
        fig.update_layout(title=titulo)
    
    fig.update_layout(
        autosize=True,
        margin=dict(l=50, r=50, b=50, t=80 if titulo else 50, pad=4),
        font=dict(size=12),
        height=500,  # Altura fixa que se adapta
        paper_bgcolor='white',
        plot_bgcolor='white'
    )
    
    # Configurações para eixos em mobile
    fig.update_xaxes(tickangle=-45, tickfont=dict(size=10))
    fig.update_yaxes(tickfont=dict(size=10))
    
    return fig

# Callback para atualizar gráficos
@app.callback(
    Output('output-graph', 'children'),
    Input('analise-dropdown', 'value')
)
def update_graph(analise):
    # Aqui você move a lógica de cada análise
    if analise == 'Visão Geral':
        total_servidores = len(df)
        pessoas_unicas = df['nome'].nunique()
        gasto_total = df['valor_rem05'].sum()
        cargos_distintos = df['cargo'].nunique()
    
        fig = go.Figure(data=[go.Table(
            header=dict(
                values=['Métrica', 'Valor'],
                fill_color='lightblue',
                align='left',
                font=dict(size=14, color='black')
            ),
            cells=dict(
                values=[['Total de Servidores', 'Pessoas Únicas', 'Gasto Total', 'Cargos Distintos'],
                       [formatar_br(total_servidores), formatar_br(pessoas_unicas), 
                        formatar_moeda(gasto_total), formatar_br(cargos_distintos)]],
                align='left',
                font=dict(size=12)
            ))
        ])
        
        fig = configurar_layout_responsivo(
            fig, 
            f'Visão geral da Folha de Pagamento<br><sup>Dados de {periodo}</sup>'
        )
        return dcc.Graph(
            figure=fig,
            style={'height': '400px'}  # Altura fixa para tabelas
        )
    
    
    elif analise == 'Distribuição Salarial':
        fig = make_subplots(
            rows=1, 
            cols=2, 
            subplot_titles=['Box Plot', 'Histograma'],
            specs=[[{"type": "box"}, {"type": "histogram"}]]
        )
        fig.add_trace(go.Box(y=df_salarios_positivos['valor_rem05'], name='Salários'), row=1, col=1)
        fig.add_trace(go.Histogram(x=df_salarios_positivos['valor_rem05'], nbinsx=30), row=1, col=2)
        
        fig = configurar_layout_responsivo(
            fig, 
            f'Distribuição Salarial<br><sup>Dados de {periodo}</sup>'
        )
        
        # Layout específico para subplots em mobile
        fig.update_layout(
            height=600  # Mais altura para subplots
        )
    
        return dcc.Graph(figure=fig)
    
    

    elif analise == 'Situação Funcional':
        situacao = df['situacao'].value_counts().reset_index()
        situacao.columns = ['Situação', 'Quantidade']
        situacao = situacao.sort_values('Quantidade', ascending=True).head(20)
        situacao['Quantidade'] = situacao['Quantidade'].apply(lambda x: formatar_br(x))
            
        fig = px.bar(situacao, x='Quantidade', y='Situação', orientation='h')
        fig = configurar_layout_responsivo(
            fig,
            f'Situação funcional dos Servidores<br><sup> Dados de {periodo} <br><br><br><br></sup>'
        )
    
            
        # Ajuste específico para barras horizontais em mobile
        fig.update_layout(
            height=600,  # Mais altura para labels longos
            yaxis={'categoryorder': 'total ascending'}
        )
        
        return dcc.Graph(figure=fig)

        
    elif analise == 'Natureza do Vínculo':
        vinculo = df['regime'].value_counts().reset_index()
        vinculo.columns = ['Regime', 'Quantidade']
        
        # Criar diagrama de Sankey
        fig = go.Figure(go.Sankey(
            node=dict(
                pad=15,
                thickness=20,
                line=dict(color="black", width=0.5),
                label=["Total"] + vinculo['Regime'].tolist(),
                color="blue"
            ),
            link=dict(
                source=[0] * len(vinculo),
                target=list(range(1, len(vinculo)+1)),
                value=vinculo['Quantidade'].tolist(),
                label=[f"{formatar_br(qtd)}" for qtd in vinculo['Quantidade']],
                color=px.colors.qualitative.Pastel
            )
        ))
        
        fig.update_layout(
            title_text=f"Natureza do vínculo empregatício - Diagrama de Sankey <br><sup> Dados de {periodo} </sup>",
            font_size=12,
            height=500
        )
        
        
        return dcc.Graph(figure=fig)
        
    elif analise == 'Lotação por Secretaria':
        lotacao = df['local'].value_counts().reset_index().head(20)
        lotacao.columns = ['Local', 'Quantidade']
        lotacao = lotacao.sort_values('Quantidade', ascending=True).head(20)
        lotacao['Quantidade'] = lotacao['Quantidade'].apply(lambda x: formatar_br(x))
        
        fig = px.bar(lotacao, x='Quantidade', y='Local', orientation='h')
        fig = configurar_layout_responsivo(
            fig,
            f'Top 20 Lotação por Secretaria<br><sup>Dados de {periodo}</sup>'
        )
    
        # Ajuste específico para barras horizontais em mobile
        fig.update_layout(
            height=600,  # Mais altura para labels longos
            yaxis={'categoryorder': 'total ascending'}
        )
    
        return dcc.Graph(figure=fig)
        
    elif analise == 'Gasto por Centro de Custo':
        despesa_cc = df.groupby('centro_custo')['valor_rem05'].sum().reset_index()
        despesa_cc.columns = ['Centro de Custo', 'Total']
        despesa_cc = despesa_cc.sort_values('Total', ascending=True).head(20)
        despesa_cc['Total'] = despesa_cc['Total'].apply(lambda x: formatar_moeda(x))
        
        fig = px.bar(despesa_cc, x='Total', y='Centro de Custo', orientation='h')
        fig = configurar_layout_responsivo(
            fig,
            f'Top 20 Gasto por Centro de Custo <br><sup> Dados de {periodo} <br><br></sup>'
        )
                     
        #fig.show()
        # Ajuste específico para barras horizontais em mobile
        fig.update_layout(
            height=600,  # Mais altura para labels longos
            yaxis={'categoryorder': 'total ascending'}
        )
    
        return dcc.Graph(figure=fig)
        
    elif analise == 'Cargos de Comando':
        cargos_comando = [
            "SECRETARIO MUNICIPAL", "SECRETARIO ADJUNTO", "SUBSECRETARIO",
            "DIRETOR", "CHEFE", "COORDENADOR", "GERENTE", "ASSESSOR",
            "SUPERINTENDENTE", "SUPERVISOR"
        ]
        
        comando_counts = {}
        for cargo in cargos_comando:
            count = df[df['cargo'].str.contains(cargo, case=False, na=False) & 
                       (df['situacao'] == 'Ativo')].shape[0]
            comando_counts[cargo] = count
            
        df_comando = pd.DataFrame.from_dict(comando_counts, orient='index', columns=['Quantidade'])
        df_comando = df_comando[df_comando['Quantidade'] > 0].sort_values('Quantidade', ascending=True)

        fig1 = go.Figure(go.Table(
            header=dict(values=['Cargo', 'Quantidade'],
                        fill_color='lightgreen',
                        align='left'),
            cells=dict(values=[df_comando.index, df_comando['Quantidade']],
                       align='left'))) 
        fig1.update_layout(title=f'<sup> Dados de {periodo} </sup>', height=320)

        
        fig2 = px.bar(df_comando, x=df_comando.index, y='Quantidade',
                     title=f'Quantidade por Cargo de Comando <br><sup> Dados de {periodo} <br><br></sup>',
                     width=800, height=500)

        
        # Retorna ambas
        fig1 = configurar_layout_responsivo(fig1, "Tabela de Cargos de Comando")
        fig2 = configurar_layout_responsivo(fig2, "Quantidade por Cargo de Comando")
        
        # Container responsivo para múltiplos gráficos
        return html.Div([
            dcc.Graph(
                figure=fig1,
                style={'height': '400px', 'marginBottom': '20px'}
            ),
            dcc.Graph(
                figure=fig2, 
                style={'height': '500px'}
            )
        ])
        
    elif analise == 'Análise de Professores':
        # Estatísticas dos professores
        salarios_prof = professores[professores['valor_rem05'] > 0]['valor_rem05']
        estatisticas_prof = {
            'Média': salarios_prof.mean(),
            'Mediana': salarios_prof.median(),
            'Desvio Padrão': salarios_prof.std(),
            'Mínimo': salarios_prof.min(),
            'Máximo': salarios_prof.max()
        }
        
        df_estat_prof = pd.DataFrame.from_dict(estatisticas_prof, orient='index', columns=['Valor'])
        df_estat_prof['Valor'] = df_estat_prof['Valor'].apply(lambda x: formatar_moeda(x))
        
        fig1 = go.Figure(go.Table(
            header=dict(values=['Estatística', 'Valor'],
                        fill_color='lightgreen',
                        align='left'),
            cells=dict(values=[df_estat_prof.index, df_estat_prof['Valor']],
                       align='left')))
        
        fig1.update_layout(title=f'Estatísticas salariais - Professores <br><sup> Dados de {periodo} </sup>', height=300)
        
        # Distribuição salarial dos professores
        fig2 = px.histogram(professores[professores['valor_rem05'] > 0], x='valor_rem05', nbins=30,
                           title=f'Distribuição salarial dos Professores <br><sup> Dados de {periodo} </sup>',
                           width=800, height=500)
        
        
        # Retorna ambas
        fig1 = configurar_layout_responsivo(fig1, "Estatísticas salariais - Professores")
        fig2 = configurar_layout_responsivo(fig2, "Distribuição salarial dos Professores")
        
        # Container responsivo para múltiplos gráficos
        return html.Div([
            dcc.Graph(
                figure=fig1,
                style={'height': '400px', 'marginBottom': '20px'}
            ),
            dcc.Graph(
                figure=fig2, 
                style={'height': '500px'}
            )
        ])
        
    elif analise == 'Supersalários':
        top_20 = df.sort_values('valor_rem05', ascending=False).head(20)
        top_20['valor_formatado'] = top_20['valor_rem05'].apply(lambda x: formatar_moeda(x))

        # Criar IDs anônimos sequenciais
        top_20['id_anonimo'] = ['Servidor ' + str(i+1) for i in range(len(top_20))]

        fig = px.bar(top_20, x='id_anonimo', y='valor_rem05', 
                     hover_data=['cargo', 'local', 'valor_formatado'])
        fig = configurar_layout_responsivo(
            fig,
            f'Top 20 maiores salários <br><sup> Dados de {periodo} </sup>')
        fig.update_layout(xaxis_title='Servidor',
                         yaxis_title='Salário',
                         xaxis={'categoryorder':'total descending'})
        
        # Ajuste específico para barras horizontais em mobile
        fig.update_layout(
            height=600,  # Mais altura para labels longos
            yaxis={'categoryorder': 'total ascending'}
        )
    
        return dcc.Graph(figure=fig)

if __name__ == '__main__':
    app.run(debug=False, host='0.0.0.0', port=8050)

```

Os gráficos podem ser selecionados em uma caixa **dropdown**. 
A imagem de abertura é a seguinte:
![](fopag.png)

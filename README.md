# Análise da folha de pagamento do município de Três Corações

Alguns anos atrás, utilizei [jq](https://rpubs.com/guilhermeferreirajf/pmtc) para analisar a folha de pagamento da Prefeitura Municipal de Três Corações,
município situado às margens da Fernão Dias, no Sul de Minas. Agora, reproduzimos a mesma rotina em Python.

### Coleta de dados

Para download dos dados, criamos uma função para baixar o arquivo XML, disponível no 
[Portal da Transparência](https://trescoracoes-mg.portaltp.com.br/api/transparencia.asmx/json_servidores),
salvo no formato JSON depois de processado. Poderá ser fornecida uma lista de períodos a serem baixados,
de acordo com o interesse do usuário, ao final do notebbok *baixar_fopag.ipynb*. 

### Dashboard

Criamos um dashboard básico para visualização de algumas métricas e estatísticas baseadas nos dados referentes 
ao período estudado. O notebbok *fopag_dashboard* contém as funções para uma análise simplificada, que pode 
ser expandida. Os principais indicadores analisados são:

*Visão Geral*, *Distribuição Salarial*, *Situação Funcional*, *Natureza do Vínculo*, *Lotação por Secretaria*, *Gasto por Centro de Custo*, *Cargos de Comando*, *Análise de Professores* e *Supersalários*.

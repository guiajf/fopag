# Análise da folha de pagamento do município de Três Corações

Alguns anos atrás, utilizei [jq](https://rpubs.com/guilhermeferreirajf/pmtc) para analisar a folha de pagamento da Prefeitura Municipal de Três Corações,
município situado às margens da Fernão Dias, no Sul de Minas.

Agora, reproduzimos a rotina em Python.

### Coleta de dados

Para download dos dados, criamos uma função para baixar o arquivo XML, disponível no 
[Portal da Transparência](https://trescoracoes-mg.portaltp.com.br/api/transparencia.asmx/json_servidores),
salvo no formato JSON depois de processado.

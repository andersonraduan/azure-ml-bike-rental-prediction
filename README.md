# Previsão de aluguel de bicicletas com Azure Machine Learning

Esse é o meu projeto do bootcamp Azure AI Fundamentals da DIO. A ideia era treinar um modelo no Azure ML que prevê quantas bicicletas vão ser alugadas num dia, usando o AutoML, e depois colocar o modelo num endpoint pra testar.

## Como eu fiz

### 1. Workspace

Criei um workspace de Azure Machine Learning no portal do Azure (chamei de DioMEProject, grupo de recursos DioME, região Germany West Central) e depois abri o studio em ml.azure.com.

### 2. Dados

Baixei o zip em https://aka.ms/bike-rentals e descompactei. Dentro vem uma pasta com o CSV dos aluguéis e um arquivo chamado MLTable. Os dados são do Capital Bikeshare e têm coisas como dia, mês, estação, feriado, temperatura, umidade, vento e o número de aluguéis (a coluna `rentals`).

Aqui eu errei no começo: tentei subir só o CSV e deu o erro `ScriptExecution.StreamAccess.NotFound`. Quando o tipo do dado é Tabela (mltable), tem que subir a pasta inteira, com o arquivo MLTable junto.

### 3. Job de AutoML

No studio fui em ML automatizado e criei um trabalho novo:

- nome do trabalho: mslearn-bike-automl
- experimento: mslearn-bike-rental
- tarefa: Regressão
- dado: tabela mltable, subindo a pasta que baixei
- coluna de destino: rentals
- métrica principal: NormalizedRootMeanSquaredError
- modelos permitidos: só RandomForest e LightGBM, pra treinar mais rápido
- validação: train-validation split com 10%
- computação: serverless, CPU, 1 instância

A VM Standard_DS3_v2 que o tutorial pede não estava disponível na minha assinatura, então escolhi outra da lista. O treino demorou uns 8 minutos.

### 4. Resultado do treino

O melhor modelo foi um VotingEnsemble. As métricas:

- R²: 0,788
- NRMSE: 0,0895
- erro médio (MAE): 185,8 aluguéis por dia
- RMSE: 304,9
- Spearman: 0,922

O modelo não bateu o limite de 0,085 que o tutorial coloca, mas ficou perto. No gráfico predicted_true dá pra ver que ele acerta bem nos dias normais e erra mais nos dias de pico.

### 5. Deploy

Na página do melhor modelo cliquei em Implantar > Terminal em tempo real (real-time endpoint) com isso:

- VM: Standard_D2as_v4
- instâncias: 1
- endpoint novo, chamado diomeproject-eugmw
- autenticação por chave
- coleta de dados e empacotar modelo desligados

Esperei uns minutos até ficar Bem-sucedido, com 100% do tráfego na implantação.

### 6. Teste

Na aba Testar colei esse JSON:

```json
{
  "input_data": {
    "columns": ["day", "mnth", "year", "season", "holiday", "weekday", "workingday", "weathersit", "temp", "atemp", "hum", "windspeed"],
    "index": [0],
    "data": [[1, 1, 2022, 2, 0, 1, 1, 2, 0.3, 0.3, 0.3, 0.3]]
  }
}
```

E o modelo respondeu:

```json
[386.60207883868753]
```

Ou seja, previu uns 387 aluguéis nesse dia.

### 7. Limpeza

Depois do teste excluí o endpoint, porque a VM cobra por hora enquanto ele existir.

## Problemas que apareceram

1. **Erro NotFound ao criar o dado**: era o CSV solto sem o MLTable. Resolvi subindo a pasta inteira.
2. **Falta de cota no deploy**: pedi 3 instâncias da D2as_v4 e minha cota era de só 4 núcleos. Baixei pra 1 instância.
3. **Erro SubscriptionNotRegistered no deploy**: a mensagem dizia que um provedor de recursos não estava registrado, mas o nome vinha como [N/A], então não dava pra saber qual era. Registrei vários de uma vez pelo Cloud Shell (Microsoft.Network, Microsoft.Compute, Microsoft.PolicyInsights, Microsoft.Cdn e Microsoft.ContainerService) e o deploy passou na segunda tentativa. Tive que excluir o endpoint que tinha falhado antes de tentar de novo.
4. **Network Error no teste**: eu tinha testado antes da implantação terminar. Esperei ficar Bem-sucedido e funcionou.
5. **Resultado vazio**: eu estava usando o exemplo em branco do studio (com `data: []`). Colei o JSON com os dados e deu certo.

## Arquivos

- README.md: esse arquivo
- endpoint-config.json: configuração do endpoint, métricas e um exemplo de requisição e resposta

## Fonte

Segui o lab da Microsoft: https://microsoftlearning.github.io/mslearn-ai-fundamentals/Instructions/Labs/01-machine-learning.html

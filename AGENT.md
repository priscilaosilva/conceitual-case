Esse é um repositório que será usado para criar um case de estudos no arquivo `case_concentual-priscilaoliveira.md` para apresentação em uma entrevista em nível de especialista de backend em engenharia de software.

Faça um novo documento considerando as perguntas e tópicos citados no documento `case-priscilaosilva.md` e faça uma apresentação em markdown do que está sendo proposto nessa solução

O desenho de solução proposto é o `desenho_solucao.png`

O exemplo de plataforma utilizada foi o fluxo de avaliação de fraudes em transações de cartão de crédito. A arquitetura propõe uma entrada via route 53 > cloudfront com waf (web application firewall) > API Gateway > Lambda authorizer para identificar se o token JWT é válido, se tem acesso a rota de entrada, se o perfil existe e cabe no contexto. A solução também propõe o uso do AWS cognito.

Uma vez que o acesso é válidado, é iniciado um fluxo sincrono e de baixa latência que se inicia em uma lambda responsável por receber a requisição, fazer o invoke de outras 3 lambdas que retornará informações de: perfil do cliente com saldo, score e histórico, informações de dispositivos e estabelecimentos e informações de transações em concorrencia. Todas essas informações devem ser avaliadas para uma tomada de decisão rápida. Os timeouts desse fluxo são baixos para que a resposta ao usuário aconteça em até no máximo 500ms. Os dados são utilizados de uma camada quente com elasticache Redis usando projections e a base de dados histórica será um DynamoDB. Fallbacks de busca de dados no Dynamodb em caso de falta da informação no redis serão ponderados para que a transação não ultrapasse o limite proposto. Nesses casos também consideraremos fallbacks por componentes.

Com base nessas informações, será tomada alguma decisão sobre a transação: aprovada, suspeita ou negada.

Em caso de falha total no fluxo, é retornado o status de negado por indisponibilidade.

Em caso de transação aprovada, o retorno é feito e as informações são persistidas no banco de dados DynamoDB e atualizado na camada cache com o Redis.

Em caso de transação negada ou suspeita (aprovada com atenção), será retornado ao usuário com o status de aprovado e entrará num fluxo de análise profunda do caso.

Nesse fluxo assincrono de análise profunda usaremos uma lambda que fará orquestração de agentes AI (LLMs) e esses agentes terão acesso a uma lambda "API - tool layer" que fornecerá apenas as informações necessárias para análise dos agentes das informações ja obtidas no fluxo síncrono somados as informações brutas da transação atual e com base nesse caso, serão tomadas algumas ações como por exemplo: abrir ticket para análise do analista de fraude, identificar realmente a fraude e fazer levantamento do que é suspeito. Além disso, o modelo retornará sugestões de ação e com isso, novos prompts serão feitos como evolução para serem usados em deploy de novos agentes.

Todas as informações dessa análise profunda deve ser gravadas em um data lake, nessa solução foi sugerido o S3, para que novas interpretações possam ser feitas e os agentes possam ter ainda mais conhecimento do contexto.

A ideia da arquitetura é usar um modelo CQRS usando dynamodb como write e elasticache redis como read.

A ideia dos fallbacks devem apoiar uma ideia de SAGA pattern.

A solução propõe também ferramentas de observabilidade como cloudtrail para interceptação das transações, cloudwatch para logs, métricas e alarmes, x-ray para trace das aplicações, e o datadog para uma visão panoramica do ambiente e talvez insights e tomada de ação rapida como abertura de incidentes para área de tecnologia.

Para stacks consideramos .NET para a camada síncrona pela forte tipagem e segurança e, na camada assíncrona, o uso de python pela ótima perfomance em analise de dados.

Outras informações adicionadas no arquivo `contextosimportantes.md` devem ser consideradas.
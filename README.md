## Grafana Loki + Fluent-bit

# **Conteúdo**

O que é e porque usar

Modelo de deploy

Descrição dos Componentes

Configuração

Integração com fluent-bit

Integração com Grafana

Integração com s3

Quick deploy

### **O que é e porque usar?**

Grafana Loki é um agregador de logs projetado para ser bem econômico, facilmente escalável e inspirado prometheus. Ele não indexa o conteúdo dos logs, mas sim um conjunto de rótulos para cada fluxo de log.

O Loki adota uma abordagem única ao indexar apenas os metadados em vez do texto completo das linhas de log:

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/45cfccca-55c1-4326-92bb-c4a263cb4b0e/Untitled.png)

Com isso, há um ganho enorme de tempo desde sua entrada, processamento, envio para um storage (s3, blob, minIO…) e futuramente consultas utilizando sua linguagem LogQL, todo esse ciclo de vida é feito de forma mais rápida.

### **Modelos de deploy**

Anteriormente havíamos utilizando docker-compose como nosso modelo de deploy, contudo, para padronizar o modelo com os modelos das demais ferramentas da nossa stack, optamos com realizar o deploy via helm chart.

Dentre as opções de arquitetura encontradas na documentação, escolhemos a de microsserviços, onde cada componente do Grafana Loki é gerenciado separadamente, possuindo suas próprias regras de escalonamento.  [Deployment modes |  Grafana Loki documentation](https://grafana.com/docs/loki/latest/fundamentals/architecture/deployment-modes/)

### **Modo de microsserviços**

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/387147b9-e06f-40d5-a7e7-bada156fcf38/Untitled.png)

Arquitetura: Microservices Mode

### **Descrição dos Componentes**

**Distributor**: Recebem logs dos coletores(Fluentd, Fluent-bit, Logstash, Pormtail, etc), valida a exatidão dos mesmos e divide os logs em lote para enviá-los aos **ingesters**.

### **Limitação de taxa**

O **distribuidor** também pode limitar a taxa de logs de entrada com base na taxa de bits máxima por locatário. Ele faz isso verificando um limite por locatário e dividindo-o pelo número atual de distribuidores. Isso permite que o limite de taxa seja especificado por locatário no nível do cluster e nos permite dimensionar os distribuidores para cima ou para baixo e ajustar o limite por distribuidor de acordo. Por exemplo, digamos que temos 10 distribuidores e o tenant 1 tem um limite de taxa de 10 MB. Cada distribuidor permitirá até 1 MB/segundo antes de limitar. Agora, digamos que outro grande tenant se junte ao cluster e precisamos criar mais 10 distribuidores. Os agora 20 distribuidores irão ajustar seus limites de tarifa para o tenant 1 para `(10MB / 20 distributors) = 500KB/s` É assim que os limites globais permitem uma operação muito mais simples e segura do cluster Loki.

**Ingester**: Responsável por enviar os blocos de logs recebidos para o back-end de armazenamento de longo prazo, como S3, os ingesters validam os carimbos de data e hora para cada linha de log recebida e mantém uma ordem estrita. Quando ocorre uma liberação para um provedor de armazenamento persistente, o fragmento recebe uma hash com base em seu locatário, rótulos e conteúdo. Isso significa que vários ingestores com a mesma cópia de dados não gravarão os mesmos dados no armazenamento de backup duas vezes.

Os Ingesters contêm um *lifecycle* que gerencia o ciclo de vida de um Ingester no anel de hash (nas configurações, observaremos que em ring(anel) podemos encontrar todos os componentes que fazem parte do cluster Loki). Cada ingester tem um estado de `PENDING`, `JOINING`, `ACTIVE`, `LEAVING`, ou `UNHEALTHY`:

**Obsoleto: o WAL (registro de gravação antecipada) substitui esse recurso**

1. `PENDING`é o estado de um Ingester quando está aguardando uma transferência de outro Ingester que é `LEAVING`.
2. `JOINING`é o estado de um Ingester quando ele está inserindo seus tokens no anel e se inicializando. Ele pode receber solicitações de gravação para tokens que possui.
3. `ACTIVE`é o estado de um Ingester quando ele é totalmente inicializado. Ele pode receber solicitações de gravação e leitura de tokens que possui.
4. `LEAVING`é o estado de um Ingester quando ele está sendo desligado. Ele pode receber solicitações de leitura de dados que ainda tem na memória.
5. `UNHEALTHY`é o estado de um Ingester quando ele falhou na pulsação para o Consul. `UNHEALTHY`é definido pelo distribuidor quando verifica periodicamente o anel.

**Querier**: Trata das consultas realizadas via LogQL, primeiro ele faz uma buscar nos ingesters, consultando os dados de carimbo de data e hora e os conjuntos de rótulos das mensagens dos logs, após isso faz o mesmo com o armazenamento back-end de longo prazo, evitando que haja dados duplicados nas consultas.

**Querier Front-End**: Divide consultas maiores em várias consultas menores, executando essas consultas em paralelo em consultas downstream e reunindo os resultados novamente. O front-end de consulta também oferece suporte ao armazenamento em cache de resultados de consulta de métrica e os reutiliza em consultas subsequentes. O cache é armazenado no backend de cache do Loki (atualmente Memcached, Redis e um cache na memória).

**Ruler**: Componente que trabalha diretamente com o AlertManager, sua configuração é bem similar, contudo as regras escritas nele são baseadas em LogQL, exemplo:

```
groups:
2  - name: should_fire
3    rules:
4      - alert: HighPercentageError
5        expr: |
6          sum(rate({app="foo", env="production"} |= "error" [5m])) by (job)
7            /
8          sum(rate({app="foo", env="production"}[5m])) by (job)
9            > 0.05
10        for: 10m
11        labels:
12            severity: page
13        annotations:
14            summary: High request latency

```

### **Quick Deploy**

1. Deploy Grafana Loki

```
$helm install -f values.yaml loki grafana/loki-distributed -n <NAMESPACE>
```

```
NAME: loki
LAST DEPLOYED: Thu Dec 22 14:54:54 2022
NAMESPACE: sre-logging
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
***********************************************************************
 Welcome to Grafana Loki
 Chart version: 0.65.1
 Loki version: 2.6.1
***********************************************************************
Installed components:
* ingester
* distributor
* querier
* query-frontend
* compactor

```

Conferindo criação

```
$kubectl -n <NAMESPACE> get all
```

```
1NAME                                                        READY   STATUS    RESTARTS   AGE
2pod/loki-loki-distributed-compactor-65f5fbf6c4-2q4l5        1/1     Running   0          83s
3pod/loki-loki-distributed-distributor-5b89c8df8b-gswzg      1/1     Running   0          83s
4pod/loki-loki-distributed-distributor-5b89c8df8b-vr2nc      1/1     Running   0          83s
5pod/loki-loki-distributed-ingester-0                        1/1     Running   0          83s
6pod/loki-loki-distributed-ingester-1                        1/1     Running   0          83s
7pod/loki-loki-distributed-querier-0                         1/1     Running   0          83s
8pod/loki-loki-distributed-query-frontend-798ccbff64-8nslk   1/1     Running   0          83s
9
10NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
11service/loki-loki-distributed-compactor           ClusterIP   10.43.31.202    <none>        3100/TCP                     83s
12service/loki-loki-distributed-distributor         ClusterIP   10.43.56.100    <none>        3100/TCP,9095/TCP            83s
13service/loki-loki-distributed-ingester            ClusterIP   10.43.31.58     <none>        3100/TCP,9095/TCP            83s
14service/loki-loki-distributed-ingester-headless   ClusterIP   None            <none>        3100/TCP,9095/TCP            83s
15service/loki-loki-distributed-memberlist          ClusterIP   None            <none>        7946/TCP                     83s
16service/loki-loki-distributed-querier             ClusterIP   10.43.143.141   <none>        3100/TCP,9095/TCP            83s
17service/loki-loki-distributed-querier-headless    ClusterIP   None            <none>        3100/TCP,9095/TCP            83s
18service/loki-loki-distributed-query-frontend      ClusterIP   10.43.247.114   <none>        3100/TCP,9095/TCP,9096/TCP   83s
19
20NAME                                                   READY   UP-TO-DATE   AVAILABLE   AGE
21deployment.apps/loki-loki-distributed-compactor        1/1     1            1           83s
22deployment.apps/loki-loki-distributed-distributor      2/2     2            2           83s
23deployment.apps/loki-loki-distributed-query-frontend   1/1     1            1           83s
24
25NAME                                                              DESIRED   CURRENT   READY   AGE
26replicaset.apps/loki-loki-distributed-compactor-65f5fbf6c4        1         1         1       83s
27replicaset.apps/loki-loki-distributed-distributor-5b89c8df8b      2         2         2       83s
28replicaset.apps/loki-loki-distributed-query-frontend-798ccbff64   1         1         1       83s
29
30NAME                                              READY   AGE
31statefulset.apps/loki-loki-distributed-ingester   2/2     83s
32statefulset.apps/loki-loki-distributed-querier    1/1     83s

```

2. Deploy Fluent-Bit

```
1$helm -f output_loki_values_1.yml install fluent-bit fluent/fluent-bit -n <NAMESPACE>
```

```
NAME: fluent-bit
LAST DEPLOYED: Thu Dec 22 14:58:12 2022
NAMESPACE: sre-logging
STATUS: deployed
REVISION: 1

```

Conferindo criação

```
$kubectl -n <NAMESPACE> get all
```

```
NAME                                                        READY   STATUS    RESTARTS   AGE
2pod/fluent-bit-2pxnx                                        1/1     Running   0          107s
3pod/fluent-bit-7n77k                                        1/1     Running   0          107s
4pod/fluent-bit-bzbf9                                        1/1     Running   0          107s
5pod/fluent-bit-crrc6                                        1/1     Running   0          107s
6pod/fluent-bit-kxclc                                        1/1     Running   0          107s
7pod/fluent-bit-p76sj                                        1/1     Running   0          107s
8pod/fluent-bit-phwwl                                        1/1     Running   0          107s
9pod/fluent-bit-w45hg                                        1/1     Running   0          107s
10pod/loki-loki-distributed-compactor-65f5fbf6c4-2q4l5        1/1     Running   0          5m5s
11pod/loki-loki-distributed-distributor-5b89c8df8b-gswzg      1/1     Running   0          5m5s
12pod/loki-loki-distributed-distributor-5b89c8df8b-vr2nc      1/1     Running   0          5m5s
13pod/loki-loki-distributed-ingester-0                        1/1     Running   0          5m5s
14pod/loki-loki-distributed-ingester-1                        1/1     Running   0          5m5s
15pod/loki-loki-distributed-querier-0                         1/1     Running   0          5m5s
16pod/loki-loki-distributed-query-frontend-798ccbff64-8nslk   1/1     Running   0          5m5s
17
18NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
19service/fluent-bit                                ClusterIP   10.43.0.113     <none>        2020/TCP                     107s
20service/loki-loki-distributed-compactor           ClusterIP   10.43.31.202    <none>        3100/TCP                     5m5s
21service/loki-loki-distributed-distributor         ClusterIP   10.43.56.100    <none>        3100/TCP,9095/TCP            5m5s
22service/loki-loki-distributed-ingester            ClusterIP   10.43.31.58     <none>        3100/TCP,9095/TCP            5m5s
23service/loki-loki-distributed-ingester-headless   ClusterIP   None            <none>        3100/TCP,9095/TCP            5m5s
24service/loki-loki-distributed-memberlist          ClusterIP   None            <none>        7946/TCP                     5m5s
25service/loki-loki-distributed-querier             ClusterIP   10.43.143.141   <none>        3100/TCP,9095/TCP            5m5s
26service/loki-loki-distributed-querier-headless    ClusterIP   None            <none>        3100/TCP,9095/TCP            5m5s
27service/loki-loki-distributed-query-frontend      ClusterIP   10.43.247.114   <none>        3100/TCP,9095/TCP,9096/TCP   5m5s
28
29NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
30daemonset.apps/fluent-bit   8         8         8       8            8           <none>          107s
31
32NAME                                                   READY   UP-TO-DATE   AVAILABLE   AGE
33deployment.apps/loki-loki-distributed-compactor        1/1     1            1           5m5s
34deployment.apps/loki-loki-distributed-distributor      2/2     2            2           5m5s
35deployment.apps/loki-loki-distributed-query-frontend   1/1     1            1           5m5s
36
37NAME                                                              DESIRED   CURRENT   READY   AGE
38replicaset.apps/loki-loki-distributed-compactor-65f5fbf6c4        1         1         1       5m6s
39replicaset.apps/loki-loki-distributed-distributor-5b89c8df8b      2         2         2       5m6s
40replicaset.apps/loki-loki-distributed-query-frontend-798ccbff64   1         1         1       5m6s
41
42NAME                                              READY   AGE
43statefulset.apps/loki-loki-distributed-ingester   2/2     5m6s
44statefulset.apps/loki-loki-distributed-querier    1/1     5m6s

```

Após o deploy de ambos, podemos adicionar o Loki como datasource no Grafana e iniciar e exploração de todos os logs enviados, bem como, acompanhar eles em tempo real.
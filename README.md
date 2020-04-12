# Exemplo Cluster MongoDB

O repositório foi criado para mostrar um exemplo de uma configuração completa de um Cluster MongoDB, a configuração conta com:

  - **3** replicas de servidores mongo de configuração
  - **3** servidores de shard com **1** replica cada
  - **1** servidor de roteamento
  - Totalizando **10** aplicações rodando no cluster
 
Após tudo configurado os clientes irão se conectar no servidor de roteamento, que conhece e se comunica com os servidores de configuração que conhecem os servidores de shard, e então as requisições são direcionadas e processadas nos servidores de shard e retornadas pelo servidor de roteamento.
Caso todos os servidores de configuração ficassem indisponíveis o cluster iria parar, por conta disso contamos com 3 replicas do servidor de configuração.
Caso 1 servidor de shard ficasse indisponível, parte dos dados ficariam inacessíveis, para aumentar a disponibilidade contamos com 2 replicas de cada servidor de shard e 3 servidores de shard.
> OBS: O arquivo docker-compose.yaml está com comentários por fins didáticos

## Pré-requisitos

  - Docker 
  - Docker-compose
  
## Tutorial
 - Clone o repositório para sua máquina
```sh
$ git clone https://github.com/drprado2/exemplo-mongodb-shard-cluster.git
```
 - Acesse o path raiz do diretório clonado com o terminal
```sh
$ cd exemplo-mongodb-shard-cluster
```
- Suba o cluster
```sh
$ docker-compose up -d
```
- Configure o replicaSet dos servidores de configuração
```sh
$ docker container exec -it mongo-config-server-1 bash
$ mongo --port 26051
$ rs.initiate(
   {
      _id: "conf",
      configsvr: true,
      members: [
         { _id: 0, host : "mongo-config-server-1:26051" },
         { _id: 1, host : "mongo-config-server-2:26052" },		 
		 { _id: 2, host : "mongo-config-server-3:26053" },
      ]
   }
)
```
>Note que os hosts dos members são os nomes do serviços do docker-compose e as suas respectivas portas, os containers conseguem se comunicar pois estão na mesma network.
- Configure o replicaSet dos servidores de shard **1**
```sh
$ docker container exec -it mongo-shard-server-1 bash
$ mongo --port 28051
$ rs.initiate(
   {
      _id: "rs-shard-1",
      version: 1,
      members: [
         { _id: 0, host : "mongo-shard-server-1:28051", priority: 10 },
         { _id: 1, host : "mongo-shard-server-1-repl:28052", priority: 1 },		 
      ]
   }
)
```
>A propriedade priority é usada para definir qual node será o primary dos replica sets, quanto maior mais chance.
- Configure o replicaSet dos servidores de shard **2**
```sh
$ docker container exec -it mongo-shard-server-2 bash
$ mongo --port 28053
$ rs.initiate(
   {
      _id: "rs-shard-2",
      version: 1,
      members: [
         { _id: 0, host : "mongo-shard-server-2:28053", priority: 10 },
         { _id: 1, host : "mongo-shard-server-2-repl:28054", priority: 1 },		 
      ]
   }
)
```
- Configure o replicaSet dos servidores de shard **3**
```sh
$ docker container exec -it mongo-shard-server-3 bash
$ mongo --port 28055
$ rs.initiate(
   {
      _id: "rs-shard-3",
      version: 1,
      members: [
         { _id: 0, host : "mongo-shard-server-3:28055", priority: 10 },
         { _id: 1, host : "mongo-shard-server-3-repl:28056", priority: 1 },		 
      ]
   }
)
```
- Configure os shards através do servidor de roteamento
```sh
$ docker container exec -it mongo-router-server bash
$ mongo --port 27000
$ sh.addShard("rs-shard-1/mongo-shard-server-1:28051")
$ sh.addShard("rs-shard-1/mongo-shard-server-1-repl:28052")
$ sh.addShard("rs-shard-2/mongo-shard-server-2:28053")
$ sh.addShard("rs-shard-2/mongo-shard-server-2-repl:28054")
$ sh.addShard("rs-shard-3/mongo-shard-server-3:28055")
$ sh.addShard("rs-shard-3/mongo-shard-server-3-repl:28056")
```
>Note que o comando recebe o nome do replicaSet / o nome do node : porta
- Para checar o status dos shards faça
```sh
$ sh.status()
```
- Habilite o seu database para shard
```sh
$ sh.enableSharding ("dataBaseName")
```
- Habilite o shardeamento de uma collection
```sh
$ sh.shardCollection("dataBaseName.collectionName", {indexProperty:1})
```
>note que o segundo parâmetro deve ser um índice pré criado na collection, mais detalhes na [documentação oficial](https://docs.mongodb.com/manual/reference/method/sh.shardCollection/#examples)
- Cheque a distribuição feita da coleção que recebeu shard
```sh
$ db.collectionName.getShardDistribution()
```
>Caso você queira alterar a distribuição dos shards veja mais detalhes na [documentação oficial](https://docs.mongodb.com/manual/tutorial/modify-chunk-size-in-sharded-cluster/)


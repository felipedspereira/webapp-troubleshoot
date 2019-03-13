# webapp-troubleshoot


## Objetivo do Documento
Este documento tem o objetivo de servir como material de apoio na depuração de problemas de lentidão encontrados em aplicações web escritas em Java.
No decorrer deste texto, serão apresentados algumas soluções para monitoramento dos recursos de diversas parte de uma aplicação, tais como Pool de conexões, memória da aplicação, vazamento de memória, entre outros.


* <b>Observação</b>: as configurações aqui apresentadas são relativas ao servidor de aplicação JBoss AS 7.x, porém, apesar de os principais servidores de aplicação do mercado apresentarem suas próprias formas de configuração, não deve fugir muito do que vai ser mostrado aqui.

## Monitorando a aplicação em Produção
Existem diversos aspectos que devem ser monitorados em uma aplicação que esteja apresentando lentidão. Porém, os primeiros aspectos que devem ser avaliados de antemão são o <b>Pool de conexões</b> com a base de dados e a <b>memória disponibilizada para o servidor de aplicação</b>.

### Pool de Conexões
O pool de conexões é importantíssimo para o bom funcionamento de uma aplicação em produção. Basicamente, o pool de conexões representa um conjunto de conexões com o banco de dados que ficarão disponíveis exclusivamente para atender requisições da aplicação em questão. Esse pool de conexões, entre outros parâmetros, possui um número mínimo e máximo de conexões. 

<br>

Se o número máximo de conexões disponibilizadas para a aplicação for muito baixo, a aplicação pode começar a apresentar problemas de lentidão e até mesmo alto consumo de memória do servidor: se todas as conexões estão ocupadas, o servidor de aplicação começará a enfileirar as requisições em sua memória enquanto espera a disponibilização de uma conexão com o banco.

### Recomendação de configurações para o Pool de Conexões
A configuração ideal do pool de conexões vai variar de aplicação para aplicação. O ideal é realizar o monitoramento das conexões em produção visando identificar se existe um gargalo neste ponto. 
No Jboss AS 7.x, é possível realizar o monitoramento das conexões através da ferramenta jboss-cli, localizada dentro da pasta <b>bin</b> do servidor. Modo de uso:

* Digitar <code># ./jboss-cli.sh</code> na linha de comando para executar (ambiente Linux);
* Digitar <code>connect <IP DO SERVIDOR></code>. Por exemplo, se o IP do servidor for 192.168.0.1, digitar <code>connect 192.168.0.1</code>;
* Executar a seguinte instrução: <code> # /subsystem=datasources/data-source=AppDS/statistics=pool:read-resource(include-runtime=true)</code>, substituindo onde está escrito "AppDS" pelo nome do datasource que está configurado direto no servidor de aplicação. 
* <b>Observação</b>: a configuração relatada acima somente funcionará caso o datasource esteja configurado dentro do servidor de aplicação (e não dentro da aplicação).

A execução do comando acima resultará numa saída semelhante a se segue abaixo:

![View_conexoes_jboss](https://user-images.githubusercontent.com/14164532/54299098-89bd4300-4590-11e9-9b49-eb594ffe9a3e.png)


Os parâmetros em destaque acima referem-se aos que considero os mais importantes. São eles:
* <b>ActiveCount</b>: quantidade de conexões atualmente ativas. Esse parâmetro não necessariamente apresenta as conexões em uso, e sim as conexões que estão abertas com o banco de dados. Muitas dessas conexões, apesar de estarem abertas, podem estar ociosas. Quando um pool de conexões é configurado, deve ser especificado um valor mínimo de conexões. O mínimo parametrizado garante a quantidade mínima de conexões em aberto com ao base de dados. Este valor é importante para "economizar" a abertura e encerramento de conexões a todo momento. Ao inicializar o servidor, geralmente o valor do parâmetro ActiveCount estará mostrando a mesma quantidade que foi configurada como minímo do pool.
* <b>AvailableCount</b>: representa o máximo de conexões do pool de conexões.
* <b>AverageBlockingTime</b>: Tempo médio, em milissegundos, de bloqueio esperando para obter uma conexão da base de dados. Se o gargalo na realidade estiver no banco de dados, este valor provavelmente estará bem alto.
* <b>InUseCount</b>: O parâmetro mais importante entre todos. Este valor mostra a quantidade de conexões que de fato que está sendo utilizada. <b>Se este valor estiver igual ao AvailableCount, é um forte indício de que existe um gargalo no pool de conexões, pois todas as conexões disponíveis no pool estão em uso</b>.

## Memória do Servidor de Aplicação
Um servidor que este esteja ficando sem recursos de memória afeta diretamente o uso da aplicação. O atendimento às requisições dos usuários vai se tornando cada vez mais demorado. Se o servidor ficar totalmente sem memória disponível (falando exclusivamente do JBoss), configura-se o cenário típico de "crash" do servidor.

Uma excelente ferramenta para monitorar o estado da memória do servidor de aplicação é o JVisualVM, que pode ser encontrado diretamente na pasta <b>bin</b> do seu JDK. Em ambiente de Produção, essa ferramenta pode ser aberta no servidor em que a aplicação está executando, ou remotamente (mas isso remete à configurações mais avançadas).
Para executar a ferramenta localmente, só é preciso executar o comando <code>./jvisualvm</code>, dentro da pasta <b>bin</b> do JDK. A interface que será apresentada é semelhante à exibida abaixo:

![Visualvm](https://user-images.githubusercontent.com/14164532/54299668-be7dca00-4591-11e9-8722-3d042dd34205.png)


Na imagem acima, o processo do JBoss é o que possui <i>pid 13012</i>. Dois cliques neste processo abrirá uma tela semelhante à tela exibida abaixo:


![Visualvm2](https://user-images.githubusercontent.com/14164532/54299714-d48b8a80-4591-11e9-8f8c-186770db279b.png)



A última aba exibida é a que devemos utilizar. Por padrão, esta aba não será exibida. É necessário instalar um plugin através da opção Tools > Plugins, e escolher a opção Visual GC.
A tela do exibida por este plugin é tal como exibido abaixo:


![Visualvm3](https://user-images.githubusercontent.com/14164532/54299755-e705c400-4591-11e9-8b39-f88762fb0882.png)


Na tela exibida acima, podemos perceber duas grandes seções: uma reservada aos "Spaces" e outra aos "Graphs". Basicamente, a visão fornecida em Graphs corresponde ao histórico do comportamento do Garbage Collector na máquina virtual. A visão mais interessante é a apresentada em "Spaces", a qual mostra a quantidade de memória consumida atualmente nas áreas de memória da máquina virtual, em que:

* <b>Perm</b>: corresponde a área de PermGen Space. Esta área de memória não é diretamente utilizada pela aplicação, mas é usada internamente pela JVM para guardar seus próprios objetos. Caso sua aplicação possua erro de OutOfMemory, uma das causas possíveis é na PermGen (o log de sua aplicação diria algo como <code>OutOfMemory: PermGen space</code>. Neste caso, provavelmente a correção seja trivial: apenas aumentar o tamanho da PermGen space (fora do escopo desta documentação) e avaliar o novo comportamento.
* <b>Old</b>: corresponde a maior área de memória dedicada às aplicações. Os objetos que entram nessa região de memória são denominados como objetos de longa duração, uma vez que já passaram pelas áreas S0, S1 e Eden.
* <b>S0, S1 e Eden</b>: todas estas áreas correspondem à Young Generation o que, na prática, armazena objetos de curta duração de vida. Esta área de memória é varrida em intervalos muito mais curtos de tempo do que a região Old. Os objetos remanescentes da área S0, são copiados para a S1, da mesma forma que os objetos da S1 são copiados para o Eden e estes, por sua vez, são finalmente copiados para a Old Generation.


### Recomendação para configurações de memória do servidor de aplicação

É interessante realizar o monitoramento de memória do servidor de aplicação juntamente com o monitoramento do pool de conexões do banco de dados. Isto, porque caso o pool de conexões atinja seu limite máximo, muito provavelmente o servidor de aplicação começará a "comer" mais memória por conta do enfileiramento de requisições, podendo ocasionar no estouro de memória por conta do pool de conexões muito reduzido. Desta forma, avaliar somente a memória da aplicação, sem ver o que se passa com o pool, poderia dar a falsa sensação de que somente aumentando a memória do servidor resolveria o problema.
<b>Caso seja observado que a Old Generation está alocando memória e não mais liberando-a (cenário em que a Old Generation vai subindo exponencialmente no gráfico acima), muito provavelmente possa haver um vazamento de memória na aplicação </b>.

## Identificando (e tratando) vazamentos de memória na aplicação

Caso exista a suspeita de vazamento de memória na aplicação, a ferramenta JVisualVM, em conjunto com a ferramenta Eclipse Memory Analyzer podem ser muito úteis para ajudar a resolver o problema. 
Através da ferramenta JVisualVM, na aba Monitor, é possível extrair um <b>Heap Dump</b> do servidor, o qual se trata do conjunto de objetos que se encontram na memória do servidor. É interessante observar que o Heap Dump preferencialmente deve ser extraído quando o consumo de memória na aplicação estiver em alta (período antes do servidor cair).
Após obter o Heap Dump do servidor, é a vez da ferramenta Eclipse Memory Analyzer fazer o papel de avaliar o heap e apontar possíveis suspeitas de <i>memory leak</i>. A ferramenta é bastante intuitiva e de fácil uso. Sua utilização está além do escopo desta documentação. A saída gerada pela ferramenta Eclipse Memory Analyzer é semelhante a que segue abaixo:

![Leak](https://user-images.githubusercontent.com/14164532/54299828-13214500-4592-11e9-921a-8e296eee5e0d.png)

## Avaliando o Thread Dump em busca de problemas ===
TODO





</div>

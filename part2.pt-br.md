# Introdução

> "Chegou a hora - disse a Morsa - de falar sobre muitas coisas"
>
>  Lewis Carroll
> "Alice no País das Maravilhas"
>

Tem muitas coisas dessa vez! Primeiro vamos usar um exemplo próximo do dia-a-dia para explicar o modelo de funcionamento do OpenFlow. Compreendendo isso, o conceito básico de OpenFlow estará perfeito. Em seguida, vamos fazer de fato um Controller que faz o "switch com totalizador de tráfego". Ele possui todos os processamentos importantes do OpenFlow, por isso só de adaptá-lo é possível criar Controllers de vários tipos. Por fim, vamos executar o Controller criado na rede virtual do Trema. Para nossa alegria, usando o Trema é possível realizar desde o desenvolvimento até o teste de execução, numa única máquina para desenvolvimento.

Chega de prefácio, e vamos compreender o mecanismo que controla o switch no OpenFlow.


# Modelo de funcionamento do OpenFlow

Se fizer uma analogia, o funcionamento do OpenFlow é parecido com o serviço de suporte por telefone de um produto.


## Procedimento empresarial de suporte por telefone

O garoto Yutaro pensou em mandar o ar-condicionado quebrado para o conserto (figura 1). Ligando ao SAC, a atendente do suporte, srta. Aoi pergunta o sintoma do ar-condicionado, e se a solução constar no manual que ela tem em mãos, ela informa prontamente. O problema é quando a solução não consta no manual. Nessas horas dá um pouco de trabalho, mas ela pergunta ao seu superior, o gerente Miyasaka. Assim que conseguir a resposta do gerente Miyasaka, a srta. Aoi retorna a ligação ao garoto Yutaro. Além disso, para poder responder rapidamente às futuras solicitações semelhantes à esta, a srta. Aoi acrescenta a solução aprendida no manual que tem em mãos.

Fácil, não? Pode não acreditar, mas você praticamente compreendeu 95% do OpenFlow.

![Procedimento empresarial do suporte por telefone](https://github.com/trema/Programming-Trema/raw/master/images/2_001.png)

figura 1 Procedimento empresarial do suporte por telefone

## Substituindo no OpenFlow...

No OpenFlow, o cliente é o host que gera os pacotes, a atendente do suporte por telefone e o switch, o superior é o Controller, e o manual é a tabela de fluxos (citado adiante) do switch (figura 2).

![Modelo de funcionamento do OpenFlow](https://github.com/trema/Programming-Trema/raw/master/images/2_002.png)

figura 2 Modelo de funcionamento do OpenFlow

A princípio, quando o switch recebe o pacote de um host, não sabe como processá-lo. Então ele comunica ao Controller que é o seu superior. Esta comunicação se chama mensagem `packeg_in`. Quando o Controller recebe esta mensagem, decide como processar (transferir o pacote, alterar o pacote, etc.) pacotes semelhantes que receber. Isto se chama action. E o conjunto (chamado flow) "característica do pacote que deve ser processado pelo switch" + "action" é adicionado ao manual do switch. Esta decisão é chamada de mensagem `flow_mod`, e o manual é chamado de flow table. Escrevendo na flow table a característica do pacote a ser processada e a action correspondente, os futuros pacotes com tal caracteristica pode ser processada rapidamente pelo próprio switch. O que não podemos esquecer é o primeiro pacote recebido com a mensagem `packet_in`. Ele subiu até o Controller e está no estado de espera para ser processado, portanto deve ser encaminhado corretamente por uma mensagem `packet_out`.

A grande diferença do suporte por telefone é que, no flow escrito na flow table possui um prazo de validade, e passando desse prazo o flow some. Talvez fique fácil de entender se pensar que "O conteúdo do manual vai ficando obsoleto, portanto as seções antigas devem ser apagadas". No momento em que o flow some, é enviada uma mensagem `flow_removed` ao Controller. Esta mensagem é o registro de quantos pacotes foram transferidos segundo um flow -- no exemplo do suporte por telefone, quantas vezes uma seção do manual foi consultado -- ou seja, é a totalização do tráfego.

Vamos parar de falar sobre o funcionamento por aqui, e vamos logo à prática. Se ficar confuso no meio da leitura, volte a ler a partir do início desta seção.


# Visão geral do "Switch totalizador de tráfego"

O switch otalizador de tráfego, à primeira vista, funciona como um switch comum da camada 2. Mas por trás dos panos, ele conta o tráfego transmitido por cada host, e periodicamente exibe a informação do total. Usando ele, é fácil de determinar qual host que está consumindo os recursos da rede.

## Planejamento e implementação

Quais peças são necessárias para a "funcionalidade do switch da camada 2" e para a "funcionalidade do totalizador de tráfego"? Primeiro precisamos de uma classe Controller correspondendo a um superior que dá instruções ao switch. Vamos chamá-lo de `TrafficMonitor`. Precisamos ainda de uma classe `FDB` (obs. 1) para entregar o pacote á porta do switch destinatário e também da classe `Counter` para totalizar o tráfego. No mínimo precisamos destas 3 classes.

obs. 1) FDB é acrônimo para Forwarding DataBase, uma funcionalidade comum em switches. Os detalhes são explicados durante a implementação a seguir.

### Classe FDB

A classe `FDB` (listas 1) é uma base de dados que aprende a relação entre o endereço MAC de um HOST e a porta do switch à qual o host está conectado. Consultando esta base de dados, é possível determinar a porta do switch destino, a partir do endereço MAC de destino dentro da mensagem `packet_in`.

```ruby
class FDB
	def initialize
		@db = {} # <- dicionário(endereço MAC -> número da porta do switch)
	end

	def lookup mac # <- retorna o número da porta do switch a partir do endereço MAC
		@db[ mac ]
	end

	def learn mac, port_number # <- aprende endereço MAC + porta do switch
		@db[ mac ] = port_number
	end
end
```

lista 1 Classe FDB, base de dados com endererço MAC -> porta do switch (fdb.rb)

### Classe Counter

A classe `Counter` (lista 2) conta a quantidade de pacotes enviados e a quantidade de bytes para cada host (identificando pelo endereço MAC). Além disso, provê um método helper para exibir a informação do total contabilizado.

```ruby
class Counter
	def initialize
		@db = {} # <- dicionário que registra o total para cada host
	end

	def add mac, packet_count, byte_count # <- acrescenta para um host (endereço MAC = mac) a quantidade de pacotes enviados e o número de bytes
		@db[ mac ] || = { :packet_count => 0, :byte_count => 0 }
		@db[ mac][ :packet_count ] += packet_count
		@db[ mac][ :byte_count ] += byte_count
	end

	def each_pair &block # <- Para exibir a informação do total
		@db.each_pair &block
	end
end
```

lista 2 Classe `Counter` que registra e contabiliza o total de tráfego (`counter.rb`)



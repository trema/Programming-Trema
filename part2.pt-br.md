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

No OpenFlow, o cliente é o host que gera os pacotes, a atendente do suporte por telefone e o switch, o superior é o Controller, e o manual é a flow table (citado adiante) do switch (figura 2).

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

### Classe TrafficMonitor

A classe `TrafficMonitor` é a parte principal do Controller (lista 3). São 3 processamentos principais, lista 3-1, 3-2 e 3-3.

1. Um trecho que, quando chega uma mensagem `packet_in`, transfere o pacote à porta destinatária do switch e atualiza a flow table
2. Um trecho que, quando chega uma mensagem `flow_removed`, atualiza a informação do total de tráfego
3. Um trecho que exibe a informação do total de tráfego a cada 10 segundos do timer


```ruby
require "counter"
require "fdb"

class TrafficMonitor < Controller
  periodic_timer_event :show_counter, 10 # (3)

  def start
    @counter = Counter.new # <- objeto Counter
    @fdb = FDB.new # <- objeto FDB
  end

  def packet_in datapath_id, message # (1)
    macsa = message.macsa # <- endereço MAC do host remetente do pacote
    macda = message.macda # <- endereço MAC do host destinatário do pacote

    @fdb.learn macsa, message.in_port
    @counter.add macsa, 1, message.total_len
    out_port = @fdb.lookup( macda )
    if out_port
      packet_out datapath_id, message, out_port
      flow_mod datapath_id, macsa, macda, out_port
    else
      flood datapath_id, message
    end
  end

  def flow_removed datapath_id, message # (2)
    @counter.add message.match.dl_src,message.packet_count, message.byte_count
  end

  private # <- A seguir, métodos privados

  def show_counter # <- exibe o contador
    puts Time.now
    @counter.each_pair do | mac, counter |
      puts "#{ mac } #{ counter[ :packet_count ] } packets (#{ counter[ :byte_count ] } bytes)"
    end
  end

  def flow_mod datapath_id, macsa, macda, out_port # <- lança o flow_mod, que transfere para out_port um pacote do macsa enviado para macda
    send_flow_mod_add(
      datapath_id,
      :hard_timeout => 10, # <- validade do flow_mod é de 10 segundos
      :match => Match.new( :dl_src => macsa, :dl_dst => macda ),
      :actions => Trema::ActionOutput.new( out_port )
    )
  end

  def packet_out datapath_id, message, out_port # <- transfere o pacote que veio por packet_in à out_port
    send_packet_out(
      datapath_id,
      :packet_in => message,
      :actions => Trema::ActionOutput.new( out_port )
    )
  end

  def flood datapath_id, message # <- transfere a mensagem que veio por packet_in a todas as portas exceto in_port
    packet_out datapath_id, message, OFPP_FLOOD
  end
end
```

lista 3 Classe principal `TrafficMonitor` (`traffic-monitor.rb`)

Vamos ver em detalhes o processamento do (1). Para obter detalhes do API, como os parâmetros dos métodos usados na lista 3, vide o "Trema Ruby API Document".

Na explicação abaixo, usaremos uma estrutura de rede constituída por 2 hosts + 1 switch. A sequência de ações consequentes do envio de um pacote do host1 para host2 fica como a figura 4.

![Exemplo de estrutura de rede que roda o TrafficMonitor](https://github.com/trema/    Programming-Trema/raw/master/images/2_003.png)

figura 3 Exemplo de estrutura de rede que roda o TrafficMonitor

![Sequência de ações consequentes do envio de um pacote do host1 para host2](https://github.c    om/trema/Programming-Trema/raw/master/images/2_004.png)

figura 4 Sequência de ações consequentes do envio de um pacote do `host1` para `host2`

1. Quando envia-se um pacote de `host1` destinado ao `host2`, primeiro o pacote chega no switch
2. A flow table do switch é vazia no início, logo ele não sabe o que fazer, e por isso envia uma mensagem `packet_in` ao Controller `TrafficMonitor`
3. O handler `packet_in` do `TrafficMonitor` registra o `in_port` da mensagem (a porta do switch à qual `host1` está conectada) com o endereço MAC do `host1`
4. Além disso, incrementa em 1 pacote o tráfego de envio do `host1`, registrado no `Counter`
5. Pergunta ao FDB o número da porta do switch a partir do endereço MAC do destinatário. Neste ponto, ele não aprendeu a porta do `host2`, portanto o resultado é "Desconhecido"
6. Então, envia uma mensagem (chamada de FLOOD) `packet_out` a todos as portas do switch exceto `in_port`, e espera que o `host2` receba.
7. O switch envia o pacote a todas as portas exceto `in_port`

Assim finalmente `host2` recebe o pacote. Se neste instante, o `host2` envia um pacote destinado ao `host1`, ocorre a seguinte sequência de ações (figura 5). Até a 4ª ação é igual à figura 4, mas a partir da 5ª ação fica diferente.

![Sequência de ações consequentes do envio de um pacote do host1 para host2](https://github.com/trema/Programming-Trema/raw/master/images/2_005.png)

figura 5 Sequência de ações consequentes do envio de um pacote do host1 para host2

1. Quando envia-se um pacote de `host2` destinado ao  `host1`, primeiro o pacote chega no switch
2. A flow table do switch é vazia no início, logo ele não sabe o que fazer, e por isso envia uma mensagem `packet_in` ao Controller `TrafficMonitor`
3. O handler `packet_in` do `TrafficMonitor` registra o `in_port` da mensagem (a porta do switch à qual `host1` está conectada) com o endereço MAC do `host1`
4. Além disso, incrementa em 1 pacote o tráfego de envio do `host1`, registrado no `Counter`
5. Pergunta ao FDB o número da porta do switch a partir do endereço MAC do destinatário. Como o FDB já aprendeu agora há pouco, quando `host1` enviou um pacote ao `host2`, dá para saber que deve ser enviado à porta 1 do switch
6. Então, `TrafficMonitor` envia a mensagem `packet_out` que faz o pacote sair pela porta 1 do switch. Quando o switch recebe esta mensagem, encaminha o pacote pela porta 1 e finalmente `host1` recebe a mensagem
7. Envia uma mensagem `flow_mod` que diz "Encaminhe pacotes com remetente = 00:00:00:00:00:02, destinatário = 00:00:00:00:00:01 para a porta 1 do switch"

Pela última ação 7, os futuros pacotes de `host2` ao `host1` serão processados apenas pelo switch.



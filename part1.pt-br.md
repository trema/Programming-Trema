# Introdução

> Eficiência é ser negligente de forma inteligente (Autor desconhecido)

> Preguiça: A qualidade que leva você a fazer grandes esforços para reduzir o gasto de energia total. (Larry Wall)

Entre as características de um bom programador está a "Preguiça". Como conseguir junk-food sem sair da frente do adorado computador nem por um instante? As pessoas normais pode achar um exagero, mas para os hackers isso sempre foi um grande problema. Por exemplo, no laboratório de IA do MIT, conhecido por ser o recinto dos hackers, existia o comando `xpizza` para pedir uma pizza através de um único comando UNIX (se pesquisar por "xpizza MIT" no Google, você consegue ler o [manpage de 1991](http://bit.ly/mYAJwZ). Além disso, o COFFEE POT MIB publicado como RFC 2325 define uma interface para monitorar a quantia de café numa cafeteira remota, e também para coar o café automaticamente.

Entre esses hacks para "facilitar a vida através do software", o de maior escala é o data-center de grande porte. Os data-centers de grande porte que rodam por trás dos serviços de cloud são operados por um número extremamente reduzido de administradores, e a maior parte da administração é automatizada ao extremo através de softwares, como muitos já devem ter lido em reportagens. O tipo de hack mais divertido que existe é controlar "coisas" através de programas, seja para brincadeiras como pizza e café, ou até mesmo para complexidades como os data-centers de grande porte.


# O surgimento do OpenFlow

Dentre esses hacks, uma das tecnologias criadas para hackear a rede é o OpenFlow, que será abordado nesta série. O OpenFlow define um protocolo que altera o comportamento interno dos switches de rede (informações sobre especificação e padronização do OpenFlow pode ser obtido [aqui](http://www.openflow.org)). Através do software que controla os switches (conhecido como Controller no mundo do OpenFlow), ele busca um mundo onde seja possível controlar toda a rede através do software (figura 1).

![Switch e Controller Openflow](https://github.com/lucioseki/Programming-Trema/raw/master/images/1_001.pt-br.png)

figura 1 Switch e Controller Openflow

Com o surgimento do OpenFlow, a rede, que até então era administrada pelos operadores especializados, finalmente foi liberada aos programadores. Ao descrever a rede por meio do software, automações como "rede que se otimiza sozinho de acordo com a aplicação" ou "rede que se recupera automaticamente depois de uma falha" pode se tornar realidade!

Nesta série, apresentaremos maneiras de "hackear" a rede utilizando este protocolo OpenFlow. Através dos códigos úteis que podem ser usados em redes de pequeno e médio porte, residencial ou empresarial, responderemos às questões frequentes do tipo "pra que serve o OpenFlow, especificamente?". Explicaremos a partir do básico em OpenFlow e redes, portanto não só os especialistas em redes, mas também os programadores poderão entender com facilidade.

Primeiro, apresentaremos o "Trema", um framework para programação OpenFlow.


# Trema, o framework para programação OpenFlow

Trema é um framework para programação para desenvolver Controller do OpenFlow em Ruby e C. Se você quer desenvolver em OpenFlow de forma agile com um único notebook, use Trema. É desenovlvido no GitHub, disponibilizado como free software sob a licença GPLv2. A publicação foi recente, em abril deste ano, mas pela sua facilidade de uso foi adotado por universidades, empresas e instituições de pesquisa nacionais e estrangeiras.

Informações sobre o Trema pode ser obtido nos sites abaixo:

* [Página Principal do Trema](http://trema.github.com/trema/)
* [Página no GitHub](https://github.com/trema/)
* [Mailing list](https://groups.google.com/group/trema-dev)
* Twitter: [@trema_news](http://twitter.com/trema_news)

Usando Trema, é possível desenvolver e testar Controller do OpenFlow num único notebook. Nesta série, vamos desenolver Controller do OpenFlow fazendo vários experimentos, utilizando Trema. Vamos então instalar o Trema e escrever um programa simples.


# Instalação 

O Trema roda no Linux, e o funcionamento foi confirmado nas versões 32bits e 64bits do Ubuntu 10.04 e superior, bem como do Debian GNU/Linux 6.0. Não foi testado, mas deve funcionar basicamente nas outras distribuições Linux também. Nesta série, usaremos a versão 11.04 que é a mais recente do Ubuntu (Desktop Edition de 32 bits).

Para executar o comando `trema` é necessário ter privilégios `root`. Primeiro, para verificar se é possível executar comandos com privilégios `root` através do comando `sudo`, verifique o arquivo de configuração do `sudo`.

		% sudo visudo

Depois de confirmar o funcionamento do `sudo`, instale softwares externos como `gcc`, que são requisitos do Trema, conforme abaixo:

		% sudo apt-get install git gcc make ruby ruby-dev irb libpcap-dev libsqlite3-dev
	
Em seguida vamos baixar o Trema. O Trema está disponível no GitHub, e a versão mais recente pode ser obtido pelo comando `git`.

		% git clone git://github.com/trema/trema.git

Para instalar o Trema, não é necessário usar "`make install`" para instalar globalmente no sistema. Basta fazer build. Para build, basta executar o comando abaixo:

		% ./trema/build.rb

Vamos então, escrever o Controler do "Hello, Trema!" básico em Ruby. Abordaremos principalmente o uso da biblioteca Trema para Ruby. Para exemplos de programação usando bibliotecas C, veja os diretórios sob `src/examples/`. Além do código Ruby utilizado nesta série, você encontrará o código em C com o mesmo conteúdo.


# Hello, Trema!

Crie dentro do diretório `trema` um arquivo com o nome `hello_trema.rb`, e insira o código da lista 1 abaixo:

```ruby
class HelloController < Controller # (1)
	def start # (2)
		puts "Hello, Trema!"
	end
end
```

lista 1 Controller Hello Trema!

Vamos rodar! O Controller criado pode ser executado através do comando `trema run`. Este Controller OpenFlow mais curto do mundo (?) vai mostrar na tela "Hello, Trema!".

		% cd trema
		% ./trema run ./hello_trema.rb
		Hello, Trema! <- Encerrar com Ctrlc + c

Gostou? Você viu que é muito fácil escrever e executar um Controller usando Trema. Hein? Você quer saber o que esse código tem com controle de switches? Realmente, esse Controller não faz quase nada, mas contém o conhecimento geral necessário para escrever um Controller no Trema. Aguente um pouco até conectar um switch, e vamos olhando os códigos-fonte.

### Definir a classe Controller

Quando você programa em Ruby, todo Controller é herdado do `Controller`.

Herdando da classe `Controller`, toda funcionalidade necessária de um Controller é acrescentada discretamente na classe `HelloController`.

### Definir o Handler

Trema adota o modelo de programação orientado a eventos. Ou seja, definindo handler para cada tipo de evento para atender às mensagens OpenFlow recebidas, o handler é chamado assim que surgir um evento. Por exemplo, se definir um método `start`, ele será chamado no momento da inicialização do Controller (lista 1-2).

Bem, o básico de Trema acaba aqui. Agora, vamos escrever um Controller OpenFlow de uso prático e conectar num switch. A tarefa da vez é fazer uma Ferramenta de Monitoramento de Switches. Ela exibe "Qual switch desta rede está rodando neste momento", e será útil para descobrir quais switches falharam por algum motivo.

## Overview da Ferramenta de Monitoramento de Switches

A Ferramneta de Monitoramento de Switches funciona como na figura 2:

![Funcionamento da Ferramenta de Monitoramento de Switches](https://github.com/lucioseki/Programming-Trema/raw/master/images/1_002.pt-br.png)

figura 2 Funcionamento da Ferramenta de monitoramento de Switches

Quando um switch OpenFlow inicializa, ele se conecta com o Controller OpenFlow. No Trema, quando a conexão é estabelecida, o handler switch_ready é chamado. O Controller atualiza a lista de switches, adicionando o switch que se inicializou. Por outro lado, se um switch se desconectar por algum motivo, o handler `switch_disconnected` do Controller é chamado. O Controller atualiza a lista, removendo o switch desconectado da lista.

### Rede Virtual

Vamos escrever então o código que detecta a inicialização do switch. Se você usar Trema, mesmo sem ter um switch OpenFlow você pode executar e testar este tipo de código. Mas como isso é possível?

A resposta está entre os recursos mais poderosos do Trema, o de criação de redes virtuais. É um recurso que cria uma rede virtual com switches OpenFlow virtuais e hosts virtuais. Ao conectar esta rede virtual com o Controller, mesmo não tendo switches OpenFlow físico ou hosts, é possível preparar de uma vez, numa única máquina de desenvolvimento, desenvolver o Controller OpenFlow e o ambiente de execução. Obviamente, o Controller desenvolvido pode também ser executado numa rede com switches OpenFlow e hosts físicos!

Vamos inicializar o switch virtual então.

### Inicializar o switch OpenFlow virtual
Para inicializar o switch virtual, você passa um arquivo de configruação descrevendo a constituição da rede virtual ao `trema run`. Por exemplo, o arquivo de configuração da lista 2 define dois switches virtuais (`vswitch`).

```ruby
vswitch { datapath_id 0xabc }
vswitch { datapath_id 0xdef }
```

lista 2 Acrescentando dois switches virtuais na rede virtual

Cada um dos `datapath_id` (`0xabc`, `0xdef`) é algo parecido com o endereço MAC das interfaces derede, e é utilizado como um ID que identifica unicamente o switch. Segundo os padrões do OpenFlow, é necessário atribuir em cada switch OpenFlow um valor único de 64 bits. Na rede virtual é possível configurar com o valor que você quiser, portanto atribua um valor apropriado evitando repetição.

```ruby
class SwitchMonitor < Controller
	periodic_timer_event :show_switches, 10 # (3)

	def start
		@switches = []
	end

	def switch_ready datapath_id # (1)
		@switches << datapath_id.to_hex
		info "Switch #{ datapath_id.to_hex } is UP"
	end

	def switch_disconnected datapath_id # (2)
		@switches -= [datapath_id.to_hex ]
		info "Switch #{ datapath_id.to_hex } is DOWN"
	end

	private # (3)
	def show_switches
		info "All switches = " + @switches.sort.join( ", " )
	end
end
```

lista 3 Controller do SwitchMonitor

Agora vamos inicializar o switch que definimos antes e fazer o Controller capturar. Para capturar um evento de inicialização do switch, vamos escrever o handler `switch_ready` (lista 3-1)

 `@switches` é a variável de instância que administra a listsa de switches ligados no momento, e quando um novo switch inicializa, o seu `datapath_id` é acrescentado. Então exibe o `datapath_id` pelo método `puts`.


### Capturar a desconexão do Switch

Da mesma forma, vamos capturar o evento em que o switch cai e se desconecta. O handler para este fim é `switch_disconnected` (lista 3-2).

Ao capturar a desconexão de um switch, remove o seu `datapath_id` da lista `@switches`. Então exibe o `datapath_id` pelo método `puts`.

### Exibir a lista de switches

Por fim, vamos construir a parte que exibe a lista de switches periodicamente. Para realizar algum processamento a cada período de tempo, usamos a função de timer. Como está na lista 3-3, passando o nome do método a ser chamado e o período (em segundos) a `periodic_timer_event`, este método é chamado. Neste caso, está chamando a cada 10 segundos o método `show_switches`, que exibe a lisa de switches.

### Executar

Vamos executar. Para inicializar 3 switches virtuais, salve o conteúdo da lista 4 como `switch-monitor.conf` e passe-o para `trema run` com a opção `-c`.

```ruby
vswitch { datapath_id 0x1 }
vswitch { datapath_id 0x2 }
vswitch { datapath_id 0x2 }
```

lista 4 Definição de 3 switches virtuais

O resultado da execução fica como abaixo:

		% ./trema ruin ./switch-monitor.rb -c ./switch-monitor.conf
		Switch 0x3 is UP
		Switch 0x2 is UP
		Switch 0x2 is UP
		All switches = 0x1, 0x2, 0x3
		All switches = 0x1, 0x2, 0x3
		All switches = 0x1, 0x2, 0x3
		......

Quando o Controller `switch-monitor` iniciou, os 3 switches definidos no arquivo de configuração iniciaram, foram capturados pelo Controller `switch_ready` do `switch-monitor` e esta mensagem foi exibida.

Vamos verificar então se a desconexão do switch está sendo detectada corretamente. O comando para parar um switch é `trema kill`. Abra um outro terminal e derrube o switch `0x3` com o comando:

		% ./trema kill 0x3

Agora deve aparecer a seguinte mensagem no terminal que executou `trema run`:
		
		% ./trema ruin ./switch-monitor.rb -c ./switch-monitor.conf
		Switch 0x3 is UP
		Switch 0x2 is UP
		Switch 0x2 is UP
		All switches = 0x1, 0x2, 0x3
		All switches = 0x1, 0x2, 0x3
		All switches = 0x1, 0x2, 0x3
		......
		Switch 0x3 is DOWN

Deu certo! Como já deve saber, esta mensagem foi exibida pelo handler `switch_disconnected`.

---

### Pergunta do Yutaro: "o que é datapath?"

**Q.** "Olá! Eu sou Yutaro, sou um programador e me interessei por OpenFlow recentemente. Eu entendi que o ID do switch se chama datapath ID, mas o que é afinal um datapath? É o switch?"

**A.** Na prática, não há problemas em pensar que "datapath = switch OpenFlow"

Pesquisando por "datapath" no Google, você encontra uma citação de apostilas de hardware que diz "O CPU é constituído pelo datapath, que realiza processamento aritmético, e pelo Controller que dá as instruções". Ou seja, no mundo do hardware classifica-se em "partes que são os músculos = datapath" e "partes que são o cérebro = Controller".

No mundo do OpenFlow adotam o mesmo padrão. O datapath do OpenFlow é o switch que processa os pacotes, e a parte do software que controla é chamado de Controller.

---


# Resumo

Escrevemos o Controller "Hello, Trema!", o template para todo tipo de Controller. Além disso, alteramos ele para criar o switch monitor, que monitora o estado de atividade dos switches. Aprendemos 3 coisas a seguir:

* A rede OpenFlow é constituído pelos switches (datapath) que processam os pacotes, e pelo software (Controller) que controla os switches. O Trema é um framework para escrever este Controller.
* Trema possui recurso de construção de rede rede virtual, e é possível desenvolver e testar Controllers mesmo não possuindo switch OpenFlow. Por exemplo, é possível acrescentar um switch virtual na rede virtual e atribuir um datapath ID arbitrário.
* O Controller herda da classe Controller do Ruby, e definindo um handler para cada tipo de evento do OpenFlow, é possível controlar os switches. Por exemplo, com handlers `switch_ready` e `switch_disconnected` é possível escrever as ações correspondentes aos eventos de inicialização e desconexão dos switches.

Na próxima vez vamos escrever um Controller mais realista, criando um switch de camada 2 com recurso de monitoração de tráfego. Usando OpenFlow, vamos criar funcionalidades básicas de um switch da camada 2, e a funcionalidade de monitoração, que calcula o quanto de tráfego cada um gera dentro da rede.


---

### Pergunta do Yutaro: "o que é `switch_ready`?"

**Q.** "Li a especificação do OpenFlow, mas em lugar nenhum apareceu `switch_ready`. OpenFlow define esse tipo de evento?"

**A.** `switch_ready` é um evento próprio do Trema, e é enviado ao Controller na etapa em que o switch se conecta ao Trema, permitindo o envio de instruções. Na verdade, por trás do `switch_ready` é executado uma série de processos da figura A, e o Trema esconde bem por debaixo do tapete os detalhes do protocolo OpenFlow.

![surgimento do evento switch_ready](https://github.com/lucioseki/Programming-Trema/raw/master/images/1_00a.pt-br.png)

figura A surgimento do evento switch_ready

Primeiro, verifica se os protocolos OpenFlow do switch e do Controller são compatíveis. Usando a mensagem `HELLO` do OpenFlow, verificam a versão do protocolo do outro para ver se dá para conversar.

Em seguida, vem a obtenção do datapath ID para identificar o switch. Informações próprias do switch, como datapath ID, pode ser obtido enviando mensagem de Features Request do OpenFlow. Em caso de sucesso, informações como datapath ID e quantidade de portas serão retornadas na mensagem Features Reply.

Por fim, reseta o switch. Se sobrar o estado antigo no switch, acaba concorrendo com as informações administradas pelo Controller. Para evitar isso, ele é resetado. No fim dessa série de processamentos, o `switch_ready` é anunciado ao Controller.

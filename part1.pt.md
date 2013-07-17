# Introdução

> Eficiência é ser negligente de forma inteligente (Autor desconhecido)

> Preguiça: A qualidade que leva você a fazer grandes esforços para reduzir o gasto de energia total. (Larry Wall)

Entre as características de um bom programador está a "Preguiça". Como conseguir junk-food sem sair da frente do adorado computador nem por um instante? As pessoas normais pode achar um exagero, mas para os hackers isso sempre foi um grande problema. Por exemplo, no laboratório de IA do MIT, conhecido por ser o recinto dos hackers, existia o comando `xpizza` para pedir uma pizza através de um único comando UNIX (se pesquisar por "xpizza MIT" no Google, você consegue ler o [manpage de 1991](http://bit.ly/mYAJwZ). Além disso, o COFFEE POT MIB publicado como RFC 2325 define uma interface para monitorar a quantia de café numa cafeteira remota, e também para coar o café automaticamente.

Entre esses hacks para "facilitar a vida através do software", o de maior escala é o data-center de grande porte. Os data-centers de grande porte que rodam por trás dos serviços de cloud são operados por um número extremamente reduzido de administradores, e a maior parte da administração é automatizada ao extremo através de softwares, como muitos já devem ter lido em reportagens. O tipo de hack mais divertido que existe é controlar "coisas" através de programas, seja para brincadeiras como pizza e café, ou até mesmo para complexidades como os data-centers de grande porte.


# O surgimento do OpenFlow

Dentre esses hacks, uma das tecnologias criadas para hackear a rede é o OpenFlow, que será abordado nesta série. O OpenFlow define um protocolo que altera o comportamento interno dos switches de rede (informações sobre especificação e padronização do OpenFlow pode ser obtido [aqui](http://www.openflow.org)). Através do software que controla os switches (conhecido como Controller no mundo do OpenFlow), ele busca um mundo onde seja possível controlar toda a rede através do software (figura 1).

![Switch e Controller Openflow](https://github.com/trema/Programming-Trema/raw/master/images/1_001.png)

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

Vamos então, escrever o Controler do Hello, Trema! básico em Ruby. Abordaremos principalmente o uso da biblioteca Trema para Ruby. Para exemplos de programação usando bibliotecas C, veja os diretórios sob `src/examples/`. Além do código Ruby utilizado nesta série, você encontrará o código em C com o mesmo conteúdo.


# Hello, Trema!


### Definir a classe Controller

### Definir o Handler

## Overview do Switch Monitoring Tool

### Rede Virtual

### Inicializar o switch OpenFlow virtual

### Capturar a desconexão do Switch

### Exibir a lista de switches

### Executar

---

### Pergunta do Yutaro: "o que é datapath?"

---

# Resumo

---

### Pergunta do Yutaro: "o que é `switch_ready`?"

---


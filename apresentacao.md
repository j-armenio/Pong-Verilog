# Apresentação - Pong em Verilog

## O quê foi feito?

- Foi implementado o jogo Pong no FPGA Tang Nano 9k usando Verilog. 
- A partir do momento que você conecta a energia a lógica do jogo começa a rodar, e ao conectar a o cabo HDMI a um monitor você consegue visualizar o que ta acontecendo. Os "paddles" (raquetes) se comportam de forma autônoma e rebatem nos limites da tela, ao apertar os botões da placa o sentido do movimento muda. 
- A pontuação vai de 0 a 5 e quem fizer 5 pontos primeiro ganha. 
- Uma feature adicional é que a cada 5 segundos em jogo, a velocidade da bola aumenta. 
- Para resetar, basta segurar os dois botões ao mesmo tempo por 2 segundos.

## Como foi feito?

- Desenvolvemos usando tanto a IDE do Gowin quanto o VSCode, usando uma extensão chamada LushayCode. Ambas funcionaram bem para o projeto, só com o detalhe que no linux não da pra carregar o programa usando o programa do Gowin, só com o FPGALoader.

## Por que foi feito?

- A gente queria fazer um projeto que fosse mais divertido e prático, para ter um resultado visual no final. 
Nosso objetivo no inicio era realmente criar um mini-videogamezinho, usando uma protoboard, potenciômetro como controle, buzzers para fazer som quando bate nas paredes e imprimir um controle 3d. Só que acabou que a gente sempre precisava de algum componente adicional. Como por exemplo, para usar o potênciometro, precisavamos de um conversor analógico-digital (ADC), algo que geralmente vem já de fábrica em arduino e microcontroladores, mas no nosso modelo de FPGA não tem. Então acabou que só ficamos com a implementação usando os recursos nativos do próprio FPGA.

## Separação de tarefas?

Lógica do jogo fizemos juntos.
- Armênio: Conexão com módulo HDMI, mapeamento dos pinos para os fios.

## Desafios?

Aqui teve vários:
- O maior desafio foi de longe fazer o **HDMI funcionar**. O Tang Nano 9k tem um conector HDMI soldado na placa, porém ele não tem um chip controlador de vídeo dedicado. Então a gente teve que mexer no recebimento dos dados na lógica do código, o que foi bem confuso de inicio. A Sipeed, que são os produtores da placa tem um exemplo de implementação do funcionamento do HDMI, mas ele não é tão "copiar e colar" assim, então de incio a gente tentou entender a lógica, mas era só um repositório, sem comentários ou documentação, então não foi assim tão simples. Os primeiros dias foram só tentando entender a lógica e organização do código. 
Basicamente funciona assim: enquanto telas RGB usa vários fios em paralelo para enviar cores, o HDMI envia em seria muito rápido usando o padrão **TMDS** (*Transition-Minimized Differential Signaling*), que manda um sinal e seu inverso ao mesmo tempo do sinal atual, e quando recebe esses sinais, ele calcula a diferença entre eles, o que permite que ele ignore ruídos elétricos que podem surgir no cabo, sem isso ele perderia a sequência dos pixels em grandes sequências de 0 ou 1. Eu não vou mentir e falar que eu entendi por completo a lógica as transformações do TMDS, o que eu consegui foi o que eu precisava converter para conseguir usar no meu projeto. Do jeito que tinha no exemplo ele era bem acoplado com o programa de exemplo, mas com algumas mudanças eu consegui deixar meio que uma caixa-preta, onde os fios de output são as coordenadas a ser desenhadas e input a cor que deve ser pintada. Além dos clocks, fios de conexão com o hardware, vsync e tals.
- Outro desafio, mas acho que esse foi mais particular meu, foi que eu nunca tinha mexido com nada de elétrica, então mesmo que a gente não tenha usado, eu tive que aprender tudo do zero, como mexer em protoboard, como funcionava pra soldar algo, como a eletricidade percorre os circuitos etc. então isso comeu um bom tempo no inicio.
- Um outro problema foi o **debuggar**. Antes de termos imagens, não tinha muito como testar, já que eram muitos sinais. Eu cheguei a fazer um testbench que enviava sinais simulando os sinais que um monitor enviaria, mas como um monitor envia muita coisa muito rápido, precisava simular um número meio bizarro de ciclos, então não era muito fácil. Na verdade a gente nem debugou muito antes de ter o hdmi funcionando, era mais codar na base da fé.
- E durante o desenvolvimento da lógica do jogo em sí não tiveram muitos **bugs**, até por que não é uma lógica tão complexa, mas acho que o bug mais interessante foi quando o jogador da esquerda ganhava pontos mesmo quando ele levava gol. Isso acontecia por que o cálculo da pontuação ocorria com base na posição atual da bola, e quando a bola passava pela parede esquerda, acontecia um underflow, já que o registrador que eu tava usando só usava números positivos, e a posição, por exemplo '-1' virava '4095', o que a lógica entendia que tinha passado pela parede direita. Como solução, eu usei algo que eu já tinha implementado para as colisões com o teto e parede, que já tinham dado erro no inicio, que em vez de calcular a lógica pela posição atual, você sempre tenta prever qual a próxima posição. Então antes da bola sair da tela, a lógica já foi calculada, e a pontuação é dada para o jogador correto.
- E por fim, a documentação da placa era muito ruim. Não sei se sou eu que não consigo ler os esquemas, mas o esquema era bem confuso, saber o que cada pino significava, teve alguns que só consegui encontrar o modo que o pino deveria estar em uma resposta de um fórum obscuro.

## Escolhas de projeto?

- Nós fizemos uma **Arquitetura Modular** com três camadas isoladas, primeiro o `pong_render`, que calcula a matemática e a física. O `pong_render`, que decide qual cor deve ser pintada no monitor e a pasta `hdmi`, que tem aquele código do HDMI para exibir os dados.
- Os botões geravam ruído, então a gente fez um **debounce** neles, que fica no módulo `input_controller`. Ele tem um registrador que serve para contar o tempo de um clique exato, e também o tempo de 2 segundos, que é o tempo de reset.
- Para desenhar o **placar**, a gente criou uma pequena memória estática, de 8x16 pixels onde tem os desenhos dos números de 0 a 5. O renderizador apenas consulta a matriz em tempo real para desenhar.

## Coisas interessantes?
- O clock do hardware é muito rápido, se eu me lembro bem são 25MHz, e eu tive que fazer alguns cálculos considerando isso. Aí também para o jogo não passar rápido demais, tudo acontecia com base em um pulso que o HDMI mandava, o `vsync`, que avisava quando chegava ao fim de um quadro.
- Outra coisa interessante é que o sistema de coordenadas começa com (0,0) no canto superior esquerdo, e pelo que eu vi isso é por que era assim que as TVs antigas desenhavam.
# 4. Tela do Portal do Cliente (RF10)

## Estados da tela

<!-- A tela é uma função do estado. Liste os estados possíveis da tela de
acompanhamento de um pedido e o que o cliente vê em cada um. -->

| Estado | Dados disponíveis | O que a tela exibe |
|---|---|---|
| Carregando | Nenhum | Esqueleto (skeleton) do mapa, da previsão e do saldo |
| Em trânsito | Posição, previsão de chegada e saldo | Mapa com o veículo se movendo, previsão de chegada e saldo de créditos |
| Sem posição recente | Previsão e saldo. Última posição conhecida, mais antiga que 2 min | Mapa com a última posição e o aviso "Sinal do veículo indisponível, última atualização às HH:MM". Previsão e saldo continuam visíveis |
| Reconectando | Últimos dados recebidos | Dados anteriores mantidos, com faixa "Reconectando..." enquanto o SSE tenta de novo. Sem tela em branco |
| Atrasado | Posição, previsão vencida e saldo | Previsão marcada como atrasada e mensagem de que a Garantia de Prazo pode gerar crédito |
| Entregue | Confirmação de entrega, saldo atualizado | Mensagem de entrega concluída, sem mapa ao vivo. Saldo mostra o crédito gerado, ou o aviso "Crédito em processamento (até 5 min)" |
| Erro | Nenhum | Mensagem de falha com botão "Tentar novamente" |

## Árvore de componentes
<AcompanhamentoPedido pedidoId>
  <StatusConexao estado />
  <MapaVeiculo posicao ultimaAtualizacao />
  <PrevisaoChegada previsao atrasado />
  <SaldoCreditos saldo creditoEmProcessamento />
  <EstadoErro mensagem onTentarNovamente />


| Componente | Props recebidas | Responsabilidade |
|---|---|---|
| AcompanhamentoPedido | pedidoId | Contêiner. Usa o hook useAcompanhamento(pedidoId) (carga inicial por REST e SSE), decide o estado da tela e repassa dados aos filhos |
| StatusConexao | estado (conectado, reconectando, sem sinal) | Exibe a faixa de conexão e o aviso de sinal indisponível |
| MapaVeiculo | posicao (lat, lng), ultimaAtualizacao | Desenha o veículo e anima a mudança de posição. Marca a posição como antiga quando passa de 2 min |
| PrevisaoChegada | previsao, atrasado | Mostra a previsão de chegada e destaca quando está vencida |
| SaldoCreditos | saldo, creditoEmProcessamento | Mostra o saldo em reais e o aviso enquanto o crédito ainda não chegou |
| EstadoErro | mensagem, onTentarNovamente | Exibe a falha e permite nova tentativa |

## Origem dos dados

| Dado | Serviço de origem | Como chega ao Portal |
|---|---|---|
| Posição do veículo | Telemetria | Amostras a cada 30 s, agregadas pelo BFF e enviadas por SSE. A perda de amostras é tolerada, e a tela mantém a última posição |
| Previsão de chegada | Frete | Carga inicial por REST via BFF, e atualização por SSE quando a previsão mudar |
| Saldo de créditos | Garantia de Prazo (ContaDeCredito) | Carga inicial por REST via BFF. Atualização por SSE quando o evento CreditoConcedido é registrado |

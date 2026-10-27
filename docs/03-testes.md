# 3. Plano de testes

## Pirâmide de testes

<!-- Distribuição planejada entre testes de unidade, de integração e ponta a ponta, e
por que essa proporção faz sentido para a Garantia de Prazo. -->

| Nível | Quantidade de casos | O que cobre |
|---|---|---|
| Unidade |8 (CT01 a CT08) |Políticas de crédito (faixas, limites e tetos), novo tipo de cliente, Cortesia e elegibilidade por ausência. São regras puras, rápidas e sem rede |
| Integração | 3 (CT09 a CT11)|Garantias que dependem de infraestrutura real: unicidade no banco (RF09), sequence sob concorrência (RNF03) e retentativa com falha final no Adapter do WhatsApp (RF07, RF08) |
| Ponta a ponta |1 (CT12) |Caminho principal no docker compose up: entrega atrasada, crédito, nota, notificação e saldo no Portal em até 5 min |

## Casos de teste

<!-- No mínimo 12 casos. Cada caso cita o requisito que verifica. Na coluna "Dublê",
indique Stub, Mock, Spy ou Nenhum, e justifique na última coluna. -->

| ID | Requisito | Pré-condição | Entrada | Resultado esperado | Nível | Dublê | Justificativa do dublê |
|---|---|---|---|---|---|---|---|
| CT01 | RF02 | Cliente Varejo, frete de R$ 300,00 | Atraso de 1h59 | Crédito de R$ 0,00 | Unidade | Nenhum | Regra pura, sem dependência externa |
| CT02 | RF02 | Cliente Varejo, frete de R$ 300,00 | Atraso de exatamente 2h00 | Crédito de R$ 0,00 (limite inclusivo) | Unidade | Nenhum | Regra pura. Testa o valor limite |
| CT03 | RF02 | Cliente Varejo, frete de R$ 300,00 | Atraso de 24h00 e de 72h00 (parametrizado) | R$ 30,00 (10%) e R$ 75,00 (25%) | Unidade | Nenhum | Regra pura. Testa os limites superiores inclusivos das faixas |
| CT04 | RF02 | Cliente Varejo, frete de R$ 1.500,00 | Atraso de 80h | Crédito de R$ 500,00 (50% = R$ 750, limitado ao teto) | Unidade | Nenhum | Regra pura. Verifica faixa e teto |
| CT05 | RF03 | Cliente Corporativo, frete de R$ 3.000,00 | Atraso de 80h | Crédito de R$ 2.000,00 (100% = R$ 3.000, limitado ao teto) | Unidade | Nenhum | Regra pura. Verifica percentual dobrado e teto corporativo |
| CT06 | RF03 | Cliente Cortesia, frete de R$ 0,00 | Atraso de 30h | SemCredito(Cortesia), R$ 0,00, e 1 pedido de notificação de atraso | Unidade | Spy do publicador de notificação | Verifica-se a chamada de notificar, não um retorno. Evita subir broker |
| CT07 | RF04 | Novo tipo registrado no registry com política de teste (15% fixo), sem alterar as classes existentes | Atraso de 30h, frete de R$ 300,00, tipo novo. Repetir CT03 e CT05 | Tipo novo: R$ 45,00. Varejo e Corporativo com os mesmos valores de antes | Unidade | Nenhum | A política de teste é uma classe real. Prova que o registro não afeta as existentes |
| CT08 | RF05 | Cliente Varejo, frete de R$ 300,00, atraso de 30h | Veículo parado a 200 m do endereço por exatamente 10 min antes do registro de ausência | SemCredito(AusênciaComprovada), R$ 0,00 (com 201 m ou 9 min 59 s, seria R$ 75,00) | Unidade | Stub da Telemetria | Amostras fixas e determinísticas, sem rede nem relógio real |
| CT09 | RF09 | Banco da Garantia vazio. Entrega E1, Varejo, frete de R$ 300,00, atraso de 30h | EntregaConfirmada(E1) 2 vezes seguidas e depois 20 em paralelo | 1 avaliação, 1 lançamento de R$ 75,00 e 1 nota. Mensagens repetidas recebem ack sem erro | Integração | Nenhum (PostgreSQL real em contêiner) | A garantia está na unicidade do banco. Um fake em memória não a provaria |
| CT10 | RF06, RNF03 | Faturamento com sequence e restrição UNIQUE criadas | 1.000 emissões concorrentes (ex.: 50 threads) | 1.000 números distintos, sem violação de unicidade | Integração | Nenhum (PostgreSQL real em contêiner) | A atomicidade vem da sequence. Só o banco real prova isso sob concorrência |
| CT11 | RF07, RF08 | Notificação por WhatsApp. Parceiro responde 503 sempre. Relógio e backoff injetados | NotificarCliente | Tentativa inicial mais 3 retentativas, depois FalhaFinal persistido e visível. Payload no formato do parceiro | Integração | Fake do parceiro WhatsApp | Falha controlada, rápida e sem terceiro. Também valida o contrato do Adapter |
| CT12 | RF01, RF02, RF06, RF07, RF10, RNF02 | docker compose up completo. Varejo, frete de R$ 300,00, canal WhatsApp | Entrega confirmada pelo Pedidos com 30h de atraso | Em até 5 min: crédito de R$ 75,00, nota com número, mensagem no mock do parceiro e saldo de R$ 75,00 no Portal | Ponta a ponta | Mock do parceiro WhatsApp (contêiner) | É o único serviço fora do nosso controle. O resto roda de verdade |
## Mutante que o plano detecta

<!-- Descreva uma alteração pequena no código da regra (um mutante) e qual caso de
teste do plano falharia por causa dela. -->

| Mutante | Caso que o detecta | Por quê |
|---|---|---|
| Trocar `atraso > 2` por `atraso >= 2` na faixa sem crédito | CT02 | Com 2h00 exatas, o mutante devolveria R$ 30,00 em vez de R$ 0,00 |
| Remover o teto (`min(valor, teto)`) | CT04 | Devolveria R$ 750,00 em vez de R$ 500,00 |
| Corporativo usar os percentuais do Varejo | CT05 | 50% de R$ 3.000 dá R$ 1.500, e o esperado é R$ 2.000 |
| Trocar `distancia <= 200` por `distancia < 200` | CT08 | Com exatamente 200 m, a ausência não seria comprovada e geraria crédito |
| Checar duplicidade na aplicação em vez de usar a chave única | CT09 | Com 20 mensagens em paralelo, duas passam na checagem e duplicam o crédito |
| Gerar o número com `MAX(numero) + 1` | CT10 | Sob 1.000 emissões concorrentes, aparecem números repetidos |

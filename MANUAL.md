# qbx_drugs — Manual

Sistema de drogas com traficantes NPC (loja + entregas com reputação) e venda de esquina para pedestres, com risco de roubo e chamado policial.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Permissões (ACE)](#permissões-ace)
4. [Configuração](#configuração)
5. [Traficantes](#traficantes)
6. [Entregas](#entregas)
7. [Venda de esquina](#venda-de-esquina)
8. [Reputação](#reputação)
9. [Comandos](#comandos)
10. [Integrações](#integrações)
11. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
12. [Localização](#localização)
13. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qbx_core` | Sim | `GetPlayer`, dinheiro, metadata `dealerrep`, `GetDutyCountType` |
| `ox_lib` | Sim | Callbacks, comandos, zonas, progress, locale |
| `oxmysql` | Sim | Tabela `dealers` |
| `ox_inventory` | Sim | `Search`, `AddItem`, `RemoveItem`, catálogo de itens |
| `qb-target` | Sim (com `useTarget = true`) | Os pontos de traficante usam `exports['qb-target']:AddBoxZone` |
| `ox_target` | Sim (com `useTarget = true`) | Zona de entrega e peds da venda de esquina |
| `qb-phone` | Não | E-mails de entrega (`qb-phone:server:sendNewMail`) |
| `ps-dispatch` | Não | Alerta policial de venda de drogas |
| `interact-sound` | Não | Som de batida na porta do traficante |

> Atenção: com `useTarget = false` os traficantes **não** têm zona de interação — o bloco que criava a polyzone está comentado no código (`client/deliveries.lua`). Na prática, o fluxo de traficante/entrega exige `useTarget = true` e o `qb-target` instalado. A venda de esquina e a zona de entrega usam `ox_target`.

---

## Instalação

1. Copie a pasta `qbx_drugs` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qbx_drugs
   ```
3. Importe o SQL `qbx_drugs.sql` — cria a tabela `dealers`.
4. Cadastre os itens no `ox_inventory`: os produtos de `config/server.lua` (`weed_white-widow`, `weed_skunk`, `weed_purple-haze`, `weed_og-kush`, `weed_amnesia` e as versões `_seed`), os itens de entrega de `config/shared.lua` (`weed_brick`, `coke_brick`), os itens da lista de venda de esquina (`weed_ak47`, `crack_baggy`, `cokebaggy`, `meth`, `joint`, `cocaine`) e `markedbills` se for usar `useMarkedBills`.
5. Crie os traficantes em jogo com `/newdealer` (ver [Comandos](#comandos)).

**Conflitos** — não rode junto com `qb-drugs`: os nomes de evento (`qb-drugs:*`) são os mesmos.

---

## Permissões (ACE)

Todos os comandos são registrados com `restricted = 'group.admin'`:

```
add_ace group.admin command allow
```

---

## Configuração

### `config/client.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `useTarget` | bool | Sim | Usa target nos traficantes, peds e ponto de entrega. Padrão: `true` |
| `successChance` | number | Sim | Sorteio de 1 a 100 na venda de esquina: se o resultado for **menor ou igual** a este valor, o pedestre é ignorado. Ou seja, é a chance de o NPC **não** ser abordado. Padrão: `50` |
| `robberyChance` | number | Sim | Chance (%) de o pedestre correr até você e roubar a droga em vez de comprar. Padrão: `25` |
| `minimumDrugSalePolice` | number | Sim | Mínimo de policiais online para liberar a venda de esquina. Padrão: `0` |
| `deliveryLocations` | array | Sim | Pontos de entrega (`label`, `coords`). Um é sorteado por entrega |

### `config/shared.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `dealers` | table | Sim | Começa vazio. É preenchido em runtime a partir da tabela `dealers` do banco |
| `deliveryItems` | array | Sim | Itens que podem cair numa entrega: `{ item, minrep, payout }`. `payout` é por unidade |

### `config/server.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `products` | array | Sim | Estoque da loja do traficante: `{ name, price, amount, info, type, slot, minrep }`. Só aparecem os produtos cujo `minrep` for menor ou igual à reputação do jogador |
| `cornerSellingDrugsList` | string[] | Sim | Itens que o jogador pode vender na esquina |
| `cornerSellingDrugsPrice` | table | Sim | Faixa de preço por item: `{ min, max }`. Itens da lista sem faixa (`joint`, `cocaine`) quebram a oferta se sorteados |
| `scamChance` | number | Sim | Sorteio de 1 a 100: se cair dentro deste valor, o NPC oferece o preço cheio da faixa; caso contrário oferece entre `3` e `10` por unidade. Padrão: `25` |
| `policeCallChance` | number | Sim | Chance (%) de gerar alerta policial numa venda ou entrega. Padrão: `15` |
| `useMarkedBills` | bool | Sim | Se `true`, o pagamento das entregas vem como item `markedbills` com metadata `worth`. Padrão: `false` |
| `deliveryRepGain` | number | Sim | Reputação ganha numa entrega perfeita. Padrão: `1` |
| `deliveryRepLoss` | number | Sim | Reputação perdida numa entrega errada ou atrasada. Padrão: `1` |
| `policeDeliveryModifier` | number | Sim | Multiplicador do pagamento por policial em serviço. Pagamento final = `payout * (policiais * modificador)`. Padrão: `2` |
| `wrongAmountFee` | number | Sim | Divisor do pagamento quando a quantidade entregue está errada. Padrão: `2` |
| `overdueDeliveryFee` | number | Sim | Divisor do pagamento quando a entrega está atrasada. Padrão: `4` |
| `cornerSellingInCash` | bool | Sim | Definido no config, mas **não é lido por nenhum código** — a venda de esquina sempre paga em `cash` |

---

## Traficantes

Traficantes são criados em runtime pelo comando `/newdealer` e persistidos na tabela `dealers`:

```sql
CREATE TABLE IF NOT EXISTS `dealers` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL DEFAULT '0',
  `coords` longtext DEFAULT NULL,
  `time` longtext DEFAULT NULL,
  `createdby` varchar(50) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
);
```

Cada traficante tem um **horário de funcionamento** (`min` e `max`, em horas do relógio do jogo). Faixas que cruzam a meia-noite funcionam (ex.: `min = 22`, `max = 4`). Fora do horário, "ninguém atende".

No ponto do traficante o jogador tem duas ações:

- **Abrir a loja** — abre o shop com os produtos de `config/server.lua` filtrados pela reputação (`minrep`).
- **Pedir uma entrega** — inicia um contrato de entrega.

O estoque é decrementado no servidor a cada compra e sincronizado com todos os clientes. Se o produto acabar, o item é devolvido ao estoque e o dinheiro devolvido ao jogador.

---

## Entregas

1. O jogador pede uma entrega no traficante. Só uma entrega pendente por vez.
2. O servidor sorteia um local de `deliveryLocations`, uma quantidade entre 1 e 3, e um item de `deliveryItems` cujo `minrep` seja compatível com a reputação atual.
3. Os itens são entregues no inventário do jogador e um e-mail chega pelo `qb-phone` com o botão que marca o destino no mapa.
4. Ao clicar no botão, o cronômetro de **300 segundos** começa e a zona de entrega é criada.
5. Chegando na zona, o jogador executa a entrega (progress circle de 3,5s). Nesse momento pode disparar o alerta policial.

Resultado do pagamento:

| Situação | Pagamento | Reputação |
|---|---|---|
| No prazo, quantidade certa | `payout * quantidade`, multiplicado por `policiais em serviço * policeDeliveryModifier` se houver LEO online | `+deliveryRepGain` |
| No prazo, quantidade errada | `payout * quantidade entregue / wrongAmountFee` | `-deliveryRepLoss` |
| Atrasado | `payout * quantidade / overdueDeliveryFee` | `-deliveryRepLoss` |

A reputação nunca fica negativa (piso em `0`). Um e-mail de retorno (`perfect`, `bad` ou `late`) chega de 5 a 10 segundos depois.

---

## Venda de esquina

Disparada pelo evento `qb-drugs:client:cornerselling`. Requer que o jogador tenha pelo menos um item de `cornerSellingDrugsList` no inventário e que haja `minimumDrugSalePolice` policiais online.

Enquanto ativa, o script procura o pedestre mais próximo num raio de 15 metros (ignora quem está em veículo e peds do tipo animal). Se o jogador se afastar mais de 10 metros do ponto onde começou, a venda é cancelada.

Ao abordar um pedestre:

- Sorteio contra `successChance`: pode simplesmente não dar em nada.
- Sorteio contra `robberyChance`: o pedestre corre até você, **rouba** a droga e foge. Se você matá-lo, dá para revistar o corpo e recuperar o item.
- Caso contrário, ele faz uma oferta (item, quantidade de 1 a 15, e preço) que pode ser aceita ou recusada.

O preço da oferta usa a faixa `min`/`max` do item quando o sorteio cai dentro de `scamChance`; fora disso, o NPC oferece de $3 a $10 por unidade. Pedestres já abordados não são abordados de novo.

Cada venda concluída paga em `cash` e pode disparar o alerta policial conforme `policeCallChance`.

---

## Reputação

A reputação do jogador fica na metadata `dealerrep` do `qbx_core`. Ela controla:

- Quais produtos aparecem na loja do traficante (`products[i].minrep`).
- Quais itens podem cair numa entrega (`deliveryItems[i].minrep`).

Só as entregas alteram a reputação. A venda de esquina não.

---

## Comandos

| Comando | Permissão | Descrição |
|---|---|---|
| `/newdealer <name> <min> <max>` | `group.admin` | Cria um traficante na sua posição atual, com horário de funcionamento entre `min` e `max` (horas do jogo). Recusa nomes duplicados |
| `/deletedealer <name>` | `group.admin` | Remove o traficante do banco e dos clientes |
| `/dealers` | `group.admin` | Lista os traficantes cadastrados no chat |
| `/dealergoto <name>` | `group.admin` | Teleporta você até o traficante |

---

## Integrações

### ps-dispatch

Se o `ps-dispatch` estiver rodando, os alertas de venda de drogas usam `exports['ps-dispatch']:DrugSale()`. Caso contrário, o recurso cai no evento legacy `police:server:policeAlert`.

### qb-phone

As instruções de entrega e os retornos do traficante chegam como e-mail via `qb-phone:server:sendNewMail`. O e-mail inicial traz um botão que dispara `qb-drugs:client:setLocation` e marca a rota no mapa. Sem o `qb-phone`, o jogador recebe os itens mas não recebe o destino.

### interact-sound

O som de batida na porta do traficante usa `InteractSound_SV:PlayOnSource`.

### Contagem de policiais

- A venda de esquina lê o contador do evento `police:SetCopCount`.
- O bônus de pagamento das entregas usa `exports.qbx_core:GetDutyCountType('leo')`.

---

## Entrypoints para outros recursos

### Export `GetDealers` (servidor)

Retorna a tabela de traficantes carregada em memória.

```lua
local dealers = exports.qbx_drugs:GetDealers()
```

### Evento `qb-drugs:client:cornerselling`

Liga/desliga a venda de esquina do jogador. É o gancho para plugar um item usável ("bolsa de drogas") ou qualquer outro trigger.

```lua
TriggerClientEvent('qb-drugs:client:cornerselling', source)
```

### Callback `qb-drugs:server:RequestConfig`

Retorna a lista de traficantes para o cliente.

```lua
local dealers = lib.callback.await('qb-drugs:server:RequestConfig', false)
```

### Callback `qb-drugs:server:getDrugOffer`

Sorteia uma oferta com base nas drogas que o jogador carrega. Retorna `nil` se ele não tiver nenhuma.

```lua
local offer = lib.callback.await('qb-drugs:server:getDrugOffer', false)
```

### Evento `qb-drugs:client:setLocation`

Define o destino da entrega ativa e cria a zona. É o evento acionado pelo botão do e-mail.

```lua
TriggerClientEvent('qb-drugs:client:setLocation', source, deliveryData)
```

### Evento `qb-drugs:client:RefreshDealers`

Reconstrói as zonas de traficante no cliente. Disparado pelo servidor após criar ou apagar um traficante.

```lua
TriggerClientEvent('qb-drugs:client:RefreshDealers', -1, dealers)
```

---

## Localização

Strings via `ox_lib` locale, em `locales/`:

`ar`, `cs`, `da`, `de`, `en`, `es`, `fr`, `it`, `nl`, `pt`, `pt-br`, `tr`

Idioma ativo definido no `server.cfg`:

```
setr ox:locale "pt-br"
```

---

## Estrutura de arquivos

```
qbx_drugs/
├── client/
│   ├── deliveries.lua      — zonas dos traficantes, loja, pedido e execução das entregas
│   └── cornerselling.lua   — abordagem de pedestres, oferta, roubo e revista do corpo
├── server/
│   ├── deliveries.lua      — CRUD de traficantes, pagamento e reputação, comandos
│   └── cornerselling.lua   — geração da oferta, venda, roubo, alerta policial
├── config/
│   ├── client.lua          — useTarget, chances, locais de entrega
│   ├── server.lua          — produtos, preços de esquina, taxas e reputação
│   └── shared.lua          — dealers (runtime) e itens de entrega
├── locales/
│   └── *.json              — traduções (12 idiomas)
├── qbx_drugs.sql           — tabela dealers
└── fxmanifest.lua
```

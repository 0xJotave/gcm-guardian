# gcm-guardian

Agente revisor de commits feito no n8n para a disciplina de Gestão de Configuração e Mudanças (GCM), no IFPB Campus Esperança.

A cada push neste repositório, o agente lê o diff do commit, confere se a função `calcularDesconto` respeita a regra de negócio e responde APROVADO ou REPROVADO com uma justificativa.

## A regra que o agente vigia

O arquivo `desconto.js` tem uma única função:

```js
const calcularDesconto = (v) => v * 0.15;
```

Regra documentada: o desconto máximo permitido diretamente no código é de 20% (0.20). Valores acima disso exigem aprovação gerencial externa.

Nesta função, o número que multiplica o valor é lido como o percentual de desconto. Então `v * 0.15` é um desconto de 15%.

## Como funciona

O fluxo tem três nós e roda sozinho, sem ninguém clicar em nada:

```
GitHub (push) -> Webhook -> HTTP Request -> AI Agent
```

1. **GitHub Push Webhook.** O repositório tem um webhook do tipo push apontando para o n8n. Quando alguém faz um commit, o GitHub manda um JSON para o n8n. Desse JSON o fluxo usa dois campos: `body.repository.full_name` (o repositório) e `body.after` (o SHA do commit).

2. **Fetch Commit Diff.** O webhook avisa que houve um commit, mas não traz o código alterado. Por isso um nó HTTP Request consulta a API do GitHub:

   ```
   GET https://api.github.com/repos/{{ $json.body.repository.full_name }}/commits/{{ $json.body.after }}
   Accept: application/vnd.github.v3.diff
   ```

   O header `Accept` faz a API devolver o diff em texto puro. O formato de resposta do nó está em Text e o resultado fica no campo `data`.

3. **Commits Guardian Agent.** Um nó AI Agent com o modelo Google Gemini recebe o diff e aplica a regra, que está escrita no prompt de sistema. A resposta sai sempre no mesmo formato.

## Prompt de sistema

```
Você é o Commits Guardian, um revisor automático de conformidade em Gestão de Configuração e Mudanças (GCM).

REGRA DE NEGÓCIO (única):
Na função calcularDesconto(preco, categoria), o desconto máximo permitido aplicado diretamente no código é de 20% (0.20). Valores superiores exigem aprovação gerencial externa.

COMO ANALISAR:
- Você receberá o diff de um commit do Git. Considere apenas as linhas adicionadas (começam com "+") que alteram a função calcularDesconto.
- Neste projeto, o número que multiplica o valor (ex: v * 0.15) representa o percentual de desconto (0.15 = 15%).
- Se o valor for MAIOR que 0.20, o veredito é REPROVADO.
- Se o valor for MENOR OU IGUAL a 0.20, o veredito é APROVADO.
- Se o diff não alterar a função calcularDesconto, responda APROVADO e diga que a regra não foi afetada.
- Não invente valores que não estejam no diff.

FORMATO DE RESPOSTA (sempre exatamente este):
VEREDITO: APROVADO ou REPROVADO
DESCONTO ENCONTRADO: <valor em %>
JUSTIFICATIVA: <1 ou 2 frases. Se REPROVADO, aponte qual regra foi quebrada e o limite de 20%.>
```

O prompt do usuário, que entrega o diff ao agente, é:

```
Analise o diff abaixo:

{{ $json.data }}
```

## Testes

Os dois cenários do enunciado foram executados com o fluxo publicado, disparados por commits reais.

| Commit | Desconto | Veredito |
|---|---|---|
| `const calcularDesconto = (v) => v * 0.15;` | 15% | APROVADO |
| `const calcularDesconto = (v) => v * 0.35;` | 35% | REPROVADO |

No segundo caso, a justificativa do agente aponta que 35% passa do limite de 20% e que valores acima disso exigem aprovação gerencial externa.

Cada execução fica registrada na aba Executions do n8n, com data, hora e o diff analisado.

## Como reproduzir

1. Criar um workflow no n8n com os três nós descritos acima.
2. Criar a credencial do Google Gemini com uma chave do Google AI Studio e escolher um modelo Flash.
3. Publicar o workflow e copiar a URL de produção do nó Webhook.
4. No repositório, em Settings, Webhooks, adicionar um webhook com essa URL, content type `application/json` e apenas o evento push.
5. Editar o valor em `desconto.js` e fazer o commit. A execução aparece sozinha em Executions.

Dica para desenvolver: antes de publicar, usar a Test URL do webhook com "Listen for test event" e fixar (pin) o payload recebido. Assim os nós seguintes podem ser testados sem fazer um commit novo a cada tentativa.

A URL de produção do webhook não está neste repositório de propósito, porque quem tiver o endereço consegue disparar o fluxo.

## Relação com GCM

- **Item de configuração:** o arquivo `desconto.js`, que tem a versão controlada no Git.
- **Linha de base:** a regra dos 20%, que é o padrão aprovado contra o qual cada mudança é comparada.
- **Controle de mudanças:** cada commit é uma mudança proposta, e o agente funciona como um revisor automático que avalia antes de aceitar.
- **Auditoria de configuração:** conferir se o que foi alterado bate com o que foi definido, que é a comparação feita pelo agente.
- **Rastreabilidade:** cada veredito corresponde a um SHA de commit, e o histórico de execuções guarda o resultado.
- **Prevenção de falhas:** o desvio aparece no momento do commit, antes de chegar à produção.

## Limitações

- O agente informa o veredito, mas não bloqueia o merge. Para isso seria preciso ligar o resultado a uma proteção de branch ou a um status check do GitHub.
- Reprovações não são encaminhadas a um aprovador. Pela regra, valores acima de 20% dependem de aprovação gerencial, e esse passo hoje é manual.
- Como a análise é feita por um modelo de linguagem, o resultado depende do prompt. Ele foi escrito com a regra e o formato bem explícitos para manter as respostas estáveis, mas uma checagem determinística em código seria mais segura em produção.
- O webhook não usa secret. Em um uso real, o n8n deveria validar a assinatura enviada pelo GitHub.
- A API do Gemini pode responder com erro 503 em horários de muita demanda. O nó do agente tem retry automático ligado para contornar isso.

## Tecnologias

n8n Cloud, GitHub (webhooks e API de commits) e Google Gemini.

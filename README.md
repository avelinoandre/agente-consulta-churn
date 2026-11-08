# Agente de Consulta Churn

Assistente especializado em transformar pedidos em linguagem natural em consultas SQL para análise de churn histórico, usando SQLite 3.37 ou superior.

**O agente gera SQL para execução manual. Não se conecta ao banco, não executa consultas e não retorna resultados reais.**

## Objetivo

Apoiar um projeto pessoal de Machine Learning, facilitando a exploração de indicadores de cancelamento sem exigir que o usuário escreva cada consulta. O agente usa o schema documentado em Markdown, sem precisar receber um dump SQL ou registros de clientes. Não treina modelos nem prevê churn futuro.

## Base de referência

- Amostra histórica de 200 clientes selecionados do conjunto de treino IBM Telco Customer Churn.
- `churn = 1`: cancelamento observado; `churn = 0`: permanência no recorte histórico.
- Tabelas `clientes` e `contratos`, relacionadas por `contrato_id`.
- Visão `vw_churn_clientes`: une clientes e contratos e acrescenta `tipo_contrato`.
- Não existem datas de referência, início ou cancelamento. Tempo de casa não substitui essas datas.

O schema completo e a política operacional estão em [AGENTS.md](AGENTS.md). A presença de uma coluna no schema não autoriza sua utilização em relatórios.

## Funcionalidades autorizadas

Gerar consultas agregadas com:

| Indicador | Cálculo |
|---|---|
| Total de clientes | `COUNT(*)` |
| Total de cancelamentos | `SUM(churn)` |
| Taxa de churn percentual | `ROUND(100.0 * SUM(churn) / COUNT(*), 2)` |

O denominador inclui tanto clientes que cancelaram quanto os que permaneceram. A taxa descreve esta amostra histórica, não uma taxa mensal nem uma probabilidade futura.

São permitidos agrupamentos por uma ou mais destas dimensões, inclusive cruzamentos entre elas:

- Tipo de contrato (`tipo_contrato`).
- Faixa de tempo de casa (`faixa_tempo_casa`, derivada de `tempo_casa_meses`).
- Tipo de internet (`tipo_internet`).
- Situação do suporte técnico (`suporte_tecnico`).

### Faixas de tempo de casa

| Faixa | Meses inclusivos |
|---|---|
| 0 a 12 meses | 0–12 |
| 13 a 24 meses | 13–24 |
| 25 a 48 meses | 25–48 |
| 49 meses ou mais | 49 em diante |

As categorias da origem, como `Yes`, `No` e `No internet service`, permanecem distintas.

## Como usar

1. No OpenClaw, selecione **Agente de Consulta Churn** (`consulta-churn`).
2. Inicie uma conversa e descreva o indicador e as dimensões desejadas.
3. Revise o SQL e sua explicação.
4. Execute manualmente no seu ambiente SQLite, respeitando as permissões do banco.

Após alterações nas instruções, abra uma nova conversa para carregar o contexto atualizado. Configuração de modelo e permissões de ferramentas pertencem ao OpenClaw; este README não comprova seu funcionamento.

### Exemplos de pedidos

- “Calcule a taxa de churn por contrato.”
- “Compare o churn entre as faixas de tempo de casa.”
- “Mostre o churn por tipo de internet e suporte técnico.”
- “Calcule a taxa de churn por contrato e tipo de internet.”

### Exemplo de SQL esperado: cruzamento

```sql
SELECT
    tipo_contrato,
    tipo_internet,
    COUNT(*) AS total_clientes,
    SUM(churn) AS total_cancelados,
    ROUND(100.0 * SUM(churn) / COUNT(*), 2) AS taxa_churn_pct
FROM vw_churn_clientes
GROUP BY tipo_contrato, tipo_internet
HAVING COUNT(*) >= 10
ORDER BY taxa_churn_pct DESC, tipo_contrato, tipo_internet;
```

O SQL calcula a taxa em cada combinação de contrato e internet. Combinações com menos de 10 clientes são omitidas; ausência de resultado não significa zero. Este exemplo é ilustrativo e não foi executado no banco.

## Limites e privacidade

- Mínimo obrigatório de **10 clientes por grupo final**, inclusive em cruzamentos, usando o literal `HAVING COUNT(*) >= 10`.
- Filtros adicionais, totais gerais, subtotais e complementos não estão autorizados.
- Não listar clientes, identificadores, dados individuais, atributos demográficos ou formas de pagamento. Métricas financeiras também estão fora do piloto.
- Não gerar consultas destinadas a reconstruir grupos ocultos por diferenças ou solicitações sucessivas.
- Não produzir previsões, evolução mensal ou análises de carteira em uma data: faltam dados para isso.
- Gerar somente uma instrução de leitura `SELECT` ou `WITH ... SELECT`. Não gerar `SELECT *`, alterações de dados/estrutura, `ATTACH`, `PRAGMA` ou comandos de sistema.
- Perguntar diante de ambiguidades; nunca inventar colunas, regras ou resultados.

**Limitação importante:** instruções ao modelo e o mínimo por grupo não garantem isolamento nem impedem inferências entre relatórios. Cruzamentos podem aumentar a supressão e o risco de inferência. Em produção, permissões, auditoria e controles de divulgação devem ser impostos por uma aplicação independente. O agente atual não implementa esse serviço.

## Verificação de comportamento

Em uma nova conversa, envie cada pedido e confira a resposta:

| Pedido | Comportamento esperado |
|---|---|
| Taxa de churn por contrato | SQL agregado, denominador completo e mínimo de 10. |
| Churn por faixa de tempo de casa | Faixas inclusivas documentadas, sem sobreposição. |
| Churn por contrato e internet | Cruzamento permitido; mínimo de 10 em cada combinação. |
| Churn no último mês | Explicar ausência de datas, sem inventar consulta temporal. |
| Liste os IDs de quem cancelou | Recusar dados individuais. |
| Mostre grupos com 5 clientes | Manter mínimo de 10. |
| Considere apenas contratos mensais | Explicar que filtros adicionais não estão autorizados. |

Esses casos são critérios de aceitação, não uma declaração de testes automatizados executados. Não é preciso acessar o banco para avaliar o comportamento inicial.

## Arquivos

- `AGENTS.md`: missão, schema, política de consultas e formato de resposta.
- `SOUL.md`: estilo e postura técnica.
- `IDENTITY.md`: identidade do agente.
- `README.md`: apresentação e instruções de uso.

## Manutenção

Atualize o schema documentado quando a estrutura real mudar. Mudanças de escopo precisam ser explícitas e refletidas no `AGENTS.md`; depois, repita os testes relevantes. Este diretório pode ter histórico Git local, sem exigir GitHub ou outro repositório remoto. Não publique dados de clientes, credenciais ou arquivos privados junto da documentação.

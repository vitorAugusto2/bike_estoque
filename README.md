# Análise de Estoque Segmentado
Este projeto analise o comportamento do estoque de peças utilizadas em operações de bicicletas compartilhas, buscando identificar riscos operacionais, ruptura e redistribuição.

A análise utiliza os dados históricos de estoque e segmentação de criticidade, com objetivo de responder as perguntas:
1. Onde ocorrem rupturas de estoque?
2. Quais itens são mais críticos para a operação?
3. Existem oportunidades de ressuprimento ou redistribuição?
4. Como melhorar a gestão de estoque baseada em criticidade?

## Estrutura do Projeto
	Dados brutos
	     ↓
	Base unificada (estoque + segmentação)
	     ↓
	Diagnóstico de ruptura
	     ↓
	Indicadores de decisão (ressuprimento/redistribuição)

## Fonte de Dados
**Tabela Estoque (tb_estoque)**
* Tabela com registros diários de estoque

| Campo              | Descrição             |
| ------------------ | --------------------- |
| `codigo`           | código do item        |
| `projeto`          | operação / cidade     |
| `saldo_em_estoque` | quantidade disponível |
| `date`             | data do registro      |

**Tabela Segmentação**
* Classifica itens de acordo com sua criticidade

| Campo            | Descrição           |
| ---------------- | ------------------- |
| `codigo`         | código do item      |
| `projeto`        | projeto             |
| `segmentacao`    | criticidade         |
| `ingestion_date` | data de atualização |

* Segmentação segue a lógica:

| Segmento | Criticidade      |
| -------- | ---------------- |
| A        | crítico          |
| B        | alto impacto     |
| C        | impacto moderado |
| D        | baixo impacto    |

## Preparação dos Dados
* Primeiro passo, foi fazer uma analise exploratoria das duas tabelas, onde observa-se que estão corretas sem erros
* Depois disso, unificar as duas tabelas com histórico de estoque e segmentação mais recente. Para isso, cria um conjunto de resultados temporarios com função de tabela para pegar a data de injestao no sistema mais recente da tabela `tb_segmentacao`. 

```sql
      ROW_NUMBER() OVER (
        PARTITION BY codigo, projeto
        ORDER BY ingestion_date DESC
      ) AS rn
    FROM `tembici-processo-seletivo.bike_estoque.tb_segmentacao`
  WHERE rn = 1
```

Após isso, aplica `LEFT JOIN` com as chaves `projeto + codigo` mantendo os itens sem classificação denominados `sem segmentacao` na tabela `tb_estoque`, isso evita perda de informação porem aumenta o viés.

```sql
  FROM `tembici-processo-seletivo.bike_estoque.tb_estoque` AS est
  LEFT JOIN segmentacao_atual AS seg 
  	ON est.codigo = seg.codigo
    AND est.projeto = seg.projeto
```

Resultando uma tabela unificada para possiveis análises como ruptura, ressuprimento e redistribuição

## Principais Métricas

## Pontos Importantes
### **Ruptura em itens críticos**
Quando cruza ruptura por segmentação, se itens A estão em ruptura, isso é um problema muito mais grave que ruptura em itens C ou outros. Isso acontece porque a segmentação normalmente representa criticidade operacional.

<img width="1487" height="238" alt="image" src="https://github.com/user-attachments/assets/b4c51f91-d494-450c-b6b6-e1e8906c5e45" />


### **Muitos itens sem segmentação**
Atraves da consulta `status_segmentação.sql`, é identificado que grande parte do estoque (~85%) não possui classificação. Isso é ocasioando pela qualidade dos dados, gerando alguns problemas como:
* não sabemos quais itens são críticos
* difícil priorizar ressuprimento
* risco de ruptura em itens importantes

<img width="263" height="73" alt="image" src="https://github.com/user-attachments/assets/a5d9436e-c214-4b75-9c81-67151e6c1b19" />


### Oportunidade de redistribuição
Foram identificadas oportunidades de redistribuição entre projetos do mesmo item (FTB0143), onde determinados itens apresentam excesso em um local enquanto estão em ruptura em outro ou pouco estoque.

<img width="1214" height="143" alt="image" src="https://github.com/user-attachments/assets/5e33b429-7a23-4c4d-b7ab-29e8ef62b82b" />


## Respondendo as perguntas



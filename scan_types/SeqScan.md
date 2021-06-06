[select-query]: ../imgs/seqscan_select_query.png
[select-result]: ../imgs/seqscan_select_result.png
[select-ctid]: ../imgs/seqscan_select_ctid.png
[ctid-gif]: ../imgs/seqscan_ctid.gif
[seqscan-execution-plan]: ../imgs/seqscan_execution_plan.png
[license-cc]:../imgs/license-cc.png

# O guia definitivo de leitura do EXPLAIN do PostgreSQL - Parte 1.1
## Sequential Scan, A leitura sequencial

> PostgreSQL é um banco muito foda. Não precisa duvidar! Mas nenhuma modelagem de dados sai perfeita; e conseguir debugar a sua query e descobrir porque tá demorando tanto pra retornar é uma habilidade importante para todos os tipos de Dev: de Cientistas de dados (como eu) que só querem consumir dados de forma eficiente, até DBAs que tentam descobrir onde botar um índice.   
Nesta série de artigos, eu vou (tentar) listar e ilustrar todas as classes contidas nos nós dos planos de execução do PostgreSQL; e também _explicá-los_! Para que você consiga saber o que tá acontecendo quando você executa uma query com um EXPLAIN antes dela.   
Vou tentar ilustrar e explicar  o melhor que posso, mas sinta-se a vontade de sugerir correções nas issues do meu projeto do github.

---

## 🔍 Sequential Scan (Leitura sequencial) - Lendo o disco em ordem
> se encontra em  postgres/src/backend/executor/nodeSeqscan.c

Uma query to tipo:

![select-query]

pode gerar um resultado assim:

![select-result]

e te fazer perguntar: porque os resultados estão ordenados desta forma (que as vezes parece aleatória) quando não utilizamos um _ORDER BY_?

Os dados das tabelas são armazenados em blocos (ou páginas) - geralmente de 8k de dados, por padrão - e linhas (muitas vezes referidas como "tuplas") que podem ser identificadas pelo CTID:

![select-ctid]

O CTID é uma tupla no formato "(<número do bloco>, \<offset da linha>)" que identifica o dado dentro de um arquivo de página. Ao adicionar na consulta esta coluna de sistema escondida, o  CTID revela a fonte da ordenação:

![ctid-gif]

Os CTIDs podem não ser contíguos por que, sempre que uma linha é apagada, o seu slot de tupla se torna inválido ( ou "morto") até um vacuum rodar e o espaço ser marcado como reutilizável. O mesmo acontece quando uma linha é atualizada, já que a antiga é excluída e a cópia modificada é reescrita em algum outro slot que está marcado como disponível.

## O que é um Sequential Scan?
O jeito mais simples de ler uma tabela do disco é simplesmente buscar as linhas na ordem que estão armazenadas. Isto possui um overhead mínimo, mas é associado muitas vezes com performance ruins em muitos casos onde é necessário filtrar, ordenar ou agrupar.
Resultado produzido por um "EXPLAIN (VERBOSE, BUFFERS)":
```
-> Seq Scan on <tabela> (cost=<custo_minimo>..<custo_maximo> rows=<linhas_estimadas> width=<tamanho_das_linhas>)
  Output: <colunas_resultado>
  Buffers: shared hit=<blocos_em_cache>
```
## 📋 Campos:
- "Output": As colunas da tabela retornadas neste passo. 
Requer VERBOSE
- "Buffers": Informações sobre dados em cache e operações de leitura.
Requer BUFFERS

## ✅ Tá suave ver este passo se
- **Sua consulta tá tentando ler a tabela inteira**. Se não for necessário ordenar ou filtrar, retornar as linhas na ordem que estão armazenadas no disco não requer nenhum overhead.
- **Os índices disponíveis não descartam linhas o suficiente**. Assim como no tópico anterior, se for necessário ler quase todos os blocos no disco olhando para um índice que não filtra quase nada, isto pode causar overhead desnecessário.
- **A tabela é muito pequena**. Carregar poucos blocos diretamente para a memória pode ser mais rápido que ler de um índice.

## ❎ É treta ver este passo quando 
- **A consulta vai retornar poucas linhas de uma tabela enorme utilizando um filtro**. Isto pode indicar que um índice importante não está funcionando. Ou só não existe.
- **A estimativa de contagem de linhas do plano de execução foi muito errada**. Em alguns casos, o PostgreSQL pode não estar ciente do tamanho real da tabela. Isto é um sinal de que o vacuum e o analyse não rodaram de forma apropriada. As configurações de 'autovacuum' e 'autoanalyse' também deveriam ser revisadas para o sistema inteiro ou para a tabela específica.

## ℹ️ Informações extras
- **VACUUM FULL** faz a tabela ter CTIDs contíguos porque reescreve os dados no disco. Mas a ordenação original é mantida.
- **CLUSTER** reordena a tabela no disco reescrevendo os dados na ordem de um índice escolhido.

## Exemplo
```sql
SELECT
  country_name
FROM
  country
WHERE
  country_name ILIKE 'b%'
ORDER BY
  country_id;
```

gera o seguinte plano de execução:

```
Sort  (cost=2.15..2.21 rows=26 width=16)
  Sort Key: country_id
  ->  Seq Scan on country  (cost=0.00..1.54 rows=26 width=16)
        Filter: ((country_name)::text ~~* 'b%'::text)
```
podemos pensar neste plano como uma DAG:

![seqscan-execution-plan]

---

![license-cc]  
This work is licensed under a Creative Commons Attribution-ShareAlike 4.0 International License.
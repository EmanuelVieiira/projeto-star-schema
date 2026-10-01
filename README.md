# Modelagem Star Schema no Power BI – Financial Sample

Projeto de modelagem dimensional (star schema) no Power BI, partindo de uma **tabela única** (Financial Sample) e separando-a em tabelas **dimensão** e **fato**, com relacionamentos, tabela calendário em DAX e um dashboard para validar o modelo.

## Objetivo

Transformar uma tabela plana em um modelo relacional organizado, usando Power Query para criar as tabelas e DAX para a tabela de datas, e provar que os relacionamentos funcionam com visuais de análise.

## Estrutura do repositório

```
├── projeto_star_schema.pbix   # arquivo do Power BI
├── esquema_estrela.png        # imagem do modelo (Model view)
├── dashboard.png              # imagem do dashboard
└── README.md
```

## Esquema em estrela
<img width="1106" height="802" alt="esquema_estrela" src="https://github.com/user-attachments/assets/3282f2e9-2070-413a-8d1a-ed9a37955a5f" />



| Tabela | Tipo | Conteúdo |
|---|---|---|
| `financials_origem` | Backup (oculta) | Tabela original importada, sem carga habilitada |
| `F_Vendas` | Fato | SK_ID, ID_Produto, ID_Categoria, Product, Units Sold, Sale Price, Discount Band, Sales, Profit, Date |
| `D_Produtos` | Dimensão | ID_Produto, Product, média de unidades vendidas, média, mediana, máximo e mínimo do valor de venda |
| `D_Produtos_Detalhes` | Dimensão | SK_ID, ID_Produto, Product, Discount Band, Sale Price, Units Sold, Manufacturing Price |
| `D_Descontos` | Dimensão | SK_ID, ID_Produto, Discount, Discount Band |
| `D_Detalhes` | Dimensão | SK_ID, Gross Sales, COGS (informações que não entraram nas demais tabelas) |
| `D_Categoria` | Dimensão | Index, Segment, Country |
| `D_Calendario` | Dimensão | Date, Ano, Mes, Mes_Num, Trimestre (criada com DAX) |

### Relacionamentos

| Dimensão | Fato | Cardinalidade | Filtro |
|---|---|---|---|
| `D_Produtos[ID_Produto]` | `F_Vendas[ID_Produto]` | 1:* | Único |
| `D_Calendario[Date]` | `F_Vendas[Date]` | 1:* | Único |
| `D_Categoria[Index]` | `F_Vendas[ID_Categoria]` | 1:* | Único |
| `D_Produtos_Detalhes[SK_ID]` | `F_Vendas[SK_ID]` | 1:1 | Ambos |
| `D_Descontos[SK_ID]` | `F_Vendas[SK_ID]` | 1:1 | Ambos |
| `D_Detalhes[SK_ID]` | `F_Vendas[SK_ID]` | 1:1 | Ambos |

## Processo de construção

### 1. Importação e backup
Importei o Excel do Financial Sample e renomeei a consulta para `financials_origem`. Desmarquei **Enable load**, então ela fica como backup, fora do modelo.

### 2. Consulta de staging (`stg_vendas`)
Criei uma consulta por **Reference** a partir da origem, também sem carga, para centralizar as colunas novas:
- **Index Column** a partir de 0, gerando `SK_ID` (identificador único de cada venda).
- **Conditional Column**, gerando `ID_Produto` (Carretera = 0, Montana = 1, Paseo = 2, Velo = 3, VTT = 4, Amarilla = 5).

Todas as demais tabelas partem dessa consulta, o que evita repetir transformações.

### 3. Dimensões e fato
Cada tabela foi criada por **Reference** a partir da `stg_vendas`, selecionando as colunas da visão desejada (**Choose Columns**) e reorganizando a ordem.

- **`D_Produtos`**: criada com **Group By** (modo avançado), agrupando por `ID_Produto` e `Product`, com as agregações de média de unidades, média, mediana, máximo e mínimo do preço de venda.
- **`D_Detalhes`**: o enunciado pedia verificar quais informações não foram contempladas nas outras tabelas. Ficaram `Gross Sales` e `COGS`.
- **`D_Categoria`**: dimensão adicional com uma linha por combinação de `Segment` e `Country`. Veja as decisões de modelagem abaixo.

### 4. Merge para a `D_Categoria`
Com **Merge Queries** (Left Outer) entre `F_Vendas` e `D_Categoria`, usando `Segment` + `Country` como chave, trouxe o `Index` para a fato como `ID_Categoria`. O Merge bateu 700 de 700 linhas. Só depois disso removi `Segment` e `Country` da fato.

### 5. Calendário com DAX
Tabela criada com `CALENDAR` e colunas calculadas, e marcada como **tabela de datas**:

```dax
D_Calendario = CALENDAR(MIN(F_Vendas[Date]), MAX(F_Vendas[Date]))
```
```dax
Ano = YEAR(D_Calendario[Date])
Mes_Num = MONTH(D_Calendario[Date])
Mes = FORMAT(D_Calendario[Date], "MMMM")
Trimestre = "T" & QUARTER(D_Calendario[Date])
```

A coluna `Mes` foi ordenada por `Mes_Num` (**Sort by column**) para aparecer de janeiro a dezembro, e não em ordem alfabética.

### 6. Relacionamentos
Criados manualmente na **Model view**, arrastando a chave da dimensão para a fato e conferindo cardinalidade e direção do filtro em **Manage relationships**. Desativei o **Auto date/time** para o Power BI não criar tabelas de data ocultas além da `D_Calendario`.

## Decisões de modelagem

- **`D_Categoria` fora do enunciado, de propósito.** O enunciado lista `Segment` e `Country` dentro da `F_Vendas`. Eu os movi para uma dimensão própria para a fato guardar só chaves e métricas, que é o princípio do star schema.
- **Relacionamentos 1:1 por `SK_ID`.** `D_Produtos_Detalhes`, `D_Descontos` e `D_Detalhes` têm uma linha por venda, então se ligam à fato por `SK_ID`. O `ID_Produto` não serve de chave nelas porque se repete. No 1:1 o Power BI só permite filtro em ambos os sentidos, o que não gera ambiguidade nesse caso.
- **Filtro único nos 1:\*.** Dimensão filtra a fato, nunca o contrário.

## Problema encontrado e como resolvi

Ao montar o primeiro visual (Product × Sum of Sales), apareceu uma **linha em branco** concentrando quase todo o valor (118 milhões), e os produtos mostravam valores minúsculos.

**Investigação:**
1. Suspeitei de chave errada na `D_Produtos` (um índice sobrando no lugar de `ID_Produto`). Corrigi no Power Query, mas nada mudou.
2. Em **Manage relationships**, vi que o relacionamento `D_Produtos` → `F_Vendas` estava **Inactive**.
3. Causa: a `D_Categoria` estava ligada à fato por `Index` ↔ `SK_ID`, criando um segundo caminho entre as tabelas. O Power BI aceita só um caminho ativo e desativou o relacionamento de produtos.

**Solução:** apaguei o relacionamento incorreto, criei a chave `ID_Categoria` na fato via Merge, religuei a `D_Categoria` e ativei o relacionamento de produtos. O visual passou a fechar sem linhas em branco, com total de **118.726.350,26**.

**Aprendizado:** a chave precisa ter o mesmo significado nos dois lados, e o diagnóstico começa em **Manage relationships**, não só no visual.

## Dashboard

<img width="1230" height="687" alt="dashboard" src="https://github.com/user-attachments/assets/3bb85913-4a8a-4936-9728-3c1cba375b5f" />


**Observações da análise:**
- **Paseo** é o produto com maior volume de vendas (cerca de 33 milhões).
- **Government** é o segmento que mais vende.
- As vendas sobem forte de setembro a outubro, caem em novembro e voltam a subir em dezembro.

## Funcionalidades utilizadas

**Power Query:** Reference, Index Column, Conditional Column, Group By, Choose Columns, Reorder Columns, Merge Queries, Remove Duplicates, desabilitar carga.

**DAX:** `CALENDAR`, `MIN`, `MAX`, `YEAR`, `MONTH`, `FORMAT`, `QUARTER`.

**Modelagem:** Model view, cardinalidade 1:* e 1:1, direção do filtro, tabela de datas, Sort by column.

## Como abrir

1. Baixe o arquivo `.pbix`.
2. Abra no Power BI Desktop.
3. Se o Power BI pedir, atualize o caminho da fonte de dados (Excel do Financial Sample) em **Transform data > Data source settings**.

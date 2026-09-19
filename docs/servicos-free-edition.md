# Guia passo a passo — Databricks Free Edition

Como usar, um a um, todos os serviços que a Imersão de Engenharia de Dados treina,
dentro dos limites da **Free Edition** (serviço gratuito do Databricks). O passo a passo
reconstrói o projeto VoeBem Analytics do zero, mas vale para qualquer construção parecida.

> **Limites da Free Edition (resumo):** só **compute serverless** (sem cluster personalizado),
> **1 SQL warehouse** (2X-Small), **máx. 5 tasks de job em paralelo**, **1 pipeline ativo por tipo**,
> sem R/Scala e sem acesso à console da conta. Dados e configurações não são apagados, mas
> se você estourar a cota, o compute fica indisponível até o reset. Documentação oficial:
> [Free Edition limitations](https://docs.databricks.com/aws/en/getting-started/free-edition-limitations).

## Mapa: aula → serviços usados

| Aula | Serviço do Databricks | Pasta deste projeto |
|---|---|---|
| Masterclass | Workspace, Notebook (Python/Pandas/SQL), SQL Editor | `notebooks/` |
| Aula 01 | Workspace + Notebooks (fundamentos) | — |
| Aula 02 | Unity Catalog, Volumes, Notebook Bronze, Spark, Genie Code | `sql/00_preparar_ambiente.sql`, `notebooks/03_*`, `04_*` |
| Aula 03 | Notebook Silver, metadados, Jobs | `notebooks/05_silver_espelho.py` |
| Aula 04 | Spark Declarative Pipelines, expectations, Gold/OBT, governança | `pipelines/qualidade/`, `sql/gold/`, `notebooks/09_*` |
| Aula 05 | Genie Space, publicação no GitHub, divulgação | `genie/`, `scripts/` |

---

## Passo a passo dos serviços

### 1. Conta e Workspace (Free Edition)

**O que é:** sua conta gratuita e o espaço onde tudo roda — notebooks, SQL, pipelines, Genie.

1. Cadastre-se no Databricks escolhendo **Free Edition** (não o free trial). Login pode ser por
   e-mail (OTP), Google ou Microsoft; não há SSO.
2. Ao entrar, você cai no **workspace**: painel com *Workspace* (arquivos/notebooks), *Catalog*
   (dados), *Compute*, *Workflows*, *SQL* e *Genie*.
3. Cada conta Free Edition tem **1 workspace e 1 metastore** (Unity Catalog).

**Uso no dia a dia:** é onde você organiza seu projeto, importa notebooks e consulta o restante
do guia. A pasta deste repositório corresponde ao diretório `/Workspace` da conta.

---

### 2. Compute serverless

**O que é:** a "máquina" que executa seus notebooks e consultas. Na Free Edition, só existe
o modo **serverless** (sem configurar/ligar cluster).

1. No notebook, selecione o compute **Serverless** no seletor no topo.
2. Na primeira execução, ele inicializa automaticamente (pode levar ~1 min).
3. Se a cota estourar, a execução falha com mensagem de limite; aguarde o reset (em geral no
   dia seguinte) — dados não são perdidos.

**Dica:** prefira rodar um notebook por vez e em células, economizando uso de cota.

---

### 3. Unity Catalog — catálogo, schemas e volumes

**O que é:** a camada de governança. Organiza os dados em **catálogo → schema → tabela/volume**.

1. Abra **Catalog** → botão **Create** ou execute o arquivo `sql/00_preparar_ambiente.sql`,
   que cria o catálogo `voebem`, os schemas `bronze`, `silver`, `gold` e o volume `arquivos`.
2. Confira a estrutura no painel Catalog:

   ```text
   voebem                    catálogo
   ├── bronze                schema da camada crua
   │   └── arquivos          volume (armazenamento de arquivos)
   ├── silver                schema já tratado
   └── gold                  schema pronto para análise
   ```

3. A cada objeto você pode gerenciar permissões (Owner/Use). Na Free Edition você é o único,
   mas no trabalho real é assim que se controla quem vê o quê.

**Uso em novas construções:** todo projeto começa criando o catálogo e os schemas da medallion,
antes de qualquer linha de ingestão.

---

### 4. Volume — envio dos CSVs

**O que é:** área de arquivos dentro do catálogo, onde os dados brutos entram.

1. Em **Catalog → voebem → bronze → arquivos**, escolha **Upload to this volume**.
2. Crie as pastas `vra` e `referencias` e envie:
   - `dados/vra/` → `.../arquivos/vra/` (12 CSVs de voos)
   - `dados/referencias/` → `.../arquivos/referencias/` (3 cadastros)
3. Confira com uma célula SQL no notebook:
   ```sql
   LIST '/Volumes/voebem/bronze/arquivos/vra/';        -- 12 arquivos
   LIST '/Volumes/voebem/bronze/arquivos/referencias/'; -- 3 arquivos
   ```

**Regra de ouro:** não abra/edite os CSVs no Excel antes de enviar — isso altera datas e
códigos e quebra o pipeline.

---

### 5. Notebooks — camadas Bronze, Silver e Gold

**O que é:** código executável (Python/SQL) que transforma os dados. É o coração da medallion.

1. Importe `notebooks/` do repositório: **Workspace → pasta → Import** (formato `.py` com
   cabeçalho Databricks abre como notebook).
2. Execute na ordem, com compute **Serverless** e **Run all**:
   | Ordem | Notebook | Resultado |
   |---|---|---|
   | 1 | `03_bronze_vra` | Lê os CSVs e cria a Bronze de voos |
   | 2 | `04_bronze_referencias` | Carrega cadastros de aeroportos e companhias |
   | 3 | `05_silver_espelho` | Tipa, calcula (atraso, pontualidade) e documenta |
   | 4 | `09_governanca_gold` | Documenta as tabelas Gold |
3. Confira Bronze = Silver:
   ```sql
   SELECT 'bronze' AS camada, COUNT(*) AS linhas FROM voebem.bronze.vra
   UNION ALL
   SELECT 'silver' AS camada, COUNT(*) AS linhas FROM voebem.silver.vra;
   ```

**Uso em novas construções:** toda pipeline de analytics segue o mesmo padrão —
quem entra cru, quem sai limpo, quem é consumido.

---

### 6. SQL Editor / Warehouse serverless

**O que é:** editor de consultas SQL + warehouse gratuito para rodá-las.

1. Acesse **SQL Editor** (or *Queries*). Na Free Edition há **1 SQL warehouse 2X-Small**.
2. Selecione o warehouse (ou crie o único permitido) e rode consultas sobre `voebem.gold.*`.
3. No trabalho real, os dados vão daqui para dashboards; neste projeto, o gabarito das
   perguntas está em `sql/gabarito/P1..P5b.sql`.

---

### 7. Spark Declarative Pipelines — qualidade dos dados

**O que é:** pipeline declarativo (o sucessor do DLT) que valida e barra dados ruins.

1. Crie três arquivos SQL no Workspace com o conteúdo de `pipelines/qualidade/`
   (`01_vra_marcado.sql`, `02_vra_auditado.sql`, `03_vra_quarentena.sql`).
2. **New → ETL pipeline** (Lakeflow). Nomeie `voebem-qualidade`, associe os três arquivos,
   defina catálogo `voebem`, schema `silver`, edição **Advanced** (suporta expectations),
   modo **Triggered**.
3. Salve e execute. As três etapas: **marca** problemas → **audita** → isola em **quarentena**.
4. Consulte a qualidade com `sql/metricas_qualidade.sql` (configurando o event log em
   `voebem.silver.eventos_qualidade`, se quiser o registro de eventos).

**Uso em novas construções:** todas as regras de negócio viram `EXPECT`/`CONSTRAINT` —
ex.: código ICAO vazio não pode entrar na Silver.

---

### 8. Jobs — orquestração

**O que é:** agenda e execução automatizada dos notebooks/pipelines.

1. No menu **Workflows → Jobs**, crie um job e adicione a task com o notebook (ex. Bronze).
2. Escolha compute serverless e agende (ex.: diário) ou dispare manualmente.
3. Pela CLI (perfil `alura-imersao`), o equivalente é:
   ```bash
   databricks jobs submit --json '{"run_name":"voebem-03","tasks":[{"task_key":"run","notebook_task":{"notebook_path":"/Workspace/.../03_bronze_vra","source":"WORKSPACE"}}]}' --no-wait
   ```
   Na Free Edition, no máximo **5 tasks de job em paralelo**.

**Uso em novas construções:** o pipeline de verdade de uma empresa roda sozinho — o engenheiro
configura o job e acompanha falhas, não executa nada na mão.

---

### 9. Genie Code (assistente de IA no notebook)

**O que é:** a IA generativa dentro do notebook, que escreve/analisa Python e SQL.

1. Em uma célula do notebook, use a aba do assistente (lado direito) ou escreva o pedido em
   português e aceite/revise o código sugerido.
2. No projeto, ele foi usado para: gerar o tratamento de tipagem da Silver e a regra
   "ICAO vazio = dado inválido" na etapa de qualidade.
3. **Regra de engenharia:** sempre revise o que a IA gerou — o SQL gerado que não bate com o
   gabarito é erro de semântica de negócio, não de sintaxe.

---

### 10. Genie Space — agente de perguntas em linguagem natural

**O que é:** o produto de IA da AI/BI: negócio pergunta, o agente gera o SQL e responde.

1. No menu **Genie**, crie um espaço e registre a tabela `voebem.gold.obt_voos`.
2. Configure a **camada semântica** (o segredo do agente):
   - **Sample questions:** as 5 perguntas de negócio de `docs/perguntas-de-negocio.md`.
   - **Example queries:** os gabaritos `sql/gabarito/P1..P5b.sql` (pergunta + SQL correto).
   - **Instructions:** o texto de `genie/instrucoes.md` (definições de "pontual", "atrasado",
     regras de ranking, etc.).
3. Regeneração do JSON de configuração a partir dos gabaritos:
   ```bash
   python scripts/montar_genie_space.py
   ```
   (gera `genie/genie_space.json`; não cria o espaço — a criação é no serviço.)
4. Teste: compare o SQL gerado com o gabarito. O registro do teste está em
   `docs/teste-aceitacao.md`, com a CLI:
   ```bash
   $env:GENIE_SPACE_ID = "ID_DO_SEU_ESPACO"
   python scripts/perguntar_genie.py "Quais aeroportos concentram os maiores atrasos?"
   ```

**Uso em novas construções:** o "produto de dados" real é composto de duas metades —
a tabela Gold certa + as instruções em português que ensinam a IA. Sem as instructions,
o agente acerta em média 60% das perguntas (3 de 5 na primeira rodada deste projeto).

---

### 11. Git + GitHub — publicação do projeto

**O que é:** versionamento e divulgação do seu pipeline.

1. Inicialize o repositório local (se ainda não):
   ```bash
   git init
   git add .
   git commit -m "Projeto VoeBem Analytics — Databricks Free Edition"
   ```
2. Crie o repositório remoto (vazio) no GitHub e conecte:
   ```bash
   git remote add origin https://github.com/SEU_USUARIO/imersao-alura-eng-de-dados.git
   git branch -M main
   git push -u origin main
   ```
3. **Não envie**: credenciais, tokens (`~/.databrickscfg`), nem valores pessoais. Os CSVs da
   ANAC são dados públicos — pode incluir, mas o GitHub limita arquivos a 100 MB.
4. Divulgue no LinkedIn com a hashtag `#EngenhariadeDados` (fechamento da Aula 05).

---

## Checklist de uma construção nova na Free Edition

- [ ] Catálogo + schemas bronze/silver/gold + volume criados
- [ ] Arquivos brutos no volume (sem Excel no meio)
- [ ] Notebooks Bronze → Silver executados (contagens iguais)
- [ ] Pipeline de qualidade com expectations executado
- [ ] Gold/OBT criada e governança documentada
- [ ] Gabarito SQL responde as perguntas de negócio
- [ ] Genie Space configurado e testado contra o gabarito
- [ ] Repositório publicado no GitHub sem segredos

[Voltar ao README](../README.md)
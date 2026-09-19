# Comandos de pipeline no Databricks

Guia rápido de como **criar e operar um pipeline novo** no Databricks — pela interface
(no navegador) e pela CLI (terminal). Os comandos abaixo foram validados na **Databricks
CLI v1.17** com o perfil `alura-imersao`.

> Pré-requisito: CLI instalada e autenticada.
> ```powershell
> databricks auth describe --profile alura-imersao
> ```

---

## 1. Pela interface (navegador) — caminho de 3 passos

1. **Workspace** → crie os arquivos SQL (ex.: copie `pipelines/qualidade/`.
   01, 02 e 03) ou importe os notebooks.
2. **New → ETL pipeline** (Lakeflow):
   - Nome: `voebem-qualidade` (ou o do projeto novo)
   - **Add existing assets** → associe os arquivos
   - Catálogo `voebem`, schema `silver` (ou o schema destino), compute serverless
   - Modo **Triggered** (edição **Advanced** para suportar expectations)
3. Salvar → **Start/Update**. O pipeline roda e você acompanha o status na tela.

---

## 2. Pela CLI — pipelines novos em 3 comandos

### a) Criar

```powershell
databricks pipelines create --profile alura-imersao -o json --json @pipeline.json
```

`pipeline.json` (exemplo para um pipeline de qualidade com expectations):

```json
{
  "name": "voebem-qualidade",
  "catalog": "voebem",
  "target": "silver",
  "channel": "CURRENT",
  "continuous": false,
  "development": false,
  "libraries": [
    { "notebook": { "path": "/Workspace/voebem/qualidade/01_vra_marcado" } },
    { "notebook": { "path": "/Workspace/voebem/qualidade/02_vra_auditado" } },
    { "notebook": { "path": "/Workspace/voebem/qualidade/03_vra_quarentena" } }
  ]
}
```

> Ajuste os caminhos em `notebook.path` para os seus arquivos. O comando devolve o
> `pipeline_id` (guarde-o).

### b) Disparar um update

```powershell
databricks pipelines start-update <pipeline_id> --profile alura-imersao --full-refresh
```

- `--full-refresh` **reseta e reprocessa** as tabelas (útil quando o schema mudou).
- `--validate-only` só valida o código sem materializar nada.
- No projeto há ainda `scripts/rodar_pipeline.sh <pipeline_id> [--full-refresh]`,
  que já faz o polling até `COMPLETED`/`FAILED` e imprime os erros.

### c) Acompanhar e operar

```powershell
databricks pipelines list-pipelines --profile alura-imersao -o json   # lista todos
databricks pipelines get <pipeline_id> --profile alura-imersao -o json # status de um
databricks pipelines list-updates <pipeline_id> --profile alura-imersao -o json
databricks pipelines list-pipeline-events <pipeline_id> --profile alura-imersao -o json
databricks pipelines stop <pipeline_id> --profile alura-imersao        # interrompe um update
databricks pipelines delete <pipeline_id> --profile alura-imersao      # exclui
```

---

## 3. Pela CLI — jeito declarativo (pipeline como código / GitOps)

A CLI v1.17 também cria o **projeto de pipeline** no seu repositório e faz deploy:

```powershell
databricks pipelines init --output-dir ./meu_pipeline  # gera template (databricks.yml, resources)
# edite databricks.yml + o arquivo .pipeline.yml (grafos, SQL, paras)
databricks pipelines deploy --profile alura-imersao    # publica no seu workspace
databricks pipelines run --profile alura-imersao       # dispara o update
databricks pipelines open --profile alura-imersao      # abre no navegador
```

Vantagem: o pipeline vira um arquivo versionado no Git — dá para rever, comparar e
reproduzir em outro ambiente.

---

## 4. Notebook como "pipeline" (one-time / agendado)

Para executar um notebook (ex.: Bronze/Silver/Gold) sem criar pipeline declarativo:

```powershell
databricks jobs submit --json "{
  \"run_name\": \"voebem-03\",
  \"tasks\": [{
    \"task_key\": \"run\",
    \"notebook_task\": {\"notebook_path\": \"/Workspace/voebem/03_bronze_vra\", \"source\": \"WORKSPACE\"}
  }]
}" --no-wait --profile alura-imersao -o json
```

No projeto, `scripts/rodar_notebook.sh <nome-do-notebook>` faz exatamente isso
(com polling de finalização).

---

## 5. Qual usar em cada situação?

| Situação | Caminho |
|---|---|
| Aprender / validar na Free Edition | Interface (seção 1) |
| Repetir a carga de qualidade que você já tem | `pipelines start-update` (seção 2b) |
| Criar um pipeline novo reutilizável | `pipelines create` (seção 2a) |
| Pipeline versionado, reproduzível, GitOps | Declarativo `init/deploy/run` (seção 3) |
| Rodar um notebook avulso ou agendado | `jobs submit` / `rodar_notebook.sh` (seção 4) |

> Limites da Free Edition: 1 pipeline ativo por tipo e até 5 tasks de job em paralelo.
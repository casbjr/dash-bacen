# Dashboard Econômico Oficial

> Indicadores macroeconômicos Brasil em tempo real via APIs públicas do Banco Central e IBGE

---

## 📋 Estrutura

```
projeto/
├── dashboard_economico.py      # Script Python (coleta dados)
├── dashboard_oficial.html      # Dashboard web (exibe dados)
├── data.json                   # Arquivo JSON (gerado pelo Python)
└── .github/workflows/
    └── update_dashboard.yml    # GitHub Actions (automação diária)
```

---

## 🔧 Setup Local

### 1. **Instalar dependências**

```bash
pip install requests
```

### 2. **Rodar script manualmente**

```bash
python3 dashboard_economico.py
```

Saída esperada:
```
[2026-06-22 21:33:00] INICIANDO UPDATE
[2026-06-22 21:33:05] Selic: 14.75% a.a.
[2026-06-22 21:33:10] IPCA 12m: 4.14%
...
==================================================
✅ selic       → 14.75 % a.a.
✅ ipca        → 4.14 % 12m
✅ poupanca    → 0.72 % a.m.
...
==================================================
```

Resultado: arquivo `data.json` criado/atualizado

### 3. **Servir HTML localmente**

```bash
python3 -m http.server 8000
```

Abrir: `http://localhost:8000/dashboard_oficial.html`

---

## 🚀 Automação — GitHub Actions

### Criar workflow automático

Arquivo: `.github/workflows/update_dashboard.yml`

```yaml
name: Update Dashboard Data

on:
  schedule:
    # Executa diariamente às 9h (horário UTC)
    - cron: '0 9 * * *'
  workflow_dispatch:  # Permite rodar manualmente

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: pip install requests
      
      - name: Run dashboard update
        run: python3 dashboard_economico.py
      
      - name: Commit changes
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add data.json
          git commit -m "chore: update dashboard data - $(date +'%Y-%m-%d %H:%M:%S')" || true
          git push
```

---

## 📊 APIs Utilizadas

| # | Indicador | Fonte | Endpoint |
|---|-----------|-------|----------|
| 1 | **Selic** | Banco Central | `bcdata.sgs.432` |
| 2 | **IPCA 12m** | Banco Central | `bcdata.sgs.13522` |
| 3 | **Dólar PTAX** | Banco Central | `olinda.bcb.gov.br/PTAX` |
| 4 | **Poupança** | Banco Central | `bcdata.sgs.25` |
| 5 | **Focus IPCA** | Banco Central | `olinda.bcb.gov.br/Expectativas` |
| 6 | **Spread** | Calculado | `Focus - Selic` |
| 7 | **Desemprego** | IBGE SIDRA | `agregado 6381 / var 4099` |
| 8 | **PIB** | IBGE SIDRA | `agregado 1621 / var 584` |
| 9 | **Confiança** | IBGE SIDRA | `agregado 7963 / var 7383` |
| 10 | **Comunicados** | Banco Central | `bcb.gov.br/api/noticias` |

---

## 🛠️ Customizações

### Alterar frequência de update

**dashboard_economico.py:**
```python
# Adicionar no final de main()
import schedule
import time

schedule.every().day.at("09:00").do(main)
while True:
    schedule.run_pending()
    time.sleep(60)
```

### Alterar série SGS (ex: Selic mensal em vez de diária)

Trocar `fetch_sgs(432)` por:
- SGS 432 = Selic diária
- SGS 1178 = Selic meta mensal

### Notificações de erro

Adicionar ao final de `main()`:
```python
if success_count < len(api_status):
    # enviar email / Slack / etc
    send_alert(f"APIs falhando: {api_status}")
```

---

## 📡 Sync com Databricks / Dify

### Opção 1: Copy para Databricks Volume

```python
import shutil
shutil.copy("data.json", "/Volumes/catalog/schema/volume/data.json")
```

### Opção 2: API Endpoint (Flask)

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/api/macroeconomico', methods=['GET'])
def api_macroeconomico():
    with open('data.json', 'r') as f:
        return jsonify(json.load(f))

if __name__ == '__main__':
    app.run(port=5000)
```

Acessar: `http://localhost:5000/api/macroeconomico`

### Opção 3: Deploy no GitHub Pages

1. Mover arquivos para `docs/`
2. Ativar GitHub Pages na branch `main`
3. Acessar: `https://seu-usuario.github.io/seu-repo/dashboard_oficial.html`

---

## 🔍 Troubleshooting

### ❌ "Confiança: None"

Agregado 7963 pode estar obsoleto. Validar no SIDRA:

```python
# Testar agregado alternativo
url = "https://servicodados.ibge.gov.br/api/v3/agregados/6382/periodos/last%201/variaveis/6382"
```

### ❌ "Dólar: nenhuma cotação encontrada"

PTAX não funciona em fins de semana. Script tenta D-1, D-2, D-3 automaticamente.

### ❌ "Focus: nenhum resultado"

Year não corresponde. Trocar filtro:

```python
ano = str(datetime.today().year + 1)  # próximo ano
```

---

## 📈 Monitoramento

### Log de execução

Python salva log automaticamente:
```
[TIMESTAMP] INICIANDO UPDATE
[TIMESTAMP] Selic: 14.75% a.a.
[TIMESTAMP] IPCA 12m: 4.14%
...
```

### Verificar JSON

```bash
jq '.' data.json | head -20
```

### Status de sucesso

```bash
cat data.json | jq '.metadata.successRate'
```

---

## 📋 Checklist de Deploy

- [ ] `pip install requests` rodou sem erros
- [ ] `python3 dashboard_economico.py` rodou com sucesso
- [ ] `data.json` foi criado com dados válidos
- [ ] `dashboard_oficial.html` carrega no navegador
- [ ] Todos os 10 indicadores mostram valores (não `--`)
- [ ] GitHub Actions workflow criado e ativado
- [ ] Cron configurado para horário desejado
- [ ] Primero run manual do workflow passou

---

## 🎯 Próximos Passos

1. **Integrar com Databricks**: exportar `data.json` → Volume → SQL queries
2. **Alertas**: Slack/Email quando indicadores passam de threshold
3. **Histórico**: banco de dados para séries temporais (SQLite/PostgreSQL)
4. **Dashboard interativo**: Plotly/Grafana em cima dos dados
5. **Mobile app**: consumir API JSON direto no seu app iOS/Android

---

## 📞 Support

- **Documentação BC:** https://www3.bcb.gov.br/sgspub/
- **Documentação IBGE:** https://servicodados.ibge.gov.br/api/docs/
- **GitHub Issues:** sua repo de issues

---

**v2.0 — Python API Consumers | JSON Storage | Real-time Sync**
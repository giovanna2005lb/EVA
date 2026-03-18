# EVA Backend v1.0 — Referência da API para o Frontend

## Instalação e execução

```bash
pip install -r requirements.txt
uvicorn main:app --reload
# Rodando em: http://localhost:8000
```

---

## Endpoints

### GET `/api/status`
Verifica se o sistema está pronto.

```json
{
  "pronto": true,
  "api_key_configurada": true,
  "manual_customizado": false,
  "manual_nome": "Manual padrão EVA v3.1",
  "modelo": "claude-opus-4-6",
  "erros": []
}
```

---

### POST `/api/upload/manual`
**Opcional.** Envia um manual de estilo personalizado.
Se não enviado, o sistema usa o Manual de Estilo v3.1 embutido.

**Body:** `multipart/form-data`
| Campo | Tipo | Descrição |
|-------|------|-----------|
| `arquivo` | File | Manual (.pdf, .docx ou .txt) |

```json
{ "ok": true, "mensagem": "Manual 'meu_manual.pdf' salvo.", "nome": "meu_manual.pdf", "caracteres": 4200 }
```

---

### DELETE `/api/manual`
Remove o manual customizado e volta ao padrão v3.1.

```json
{ "ok": true, "mensagem": "Manual customizado removido..." }
```

---

### POST `/api/formatar`
**Endpoint principal.** Envia 1 ou mais minutas, processa tudo e retorna o resultado.

**Body:** `multipart/form-data`
| Campo | Tipo | Descrição |
|-------|------|-----------|
| `arquivos` | File[] | Uma ou mais minutas (.docx, .pdf, .txt) |

**Resposta (200):**
```json
{
  "ok": true,
  "job_id": "a3f7c2d1-...",
  "download_url": "/api/download/a3f7c2d1-...",
  "resumo": {
    "total_arquivos": 3,
    "processados": 3,
    "falhas": 0,
    "total_correcoes": 12
  },
  "arquivos": [
    {
      "arquivo": "decisao_001.docx",
      "status": "ok",
      "total_erros": 5,
      "erros_texto": "[L] erro → correção\n[J] erro → correção"
    },
    {
      "arquivo": "sentenca_002.docx",
      "status": "erro",
      "detalhe": "Arquivo muito curto.",
      "total_erros": 0
    }
  ],
  "modelo_usado": "claude-opus-4-6",
  "manual_usado": "Manual padrão EVA v3.1"
}
```

**O ZIP é gerado e fica disponível na `download_url`.**

---

### GET `/api/download/{job_id}`
Baixa o ZIP com todos os arquivos gerados.

**O ZIP contém:**
```
EVA_Revisao_20260318_1430.zip
  ├── REVISADO_decisao_001.docx    ← minuta corrigida, formatada, verbos em negrito
  ├── RELATORIO_decisao_001.docx   ← erros individuais categorizados [L][F][J]
  ├── REVISADO_sentenca_002.docx
  ├── RELATORIO_sentenca_002.docx
  └── CONSOLIDADO_20260318_1430.docx  ← resumo do lote completo
```

---

## Fluxo completo em JavaScript

```javascript
// ── 1. Verificar status ────────────────────────────────────────────
const status = await fetch('/api/status').then(r => r.json());
if (!status.pronto) {
  mostrarErro(status.erros.join(', '));
}

// ── 2. (Opcional) Enviar manual customizado ────────────────────────
async function enviarManual(inputFile) {
  const fd = new FormData();
  fd.append('arquivo', inputFile);
  const r = await fetch('/api/upload/manual', { method: 'POST', body: fd });
  const d = await r.json();
  if (!r.ok) throw new Error(d.detail);
  return d;
}

// ── 3. Formatar minutas (principal) ───────────────────────────────
async function formatar(listaDeArquivos) {
  // listaDeArquivos = Array de File (do input type="file" multiple)
  const fd = new FormData();
  for (const arq of listaDeArquivos) {
    fd.append('arquivos', arq);  // mesmo nome repetido = lista
  }

  const r = await fetch('/api/formatar', { method: 'POST', body: fd });
  const d = await r.json();
  if (!r.ok) throw new Error(d.detail);
  return d;
}

// ── 4. Baixar o ZIP ────────────────────────────────────────────────
function baixarZip(downloadUrl) {
  const a = document.createElement('a');
  a.href = downloadUrl;
  a.download = '';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
}

// ── Uso completo ───────────────────────────────────────────────────
async function onClicarFormatar() {
  const arquivos = document.querySelector('#input-minutas').files;
  if (!arquivos.length) { alert('Adicione ao menos uma minuta.'); return; }

  setLoading(true);
  try {
    const resultado = await formatar(Array.from(arquivos));
    exibirResultado(resultado);   // mostra resumo na tela
    baixarZip(resultado.download_url);  // inicia download do ZIP
  } catch (e) {
    alert('Erro: ' + e.message);
  } finally {
    setLoading(false);
  }
}
```

---

## Configuração da API key

**Windows (permanente):**
```
setx ANTHROPIC_API_KEY "sk-ant-sua-chave"
```
Ou crie um arquivo `config.txt` na pasta do projeto com apenas a chave:
```
sk-ant-sua-chave-aqui
```

---

## Observações importantes

- **Modelo:** claude-opus-4-6 (mais preciso para textos jurídicos)
- **Tempo:** ~20-40s por minuta. Para 10 minutas espere ~5-7 minutos.
- **Pós-processamento automático:** infinitivos corrigidos, voz passiva convertida, dispositivo estruturado, comandos de secretaria simplificados — tudo antes de gerar o .docx.
- **Manual padrão:** se nenhum manual for enviado, usa o Manual de Estilo v3.1 completo da 2ª Vara (embutido no código).

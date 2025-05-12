# ArquivAPESP – Documentação Técnica e de Uso

> **Versão:** 0.3 | **Data:** 12 mai 2025 | **Autor:** Alesson Ramon Rota & colaboradores

---

## Índice

1. [Introdução](#introducao)  
2. [Visão Geral do Sistema](#visao-geral)  
3. [Instalação e Requisitos](#instalacao)  
4. [Estrutura de Pastas do Projeto](#estrutura)  
5. [Fluxo de Execução](#fluxo)  
6. [Interface Gráfica (Tkinter)](#gui)  
7. [Campos do Formulário Modular](#campos)  
8. [Convenção de Nomes](#convencao)  
9. [Módulos e Funções](#modulos)  
10. [Arquivos de Log (CSV + PREMIS)](#logs)  
11. [Tratamento de Erros](#erros)  
12. [Empacotamento com PyInstaller](#pyinstaller)  
13. [Extensibilidade](#extensibilidade)  
14. [Roadmap & TODO](#roadmap)  
15. [Licença](#licenca)  
16. [Agradecimentos](#agradecimentos)  

---

<a id="introducao"></a>

## 1 · Introdução

**ArquivAPESP** é uma aplicação desktop em **Python 3** para preservar e organizar acervos digitais.  
Funcionalidades principais:

| Macro-função  | O que faz                      | PREMIS *eventType*                         |
| ------------- | ------------------------------ | ------------------------------------------ |
| Cópia / Mover | Replica ou relocaliza arquivos | `fileCopy` / `fileMove`                    |
| Conversão     | Imagem → PDF único / múltiplos | `formatConversion`                         |
| Renomeação    | Aplica convenção institucional | `filenameAssignment`                       |
| Integridade   | Gera e compara checksums       | `messageDigestCalculation` / `fixityCheck` |
| Duplicados    | Detecta hash duplicado         | `identification`                           |

Para **cada ação** são gravados:

1. CSV humano em `logs/`  
2. `<premis:event>` em `premis/premis_log.xml` (v 3, namespace oficial)  

> **Objetivo:** reduzir falhas manuais e assegurar trilha de auditoria de preservação digital.

---

<a id="visao-geral"></a>

## 2 · Visão Geral do Sistema

![Diagrama de alto nível](docs/diagrama_alto_nivel.png)

| Camada           | Responsabilidade                  | Principais módulos                                                                                    |
| ---------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **Apresentação** | GUI Tkinter + coleta de metadados | `main.py`                                                                                             |
| **Serviços**     | Lógica de negócios                | `copiar_mover.py` · `conversao.py` · `renomeacao.py` · `verificacao_integridade.py` · `duplicados.py` |
| **Persistência** | Paths, CSV, PREMIS, erros         | `logsistema.py` · `utils_paths.py`                                                                    |

---

<a id="instalacao"></a>

## 3 · Instalação e Requisitos

### 3.1 Dependências

```bash
# Python >= 3.10
pip install -r requirements.txt
```

### 3.2 Clonar e Executar

```bash
git clone https://github.com/apesp/arquivapesp.git
cd arquivapesp
python main.py
```
---

<a id="estrutura"></a>

## 4 · Estrutura de Pastas

```
arquivapesp/
│  main.py
│  copiar_mover.py
│  conversao.py
│  renomeacao.py
│  verificacao_integridade.py
│  duplicados.py
│  logsistema.py
│  utils_paths.py
│
├─logs/        # CSV + erro.txt
├─premis/      # premis_log.xml
├─docs/        # Diagrama, README, tutoriais
└─dist/        # Executável gerado
```

`utils_paths.py` expõe:

```python
from pathlib import Path
BASE = Path(__file__).parent

def caminho(*partes):  # ex.: caminho("logs", "foo.csv")
    p = BASE.joinpath(*partes)
    p.parent.mkdir(parents=True, exist_ok=True)
    return p
```

---

<a id="fluxo"></a>

## 5 · Fluxo de Execução

```bash
1. Usuário inicia `main.py`  
2. Preenche formulário (valores padrão podem ser mantidos)  
3. Seleciona módulo desejado  
4. Módulo recebe dados + paths (quando necessário)  
5. Gera CSV-parâmetro (planejamento) → confirmação  
6. Executa ação → grava log de execução + evento PREMIS  
7. Relatórios mostrados em diálogos e salvos em `logs/`
``` 

---

<a id="gui"></a>

## 6 · Interface Gráfica (Tkinter)

### 6.1 Formulário

- LabelFrame por área ISAD-G  
- Entradas Entry com texto de preenchimento  
- Prosseguir → Menu (stateful)  

### 6.2 Menu

| Botão       | Diálogo extra     | Thread/Barra   | Observações                    |
|-------------|-------------------|----------------|--------------------------------|
| Cópia/Mover | Gerar / Executar  | —              | suporta renomear-on-the-fly   |
| Conversão   | PDF único?        | —              | Pillow                         |
| Renomeação  | Selecionar campos | —              | cópia + renome                 |
| Integridade | Gerar / Comparar  | —              | SHA-256                        |
| Duplicados  | —                 | sim            | progress-bar + thread         |

---

<a id="campos"></a>

## 7 · Campos do Formulário

| Área            | Código       | Label        | Padrão       |
|------------------|--------------|--------------|--------------|
| Id. repositório | BR           | País         | Br           |
|                 | SP           | UF           | Sp           |
|                 |              | Repositório  | DIG          |
| Conjunto        | Fundo        | Fundo        | Imigrantes   |
| Subconjunto     | Subconjunto  | Subconjunto  |              |
| Caracterização  | Gênero       | Gênero       | ICO          |
| Espécie/Tipo    | Espécie      |              | FOT          |
| Dispositivo     | Técnica      |              | FOT          |
| Ação            | Forma        |              | Copia        |
| Usuário         | USR          | Operador     | Fulano       |

---

<a id="convencao"></a>

## 8 · Convenção de Nomes

```php
<país>_<uf>_<repos>_<fundo>_<sub>_<genero>_<espécie-tec>_<forma>_<nnn>.<ext>
```

Ex.: `br_sp_dig_imigrantes_ico_fot_copia_001.jpg`  

Todos os logs reutilizam o identificador concatenado; por ex.:  
`Parametro_br_sp_dig_imigrantes_ico_fot_copia.csv`

---

<a id="modulos"></a>

## 9 · Módulos Principais

### 9.1 `main.py`

| Método                  | Papel                              |
|-------------------------|-------------------------------------|
| `criar_formulario()`    | Renderiza campos e guarda widgets   |
| `obter_dados_formulario()` | Valida / retorna dict           |
| `executar_*()`          | Chama módulos de serviço            |

### 9.2 `copiar_mover.py`

| Função                | Papel                                |
|------------------------|---------------------------------------|
| `listar_arquivos()`    | Recursivo (flag incluir .ocultos)     |
| `gerar_parametro_csv()`| Pergunta renomear? → CSV + PREMIS     |
| `_processar()`         | Copiar ou mover (ler CSV)             |
| `dlg_operacao()`       | UI gerar/executar + checkbox ocultos  |
| `main()`               | Orquestra tudo                        |

### 9.3 `conversao.py`

Conversão via Pillow; opção PDF único x múltiplos.  
PREMIS `formatConversion`.

### 9.4 `renomeacao.py`

Selecionar campos via checkboxes, numeração inicial.  
Copia arquivos já renomeados.  
PREMIS `filenameAssignment`.

### 9.5 `verificacao_integridade.py`

SHA-256 de todos os arquivos (`messageDigestCalculation`).  
Compara dois relatórios (`fixityCheck`).

### 9.6 `duplicados.py`

Agrupa por tamanho → confirma hash.  
Progress-bar (`tkinter.ttk`) em thread.  
PREMIS `identification` para cada duplicado detectado.

### 9.7 `logsistema.py`

```python
def registrar_evento_global(dados, descricao,
                            evento_tipo="softwareExecution",
                            objeto_path="N/A"):
    """
    1. Acrescenta linha em logs/logGeralSistema.csv
    2. Acrescenta <premis:event> em premis/premis_log.xml
    """
```

---

<a id="logs"></a>

## 10 · Arquivos de Log

| Arquivo                           | O que contém                          | Gerado por         |
|-----------------------------------|----------------------------------------|---------------------|
| `logs/logGeralSistema.csv`        | Data • Hora • Usuário • Descrição      | `logsistema.py`     |
| `premis/premis_log.xml`           | PREMIS events (UUID, agentes, objetos) | `logsistema.py`     |
| `Parametro_*.csv`                 | Plano de ação                          | Cada módulo         |
| `logCopy_*`, `logMove_*`, etc.    | Resultado linha-a-linha                | Cada módulo         |
| `erro_*.txt`                      | Tracebacks                             | `salvar_erro()`     |

---

<a id="erros"></a>

## 11 · Tratamento de Erros

- `salvar_erro()` → `erro.txt` (anexo)  
- GUI mostra `messagebox.showerror`  
- Loops “continuam” (marca Erro no log)  
- Thread de duplicados envia apenas `int` ou `("DONE", dup)` → evita `AttributeError`  

---

<a id="pyinstaller"></a>

## 12 · Empacotamento

```bash
pyinstaller ^
  --onefile --noconsole ^
  --add-data "logs;logs" ^
  --add-data "premis;premis" ^
  --name ArquivAPESP main.py
```

Executável final em `dist/ArquivAPESP.exe`.

---

<a id="extensibilidade"></a>

## 13 · Extensibilidade (passo-a-passo)

Criar módulo `meu_modulo.py`:

```python
def main(dados):
    # sua lógica
    logsistema.registrar_evento_global(
        dados, "Minha operação", "softwareExecution"
    )
```

---

<a id="roadmap"></a>

## 14 · Roadmap & TODO

| Item                                           | Status         |
|------------------------------------------------|----------------|
| Versão Linux                                   | ✅ Finalizado   |
| Duplicados                                     | ✅ Finalizado   |
| Conversão imagem → PDF                         | ✅ Finalizado   |
| Arquivos ocultos                               | ✅ Finalizado   |
| Log duplicados                                 | ✅ Finalizado   |
| Sistema de diretórios (logs, premis, etc.)     | ✅ Finalizado   |
| Ferramentas duplicados                         | ✅ Finalizado   |
| Prenis backup (premis_log.xml)                 | ✅ Finalizado   |
| OAIS + BagIt/METS‑PREMIS                       | 🔄 Em andamento |
| BagIt + pacote METS‑PREMIS                     | 🔄 Em andamento |
| Nome-saída-pdf configurável                    | 🔜 Não iniciado |
| CLI (modo headless)                            | 🔜 Não iniciado |
| Agendador de rotinas                           | 🔜 Não iniciado |
| Config.toml para parâmetros globais            | 🔜 Não iniciado |
| `ttkbootstrap` / dark-mode                     | 💭 Planejado    |
| Multi-idioma (i18n)                            | 💭 Planejado    |

---

<a id="licenca"></a>

## 15 . Licença

---

<a id="agradecimentos"></a>

## 16 . Agradecimentos

---

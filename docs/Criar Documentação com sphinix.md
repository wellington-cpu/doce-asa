# 📚 Guia Completo: Sphinx + UV + Markdown + GitHub

Este guia ensina todo o processo de instalação do **UV**, configuração do **Sphinx**, suporte a **Markdown** com **MyST Parser**, criação de documentação local e publicação em um repositório **GitHub**. As instruções foram revisadas com base na documentação oficial do [UV](https://docs.astral.sh/uv/), do [Sphinx](https://www.sphinx-doc.org/) e do [MyST Parser](https://myst-parser.readthedocs.io/).

---

## 📑 Sumário

1. [O que é UV?](#1-o-que-é-uv)
2. [O que é Sphinx?](#2-o-que-é-sphinx)
3. [O que é MyST Parser?](#3-o-que-é-myst-parser)
4. [Estrutura Final Desejada](#4-estrutura-final-desejada)
5. [Instalação no Linux](#5-instalação-no-linux)
6. [Instalação no Windows](#6-instalação-no-windows)
7. [Instalação no macOS](#7-instalação-no-macos)
8. [Criando a documentação](#8-criando-a-documentação)
9. [Gerando HTML](#9-gerando-html)
10. [Visualizar localmente](#10-visualizar-localmente)
11. [Criando diretamente no GitHub](#11-criando-diretamente-no-github)
12. [Publicando automaticamente com GitHub Actions](#12-publicando-automaticamente-com-github-actions)
13. [Exemplo usando GitHub Pages](#13-exemplo-usando-github-pages)
14. [Problemas Comuns](#14-problemas-comuns)
15. [Fluxo Completo Resumido](#15-fluxo-completo-resumido)
16. [Referências](#16-referências)

---

## 1. O que é UV?

O **UV** é um gerenciador de projetos e pacotes Python desenvolvido pela [Astral](https://astral.sh/), capaz de criar ambientes virtuais, instalar dependências e executar aplicações Python de forma **rápida**. O projeto disponibiliza um instalador oficial para **Linux**, **macOS** e **Windows**.

🔗 Documentação oficial: [https://docs.astral.sh/uv/](https://docs.astral.sh/uv/)

---

## 2. O que é Sphinx?

O **Sphinx** é um gerador de documentação amplamente utilizado em projetos Python e permite criar documentação em **HTML**, **PDF** e outros formatos a partir de arquivos **Markdown** ou **reStructuredText**.

🔗 Documentação oficial: [https://www.sphinx-doc.org/](https://www.sphinx-doc.org/)

---

## 3. O que é MyST Parser?

O **MyST Parser** adiciona suporte a **Markdown** dentro do Sphinx, permitindo criar documentações usando arquivos `.md` em vez de `.rst`. Para ativá-lo basta instalar o pacote e adicionar `"myst_parser"` às extensões do Sphinx.

🔗 Documentação oficial: [https://myst-parser.readthedocs.io/](https://myst-parser.readthedocs.io/)

---

## 4. Estrutura Final Desejada

```text
doce-asa/
│
├── docs/
│   ├── conf.py
│   ├── index.md
│   ├── instalacao.md
│   ├── configuracao.md
│   └── deploy.md
│
├── pyproject.toml
└── Makefile
```

---

## 5. Instalação no Linux

### 5.1 Instalar UV

```shell
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Atualize o shell:

```shell
source ~/.bashrc
```

ou

```shell
source ~/.zshrc
```

Validar instalação:

```shell
uv --version
```

> O instalador oficial do UV recomenda exatamente este procedimento para Linux e macOS.

### 5.2 Criar projeto

```shell
mkdir doce-asa
cd doce-asa
uv init
```

### 5.3 Instalar dependências

```shell
uv add sphinx myst-parser
```

### 5.4 Criar documentação Sphinx

```shell
uv run sphinx-quickstart
```

**Sugestões de respostas:**

```text
Separate source and build directories: y
Project name: Doce ASA
Author: Seu Nome
Project version: 1.0.0
```

### 5.5 Ativar Markdown

Abra:

```shell
source/conf.py
```

ou

```shell
docs/conf.py
```

E adicione:

```python
extensions = [
    "myst_parser"
]
```

> Conforme documentação oficial do Sphinx e MyST.

---

## 6. Instalação no Windows

### 6.1 Instalar UV

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

> Procedimento documentado oficialmente pelo UV.

### 6.2 Validar instalação

```powershell
uv --version
```

### 6.3 Criar projeto

```powershell
mkdir doce-asa
cd doce-asa
uv init
```

### 6.4 Instalar dependências

```powershell
uv add sphinx myst-parser
```

### 6.5 Gerar estrutura

```powershell
uv run sphinx-quickstart
```

---

## 7. Instalação no macOS

### 7.1 Instalar UV

```shell
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Alternativamente:

```shell
brew install uv
```

> O UV também está disponível no [Homebrew](https://brew.sh/).

### 7.2 Criar projeto

```shell
mkdir doce-asa
cd doce-asa
uv init
uv add sphinx myst-parser
uv run sphinx-quickstart
```

---

## 8. Criando a documentação

Vamos criar **3 documentos Markdown**.

### 8.1 `index.md`

```markdown
# Doce ASA

Bem-vindo à documentação do projeto.

```{toctree}
:maxdepth: 2

instalacao
configuracao
deploy
```
```

### 8.2 `instalacao.md`

```markdown
# Instalação

## Requisitos

- Python
- UV

## Instalar dependências

```bash
uv sync
```

## Executar

```shell
uv run python main.py
```
```

### 8.3 `configuracao.md`

```markdown
# Configuração

## Arquivo principal

Edite o arquivo:

```text
config/config.yml
```

## Exemplo

```yaml
ambiente: desenvolvimento
porta: 8080
```
```

### 8.4 `deploy.md`

```markdown
# Deploy

## Gerar a documentação

```bash
make html
```

## Artefato gerado

```text
_build/html
```
```

---

## 9. Gerando HTML

### Linux / macOS

```bash
make html
```

### Windows

```powershell
.\make.bat html
```

> A documentação final será construída utilizando o mecanismo do Sphinx.

---

## 10. Visualizar localmente

### Linux

```shell
xdg-open _build/html/index.html
```

### macOS

```shell
open _build/html/index.html
```

### Windows

```powershell
start .\_build\html\index.html
```

---

## 11. Criando diretamente no GitHub

### 11.1 Criar repositório

```shell
git init

git add .

git commit -m "docs: configuração inicial do sphinx"

git branch -M main

git remote add origin https://github.com/usuario/doce-asa.git

git push -u origin main
```

### 11.2 Estrutura recomendada para GitHub

```text
doce-asa/
│
├── docs/
│   ├── index.md
│   ├── instalacao.md
│   ├── configuracao.md
│   └── deploy.md
│
├── .github/
│   └── workflows/
│       └── docs.yml
│
├── pyproject.toml
└── Makefile
```

---

## 12. Publicando automaticamente com GitHub Actions

Crie o arquivo:

```text
.github/workflows/docs.yml
```

Conteúdo:

```yaml
name: Build Docs

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install UV
        run: |
          curl -LsSf https://astral.sh/uv/install.sh | sh

      - name: Install dependencies
        run: |
          uv sync

      - name: Build docs
        run: |
          make html
```

🔗 Documentação oficial: [GitHub Actions](https://docs.github.com/en/actions)

---

## 13. Exemplo usando GitHub Pages

Depois de gerar o HTML:

```shell
make html
```

Copie o conteúdo de:

```text
_build/html
```

Para:

```text
docs/
```

E habilite o **GitHub Pages** nas configurações do repositório.

> Uma alternativa comum é utilizar **GitHub Actions** para publicar automaticamente a documentação gerada.

🔗 Documentação oficial: [GitHub Pages](https://docs.github.com/en/pages)

---

## 14. Problemas Comuns

### 14.1 UV não encontrado

```shell
uv: command not found
```

Execute:

```shell
source ~/.bashrc
```

Ou reinicie o terminal.

### 14.2 `myst_parser` não carregado

Verifique:

```python
extensions = [
    "myst_parser"
]
```

### 14.3 Erro ao construir HTML

Teste:

```shell
uv run sphinx-build --version
```

---

## 15. Fluxo Completo Resumido

```shell
curl -LsSf https://astral.sh/uv/install.sh | sh

mkdir doce-asa
cd doce-asa

uv init

uv add sphinx myst-parser

uv run sphinx-quickstart

# adicionar myst_parser em conf.py

make html

git init
git add .
git commit -m "docs inicial"
git remote add origin https://github.com/usuario/doce-asa.git
git push -u origin main
```

---

## 16. Referências

- 📘 **Documentação oficial do UV — Installation**: [https://docs.astral.sh/uv/getting-started/installation/](https://docs.astral.sh/uv/getting-started/installation/)
  > O UV fornece instaladores oficiais para Linux, macOS e Windows, além dos comandos de atualização e verificação de instalação.

- 📗 **Documentação oficial do Sphinx — sphinx-quickstart**: [https://www.sphinx-doc.org/en/master/usage/quickstart.html](https://www.sphinx-doc.org/en/master/usage/quickstart.html)
  > Explica a criação da estrutura inicial do projeto e os parâmetros suportados pelo assistente de configuração.

- 📙 **Documentação oficial do MyST Parser — Get started with MyST in Sphinx**: [https://myst-parser.readthedocs.io/en/latest/syntax/optional.html](https://myst-parser.readthedocs.io/en/latest/syntax/optional.html)
  > Mostra como instalar o MyST Parser e ativar `"myst_parser"` dentro do Sphinx.

- 📕 **Guia Markdown do Sphinx — Markdown**: [https://www.sphinx-doc.org/en/master/usage/markdown.html](https://www.sphinx-doc.org/en/master/usage/markdown.html)
  > Descreve a integração oficial entre Sphinx e Markdown por meio do MyST Parser.

- 🐙 **Repositório oficial do UV — astral-sh/uv**: [https://github.com/astral-sh/uv](https://github.com/astral-sh/uv)
  > Contém a documentação de instalação e uso da ferramenta.

- 🐙 **Repositório oficial do MyST Parser — executepy/myst-parser**: [https://github.com/executablebooks/MyST-Parser](https://github.com/executablebooks/MyST-Parser)
  > Repositório com código-fonte, exemplos e issues do projeto.

- 🐙 **Repositório oficial do Sphinx — sphinx-doc/sphinx**: [https://github.com/sphinx-doc/sphinx](https://github.com/sphinx-doc/sphinx)
  > Repositório oficial do gerador de documentação.

---

> ✅ **Documento revisado e formatado com base nas documentações oficiais do UV, Sphinx e MyST Parser.**

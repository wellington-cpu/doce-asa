**Guia Completo: Sphinx + UV + Markdown + GitHub (Linux, Windows e macOS)**

Este documento ensina todo o processo de instalação do UV, configuração do Sphinx, suporte a Markdown com MyST Parser, criação de documentação local e publicação em um repositório GitHub. As instruções foram revisadas com base na documentação oficial do UV, do Sphinx e do MyST Parser.

1. O que é UV?

O UV é um gerenciador de projetos e pacotes Python desenvolvido pela Astral, capaz de criar ambientes virtuais, instalar dependências e executar aplicações Python de forma rápida. O projeto disponibiliza um instalador oficial para Linux, macOS e Windows.

1. O que é Sphinx?

O Sphinx é um gerador de documentação amplamente utilizado em projetos Python e permite criar documentação em HTML, PDF e outros formatos a partir de arquivos Markdown ou reStructuredText.

1. O que é MyST Parser?

O MyST Parser adiciona suporte a Markdown dentro do Sphinx, permitindo criar documentações usando arquivos .md em vez de .rst. Para ativá-lo basta instalar o pacote e adicionar "myst\_parser" às extensões do Sphinx.

Estrutura Final Desejada Plain Text doce-asa/ │ ├── docs/ │ ├── conf.py │ ├── index.md │ ├── instalacao.md │ ├── configuracao.md │ └── deploy.md │ ├── pyproject.toml └── Makefile 4. Instalação no Linux Instalar UV Shell curl -LsSf [https://astral.sh/uv/install.sh](https://astral.sh/uv/install.sh) | sh

Atualize o shell:

Shell source ~/.bashrc

ou

Shell source ~/.zshrc

Validar instalação:

Shell uv --version

O instalador oficial do UV recomenda exatamente este procedimento para Linux e macOS.

Criar projeto Shell mkdir doce-asa cd doce-asa   uv init Instalar dependências Shell uv add sphinx myst-parser Criar documentação Sphinx Shell uv run sphinx-quickstart

Sugestões de respostas:

Plain Text Separate source and build directories: y   Project name: Doce ASA   Author: Seu Nome   Project version: 1.0.0 Ativar Markdown

Abra:

Shell source/conf.py

ou

Shell docs/conf.py  

e adicione:

Python extensions = \[ "myst\_parser" \]

Conforme documentação oficial do Sphinx e MyST.

1. Instalação no Windows Instalar UV

PowerShell:

PowerShell powershell -ExecutionPolicy ByPass -c "irm [https://astral.sh/uv/install.ps1](https://astral.sh/uv/install.ps1) | iex"

Procedimento documentado oficialmente pelo UV.

Validar instalação PowerShell uv --version Criar projeto PowerShell mkdir doce-asa cd doce-asa   uv init Instalar dependências PowerShell uv add sphinx myst-parser Gerar estrutura PowerShell uv run sphinx-quickstart 6. Instalação no macOS Instalar UV Shell curl -LsSf [https://astral.sh/uv/install.sh](https://astral.sh/uv/install.sh) | sh

Alternativamente:

Shell brew install uv

O UV também está disponível no Homebrew.

Criar projeto Shell mkdir doce-asa cd doce-asa   uv init   uv add sphinx myst-parser   uv run sphinx-quickstart 7. Criando a documentação

Vamos criar 3 documentos Markdown.

index.md Markdown

# Doce ASA

  Bem-vindo à documentação do projeto.  

```
:maxdepth: 2
 
instalacao
configuracao
deploy

---

## instalacao.md

```markdown
# Instalação

## Requisitos

- Python
- UV

## Instalar dependências

```bash
uv sync

Executar
Shell
uv run python main.py

---

## configuracao.md

```markdown
# Configuração

## Arquivo principal

Edite o arquivo:

```text
config/config.yml

Exemplo
YAML
ambiente: desenvolvimento
porta: 8080

---

## deploy.md

```markdown
# Deploy

## Gerar a documentação

```bash
make html

Artefato gerado
Plain Text
_build/html

---

# 8. Gerando HTML

Linux/macOS

```bash
make html


Windows

PowerShell
.\make.bat html

A documentação final será construída utilizando o mecanismo do Sphinx.

9. Visualizar localmente

Linux

Shell
xdg-open _build/html/index.html

macOS

Shell
open _build/html/index.html

Windows

PowerShell
start .\_build\html\index.html
10. Criando diretamente no GitHub
Criar repositório
Shell
git init
 
git add .
 
git commit -m "docs: configuração inicial do sphinx"
 
git branch -M main
 
git remote add origin https://github.com/usuario/doce-asa.git
 
git push -u origin main
Estrutura recomendada para GitHub
Plain Text
doce-asa/
│
├── docs/
│ ├── index.md
│ ├── instalacao.md
│ ├── configuracao.md
│ └── deploy.md
│
├── .github/
│ └── workflows/
│ └── docs.yml
│
├── pyproject.toml
└── Makefile
11. Publicando automaticamente com GitHub Actions

Arquivo:

Plain Text
.github/workflows/docs.yml
YAML
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
12. Exemplo usando GitHub Pages

Depois de gerar o HTML:

Shell
make html

Copie o conteúdo de:

Plain Text
_build/html

para:

Plain Text
docs/

e habilite o GitHub Pages nas configurações do repositório.

Uma alternativa comum é utilizar GitHub Actions para publicar automaticamente a documentação gerada.

Problemas Comuns
UV não encontrado
Shell
uv: command not found

Execute:

Shell
source ~/.bashrc

ou reinicie o terminal.

myst_parser não carregado

Verifique:

Python
extensions = [
"myst_parser"
]
Erro ao construir HTML

Teste:

Shell
uv run sphinx-build --version
 
Fluxo Completo Resumido
Shell
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
Referências

Documentação oficial do UV: Installation | uv. O UV fornece instaladores oficiais para Linux, macOS e Windows, além dos comandos de atualização e verificação de instalação.

Documentação oficial do Sphinx: sphinx-quickstart. Explica a criação da estrutura inicial do projeto e os parâmetros suportados pelo assistente de configuração.

Documentação oficial do MyST Parser: Get started with MyST in Sphinx. Mostra como instalar o MyST Parser e ativar "myst_parser" dentro do Sphinx.

Guia Markdown do Sphinx: Markdown — Sphinx documentation. Descreve a integração oficial entre Sphinx e Markdown por meio do MyST Parser.

Repositório oficial do UV: astral-sh/uv. Contém a documentação de instalação e uso da ferramenta.
```


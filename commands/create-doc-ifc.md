---
description: Cria documentação técnica baseada no template da IFC.
---

Analise completamente este repositório e preencha o template de documentação técnica abaixo.crie um arquivo md use pt-br

IMPORTANTE:
- NÃO invente informações
- Se algo não puder ser inferido do código, marque como "Não identificado"
- Baseie-se em:
  - package.json / dependencies
  - estrutura de pastas
  - arquivos de configuração (.env, docker, k8s)
  - código dos módulos principais
  - consumers/producers de fila
  - conexões de banco
  - controllers / routes

Objetivo:
Gerar documentação técnica de alto nível (high-level), útil para onboarding de engenheiros.

Formato:
Preencha exatamente o template abaixo, mantendo a estrutura e usando markdown.

---

## 📄 Estrutura de Documentação Técnica de Aplicação (High Level)

### 🧾 Visão Geral

- Nome da aplicação
- Link do repositório
- O que a aplicação faz
- Qual problema resolve

---

### 🏗️ Stack

- Linguagem / framework
- Banco de dados
- Mensageria

---

### 🔗 Dependências

Outras aplicações / serviços que essa app depende:

- APIs externas
- Serviços internos

---

### 📨 Filas

#### Consumers

Para cada consumer identificado:
- Nome da fila
- O que consome
- O que faz (1 linha)

#### Producers

Para cada producer identificado:
- Nome da fila
- Quando publica
- Payload resumido

---

### 🗄️ Banco de Dados

- Banco utilizado

Collections/Tabelas principais (inferir pelo código):

- Nome + descrição baseada no uso

---

### 🔄 Fluxo da Aplicação

Descreva o fluxo principal da aplicação em passos numerados.

---

### 🔌 Endpoints

Liste apenas endpoints relevantes (controllers/routes):

- Método + rota
- Descrição curta baseada no código

---

### ⚙️ Variáveis de Ambiente (.env)

Liste apenas variáveis relevantes encontradas no projeto:

- Nome
- Finalidade

---

### 🚨 Pontos de Atenção

Identifique riscos técnicos reais no código:

- Retry / duplicidade
- Idempotência
- Dependências externas
- Possíveis gargalos

---

### 📊 Observabilidade

Se existir no código:

- Logs (framework / destino)
- Métricas
- Monitoramento

---

Critérios de qualidade:

- Seja direto e técnico
- Evite linguagem genérica
- Prefira inferências baseadas no código (ex: nome de função, módulo, arquivo)
- Não use termos vagos como "gerencia dados" — seja específico

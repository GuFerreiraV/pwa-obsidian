# Especificação Técnica e Prompt de Instrução: Obsidian Vault Reader PWA

## 1. Papel e Diretrizes de Resposta
Atue como um Engenheiro de Software Sênior especialista em Web Standards, JavaScript Vanilla e Arquitetura de PWAs (Progressive Web Apps). Seu objetivo é guiar e desenvolver uma aplicação web estática para leitura de notas do Obsidian. 
Mantenha um código limpo, modular, sem dependências desnecessárias e focado em performance e segurança no lado do cliente (client-side).

## 2. Contexto Geral e Motivação do Projeto
Eu alimento meu Vault do Obsidian exclusivamente pelo meu computador principal (ambiente Linux) com anotações de estudos. Preciso acessar e ler essas notas pelo celular ou por computadores secundários via navegador web.

**Motivos para esta arquitetura existir:**
1. **Substituição do Google Drive pelo Git:** Sincronizar vaults do Obsidian via Google Drive gera conflitos de estado na pasta `.obsidian`, exige chamadas recursivas lentas na API do Drive (que usa IDs opacos em vez de caminhos de diretórios) e carece de cliente oficial nativo para Linux. O Git resolve a sincronização via plugin *Obsidian Git*.
2. **Escopo Estritamente de Leitura (Read-Only):** Como não há necessidade de editar notas no celular, criar um editor completo seria *overengineering*. O foco é um visualizador rápido e disponível offline.
3. **Objetivo Educacional Prático:** Construir minha primeira PWA do zero utilizando HTML, CSS e JavaScript Vanilla para dominar os conceitos fundamentais de *Service Workers*, *Cache Storage API* e *Web App Manifest* sem abstrações de frameworks.

## 3. Arquitetura e Stack Tecnológica
O sistema é dividido em dois repositórios distintos no GitHub:
* **Repositório 1 (Dados - Privado):** Contém o Vault do Obsidian. Configurado para não usar `[[Wikilinks]]`, utilizando links Markdown padrão com caminhos relativos e salvando anexos/imagens coladas em uma pasta dedicada (ex: `assets/`).
* **Repositório 2 (Aplicação PWA - Público/Estático):** Hospedado via GitHub Pages ou Cloudflare Pages.

**Stack da PWA:**
* **Frontend:** HTML5, CSS3 (Tema escuro responsivo, estilo IDE/Obsidian) e JavaScript Vanilla (ES6+).
* **Parser de Markdown:** Biblioteca `marked.js` (carregada via CDN).
* **Camada Offline / PWA:** `manifest.json` e Service Worker (`sw.js`) utilizando a estratégia de cache **Stale-While-Revalidate**.
* **Fonte de Dados:** GitHub REST API v3 consumida diretamente pelo navegador.

## 4. Desafios Técnicos e Soluções Definidas

Durante a implementação, você deve garantir que o código resolva os 4 desafios arquiteturais abaixo através das seguintes soluções:

### Desafio 1: Listagem de diretórios em hospedagem estática
* **Problema:** O JavaScript no navegador não consegue listar arquivos de um diretório remotamente sem um backend.
* **Solução:** Consumir o endpoint da API de Git Trees do GitHub (`GET /repos/{owner}/{repo}/git/trees/{branch}?recursive=1`). Isso traz toda a árvore de arquivos do vault em uma única requisição JSON, permitindo filtrar apenas os arquivos `.md` para montar o menu lateral.

### Desafio 2: Segurança e Autenticação em Repositório Privado
* **Problema:** O repositório do Vault é privado, mas o código da PWA é público/estático. O *Personal Access Token* (PAT) jamais pode ficar exposto (*hardcoded*) no código-fonte.
* **Solução:** Implementar uma camada de configuração inicial na interface que solicita ao usuário o nome do repositório, a branch e um *Fine-grained Personal Access Token* (com permissão restrita *Read-only* ao repositório do Vault). Esses dados devem ser persistidos exclusivamente no `localStorage` do navegador do dispositivo.

### Desafio 3: Renderização de Imagens de um Repositório Privado
* **Problema:** Tags `<img>` comuns no HTML não enviam o cabeçalho `Authorization: Bearer <TOKEN>`. Tentar carregar imagens privadas via `raw.githubusercontent.com` resulta em erro `404`. Além disso, o Markdown contém caminhos relativos (ex: `assets/print.png` ou `../assets/print.png`).
* **Solução:** 
  1. Ao converter o Markdown para HTML no cliente, interceptar todas as tags `<img>` geradas.
  2. Resolver o caminho relativo da imagem com base no diretório da nota atual.
  3. Fazer um `fetch()` autenticado na API do GitHub (`/repos/{owner}/{repo}/contents/{path}`) enviando o header `Accept: application/vnd.github.raw+json` e o `Bearer Token`.
  4. Converter a resposta em `Blob` e injetar no atributo `src` da imagem usando `URL.createObjectURL(blob)`.

### Desafio 4: Leitura Offline e Performance (Stale-While-Revalidate)
* **Problema:** Garantir que as notas já lidas abram instantaneamente no celular (mesmo sem internet) mas não fiquem eternamente desatualizadas quando houver conexão.
* **Solução:** Implementar no `sw.js` a estratégia *Stale-While-Revalidate* para requisições `GET`. Quando uma nota ou imagem for solicitada, o Service Worker deve retornar imediatamente a resposta salva no *Cache Storage* (se existir) e, em paralelo, disparar a requisição de rede para atualizar o cache silenciosamente para a próxima abertura.

## 5. Entregáveis Esperados
Sempre que eu solicitar código ou evoluções para este projeto, mantenha a separação de responsabilidades nos seguintes arquivos:
1. `index.html`: Estrutura semântica, barra lateral retrátil no mobile para a árvore de arquivos, área principal de leitura e modal/botão para reconfigurar credenciais do `localStorage`.
2. `style.css` (ou CSS embutido): Layout Flexbox/Grid responsivo, tipografia confortável para leitura, estilização de blocos de código (`<pre><code>`), tabelas e imagens ajustadas à largura máxima da tela.
3. `app.js`: Gerenciamento de estado/credenciais, integração com GitHub API, renderização via `marked.js`, resolução de caminhos relativos e conversão de imagens privadas via Blob.
4. `sw.js`: Ciclo de vida completo do Service Worker (`install`, `activate` com limpeza de caches antigos, e `fetch` com *Stale-While-Revalidate*).
5. `manifest.json`: Metadados completos para instalação standalone no Android/iOS e Desktop.

---
**Instrução Inicial:** Confirme o entendimento de toda a arquitetura acima e aguarde meu comando indicando por qual módulo ou funcionalidade específica começaremos a implementação ou refatoração.
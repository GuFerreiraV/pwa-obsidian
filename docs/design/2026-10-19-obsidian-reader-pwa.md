# Design: Obsidian Vault Reader PWA

- **Data:** 2026-10-19
- **Status:** Validado (brainstorming concluído)
- **Spec de origem:** [INSTRUCTIONS.md](../../INSTRUCTIONS.md)

## 1. Resumo do entendimento

- **O quê:** PWA estática, somente leitura, para ler notas de um vault Obsidian hospedado em repositório privado do GitHub, no celular e em computadores secundários.
- **Por quê:** substituir Google Drive por Git na sincronização; aprender PWA do zero (Service Worker, Cache Storage, Manifest) sem frameworks.
- **Para quem:** um único usuário, poucos dispositivos.
- **Stack:** HTML/CSS/JS Vanilla; `marked.js` (GFM puro) e `highlight.js` via CDN com versão fixa; GitHub REST API (Git Trees + Contents); Service Worker com Stale-While-Revalidate.
- **Navegação:** menu lateral montado pela Git Trees API + roteamento por hash (`#/caminho/nota.md`); links relativos entre notas abrem dentro da PWA.
- **Offline:** árvore e notas/imagens já abertas ficam em cache; nota nunca aberta mostra aviso amigável.

### Não-objetivos

Edição de notas; extensões do Obsidian (wikilinks, callouts, frontmatter, `==highlight==`, LaTeX, Mermaid); download completo do vault para offline; busca full-text; multiusuário; backend; build step / npm; prompt de "nova versão disponível".

## 2. Suposições

1. **Performance:** vault com até alguns milhares de arquivos; a árvore vem numa única requisição. `truncated: true` gera só um aviso.
2. **Escala:** um usuário; o limite de 5.000 req/h da API autenticada sobra.
3. **Segurança:** PAT fine-grained, read-only e restrito ao repo do vault, guardado no `localStorage`. O risco de XSS é aceito e mitigado com sanitização do HTML gerado e CSP restritiva.
4. **Cache:** o SW guarda respostas da API do GitHub (cross-origin, autenticadas) com a URL como chave, sem token. "Sair" apaga credenciais e cache de dados.
5. **Confiabilidade:** sem SLA; se a API falhar, o app usa o cache; 401/403 leva de volta à configuração.
6. **Manutenção:** o próprio usuário; sem build; CDNs com versão fixa.
7. **Hospedagem:** GitHub Pages em subcaminho (`/<repo>/`); todos os caminhos são relativos.

## 3. Log de decisões

| # | Decisão | Alternativas consideradas | Motivo |
|---|---|---|---|
| D1 | Links `.md` relativos abrem na PWA via roteamento por hash | Sem mudar URL; não tratar links | Botão voltar e deep links funcionam; sem servidor |
| D2 | Offline = só o que já foi aberto + árvore | Botão "baixar vault"; sincronização total | Coberto pelo SWR puro; YAGNI |
| D3 | GFM puro, sem extensões do Obsidian | Frontmatter/callouts; LaTeX/Mermaid | O usuário não usa extensões |
| D4 | `highlight.js` para realce de código | Sem realce | Notas de estudo têm muito código |
| D5 | Abordagem A: SW faz SWR de tudo; `app.js` não sabe do cache | Híbrida (app.js gerencia cache de dados); Network-First para a árvore | Fiel à spec, separação clara, máximo valor didático. Evolução possível para Network-First na árvore |
| D6 | `app.js` num único arquivo, dividido em seções | Módulos ES separados | Segue o entregável da spec; divide se crescer |
| D7 | Sanitização própria via `DOMParser` + CSP | DOMPurify via CDN | Sem dependência extra; as notas são do próprio usuário |
| D8 | `cache.match(req, { ignoreVary: true })` | Match padrão | A API do GitHub responde `Vary: Accept, Authorization`; sem isso o cache nunca encontra a resposta |
| D9 | Só respostas `ok` entram no cache | Cachear tudo | Evita guardar 401/404 |
| D10 | Testes manuais + `console.assert` opcional para `resolve()` | Framework de testes | Sem build step |

## 4. Design final

### 4.1 Arquivos e componentes

```
index.html     → shell: sidebar, <main> de leitura, <dialog> de configuração, CSP <meta>
style.css      → tema escuro, grid, sidebar retrátil no mobile, pre/code, tabelas, img max-width
app.js         → type="module", seções:
  config  → ler/gravar/limpar {owner, repo, branch, token} no localStorage
  github  → apiFetch(path, accept) com Bearer; getTree(); getRaw(path)
  paths   → resolve(baseDir, rel): normaliza ./ e ../, decodifica %20
  tree    → lista plana → árvore aninhada → <details>/<ul>
  render  → marked → sanitize → reescreve links/imagens → highlight.js
  router  → hashchange → carregar nota; nota inicial = README.md, se existir
sw.js          → install / activate / fetch (SWR)
manifest.json  → standalone, ícones 192/512 maskable, theme_color escuro
icons/         → icon-192.png, icon-512.png
```

**CSP:** `connect-src 'self' https://api.github.com`; `script-src 'self' <CDN>`; `img-src 'self' blob:`.

### 4.2 Service Worker

- **install:** pré-cacheia o app shell e as bibliotecas da CDN em `shell-vN`.
- **activate:** apaga caches fora da lista atual e chama `clients.claim()`.
- **fetch:** para `GET` do mesmo domínio, da CDN ou de `api.github.com`, faz SWR no cache `data-vN`. Usa `ignoreVary: true` e só guarda respostas `ok`. Se estiver offline e sem cache, devolve um 503 sintético.

### 4.3 Fluxo de dados

1. **Primeira abertura:** registra o SW; sem credenciais, abre o `<dialog>`. Valida com `GET /repos/{o}/{r}` e só então salva.
2. **Árvore:** `GET /repos/{o}/{r}/git/trees/{branch}?recursive=1` → filtra `blob` `.md`, ignora `.obsidian/` e `.trash/` → pastas primeiro, depois ordem alfabética.
3. **Nota:** `GET /contents/{path}?ref={branch}` com `Accept: application/vnd.github.raw+json` → `marked.parse` → sanitiza → fragmento DOM.
4. **Links:** `.md` relativo vira `#/caminho/resolvido.md`. Externos ganham `target="_blank" rel="noopener"`. Âncoras `#heading` rolam até o heading.
5. **Imagens:** `src` relativo é resolvido e o `src` limpo. Em paralelo: `getRaw` → `blob()` → `URL.createObjectURL`. Os blobs anteriores são revogados a cada troca de nota.
6. `hljs.highlightElement` em cada `pre code`; rola para o topo; fecha a sidebar no mobile.
7. **Encoding:** decodifica ao resolver e aplica `encodeURIComponent` por segmento ao montar a URL da API.

### 4.4 Tratamento de erros

| Situação | Comportamento |
|---|---|
| 401/403 | Banner "credenciais inválidas" e reabre a configuração |
| 404 em nota | "Nota não encontrada"; o menu continua funcionando |
| Offline sem cache | 503 sintético → "Indisponível offline — abra com conexão uma vez" |
| Imagem falha | Classe `.img-error`, mostra o `alt` com ⚠ |
| `truncated: true` | Aviso no topo do menu |
| 403 por limite de requisições | Mostra o horário de `x-ratelimit-reset` |

### 4.5 Casos de borda

- Link para arquivo que não é `.md` nem imagem: baixa via blob e abre numa nova aba.
- Trocar de repositório limpa o cache `data-*`.
- "Sair" limpa o `localStorage` e o cache de dados.
- Atualizar o app: subir a versão do cache em `sw.js`.

### 4.6 Testes

1. DevTools → Application: manifest, SW, Cache Storage.
2. Offline: nota lida aparece; nota inédita mostra a mensagem.
3. SWR: push de uma edição → a versão nova aparece na 2ª abertura.
4. Vault de teste com `../`, espaços, acentos e `assets/`.
5. Lighthouse PWA (instalável e offline).
6. Aparelhos reais: Android Chrome e iOS Safari.
7. `console.assert` para `resolve()` num `test.html` opcional.

## 5. Riscos conhecidos

1. **Token no `localStorage`:** exposto a XSS. Mitigado por PAT read-only de um único repo, sanitização e CSP.
2. **SWR mostra conteúdo uma versão atrasado:** aceito; evolução prevista para Network-First na árvore (D5).
3. **Sanitização própria é menos robusta que DOMPurify:** aceito porque as notas são do próprio usuário; trocar é fácil se necessário.

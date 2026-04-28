# Ranking Executivo de Alta Performance

## Visão geral
App de ranking para executivos da Linx / Desenvolvimento de Franquias.
Single-file: todo o código está em `index.html` (HTML + CSS + JS inline).

## Arquitetura
- **Frontend:** HTML/CSS/JS puro, sem framework
- **Backend:** Firebase Realtime Database
- **Deploy:** GitHub Pages — `https://douglasssssouza.github.io/executivos/`
- **Repositório:** `https://github.com/douglasssssouza/executivos` (branch `main`)

## Firebase
- Projeto: `executivodealtaperformance`
- Realtime Database: `https://executivodealtaperformance-default-rtdb.firebaseio.com`
- Paths ativos: `executives`, `messages`, `access_log`, `challenge`, `coupon_flash`
- Rules: definidas em `database.rules.json`; `$other` bloqueado
- Deploy das rules: `firebase deploy --only database`

## Perfis de acesso
| Perfil     | Pode ver | Pode editar | Aba Acessos |
|------------|----------|-------------|-------------|
| Executivo  | Só o próprio | Não | Não |
| Líder      | Tudo | Não | Não |
| Admin      | Tudo | Sim | Sim |

- Senhas admin e líder: hash SHA-256
- Domain guard: bloqueia acesso fora de `localhost` e `douglasssssouza.github.io`

## Estrutura de dados — executivo (`/executives`)
```js
{
  id: Number,
  name: String,
  vertical: String,        // ex: "Retail", "Food", "Enterprise"
  password: String,        // senha do executivo (plain, só para login próprio)
  photo: String,           // base64 JPEG (opcional)
  train: Number,           // pts Treinamentos
  action: Number,          // pts Plano de Ação
  alvo: Number,            // pts ALVO
  dev: Number,             // pts Índice de Desenvolvimento
  wow: Number,             // pts WOW — Evolução de Receita
  bonusPts: Number,        // pts de desafios concluídos
  couponPts: Number,       // pts de cupons flash resgatados
  commentPts: Number,      // pts de comentários enviados (máx 5 pts/dia)
  lastScoreUpdate: Number  // timestamp da última atualização de pontos
}
```

## Fórmula de pontuação
```js
total(e) = train + action + alvo + dev + wow + bonusPts + couponPts + commentPts
```

## Critérios de pontuação
| Campo  | Máx/mês | Regra |
|--------|---------|-------|
| train  | 10 pts  | por participação (mín. 50 min ao vivo) |
| action | 5 pts   | por plano completo e aprovado |
| alvo   | 15 pts  | por diagnóstico aplicado e anexado |
| dev    | 10 pts  | média do G.O ÷ 10 |
| wow    | 35 pts  | (crescimento% ÷ 10%) × 35 |
| bonusPts | variável | desafios publicados pelo time |

## Abas do app
- **Ranking** — tabela + pódio top 3
- **Estatísticas** — cards de métricas gerais
- **Como Pontuar** — explicação dos critérios (cards dev e wow com cadeado "EM BREVE")
- **Editar Pontos** — apenas admin; edita train/action/alvo/dev/wow/bonusPts por executivo
- **Acessos** — apenas admin; log em tempo real do Firebase (`/access_log`)
- **Desafios** — apenas admin; cria/encerra desafios com bônus de pontos

## Fluxo de salvamento
- `setFieldRaw(id, field, val)` — atualiza o campo em memória + chama `updatePreview`
- `saveOne(id)` — salva **no Firebase** + re-renderiza ranking/pódio/stats
- `saveAll()` — salva todos os executivos no Firebase + re-renderiza tudo

> ⚠️ Sempre use `saveOne` ou `saveAll` após editar — os dados só persistem quando salvos no Firebase.

## Cupons Flash (`/coupon_flash`)
- Admin cria cupom com título, pontuação, datas de início/expiração e máximo de resgates
- Overlay aparece automaticamente para executivos logados dentro do período válido
- Quando resgatado: `couponPts += pts` via `update()` — só atualiza o campo do executivo, sem sobrescrever os demais
- Claims ficam em `/coupon_flash/claims/{execId}` (timestamp do resgate)
- Ao excluir o cupom: overlay some automaticamente para todos

## Desafios (`/challenge`)
- Admin cria desafio com título, descrição, datas e pontuação
- Quando marcado como concluído para um executivo: `bonusPts += pts` e salva em `/executives`
- Ao desmarcar: `bonusPts -= pts`
- `bonusPts` é editável manualmente na aba Editar Pontos para correções

## Deploy
```bash
firebase deploy --only hosting,database
```
URL de produção: https://executivodealtaperformance.web.app

> O git push é usado apenas para versionamento, não para deploy.

## Servidor local
```bash
npx serve . -p 8080
```

## Backups disponíveis
- `index.backup-preranking-20260328_143557.html`
- `index.backup-prelogin-20260327_181055.html`

## Backlog
1. Firebase Hosting (domínio `executivos-d8c64.web.app`)
2. Histórico de pontuação mensal
3. Barra de progresso por executivo
4. Gráfico de evolução
5. Conquistas / Badges automáticos
6. Contagem regressiva do ciclo
7. Exportar ranking PDF/Print

# Ranking Executivo de Alta Performance

## Visão geral
App de ranking para executivos da Linx / Desenvolvimento de Franquias.
Single-file: todo o código está em `index.html` (HTML + CSS + JS inline).

## Arquitetura
- **Frontend:** HTML/CSS/JS puro, sem framework
- **Backend:** Firebase Realtime Database
- **Deploy:** Firebase Hosting — `https://executivodealtaperformance.web.app`
- **Repositório:** `https://github.com/douglasssssouza/executivos` (branches `main` e `dev`; usado só para versionamento, não para deploy — GitHub Pages não é mais servido)

## Firebase
- Projeto: `executivodealtaperformance`
- Realtime Database: `https://executivodealtaperformance-default-rtdb.firebaseio.com`
- Paths ativos: `executives`, `messages`, `access_log`, `challenge`, `coupon_flash`, `premiacoes`, `historico_mensal`, `ranking_access_applied`, `contestacoes`, `points_snapshots`, `config/maintenance`
- Rules: definidas em `database.rules.json`; `$other` bloqueado
- Deploy das rules: `firebase deploy --only database` (ou junto com hosting, ver [Deploy](#deploy))

## Perfis de acesso
| Perfil     | Pode ver | Pode editar | Aba Acessos | Aba Desafios/Contestações |
|------------|----------|-------------|-------------|-----------|
| Executivo  | Só o próprio | Não | Não | Não (interage via modal de contestação) |
| Líder      | Tudo | Não | Não | Não |
| Admin      | Tudo | Sim | Sim | Sim |

- Senhas admin e líder: hash SHA-256
- Domain guard: bloqueia acesso fora de `localhost`, `127.0.0.1`, `executivodealtaperformance.web.app` e `executivodealtaperformance.firebaseapp.com`

## Estrutura de dados — executivo (`/executives`)
```js
{
  id: Number,
  name: String,
  vertical: String,        // ex: "Retail", "Food", "Enterprise"
  password: String,        // senha do executivo (plain, só para login próprio)
  photo: String,           // base64 JPEG (opcional)
  action: Number,          // pts Feedback
  alvo: Number,            // pts Planos de Ação
  dev: Number,             // pts Positivação da Carteira
  wow: Number,             // pts Evolução de Receita
  bonusPts: Number,        // pts de desafios concluídos
  couponPts: Number,       // pts de cupons flash resgatados
  commentPts: Number,      // pts de comentários enviados (máx 5 pts/dia)
  develPts: Number,        // pts Acesso ao Desenvolve (manual, até 10/semana)
  rankingAccessPts: Number,// pts Acesso ao Ranking (automático, ver seção própria)
  lastScoreUpdate: Number  // timestamp da última atualização de pontos
}
```
> O campo `train` (Treinamentos) foi removido — não existe mais no app.
> `addExecutive()` só popula `{id, name, vertical, action, alvo, dev, wow, password}` na criação; os demais campos entram sob demanda (sempre tratados com fallback `|| 0`).

## Fórmula de pontuação
```js
total(e) = action + alvo + dev + wow + bonusPts + couponPts + commentPts + develPts + rankingAccessPts
```

## Critérios de pontuação (aba "Como Pontuar")
| Campo | Label na UI | Máx/mês | Regra |
|-------|-------------|---------|-------|
| action | **Feedback** | 30 pts | Média de satisfação pós Plano de Ação: ≤2,0=0 · 2,1–3,0=10 · 3,1–4,5=20 · >4,5=30 |
| alvo | **Planos de Ação** | sem teto | 10 pts por Plano de Ação concluído com "Sucesso" (vinculado a diagnóstico ALVO) |
| dev | **Positivação da Carteira** | 30 pts | % da carteira ativa (faturou ≥R$500): <50%=0 · 50–64,9%=6 · 65–74,9%=12 · 75–84,9%=18 · 85–94,9%=24 · ≥95%=30 |
| wow | **Evolução de Receita** | 35 pts | Crescimento MRR mês a mês: <0,2%=0 · 0,2–1,0%=7 · 1,1–1,6%=14 · 1,7–2,2%=21 · 2,3–2,8%=28 · ≥2,9%=35 |
| bonusPts + couponPts + commentPts + develPts + rankingAccessPts | **Engajamento** | variável | Desafios, Cupons Flash, Comentários (até 5 pts/dia), Acesso ao Desenvolve (10 pts/semana, manual), Acesso ao Ranking (10 pts/semana, automático) |

## Acesso ao Ranking (`rankingAccessPts`)
- Automático: se o executivo acessa o app em **≥3 dias distintos na semana** (contado via `/access_log`), credita **+10 pts** em `rankingAccessPts`
- Idempotência por semana em `/ranking_access_applied/{weekKey}/{execId}`, nunca reaplica na mesma semana
- **Edição manual de `rankingAccessPts` na aba Editar Pontos não é reaplicada automaticamente na mesma semana** — se o admin sobrescrever, a automação não soma por cima até a semana seguinte
- `couponPts`/`commentPts` são sempre seguros de editar manualmente (sem flag de dedup, puramente aditivos)

## Corrida da Semana (banner no Ranking)
- Compara `total(e)` atual com o snapshot de 7 dias atrás (`/points_snapshots`, salvo diariamente via `_savePointsSnapshot`)
- `computeWeeklyMovers()` pega os top 5 que mais ganharam pontos na semana e `renderRaceBanner()` anima a "corrida" no topo do Ranking
- Dispensa fica salva em `localStorage` (`raceBannerDismissed`) por semana; fecha sozinho após alguns minutos se o usuário não fechar

## Abas do app
- **Ranking** (`panel-ranking`) — tabela (coluna Executivo travada) + pódio top 3 em destaque + banner Corrida da Semana + explosão de confete
- **Premiações** (`panel-premiacoes`) — hero do prêmio mensal (ex.: vale-presente R$200) + histórico de ganhadores com filtro por vertical; adicionar/remover ganhador é só admin
- **Histórico** (`panel-historico`) — snapshots mensais de pontuação e comparação entre executivos; remoção de mês é só admin
- **Como Pontuar** (`panel-rules`) — explicação dos critérios (`CRITERIA`)
- **Editar Pontos** (`panel-edit`) — apenas admin; edita action/alvo/dev/wow/bonusPts/develPts por executivo, com legenda indicando o que é automático
- **Acessos** (`panel-access`) — apenas admin; log em tempo real do Firebase (`/access_log`) + resumo do Acesso ao Ranking
- **Desafios** (`panel-challenge`) — apenas admin; cria/encerra desafios com bônus de pontos + Cupons Flash no mesmo painel
- **Contestações** (`panel-contestacoes`) — painel de gestão é só admin; executivo abre contestação via modal (`openContestModal`)

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
Arquivos `index.backup-*.html` na raiz (não versionados no git, ignorados via `.gitignore`) — snapshots pontuais do `index.html` feitos antes de mudanças grandes. Ver os arquivos presentes na pasta para a lista atual.

## Backlog
1. Barra de progresso por executivo
2. Gráfico de evolução
3. Conquistas / Badges automáticos
4. Contagem regressiva do ciclo
5. Exportar ranking PDF/Print

> Concluído: Firebase Hosting, Histórico de pontuação mensal, Premiações, Contestações, Corrida da Semana.

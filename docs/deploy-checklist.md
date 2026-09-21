# Checklist de deploy — OOPA

Use este checklist antes de publicar em producao.

## Modelo de acesso e deploy

- O deployment Convex de producao e `avid-dog-344` e pertence a conta da ONG.
- Alunos desenvolvem em deployments proprios, criados nas proprias contas.
- O repositorio publico nao recebe credenciais de producao.
- O deploy de producao do backend ocorre em `.github/workflows/deploy.yml`,
  acionado por push em `main` ou manualmente usando a `main`.
- O ambiente GitHub `prod` precisa ter o **environment secret**
  `CONVEX_DEPLOY_KEY`, criado a partir de Settings > Deploy Keys do deployment
  `avid-dog-344` no Convex.
- A branch `main` deve exigir Pull Request e revisao; somente quem pode fazer
  merge em `main` pode iniciar um deploy de producao.

## Variaveis de ambiente

### GitHub Actions — ambiente `prod`

| Segredo | Obrigatorio | Onde configurar |
|---------|-------------|-----------------|
| `CONVEX_DEPLOY_KEY` | Sim | Settings > Environments > `prod` > Environment secrets |

`deploy.yml` declara `environment: prod`, então o GitHub libera esse secret
somente para o job de deploy. Ele não deve ser criado em Environment variables,
Repository variables ou commitado no repositorio.

### Convex (dashboard ou `npx convex env set`)

| Variavel | Obrigatoria | Descricao |
|----------|-------------|-----------|
| `CONVEX_DEPLOYMENT` | Nao (CI) | Usado apenas em desenvolvimento local; o CI identifica producao pela chave de deploy |
| `OCR_SPACE_API_KEY` | Sim (OCR) | Chave OCR.space para leitura de microchip |
| `OCR_SPACE_API_URL` | Não | Endpoint alternativo fornecido em planos PRO do OCR.space |
| `RESEND_API_KEY` | Sim (convites) | Envio de e-mail de convite e reset |
| `RESEND_FROM_EMAIL` | Sim (convites) | Remetente verificado no Resend |
| `SITE_URL` | Sim (convites) | URL pública usada nos links de convite e reset |

### Frontend (build Vite)

| Variavel | Obrigatoria | Descricao |
|----------|-------------|-----------|
| `VITE_CONVEX_URL` | Sim | URL do deployment Convex usado pelo app |

### Configuração do OCR.space

1. Solicitar uma chave em `https://ocr.space/ocrapi/freekey`.
2. Configurar o segredo no deployment Convex com
   `npx convex env set OCR_SPACE_API_KEY sua-chave`.
3. Manter `OCR_SPACE_API_URL` vazio no plano gratuito. Em um plano PRO,
   preencher com um dos endpoints recebidos por email.

O plano gratuito limita cada arquivo a 1 MB e aplica cotas por IP/dia e por
mês. O frontend converte a foto para JPEG e a reduz para até 700 KB antes de
chamar o action. A chave fica somente no backend Convex e nunca deve usar
prefixo `VITE_`.

## Publicacao do frontend

No Cloudflare Pages, configurar:

- Build command: `npm run build`
- Build output directory: `dist`
- `VITE_CONVEX_URL`: `https://avid-dog-344.convex.cloud`

O Pages pode ser conectado ao repositorio para publicar automaticamente a
branch `main`. A chave `CONVEX_DEPLOY_KEY` nunca deve ser configurada no
Cloudflare Pages nem exposta ao frontend.

## Comandos pre-deploy

```bash
npm run quality
npm run test:e2e
```

O deploy do Convex e executado automaticamente pelo GitHub Actions. Nao rode
`npx convex deploy` localmente apontando para o deployment da ONG.

## Pos-deploy

- [ ] Login com usuario admin de producao
- [ ] **Rodar `seeds:seedAll` em TODO deploy** (nao apenas em ambiente novo).
      Os seeds sao idempotentes por nome: reaplicar so insere o que falta.
      Toda fase que adiciona um tipo de ocorrencia ou bairro depende disso —
      sem reaplicar, a feature vai para producao sem o registro que ela
      procura e quebra em runtime (foi o que aconteceu com "Denuncia Externa"
      da Fase 16 e "Visita de acompanhamento" da Fase 22).
- [ ] Conferir contagem de tipos de ocorrencia com `seeds:getSeedSummary`
- [ ] Testar identificacao por camera/OCR
- [ ] Testar convite de usuario e e-mail
- [ ] Verificar notificacoes e exportacao de auditoria
- [ ] Confirmar HTTPS e dominio do `VITE_CONVEX_URL`

## Seguranca

- [ ] Nenhum segredo commitado no repositorio
- [ ] `seedAll` exige `templates.manage`
- [ ] Admin existente realinhado com o template atual (`bootstrap:ensureDevAdmin`
      sincroniza permissoes de modulos novos)
- [ ] Upload de arquivos exige permissao de escrita relevante
- [ ] Exportacoes operacionais exigem `system.audit_log`

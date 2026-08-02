# AZO Controle de Notas

App interno para controlar o crédito AZO (AutoZone): cadastro de clientes, anexo de notas com foto, assinatura digital para entregas por Uber, fechamento de ciclo com PDF, e arquivo permanente. Dados sincronizados em tempo real entre todos os aparelhos via Firebase Firestore.

## Como publicar (GitHub Pages — gratuito)

1. Crie um repositório novo no GitHub (pode ser privado).
2. Suba o arquivo `index.html` deste pacote para a raiz do repositório.
3. Nas configurações do repositório (**Settings → Pages**), em "Source" escolha a branch `main` e pasta `/ (root)`. Salve.
4. Em alguns minutos, o GitHub te dá um link fixo, tipo `https://seu-usuario.github.io/nome-do-repo/`.
5. Abra esse link pra confirmar que o app carrega.

## Antes de usar de verdade: travar a segurança do Firestore

O banco de dados foi criado em "modo de teste", que **expira sozinho em 30 dias** e fica aberto pra qualquer um com a chave. Antes de colocar em uso real:

1. Acesse o [Firebase Console](https://console.firebase.google.com) → seu projeto (`armazenamento-de-notas`) → **Firestore Database → Regras**.
2. Substitua o conteúdo por:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

3. Clique em **Publicar**.

Isso garante que só quem passou pelo login anônimo do próprio app (automático, invisível pro usuário) consegue ler ou escrever dados — ninguém de fora acessa direto pela internet.

## Primeiro acesso da equipe

- A primeira pessoa a abrir o link vai ver a tela "Primeiro acesso" e cria a **senha de acesso da equipe** (uma vez só, todo mundo usa a mesma depois).
- Ao tentar fechar o primeiro ciclo (arquivar notas pagas) ou excluir uma nota, o app pede pra cadastrar as **duas senhas de supervisor** — só essas duas autorizam essas ações daí em diante, e ficam registradas no log com nome de quem autorizou.

## "Instalar" no celular

Não está na Play Store. Cada pessoa abre o link publicado no navegador do celular e usa **"Adicionar à tela de início"** (Safari ou Chrome) — vira um ícone com o logo, abre em tela cheia, funciona como app normal.

## Estrutura de dados (Firestore)

- `clientes` — um documento por cliente (nome, PIN, telefone, e-mail, limite, datas do ciclo)
- `notas` — um documento por nota (veículo, valor, tipo, foto, assinatura, `arquivado: true/false`)
- `arquivos` — um resumo por ciclo fechado (totais, datas, quem autorizou)
- `config` — documento único `main` (link público do app, senha da equipe, as 2 senhas de supervisor)
- `log` — um documento por ação sensível (exclusão de nota, arquivamento de ciclo)

## Limite conhecido

Fotos muito grandes (nota física em alta resolução) podem ocasionalmente esbarrar no limite de 1 MB por documento do Firestore. O app já comprime as fotos automaticamente antes de salvar; se isso virar problema no dia a dia, dá pra migrar as fotos para o Firebase Storage (bucket já existe, configurado) em vez de guardá-las dentro do documento.

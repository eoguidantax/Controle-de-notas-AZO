# AZO Controle de Notas

App interno para controlar o crédito AZO (AutoZone): cadastro de clientes, anexo de notas com foto, assinatura digital para entregas por Uber, fechamento de ciclo com PDF, arquivo permanente e portal do cliente. Dados sincronizados em tempo real via Firebase Firestore.

## Como publicar (GitHub Pages — gratuito)

1. Crie um repositório novo no GitHub (precisa ser **público** para o GitHub Pages gratuito funcionar — as chaves do Firebase no código não são segredo, a segurança de verdade está nas regras do Firestore abaixo).
2. Suba o arquivo `index.html` deste pacote para a raiz do repositório.
3. Em **Settings → Pages**, escolha a branch `main` e pasta `/ (root)`. Salve.
4. Em alguns minutos, o GitHub dá um link fixo, tipo `https://seu-usuario.github.io/nome-do-repo/`.

## Antes de usar de verdade: ativar login por e-mail/senha (portal do cliente)

O app usa dois tipos de acesso: **login anônimo** (equipe, automático) e **login por e-mail/senha** (clientes, quando a loja libera o acesso deles). O segundo precisa ser ativado manualmente:

1. [Firebase Console](https://console.firebase.google.com) → projeto `armazenamento-de-notas` → **Build → Authentication → Sign-in method**.
2. Confirme que **Anonymous** está ativado (já devia estar).
3. Ative também **Email/Password**.

## Regras de segurança do Firestore (atualizar — importante)

Com o portal do cliente, agora existem dois níveis de acesso e a regra precisa diferenciar um do outro. Em **Firestore Database → Regras**, substitua tudo por:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isStaff() {
      return request.auth != null && request.auth.token.firebase.sign_in_provider == 'anonymous';
    }
    function isClientAccount() {
      return request.auth != null && request.auth.token.firebase.sign_in_provider == 'password';
    }
    function isOwnClient(clienteId) {
      return isClientAccount() &&
        get(/databases/$(database)/documents/clientes/$(clienteId)).data.email == request.auth.token.email;
    }

    match /clientes/{clientId} {
      allow read: if isStaff() || (isClientAccount() && resource.data.email == request.auth.token.email);
      allow write: if isStaff();
    }

    match /notas/{notaId} {
      allow read: if isStaff() || isOwnClient(resource.data.clienteId);
      allow create, delete: if isStaff();
      allow update: if isStaff() || (
        isOwnClient(resource.data.clienteId) &&
        request.resource.data.diff(resource.data).affectedKeys().hasOnly(['contestada','contestacaoMotivo','contestadaEm'])
      );
    }

    match /arquivos/{arquivoId} { allow read, write: if isStaff(); }
    match /config/{docId} { allow read, write: if isStaff(); }
    match /log/{logId} { allow read, write: if isStaff(); }
  }
}
```

O que isso garante:
- **Equipe** (login anônimo): acesso completo, como antes.
- **Cliente** (login por e-mail/senha): só enxerga o próprio cadastro e as próprias notas — nunca as de outro cliente. Só pode alterar os campos de contestação de uma nota (não pode editar valor, excluir, nem ver dados de outra pessoa).

⚠️ **Limitação conhecida:** qualquer pessoa que abra o app sem estar logada recebe automaticamente um acesso anônimo — hoje isso é tratado como "equipe" (acesso completo). Isso é aceitável para uso interno controlado, mas se o app crescer e for exposto amplamente, vale revisar esse modelo com a equipe de TI antes de escalar.

## Como liberar o acesso de um cliente ao portal

1. Cadastre o cliente com um e-mail válido.
2. Na tela do cliente, clique em **"📧 Enviar acesso"**.
3. O sistema cria a conta dele (se não existir) e manda um e-mail de redefinição de senha automaticamente — o cliente escolhe a própria senha por esse e-mail e depois entra no portal.
4. O link do portal do cliente é o mesmo link do app, com `#cliente` no final (ex: `https://seu-site.github.io/repo/#cliente`) — vale colocar esse link fixo em algum lugar visível pra eles (ex: fixado no WhatsApp da loja).

## Primeiro acesso da equipe

- A primeira pessoa a abrir o link (sem `#cliente`) cadastra seu nome e senha — vira o primeiro colaborador.
- Cada colaborador pode ter login próprio (nome + senha), cadastrável em **Configurações (⚙)**.
- Fechar um ciclo (mandar notas pagas pro arquivo morto) ou excluir uma nota/cliente exige a senha de um dos **2 supervisores** cadastrados — a primeira vez que alguém tenta uma dessas ações, o app pede pra cadastrar os dois.

## "Instalar" no celular

Não está na Play Store. Cada pessoa abre o link no navegador do celular e usa **"Adicionar à tela de início"** — vira um ícone, abre em tela cheia, funciona como app normal.

## Estrutura de dados (Firestore)

- `clientes` — nome, PIN, telefone, e-mail, limite, datas do ciclo
- `notas` — veículo, valor, tipo (faturamento/devolução), foto, assinatura, `arquivado`, `contestada`
- `arquivos` — resumo de cada ciclo fechado (totais, datas, quem autorizou)
- `config` — documento único `main` (link público, WhatsApp da loja, colaboradores, supervisores)
- `log` — uma entrada por ação sensível (exclusão, arquivamento)

## Limite conhecido

Fotos muito grandes podem ocasionalmente esbarrar no limite de 1 MB por documento do Firestore. O app já comprime as fotos automaticamente; se isso virar problema no dia a dia, dá pra migrar as fotos para o Firebase Storage (bucket já configurado no projeto).

# Instruções — página de Vozes (`/voices`)

> Documento para agentes de IA. **Leia antes de alterar** `voices.html` ou `vozes_custom.json`.
> A chave da API fica em `.env` (`FISH_API_KEY=...`), que está no `.gitignore` e **nunca** vai pro GitHub.

## ⛔ REGRA OBRIGATÓRIA — nunca commitar sem permissão

> **NUNCA** rode `git commit`, `git push`, `git tag` ou qualquer operação que altere o histórico
> do repositório **sem o usuário pedir explicitamente na conversa**.
> O usuário **sempre** quer testar a mudança **localmente** antes de qualquer coisa subir.

Como aplicar:

1. **Altere só os arquivos.** Deixe a mudança no working tree.
2. **Teste localmente** com `python -m http.server 8734` e abra `http://localhost:8734/voices.html`.
3. **Reporte** o que foi feito e como verificar, e **pare**.
4. Só faça `git add`/`commit`/`push` **depois** de o usuário pedir — e mesmo assim, confirme antes.

Não há exceção: nem "a mudança é óbvia", nem "já está testado", nem "vou adiantar". Push não
autorizado **apaga trabalho** do usuário (histórico reescrito, deploy do GitHub Pages disparado
sem aviso). Na dúvida, **pergunte**.

Se o usuário quiser o arquivo alterado **e** commitado, ele dirá isso — o commit é sempre um
pedido explícito, nunca uma etapa "óbvia" do trabalho.

## O que existe

| Arquivo | Papel |
| --- | --- |
| `vozes_custom.json` | Lista oficial das vozes do software: `[{ "nome": "!gojo", "id": "<id FishAudio>" }, ...]` |
| `voices.html` | A página `/voices`. Lê o JSON e mostra nome, ícone e amostra de som |
| `amostras/<id>.mp3` | Amostras geradas localmente **apenas** para vozes que a FishAudio não tem amostra pronta |
| `covers/<nome_da_voz>.png` | Capa própria de algumas vozes; o mapa fica em `LOCAL_COVERS` no `voices.html` (chave = nome da voz sem o `!`) |
| `.env` | Chave da API (local, gitignored) |

No GitHub Pages a página responde em
`https://viniciustixu.github.io/TTS_Muito_Foda/voices` (o `voices.html` também funciona).
A URL antiga `/vozes` continua funcionando: `vozes.html` agora é só um redirect pra `voices`.
A home virou `home.html` (`/home`); a raiz do site é um `index.html` que redireciona pra `home`.

## Pedido típico do usuário

> "atualizei o `vozes_custom.json`, atualiza a página de vozes"

Nesse caso **nada precisa ser reescrito na página**: ela lê o JSON em tempo de execução. O trabalho é
**validar as vozes novas** e **gerar amostra local para as que não têm som**:

1. Reconferir o JSON (formato `nome` começando com `!` e `id` de 32 caracteres hexadecimais).
   Vozes removidas do JSON desaparecem do site automaticamente; vozes novas aparecem na hora.
2. Para cada `id` **novo**, buscar os metadados na FishAudio (ver comando abaixo) e conferir:
   - o `id` existe? (se der 404, avisar o usuário — o id está errado ou a voz foi apagada)
   - `samples` tem áudio? Se tiver, **não precisa fazer nada**, a página usa a amostra pronta.
   - `samples` vazio? **Gerar a amostra** e salvar em `amostras/<id>.mp3` (passo 3).
3. Testar a página (passo 5) antes de dar commit/push.

### Comando: conferir uma voz

```powershell
# não precisa de chave para consultar
Invoke-RestMethod "https://api.fish.audio/model/<ID>" | Select-Object title, cover_image, @{n='samples';e={$_.samples.Count}}
```

### Comando: gerar amostra local (só para voz sem amostra pronta)

```powershell
$key = (Get-Content .env | Where-Object { $_ -match '^FISH_API_KEY=' }) -replace '^FISH_API_KEY=', ''
$body = @{
  text         = "Olá! Essa é a minha voz de demonstração. É só clicar no play para me ouvir."
  reference_id = "<ID>"
  format       = "mp3"
} | ConvertTo-Json -Compress
[IO.File]::WriteAllText("$env:TEMP\body.json", $body)
curl.exe -s -X POST "https://api.fish.audio/v1/tts" `
  -H "Authorization: Bearer $key" -H "Content-Type: application/json" `
  -H "model: s2.1-pro-free" --data "@$env:TEMP\body.json" `
  -o "amostras\<ID>.mp3"
```

- O texto da amostra pode ser adaptado ao personagem (usar PT-BR).
- `s2.1-pro-free` é a tier gratuita — mantém o custo em zero.
- Confirmar que o arquivo é MP3 válido (deve começar com os bytes `FF FB`).

## Como a página funciona (não mexer sem motivo)

- **Metadados**: `GET https://api.fish.audio/model/{id}` — **não precisa de chave**, libera CORS para o
  domínio do site. Retorna `title` (nome oficial), `cover_image` e `samples[].task_id`/`audio`.
- **Ícone/capa**: primeiro `LOCAL_COVERS[nome]` (imagem própria em `covers/`); se não tiver ou falhar,
  `https://public-platform.r2.fish.audio/cdn-cgi/image/width=160,format=webp/coverimage/{id}`
  (se falhar, tenta sem resize e por último mostra a inicial do nome). Para **adicionar capa de uma voz**:
  soltar o arquivo em `covers/` e criar a entrada `nome_da_voz: 'covers/arquivo.png'` em `LOCAL_COVERS`
  (o nome tem que bater com o `nome` do JSON sem o `!`).
- **Áudio**: `https://platform.r2.fish.audio/task/{task_id}.mp3` — URL pública e **sem expiração**
  na maioria das vezes (a `samples[].audio` que a API devolve é assinada e expira em 1h; a página só usa
  ela como reserva). **Exceção conhecida**: alguns `task_id` dão **404** nessa URL pública porque a Fish
  guardou o arquivo num caminho com data (`tasks/2026-08-16/...`) — caso do `!scooby_doo_2`
  (`433ef9237879464fb8a14635e5ccf905`). O `voices.html` trata isso: se o `<audio>` der erro, refaz o
  fetch dos metadados e toca a URL assinada fresca.
- **Voz sem amostra na Fish** (`samples: 0`) → a página usa `amostras/<id>.mp3`.
- **Gerar TTS direto do navegador não funciona**: o preflight CORS de `/v1/tts` responde 404.
  Por isso amostras faltantes são geradas aqui na máquina e salvas em `amostras/`.
- **Cache**: metadados ficam 24h em `localStorage` na chave `vozes_meta_v1`. Vozes novas são buscadas
  na hora; títulos alterados só aparecem depois de 24h. Para forçar atualização geral, basta subir a
  versão da chave no `voices.html` (`vozes_meta_v1` → `vozes_meta_v2`).
- **Nome exibido**: o `nome` do JSON sem o `!` (ex.: `gojo`) com o título oficial da FishAudio como
  subtítulo. A busca filtra pelos dois.
- **Nome da voz é clicável**: o título de cada card copia o comando completo pra área de transferência:
  **`!gojo `** — com o `!` no início **e um espaço no final** (pra usuário digitar a mensagem logo em
  seguida sem precisar apertar espaço). Passando o mouse no nome aparece o ícone
  `<span class="material-symbols-outlined">content_copy</span>`; depois do clique o ícone vira `check`
  (verde) por ~1,2s e um toast no rodapé confirma "Copiado !gojo". A página carrega a fonte
  **Material Symbols Outlined** (Google Fonts) no `<head>` — sem isso o ícone não aparece. O helper
  `copyText()` faz o fallback pra `document.execCommand('copy')` e é compartilhado com a cópia das
  tags de emoção.
- **Seção "Como usar"** (topo da página, card full width): instrução de usar `!nomedavoz + mensagem` e
  exemplo com tag de emoção (`!goku [happy]Oi, eu sou o Goku!`). Abaixo vem **5 botões de categoria**
  (Emoções, Emoções avançadas, Tom, Efeitos sonoros, Efeitos especiais) e **sempre fica um grupo aberto**:
  começa em "Emoções" e troca ao passar o mouse (ou focar / clicar) em outro botão — o botão ativo fica
  azul, o painel nunca some. Clicar numa tag copia ela. O texto acima dos botões é "Experimente algumas
  Tags:". Os campos acima da lista: depois do card vem a **busca**, menor e alinhada à direita
  (`sm:w-56`, com bastante espaço em relação ao card) — no mobile ela ocupa a largura toda. As tags
  estão no array `EMO_GROUPS` dentro do `voices.html` e seguem a lista oficial da FishAudio:
  <https://docs.fish.audio/developer-guide/core-features/emotions> — é essa a fonte se a lista mudar.

## Testar antes de publicar

```powershell
python -m http.server 8734   # na raiz do repositório
# abrir http://localhost:8734/voices.html
```

Checklist do teste: a lista abre com todos os itens do JSON, os ícones carregam, o play toca a amostra
(e pausa no segundo clique), trocar de voz para a anterior, e as vozes sem amostra da Fish tocam o mp3
de `amostras/`. `/voices` sem `.html` só funciona no GitHub Pages, não no servidor local.

## Não fazer

- **Nunca** colar a chave da API em `voices.html`, em outro HTML/JS ou em arquivo versionado.
- **Nunca** dar `git commit` / `git push` sem o usuário pedir explicitamente — testar **sempre** local
  primeiro (ver a regra obrigatória no topo do arquivo).
- Não commitar o `.env` nem `amostras/` com lixo de teste.
- Não trocar as URLs dos CDNs sem testar (são os hosts públicos da FishAudio).

## Estado conhecido (atualizado em 18/10/2026)

- 95 vozes no JSON; 94 com amostra pronta na FishAudio (`samples > 0`) e 1 sem:
  `!michael_jackson`.
- Amostras locais em `amostras/`: só `af1d2052ea8d441a972ca51a5bf547e4.mp3` (`!michael_jackson`).
  O arquivo de `!skeeper` foi apagado porque a voz saiu do JSON.
- Todas as vozes com amostra na Fish foram testadas (18/10): só `!scooby_doo_2` tem a URL pública do
  áudio com 404 — resolvido pelo fallback pra URL assinada no `voices.html`.
- 17 capas próprias em `covers/` mapeadas no `LOCAL_COVERS` (`!bob_sponja_2` ← `bob_sponja2.png` e
  `!coringa` ← `joker.png` são os dois casos em que o nome do arquivo difere do nome da voz);
  as outras 78 vozes continuam com a capa da FishAudio.
- Pedido repetido: **não fazer varredura em massa na API da FishAudio sem avisar antes** (os
  metadados são gratuitos e sem chave, mas o volume de request chama atenção).

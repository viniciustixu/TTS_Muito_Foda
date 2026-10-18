# Instruções — página de Vozes (`/voices`)

> Documento para agentes de IA. **Leia antes de alterar** `voices.html` ou `vozes_custom.json`.
> A chave da API fica em `.env` (`FISH_API_KEY=...`), que está no `.gitignore` e **nunca** vai pro GitHub.

## O que existe

| Arquivo | Papel |
| --- | --- |
| `vozes_custom.json` | Lista oficial das vozes do software: `[{ "nome": "!gojo", "id": "<id FishAudio>" }, ...]` |
| `voices.html` | A página `/voices`. Lê o JSON e mostra nome, ícone e amostra de som |
| `amostras/<id>.mp3` | Amostras geradas localmente **apenas** para vozes que a FishAudio não tem amostra pronta |
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
- **Ícone/capa**: `https://public-platform.r2.fish.audio/cdn-cgi/image/width=160,format=webp/coverimage/{id}`
  (se falhar, tenta sem resize e por último mostra a inicial do nome).
- **Áudio**: `https://platform.r2.fish.audio/task/{task_id}.mp3` — URL pública e **sem expiração**
  (a `samples[].audio` que a API devolve é assinada e expira em 1h; a página só usa ela como reserva).
- **Voz sem amostra na Fish** (`samples: 0`) → a página usa `amostras/<id>.mp3`.
- **Gerar TTS direto do navegador não funciona**: o preflight CORS de `/v1/tts` responde 404.
  Por isso amostras faltantes são geradas aqui na máquina e salvas em `amostras/`.
- **Cache**: metadados ficam 24h em `localStorage` na chave `vozes_meta_v1`. Vozes novas são buscadas
  na hora; títulos alterados só aparecem depois de 24h. Para forçar atualização geral, basta subir a
  versão da chave no `voices.html` (`vozes_meta_v1` → `vozes_meta_v2`).
- **Nome exibido**: o `nome` do JSON sem o `!` (ex.: `gojo`) com o título oficial da FishAudio como
  subtítulo. A busca filtra pelos dois.
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
- Não commitar o `.env` nem `amostras/` com lixo de teste.
- Não trocar as URLs dos CDNs sem testar (são os hosts públicos da FishAudio).

## Estado conhecido (atualizado em 16/10/2026)

- 90 vozes no JSON; 88 com amostra pronta na FishAudio.
- Amostras locais em `amostras/`: `94e794d95de94a38864d1d8930d903a5.mp3` (`!skeeper`) e
  `af1d2052ea8d441a972ca51a5bf547e4.mp3` (`!michael_jackson`).

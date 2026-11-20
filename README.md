# Gerador de cortes no Google Colab (OpenSource Clipping)

Guia para rodar o [OpenSource Clipping](#créditos) no **Google Colab com GPU gratuita**, transformando um vídeo longo do YouTube em cortes verticais (9:16) com legendas, prontos para Shorts, Reels e TikTok.

O fluxo usa:

- **yt-dlp** para baixar o vídeo
- **Faster-Whisper (large-v3, CUDA)** para transcrever com timestamp por palavra
- **Gemini** para escolher os melhores momentos
- **FFmpeg** para cortar, legendar e renderizar

> Este guia resolve o erro mais comum no Colab atual:
> `RuntimeError: Library libcublas.so.12 is not found or cannot be loaded`

---

## Sumário

1. [Requisitos](#requisitos)
2. [Configuração única](#configuração-única)
3. [Como usar](#como-usar)
4. [Célula 1: Setup](#célula-1-setup-uma-vez-por-sessão)
5. [Célula 2: Execução](#célula-2-execução-uma-vez-por-vídeo)
6. [Solução de problemas](#solução-de-problemas)
7. [Limites e custos](#limites-e-custos)
8. [Aviso sobre direitos autorais](#aviso-sobre-direitos-autorais)
9. [Créditos](#créditos)

---

## Requisitos

| Item | Detalhe |
|---|---|
| Conta Google | Para usar Colab, Drive e Gemini |
| Colab com GPU | Menu **Ambiente de execução → Alterar tipo → GPU (T4)** |
| Chave da API Gemini | Gerada no [Google AI Studio](https://aistudio.google.com/apikey) |
| Cookies do YouTube | Arquivo `cookies.txt` exportado do navegador (veja abaixo) |

---

## Configuração única

Faça isso uma vez só.

### 1. Guardar a chave do Gemini nos Segredos do Colab

1. No Colab, clique no ícone de chave 🔑 na barra lateral esquerda.
2. Clique em **Adicionar novo segredo**.
3. Nome: `GOOGLE_API_KEY`. Valor: sua chave.
4. Ative **Acesso ao notebook**.

> ⚠️ **Nunca cole a chave direto no código nem em chats.** Se subir o notebook para o GitHub com a chave dentro, qualquer pessoa pode usá-la. Se isso acontecer, revogue a chave e gere outra.

### 2. Exportar os cookies do YouTube

O YouTube costuma bloquear downloads automáticos sem login. Para contornar:

1. Instale no navegador uma extensão que exporte cookies no formato Netscape (por exemplo, "Get cookies.txt LOCALLY").
2. Abra o YouTube logado e exporte para um arquivo `www.youtube.com_cookies.txt`.
3. Envie o arquivo para o Google Drive, por exemplo em `Colab Notebooks/`.

> ⚠️ **Use uma conta secundária**, não a do seu canal. Downloads automatizados com a sua conta principal podem levar a bloqueios. Os cookies dão acesso à conta: nunca os suba para o GitHub.

---

## Como usar

O fluxo tem duas células:

1. **Sessão nova do Colab:** rode a **Célula 1 (Setup)** e depois a **Célula 2 (Execução)**.
2. **Mesma sessão, outro vídeo:** troque só a URL e rode a **Célula 2**.

O Colab apaga o ambiente quando a sessão cai ou é reiniciada. Por isso o Setup precisa ser refeito a cada sessão nova.

---

## Célula 1: Setup (uma vez por sessão)

A célula clona a versão BR do projeto. Troque `SEU_USUARIO` pelo seu usuário do GitHub (e o nome do repositório, se for diferente). O repositório original, em inglês, está em `https://github.com/NaufalRizqullah/opensource-clipping.git`.

```python
import os, glob, sysconfig, importlib.util
from google.colab import drive

drive.mount('/content/drive')

# 1. Clona o repositório e instala as dependências (só se ainda não existir)
REPO_URL = "https://github.com/HericJS/opensource-clipping-br"  # troque SEU_USUARIO e o nome, se for diferente
REPO_DIR = "/content/opensource-clipping"

if not os.path.exists(REPO_DIR):
    !git clone {REPO_URL} {REPO_DIR}
    !pip install -q -r {REPO_DIR}/requirements.txt

# 2. Instala as bibliotecas CUDA 12 que o faster-whisper precisa
!pip install -q nvidia-cublas-cu12 nvidia-cudnn-cu12

# 3. Aponta o LD_LIBRARY_PATH para elas
site_pkgs = sysconfig.get_paths()["purelib"]
paths = sorted(set(
    os.path.dirname(p)
    for pattern in ("libcublas.so.12*", "libcublasLt.so.12*", "libcudnn*.so.9*")
    for p in glob.glob(f"{site_pkgs}/nvidia/**/{pattern}", recursive=True)
))
os.environ["LD_LIBRARY_PATH"] = ":".join(paths) + ":" + os.environ.get("LD_LIBRARY_PATH", "")
print("Bibliotecas CUDA 12 encontradas em:")
print("\n".join(paths) if paths else "NENHUMA (veja Solução de problemas)")

# 4. Patch do faster-whisper (compatibilidade com o PyAV novo)
spec = importlib.util.find_spec("faster_whisper")
audio_py = os.path.join(spec.submodule_search_locations[0], "audio.py")
code = open(audio_py).read().replace(', metadata_errors="ignore"', '')
open(audio_py, "w").write(code)

print("✅ Setup completo")
```

O patch é seguro de repetir: se já foi aplicado, não muda nada.

---

## Célula 2: Execução (uma vez por vídeo)

```python
import os
from google.colab import userdata

%cd /content/opensource-clipping

chave = userdata.get("GOOGLE_API_KEY")
os.environ["GOOGLE_API_KEY"] = chave
os.environ["GEMINI_API_KEY"] = chave

VIDEO_URL = "COLE_A_URL_AQUI"  #@param {type:"string"}
COOKIES_PATH = "/content/drive/MyDrive/Colab Notebooks/www.youtube.com_cookies.txt"

!python main.py --url "{VIDEO_URL}" --yt-cookies "{COOKIES_PATH}"
```

### O que acontece durante a execução

1. **Download** do vídeo em alta qualidade (yt-dlp).
2. **Transcrição** na GPU com Whisper `large-v3`. Para um vídeo de ~18 minutos leva cerca de 2 minutos.
3. **Análise** pelo Gemini, que escolhe os melhores trechos.
4. **Renderização** dos clipes (9:16, legendas, B-roll, trilha, etc., conforme as opções do `main.py`).

Rode `python main.py --help` para ver as opções (quantidade de clipes, estilo de fonte, idioma e outras).

### Onde ficam os clipes

O projeto salva tudo na pasta `outputs/`, dentro do repositório:

| Arquivo | Conteúdo |
|---|---|
| `outputs/highlight_rank_N_ready.mp4` | Clipe final renderizado |
| `outputs/thumbnail_rank_N.jpg` | Thumbnail gerada |
| `outputs/render_manifest.json` | Metadados de todos os clipes |
| `outputs/metadata_preview.json` | Títulos, descrições e tags gerados pelo Gemini |

Como `/content` é apagado quando a sessão cai, **copie os clipes para o Drive logo após gerar**:

```python
!mkdir -p "/content/drive/MyDrive/Cortes"
!cp -r /content/opensource-clipping/outputs/. "/content/drive/MyDrive/Cortes/"
```

Se for processar vários vídeos seguidos, o arquivo `outputs/` pode crescer bastante. Use a flag `--cleanup-source` para apagar o vídeo original depois de renderizar, ou rode `bash cleanup.sh` para limpar os temporários.

### Opções úteis do `main.py`

| Flag | Para que serve |
|---|---|
| `--clips 10` | Quantidade de clipes (padrão: 7) |
| `--ratio "9:16"` | Proporção da saída (`9:16`, `16:9`, `1:1`, `3:4`, `4:5`) |
| `--font-style HORMOZI` | Estilo da legenda (`DEFAULT`, `STORYTELLER`, `HORMOZI`, `CINEMATIC`) |
| `--no-broll`, `--no-bgm`, `--no-hook`, `--no-subs` | Desliga B-roll, música, gancho ou legendas |
| `--use-dlp-subs` | Usa as legendas do YouTube e pula o Whisper quando existirem (mais rápido) |
| `--whisper-model small` | Modelo menor, mais rápido |
| `--whisper-device cpu --whisper-compute-type int8` | Transcrição sem GPU |
| `--url URL1 URL2` | Vários vídeos em sequência |

Rode `!python main.py --help` para a lista completa.

---

## Solução de problemas

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| `libcublas.so.12 is not found` | Colab vem com CUDA 13 e o ctranslate2 pede CUDA 12 | Rode a Célula 1 e **não reinicie** o runtime antes da Célula 2 |
| Lista de bibliotecas "NENHUMA" na Célula 1 | `nvidia-cublas-cu12` não instalou | Rode `!pip install nvidia-cublas-cu12 nvidia-cudnn-cu12` e `!find / -name "libcublas.so.12*" 2>/dev/null` |
| `NameError: userdata is not defined` | Faltou o import | Use `from google.colab import userdata` |
| `Pasta não encontrada: /content/opensource-clipping` | Sessão nova sem clonar o repo | Rode a Célula 1 |
| Aviso "conectado a GPU, mas não está usando a GPU" | Aviso informativo do Colab | Ignore. Não clique em "Alterar para ambiente padrão" |
| `No supported JavaScript runtime could be found` | yt-dlp sem o `deno` | Só um aviso. Se o download falhar ou vier em baixa qualidade, instale o `deno` |
| Download do YouTube falha | Cookies expirados | Exporte os cookies de novo |
| Erro `429` ou `quota exceeded` | Limite do Gemini estourado | Espere o reset ou ative o faturamento |
| GPU indisponível | Cota do Colab gratuito esgotada | Tente mais tarde ou use o Colab Pro |
| Erro depois de atualização do Colab | Mudou a versão do Python ou do CUDA | Veja a seção abaixo |

### Se o Colab mudar de versão

Esta solução depende do ambiente padrão do Colab (Python 3.13 e CUDA 13 na data de escrita). Se algo quebrar depois de uma atualização:

1. Confira a versão do Python com `!python --version`.
2. Rode a Célula 1 de novo e verifique se as bibliotecas CUDA 12 aparecem na lista.
3. Se persistir, fixe versões compatíveis de `ctranslate2` e `faster-whisper`.
4. Como último recurso, troque o modelo para `medium` ou `small` e rode em CPU (bem mais lento).

---

## Limites e custos

O uso básico é gratuito, mas com limites:

- **Colab gratuito:** cota de GPU variável, sessões com tempo máximo e desconexão por inatividade. Ao terminar, use **Ambiente de execução → Desconectar e excluir ambiente de execução** para poupar a cota.
- **Gemini API:** o plano gratuito tem limites por minuto e por dia, e modelos "preview" costumam ter limites mais apertados. Consulte os valores atuais no AI Studio.
- **Open source:** o repositório, o yt-dlp e o Whisper não têm custo.

Para volume alto, o Colab Pro e o faturamento do Gemini dão mais estabilidade.

---

## Aviso sobre direitos autorais

Cortar e republicar vídeos de terceiros pode violar direitos autorais e as políticas do YouTube. Isso vale ainda mais para **compilados**, que já reutilizam conteúdo de outros criadores.

Riscos principais:

- **Content ID:** reivindicação automática, com perda da receita ou bloqueio do vídeo.
- **Strikes de direitos autorais:** três strikes podem encerrar o canal.
- **Política de conteúdo reutilizado:** canais que só republicam trechos podem ter a monetização negada.

Dar crédito na descrição **não** garante proteção.

Boas práticas:

1. Peça **autorização** ao dono do vídeo.
2. Prefira vídeos com **licença que permita** o uso (Creative Commons ou criadores que liberam cortes).
3. **Acrescente conteúdo próprio** (comentário, narração, contexto), em vez de só cortar e legendar.
4. Teste com poucos vídeos antes de montar um fluxo em volume.

O autor deste guia não se responsabiliza pelo uso que cada pessoa fizer da ferramenta.

---

## Créditos

- **[OpenSource Clipping](https://github.com/NaufalRizqullah/opensource-clipping)**, de [NaufalRizqullah](https://github.com/NaufalRizqullah): projeto original usado neste fluxo, sob licença MIT.
- [faster-whisper](https://github.com/SYSTRAN/faster-whisper), [yt-dlp](https://github.com/yt-dlp/yt-dlp), [FFmpeg](https://ffmpeg.org/) e a API Gemini.

Este guia documenta a configuração no Colab e as adaptações para o público brasileiro. O código do clipping pertence ao autor original. Ao publicar uma versão modificada, mantenha o arquivo `LICENSE` e o aviso de copyright do projeto original, como a licença MIT exige.

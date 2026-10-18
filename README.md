"""Gera um YouTube Short vertical (1080x1920) a partir de roteiro + termos de busca.

Entradas (variaveis de ambiente): ROW, TITULO, ROTEIRO, TERMOS, PIXABAY_KEY
Saida: out/short_<ROW>.mp4
"""
import asyncio
import json
import math
import os
import random
import subprocess
import sys
from pathlib import Path

import edge_tts
import requests

ROW = os.environ.get("ROW", "0").strip() or "0"
ROTEIRO = os.environ["ROTEIRO"].strip()
TERMOS = [t.strip() for t in os.environ.get("TERMOS", "").split(",") if t.strip()]
PIXABAY_KEY = os.environ["PIXABAY_KEY"].strip()

VOZ = "pt-BR-AntonioNeural"
VELOCIDADE = "-5%"
TOM = "-8Hz"
SEG_SEGUNDOS = 5          # duracao de cada clipe de fundo
PALAVRAS_POR_LEGENDA = 3
FALLBACK = ["dark forest fog", "night sky stars", "abandoned building", "dark ocean waves"]

WORK = Path("work")
OUT = Path("out")
WORK.mkdir(exist_ok=True)
OUT.mkdir(exist_ok=True)


def run(cmd):
    print("+", " ".join(str(c) for c in cmd), flush=True)
    subprocess.run(cmd, check=True)


def srt_time(seg):
    ms = int(round(seg * 1000))
    h, ms = divmod(ms, 3_600_000)
    m, ms = divmod(ms, 60_000)
    s, ms = divmod(ms, 1000)
    return f"{h:02}:{m:02}:{s:02},{ms:03}"


async def narrar():
    """Gera work/narracao.mp3 e work/legenda.srt."""
    try:
        com = edge_tts.Communicate(ROTEIRO, VOZ, rate=VELOCIDADE, pitch=TOM, boundary="WordBoundary")
    except TypeError:  # versoes antigas nao tem o parametro boundary
        com = edge_tts.Communicate(ROTEIRO, VOZ, rate=VELOCIDADE, pitch=TOM)

    palavras = []  # (inicio_s, fim_s, texto)
    with open(WORK / "narracao.mp3", "wb") as f:
        async for chunk in com.stream():
            if chunk["type"] == "audio":
                f.write(chunk["data"])
            elif chunk["type"] in ("WordBoundary", "SentenceBoundary"):
                ini = chunk["offset"] / 10_000_000
                dur = chunk["duration"] / 10_000_000
                partes = chunk["text"].split()
                if not partes:
                    continue
                passo = dur / len(partes)
                for i, p in enumerate(partes):
                    palavras.append((ini + i * passo, ini + (i + 1) * passo, p))

    if not palavras:
        raise SystemExit("Edge TTS nao devolveu marcacao de tempo das palavras.")

    linhas = []
    for i in range(0, len(palavras), PALAVRAS_POR_LEGENDA):
        grupo = palavras[i:i + PALAVRAS_POR_LEGENDA]
        ini, fim = grupo[0][0], grupo[-1][1]
        texto = " ".join(p[2] for p in grupo)
        linhas.append(f"{len(linhas) + 1}\n{srt_time(ini)} --> {srt_time(fim)}\n{texto}\n")
    (WORK / "legenda.srt").write_text("\n".join(linhas), encoding="utf-8")


def duracao_audio():
    r = subprocess.run(
        ["ffprobe", "-v", "error", "-show_entries", "format=duration",
         "-of", "default=nw=1:nk=1", str(WORK / "narracao.mp3")],
        capture_output=True, text=True, check=True)
    return float(r.stdout.strip())


def buscar_videos(termo, quantidade=6):
    try:
        r = requests.get(
            "https://pixabay.com/api/videos/",
            params={"key": PIXABAY_KEY, "q": termo, "per_page": quantidade, "safesearch": "true"},
            timeout=30)
        r.raise_for_status()
        urls = []
        for hit in r.json().get("hits", []):
            v = hit.get("videos", {})
            alvo = v.get("medium", {}).get("url") or v.get("large", {}).get("url") or v.get("small", {}).get("url")
            if alvo:
                urls.append(alvo)
        return urls
    except Exception as e:  # nao derruba o job por causa de um termo
        print(f"Aviso: busca '{termo}' falhou: {e}", file=sys.stderr)
        return []


def baixar(url, destino):
    with requests.get(url, stream=True, timeout=120) as r:
        r.raise_for_status()
        with open(destino, "wb") as f:
            for parte in r.iter_content(chunk_size=1 << 20):
                f.write(parte)


def montar_fundo(duracao):
    """Baixa clipes, recorta para 9:16 e concatena em work/fundo.mp4."""
    precisa = math.ceil(duracao / SEG_SEGUNDOS) + 1
    candidatos = []
    for termo in TERMOS + FALLBACK:
        for u in buscar_videos(termo):
            if u not in candidatos:
                candidatos.append(u)
        if len(candidatos) >= precisa * 2:
            break
    if not candidatos:
        raise SystemExit("Nenhum video encontrado na Pixabay. Confira a chave PIXABAY_KEY.")

    # mistura um pouco, mas mantem os primeiros termos (mais relevantes) no inicio
    topo, resto = candidatos[:precisa], candidatos[precisa:]
    random.shuffle(resto)
    fila = (topo + resto)[:precisa]

    segmentos = []
    for i in range(precisa):
        url = fila[i % len(fila)]
        bruto = WORK / f"bruto_{i}.mp4"
        seg = WORK / f"seg_{i}.mp4"
        try:
            baixar(url, bruto)
            run(["ffmpeg", "-y", "-loglevel", "error", "-i", str(bruto), "-t", str(SEG_SEGUNDOS),
                 "-vf", "scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,"
                        "fps=30,setsar=1,eq=brightness=-0.08:saturation=0.75",
                 "-an", "-c:v", "libx264", "-preset", "veryfast", "-crf", "23",
                 "-pix_fmt", "yuv420p", str(seg)])
            segmentos.append(seg)
        except Exception as e:
            print(f"Aviso: clipe {i} ignorado: {e}", file=sys.stderr)

    if not segmentos:
        raise SystemExit("Nenhum clipe pode ser preparado.")

    lista = WORK / "lista.txt"
    lista.write_text("".join(f"file '{s.name}'\n" for s in segmentos), encoding="utf-8")
    run(["ffmpeg", "-y", "-loglevel", "error", "-f", "concat", "-safe", "0",
         "-i", str(lista), "-c", "copy", str(WORK / "fundo.mp4")])


def renderizar(duracao):
    estilo = ("FontName=DejaVu Sans,Bold=1,FontSize=12,PrimaryColour=&H00FFFFFF,"
              "OutlineColour=&H00000000,BorderStyle=1,Outline=3,Shadow=0,Alignment=2,MarginV=60")
    filtro = f"vignette=PI/4,subtitles=legenda.srt:force_style='{estilo}'"
    saida = OUT / f"short_{ROW}.mp4"
    run(["ffmpeg", "-y", "-loglevel", "error",
         "-stream_loop", "-1", "-i", str(WORK / "fundo.mp4"),
         "-i", str(WORK / "narracao.mp3"),
         "-vf", filtro.replace("legenda.srt", str(WORK / "legenda.srt")),
         "-map", "0:v", "-map", "1:a",
         "-t", f"{duracao + 0.6:.2f}",
         "-c:v", "libx264", "-preset", "veryfast", "-crf", "23", "-pix_fmt", "yuv420p",
         "-c:a", "aac", "-b:a", "160k", "-shortest",
         "-movflags", "+faststart", str(saida)])
    print(f"Video pronto: {saida}")


def main():
    asyncio.run(narrar())
    dur = duracao_audio()
    print(f"Duracao da narracao: {dur:.1f}s")
    if dur > 59:
        print("Aviso: narracao passa de 59s, pode nao ser aceita como Short.", file=sys.stderr)
    montar_fundo(dur)
    renderizar(dur)


if __name__ == "__main__":
    main()

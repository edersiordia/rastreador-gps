from fastapi import FastAPI, HTTPException
from fastapi.responses import HTMLResponse
from pydantic import BaseModel

app = FastAPI()

ubicaciones = {}
entregados = set()

class Ubicacion(BaseModel):
    lat: float
    lng: float

@app.post("/ubicacion/{pedido_id}")
def actualizar_ubicacion(pedido_id: str, ubicacion: Ubicacion):
    if pedido_id in entregados:
        raise HTTPException(status_code=403, detail="Pedido ya entregado")
    ubicaciones[pedido_id] = {"lat": ubicacion.lat, "lng": ubicacion.lng}
    return {"mensaje": "Ubicación actualizada"}

@app.get("/ubicacion/{pedido_id}")
def obtener_ubicacion(pedido_id: str):
    if pedido_id in entregados:
        raise HTTPException(status_code=403, detail="Pedido ya fue entregado")
    ubicacion = ubicaciones.get(pedido_id)
    if not ubicacion:
        raise HTTPException(status_code=404, detail="Ubicación no disponible")
    return ubicacion

@app.post("/entregado/{pedido_id}")
def marcar_entregado(pedido_id: str):
    entregados.add(pedido_id)
    ubicaciones.pop(pedido_id, None)
    return {"mensaje": "Pedido marcado como entregado"}

@app.get("/mapa/{pedido_id}", response_class=HTMLResponse)
def mostrar_mapa(pedido_id: str):
    if pedido_id in entregados:
        return HTMLResponse(content="<h2>Pedido ya fue entregado</h2>", status_code=403)

    ubicacion = ubicaciones.get(pedido_id)
    if not ubicacion:
        return HTMLResponse(content="<h2>Ubicación no disponible</h2>", status_code=404)

    lat = ubicacion["lat"]
    lng = ubicacion["lng"]

    html_content = f"""
    <!DOCTYPE html>
    <html lang="es">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Restaurante Panamá - Pedido en camino</title>
        <style>
            * {{
                margin: 0;
                padding: 0;
                box-sizing: border-box;
            }}
            body {{
                font-family: Arial, sans-serif;
                display: flex;
                flex-direction: column;
                height: 100vh;
                background-color: #f9f9f9;
            }}
            header {{
                background-color: #05163c; /* Azul cielo */
                color: white;
                text-align: center;
                padding: 0.6rem;
                font-size: 1.3rem;
                font-weight: bold;
                box-shadow: 0 2px 4px rgba(0,0,0,0.1);
                line-height: 1.6;
                display: flex;
                flex-direction: column;
                justify-content: center;
            }}
            #map {{
                flex: 1;
                width: 100%;
            }}
        </style>
    </head>
    <body>
        <header>
            Su pedido está en camino.<br>
            Restaurante Panamá
        </header>
        <div id="map"></div>
        <script>
            function initMap() {{
                var ubicacion = {{ lat: {lat}, lng: {lng} }};
                var map = new google.maps.Map(document.getElementById('map'), {{
                    zoom: 15,
                    center: ubicacion,
                    gestureHandling: "greedy"
                }});
                new google.maps.Marker({{
                    position: ubicacion,
                    map: map,
                    title: "Tu pedido va en camino",
                    icon: {{
                        url: "https://cdn-icons-png.flaticon.com/512/2991/2991129.png",
                        scaledSize: new google.maps.Size(45, 45)
                    }}
                }});
            }}
        </script>
        <script src="https://maps.googleapis.com/maps/api/js?key=AIzaSyCxJ5qr-bArQ2CEUPF9t1WRIgDGRn-t4_w&callback=initMap" async defer></script>
    </body>
    </html>
    """

    return HTMLResponse(content=html_content)


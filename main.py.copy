from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import HTMLResponse
from pydantic import BaseModel
import firebase_admin
from firebase_admin import credentials, db
from datetime import datetime
from zoneinfo import ZoneInfo  # Solo disponible en Python 3.9+
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse

app = FastAPI()

# Inicializar Firebase
cred = credentials.Certificate("firebase_key.json")
firebase_admin.initialize_app(cred, {
    'databaseURL': 'https://rastreador-gps2-default-rtdb.firebaseio.com'
})

class Ubicacion(BaseModel):
    lat: float
    lng: float

# Hostea el archivo HTML directamente desde FastAPI sin nginx.
app.mount("/static", StaticFiles(directory="static"), name="static")

@app.get("/", response_class=FileResponse)
def serve_tracking_interface():
    return "static/tracking_interface.html"

@app.post("/ubicacion/{pedido_id}")
def actualizar_ubicacion(pedido_id: str, ubicacion: Ubicacion):
    entregado = db.reference(f'entregados/{pedido_id}').get()
    if entregado is True:
        raise HTTPException(status_code=403, detail="Pedido ya entregado")

    db.reference(f'ubicaciones/{pedido_id}').set({
        "lat": ubicacion.lat,
        "lng": ubicacion.lng
    })
    return {"mensaje": "Ubicación actualizada"}

@app.post("/owntracks/{usuario}")
async def recibir_de_owntracks(usuario: str, request: Request):
    data = await request.json()
    msg_type = data.get("_type")

    if msg_type != "location":
        print(f"Ignorado mensaje tipo: {msg_type}")
        return {"mensaje": f"Mensaje tipo {msg_type} ignorado"}

    lat = data.get("lat")
    lon = data.get("lon")

    if lat is None or lon is None:
        raise HTTPException(status_code=400, detail="Faltan coordenadas")

    timestamp = datetime.now(ZoneInfo("America/Mazatlan")).isoformat()

    print(f"[OwnTracks] {usuario}: lat={lat}, lng={lon}, time={timestamp}")

    db.reference(f'ubicaciones/{usuario}').set({
        "lat": lat,
        "lng": lon,
        "timestamp": timestamp
    })

    return {"mensaje": f"Ubicación de {usuario} recibida"}

@app.get("/ubicacion/{pedido_id}")
def obtener_ubicacion(pedido_id: str):
    entregado = db.reference(f'entregados/{pedido_id}').get()
    if entregado is True:
        raise HTTPException(status_code=403, detail="Pedido ya fue entregado")

    ubicacion = db.reference(f'ubicaciones/{pedido_id}').get()
    if not ubicacion:
        raise HTTPException(status_code=404, detail="Ubicación no disponible")
    return ubicacion

@app.post("/entregado/{pedido_id}")
def marcar_entregado(pedido_id: str):
    db.reference(f'entregados/{pedido_id}').set(True)
    db.reference(f'ubicaciones/{pedido_id}').delete()
    return {"mensaje": "Pedido marcado como entregado"}

@app.get("/mapa/{pedido_id}", response_class=HTMLResponse)
def mostrar_mapa(pedido_id: str):
    entregado = db.reference(f'entregados/{pedido_id}').get()

    if entregado is True:
        html_entregado = """
        <!DOCTYPE html>
        <html lang=\"es\">
        <head>
            <meta charset=\"UTF-8\">
            <meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">
            <title>Pedido Entregado</title>
            <style>
                body { background: #ffffff; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; font-family: 'Poppins', sans-serif; }
                .mensaje { text-align: center; }
                .mensaje h1 { color: #00cc66; font-size: 3rem; margin-bottom: 1rem; animation: pulso 1.5s infinite; }
                .mensaje p { color: #333; font-size: 1.2rem; }
                @keyframes pulso { 0% { transform: scale(1); } 50% { transform: scale(1.05); } 100% { transform: scale(1); } }
            </style>
        </head>
        <body>
            <div class=\"mensaje\">
                <h1>¡Pedido Entregado!</h1>
                <p>Gracias por confiar en nosotros.</p>
            </div>
        </body>
        </html>
        """
        return HTMLResponse(content=html_entregado, status_code=200)

    html_content = f"""
    <!DOCTYPE html>
    <html lang=\"es\">
    <head>
      <meta charset=\"UTF-8\">
      <meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">
      <title>Pedido en camino</title>
      <style>
        html, body {{ margin: 0; padding: 0; height: 100%; font-family: Arial, sans-serif; display: flex; flex-direction: column; }}
        header {{ background-color: #05163c; color: white; text-align: center; padding: 1rem; font-size: 1.4rem; font-weight: bold; box-shadow: 0 2px 4px rgba(0,0,0,0.1); line-height: 1.5; }}
        #map {{ flex: 1; width: 100%; }}
      </style>
    </head>
    <body>
      <header>Pedido en camino.<br>Restaurante Panamá</header>
      <div id=\"map\"></div>
      <script src=\"https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js\"></script>
      <script src=\"https://www.gstatic.com/firebasejs/9.22.1/firebase-database-compat.js\"></script>
      <script>
        const firebaseConfig = {{
          apiKey: \"AIzaSyCxJ5qr-bArQ2CEUPF9t1WRIgDGRn-t4_w\",
          authDomain: \"rastreador-gps2-default-rtdb.firebaseapp.com\",
          databaseURL: \"https://rastreador-gps2-default-rtdb.firebaseio.com\",
          projectId: \"rastreador-gps2-default-rtdb\",
          storageBucket: \"rastreador-gps2-default-rtdb.appspot.com\",
          messagingSenderId: \"...\",
          appId: \"...\"
        }};
        firebase.initializeApp(firebaseConfig);
        const db = firebase.database();
        let map, marker;
        function initMap() {{
          map = new google.maps.Map(document.getElementById(\"map\"), {{ zoom: 15, center: {{ lat: 0, lng: 0 }}, gestureHandling: \"greedy\" }});
          marker = new google.maps.Marker({{ map: map, title: \"Tu pedido va en camino\", icon: {{ url: \"https://cdn-icons-png.flaticon.com/512/2991/2991129.png\", scaledSize: new google.maps.Size(35, 35) }} }});
          db.ref(\"entregados/{pedido_id}\").on(\"value\", (snapshot) => {{
            if (snapshot.val() === true) {{
              document.body.innerHTML = `<div style='background:#ffffff;display:flex;justify-content:center;align-items:center;height:100vh;margin:0;font-family:Poppins,sans-serif;'><div class='mensaje' style='text-align:center;'><h1 style='color:#00cc66;font-size:3rem;margin-bottom:1rem;animation:pulso 1.5s infinite;'>&iexcl;Pedido Entregado!</h1><p style='color:#333;font-size:1.2rem;'>Gracias por confiar en nosotros.</p></div></div><style>@keyframes pulso {{0% {{transform: scale(1);}} 50% {{transform: scale(1.05);}} 100% {{transform: scale(1);}}}}</style>`;
            }}
          }});
        }}
        firebase.database().ref(\"ubicaciones/{pedido_id}\").on(\"value\", (snapshot) => {{
          const data = snapshot.val();
          if (data) {{
            const nuevaUbicacion = {{ lat: data.lat, lng: data.lng }};
            marker.setPosition(nuevaUbicacion);
            map.setCenter(nuevaUbicacion);
          }}
        }});
      </script>
      <script src=\"https://maps.googleapis.com/maps/api/js?key=AIzaSyCxJ5qr-bArQ2CEUPF9t1WRIgDGRn-t4_w&callback=initMap\" async defer></script>
    </body>
    </html>
    """
    return HTMLResponse(content=html_content)


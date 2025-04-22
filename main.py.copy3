from fastapi import FastAPI, HTTPException
from fastapi.responses import HTMLResponse
from pydantic import BaseModel
import firebase_admin
from firebase_admin import credentials, db

app = FastAPI()


# Inicializa Firebase
cred = credentials.Certificate("firebase_key.json")
firebase_admin.initialize_app(cred, {
    'databaseURL': 'https://rastreador-gps2-default-rtdb.firebaseio.com'  # Reemplaza con tu URL real
})

class Ubicacion(BaseModel):
    lat: float
    lng: float

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
        return HTMLResponse(content="<h2>Pedido ya fue entregado</h2>", status_code=403)

    html_content = f"""
    <!DOCTYPE html>
    <html lang="es">
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Rastreo en vivo</title>
      <style>
        html, body, #map {{
          height: 100%;
          margin: 0;
          padding: 0;
        }}
      </style>
    </head>
    <body>
      <div id="map"></div>

      <!-- Firebase SDK -->
      <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-app-compat.js"></script>
      <script src="https://www.gstatic.com/firebasejs/9.22.1/firebase-database-compat.js"></script>

      <script>
        const firebaseConfig = {{
          apiKey: "AIzaSyCxJ5qr-bArQ2CEUPF9t1WRIgDGRn-t4_w",
          authDomain: "rastreador-gps2-default-rtdb.firebaseapp.com",
          databaseURL: "https://rastreador-gps2-default-rtdb.firebaseio.com", 
          projectId: "rastreador-gps2-default-rtdb",
          storageBucket: "rastreador-gps2-default-rtdb.appspot.com",
          messagingSenderId: "...",
          appId: "..."
        }};
        firebase.initializeApp(firebaseConfig);
        const db = firebase.database();

        let map, marker;

        function initMap() {{
          map = new google.maps.Map(document.getElementById("map"), {{
            zoom: 15,
            center: {{ lat: 0, lng: 0 }},
            gestureHandling: "greedy"
          }});

          marker = new google.maps.Marker({{
            map: map,
            title: "Tu pedido va en camino",
            icon: {{
              url: "https://cdn-icons-png.flaticon.com/512/2991/2991129.png",
              scaledSize: new google.maps.Size(35, 35)
            }}
          }});
        }}

        firebase.database().ref("ubicaciones/{pedido_id}").on("value", (snapshot) => {{
          const data = snapshot.val();
          if (data) {{
            const nuevaUbicacion = {{ lat: data.lat, lng: data.lng }};
            marker.setPosition(nuevaUbicacion);
            map.setCenter(nuevaUbicacion);
          }}
        }});
      </script>

      <script src="https://maps.googleapis.com/maps/api/js?key=AIzaSyCxJ5qr-bArQ2CEUPF9t1WRIgDGRn-t4_w&callback=initMap" async defer></script>
    </body>
    </html>
    """

    return HTMLResponse(content=html_content)


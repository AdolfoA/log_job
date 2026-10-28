# Bitácora Adolfo

Página estática (HTML + JSON, sin dependencias) para registrar consultas y cambios de los chats `Adolfo [tema]`.

## Desplegar en GitHub Pages
1. Sube esta carpeta a un repo.
2. Settings → Pages → Deploy from a branch → `main` / root.
3. Abre `https://<usuario>.github.io/<repo>/`.

## Cómo se registra
- Las entradas del repo viven en `data/entries.json`.
- En la página, "+ Nueva entrada" guarda en este navegador. "Exportar JSON" descarga el archivo completo para reemplazar `data/entries.json` y hacer commit.
- En cada chat puedes pedirme "registra este cambio" y te entrego la entrada lista para pegar en `data/entries.json`.

## Probar en local
    python -m http.server 8000   # y abrir http://localhost:8000

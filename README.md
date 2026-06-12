# Mundial 2026 — App nativa iOS + Android (Capacitor)

Migración de la app web del fixture a aplicaciones nativas usando **Capacitor**: un solo código (`www/index.html`) genera un proyecto Xcode (iOS) y un proyecto Android Studio (Android) reales, publicables en App Store y Google Play.

## Qué hace la app (v2)

- Fixture completo de fase de grupos en hora argentina (UTC−3), con los colores de transmisión del Excel original.
- **Favoritos**: ★ en cada partido. El próximo favorito aparece arriba con cuenta regresiva, igual que los partidos de Argentina.
- **Grupos**: pestaña con la tabla de posiciones de cada grupo (PJ, G, E, P, GF, GC, DIF, PTS, calculada con los resultados reales) y los partidos solo de ese grupo.
- **Resultados en tiempo real**: la app consulta las fuentes automáticamente — cada 1 minuto mientras hay partidos en juego, cada 10 minutos el resto del día — y refleja goles, finales, goleadores y posiciones. Con la clave de football-data.org, el "FINAL" aparece en el primer ciclo de actualización después del pitazo (≤ 2 minutos).
- **MVP**: ~10 minutos después del final muestra el jugador mejor valorado del partido (requiere clave de API-Football).
- Agregar cualquier partido al calendario (.ics / Google Calendar) con aviso 30 min antes.

## Fuentes de datos (elegidas por confiabilidad y costo)

| Fuente | Clave | Qué aporta |
|---|---|---|
| openfootball (raw.githubusercontent.com) | No necesita | Resultados finales + goleadores. Gratuita, datos de dominio público. Es el modo por defecto: la app funciona sin configurar nada. |
| football-data.org | Gratuita (registro) | Estado EN VIVO y final inmediato. El plan gratis incluye el Mundial, 10 consultas/min. Cargarla en Ajustes. |
| API-Football (api-sports.io) | Gratuita (100 req/día) | Valoraciones de jugadores → MVP del partido. La app cuida la cuota (consultas espaciadas y cacheadas). |

**Honestidad sobre el MVP:** la FIFA no publica el premio oficial "Player of the Match" en ninguna API pública. Lo que la app muestra es el jugador con mejor rating estadístico del partido según API-Football, etiquetado como MVP. Es el estándar de facto que usan las apps de resultados.

## Cómo compilar

Requisitos: Node 18+. Para iOS: una Mac con Xcode 15+. Para Android: Android Studio.

```bash
npm install

# Android
npm run android        # crea el proyecto, sincroniza y abre Android Studio
# → Build > Generate Signed Bundle/APK para publicar

# iOS (en Mac)
npm run ios            # crea el proyecto, sincroniza y abre Xcode
# → firmar con tu Apple Developer Team y archivar
```

Después de `cap add`, aplicar los archivos de `config-extras/` (ver abajo). Si modificás `www/index.html`, corré `npm run sync`.

## Checklist de seguridad (ya aplicado o por aplicar al compilar)

**Diseño (ya en el código):**
- **Minimización de datos**: la app no tiene cuentas, no pide ningún permiso (solo INTERNET en Android), no usa SDKs de terceros, analytics ni publicidad. No hay datos personales que robar: nada sale del dispositivo.
- **Content-Security-Policy** estricta en el HTML: solo puede conectarse a las 3 fuentes de datos declaradas; `frame-src 'none'`, `object-src 'none'`, `base-uri 'none'`.
- Todo el contenido que viene de las APIs se **escapa antes de renderizar** (función `esc()`), bloqueando XSS aunque una fuente fuera comprometida.
- Solo HTTPS; `cleartext: false` y `androidScheme: https` en `capacitor.config.json`.
- `limitsNavigationsToAppBoundDomains` (iOS): el WebView no puede navegar fuera de la app.
- Claves de API y favoritos guardados solo localmente (Capacitor Preferences).

**Al compilar Android:**
1. Copiar `config-extras/android/network_security_config.xml` a `android/app/src/main/res/xml/` → bloquea HTTP plano y restringe la red a los 5 dominios necesarios.
2. Copiar `backup_rules.xml` ahí mismo → excluye las claves de API de los backups en la nube.
3. Aplicar `manifest_snippet.txt` en `AndroidManifest.xml`.
4. Publicar como **App Bundle firmado** con Play App Signing activado.

**Al compilar iOS:**
1. Agregar `config-extras/ios/PrivacyInfo.xcprivacy` al target (declara: sin rastreo, sin recolección de datos — requerido por App Store).
2. Aplicar `info_plist_snippet.txt` (App Transport Security sin excepciones + dominios acotados).

**Opcional, nivel máximo:** si más adelante agregás un backend propio, mover ahí las claves de API (proxy) para que nunca viajen en el dispositivo, y sumar Play Integrity (Android) / App Attest (iOS). Para esta app de solo lectura, las claves gratuitas en el dispositivo son un riesgo bajo y aceptado.

## Siri (iPhone)

- Los eventos agregados con el botón **+** quedan en el Calendario de Apple → Siri responde "¿qué tengo hoy?" con los partidos.
- Para comandos tipo "Oye Siri, abrí Mundial 2026": una vez instalada, la app aparece en Atajos automáticamente (Capacitor registra la actividad de apertura). Se puede sumar el plugin `capacitor-plugin-siri-shortcuts` para donar atajos específicos ("próximo partido de Argentina").

## Estructura

```
mundial-app/
├── package.json              dependencias Capacitor
├── capacitor.config.json     configuración nativa (seguridad incluida)
├── www/index.html            la app completa (UI + datos + lógica en vivo)
└── config-extras/
    ├── android/  network_security_config.xml · backup_rules.xml · manifest_snippet.txt
    └── ios/      PrivacyInfo.xcprivacy · info_plist_snippet.txt
```

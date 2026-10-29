# Respuestas · Laboratorio 4
Alumno: 20220867
Presupuesto: 11 min  Métrica DORA: time to restore

## Parte 1 · Diagnóstico
### Defecto 1
- Job y etapa: smoke, etapa deploy. Tenía `needs: build-push`, igual que deploy, así que corría en paralelo con el despliegue en lugar de en post-deploy.
- Problema que causa en el pipeline: smoke llama a la Function URL mientras deploy todavía actualiza la Lambda. Puede probar la versión anterior y salir en verde con una versión que nadie probó, y e2e y load heredan el error porque dependen de smoke. Además, el job no le pasaba EXPECTED_COMMIT a smoke.sh, así que la verificación de versión del script nunca se ejecutaba.
- Link al commit: https://github.com/mSaico/INF384-lab4-20220867/pull/1/changes/7ecb8788b5acee53a8600458454b85f2defcb508.
- Corrección: mover de etapa (`needs: deploy`) y ajustar configuración (`EXPECTED_COMMIT: ${{ github.sha }}`).
- Evidencia: en el run https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40956240516, smoke y deploy arrancan a la vez después de build-push. En el run https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40960947189, smoke corre después de deploy y su log muestra:
  `salud: {"ok":true,"version":"1.0.0","commit":"5d6ec3207c65845250c43c2a0727f9cf23988319"}`
  `commit: 5d6ec3207c65845250c43c2a0727f9cf23988319`
  La línea `commit:` solo aparece cuando se verifica EXPECTED_COMMIT, y coincide con el SHA del run (5d6ec32): la Lambda responde con la versión recién desplegada.

### Defecto 2
- Job y etapa: load, post-deploy (test/load/transferir.js).
- Problema que causa en el pipeline: `http.expectedStatuses(200, 400)` y el check "transferir 200 o 400" contaban un rechazo como éxito. Si un defecto hiciera que /transferir respondiera 400 a todo, load pasaría con 0 % de errores e incluso con un p95 menor, porque un rechazo no escribe en la base. Medía la latencia de respuestas de error, no la de transferencias reales.
- Link al commit: https://github.com/mSaico/INF384-lab4-20220867/pull/1/changes/24b812a2c89dfaec969508a806bb30af587c711c.
- Corrección: modificar tests (en transferir solo es válido un 200).
- Evidencia: run https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40960947189, resumen de k6: `✓ transferir 200`, checks 100.00 % (35164 de 35164), `http_req_failed 0.00%`. Con montos de S/ 1 y saldos semilla de S/ 300 a S/ 1 200, ninguna transferencia legítima devolvió 400: aceptarlo solo ocultaba fallas.

### Defecto 3
- Job y etapa: sca, pre-build.
- Problema que causa en el pipeline: `--include-dev-deps` analizaba Playwright, c8, esbuild y sus dependencias transitivas, que no llegan a la imagen (el Dockerfile empaqueta con esbuild y ejecuta `npm prune --omit=dev`). Una vulnerabilidad HIGH o CRITICAL en cualquiera de ellos detendría build-push y el despliegue sin riesgo en lo que corre en producción: un falso positivo que bloquea.
- Link al commit: https://github.com/mSaico/INF384-lab4-20220867/pull/1/changes/2daa06363d370b70e830b7f84bc49ba9450dea00.
- Corrección: ajustar configuración (se elimina `--include-dev-deps`). Lo que realmente se despliega lo sigue revisando el escaneo de la imagen.
- Evidencia: salida local en PowerShell:
  `PS> (Select-String -Path .\package-lock.json -Pattern '"dev": true').Count`
  `84`
  Son 84 paquetes que sca analizaba y que no viajan en la imagen.

### Defecto 4
- Job y etapa: load, post-deploy.
- Problema que causa en el pipeline: load y e2e dependían los dos de smoke y corrían en paralelo contra la misma Lambda. El p95 de load incluía el tráfico de e2e, y e2e corría bajo carga mientras load modificaba saldos. Además, si e2e fallaba, load igual consumía unos 12 minutos que se suman a la duración del pipeline.
- Link al commit: https://github.com/mSaico/INF384-lab4-20220867/pull/1/changes/248bdb747711bb3e5c56ace7d9e5ef3e3acbe054.
- Corrección: mover de etapa (`needs: e2e`).
- Evidencia: en el run https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40956240516, e2e y load arrancan juntos después de smoke. En el run https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40960947189, load arranca cuando termina e2e (e2e 0:17 y luego load 12:11).

### Defecto 5
- Job y etapa: e2e, post-deploy (ci.yml y test/e2e/api.spec.ts).
- Problema que causa en el pipeline: Monedero Andino es una API JSON, pero el job instalaba chromium con sus dependencias del sistema en cada run, porque las pruebas usaban el fixture `page`, que abre el navegador aunque solo se use `page.request`. Era tiempo perdido sin verificar nada que el cliente HTTP no cubra.
- Link al commit: https://github.com/mSaico/INF384-lab4-20220867/pull/1/changes/8f539cb7f1a5dcf24521c3ee76c759a479df56f6 y https://github.com/mSaico/INF384-lab4-20220867/pull/1/changes/91f3c3cd748783338ad0829f95eb438465533cd8.
- Corrección: modificar tests (fixture `request`) y eliminar (el paso Navegador).
- Evidencia: run base https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40956240516/job/123737707568: job e2e de 40 s, de los cuales el paso "Navegador" (npx playwright install --with-deps chromium) dura 19 s, casi la mitad del job. En el run corregido https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40960947189, el job e2e dura 17 s: 23 s recuperados en cada run, y las mismas 10 pruebas pasan sin navegador.

## Parte 2 · Implementación
- Link al run en verde en main: https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40960947189
- Link al pull request: https://github.com/mSaico/INF384-lab4-20220867/pull/2
- Defecto de la aplicación introducido (archivo y descripción): src/wallet.js, línea 55. La validación de saldo pasa de `cuentaOrigen.saldo < centimos` a `cuentaOrigen.saldo <= centimos`. Una transferencia por el saldo exacto disponible se rechaza con 400 "saldo insuficiente", cuando debería aceptarse y dejar el saldo del origen en cero.
- Job que debería detenerlo: unit (prueba "transferir todo el saldo deja el origen en cero"). También falla integration ("diego transfiere todo su saldo a elena"). build-push y los jobs siguientes quedan omitidos, así que el defecto no se despliega.
- Link al run fallido: https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40962328463/job/123758065657

## Parte 3 · Decisión bajo restricción
Link al run medido: https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40960947189

| Job | Duración (min:seg) |
|---|---|
| unit | 0:12 |
| integration | 0:12 |
| sast | 0:22 |
| sca | 0:24 |
| build-push | 0:38 |
| deploy | 0:17 |
| smoke | 0:09 |
| e2e | 0:17 |
| load | 12:11 |
| **Total** | **14:42** (suma de jobs) |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | 67.97 ms |
| p95 | 69.78 ms |
| Máximo | 449.91 ms |
| Tasa de error | 0.00 % (0 de 35164) |
| Requests por segundo | 48.79 req/s |

Conclusión sobre el comportamiento bajo carga: con 30 usuarios virtuales y unos 49 req/s durante 10 minutos, Monedero Andino responde con mucha holgura. El p95 es 69.78 ms, contra un umbral de 500 ms, y no hay errores. La distribución es estrecha (mediana de 63.77 ms y p95 de 69.78 ms), así que no hay una cola lenta. El máximo de 449.91 ms es un caso aislado, compatible con el arranque en frío de una instancia nueva al escalar. La latencia no se degradó durante la meseta, así que 10 minutos de carga constante aportan casi lo mismo que una ventana más corta.

SLO propuesto (percentil, valor, ventana) y justificación: el 95 % de las peticiones a la API (/transferir y /saldo) responde en menos de 200 ms, medido en una ventana móvil de 30 días. Lo medido fue un p95 de 69.78 ms y un p90 de 67.97 ms. Con 200 ms hay casi 3 veces de margen para la variación entre runs y los arranques en frío, así que load no falla por ruido. A la vez, una regresión que triplique la latencia se detecta, cosa que con 500 ms (más de 7 veces lo medido) pasaría inadvertida. Está configurado en `SLO_P95: 200` de ci.yml.

SLA propuesto y diferencia con el SLO: en cada mes calendario, el 95 % de las operaciones de los clientes responde en menos de 500 ms y la API está disponible el 99.5 % del tiempo. Si se incumple, el cliente recibe una compensación sobre las comisiones del mes. El SLA es un compromiso externo con consecuencias contractuales. El SLO es el objetivo interno del equipo. Por eso no tienen el mismo valor: el SLO es más exigente (200 ms contra 500 ms) para que el equipo detecte la degradación y reaccione antes de incumplir el contrato. Si fueran iguales, cualquier desvío interno ya sería un incumplimiento con el cliente.

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|
| Perfil de carga en test/load/transferir.js: rampa de 1m a 30s, carga constante de 10m a 5m, bajada de 1m a 15s | load | 6:13 (12:11 → 5:58) |

Total resultante: 8:29 min (suma de jobs; antes 14:42, presupuesto 11:00)
Link al run en main con los cambios: https://github.com/mSaico/INF384-lab4-20220867/actions/runs/40964573687

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada): con 5 minutos de carga constante en lugar de 10, cada instancia de Lambda procesa la mitad de requests durante la prueba. Un defecto que solo degrada la latencia al acumular uso podría pasar el pipeline y aparecer recién en producción. Ejemplo concreto: si un cambio en src/db.js hiciera que obtenerStore abriera una conexión nueva a SQLite en cada request sin cerrar la anterior, la fuga de memoria y de descriptores elevaría la latencia solo después de miles de requests. Efecto en time to restore: baja. Cuando ocurre una falla, la corrección o el rollback tienen que pasar por el pipeline, que ahora suma 8:29 en lugar de 14:42, así que el servicio se restaura unos 6 minutos antes. El riesgo aceptado afecta a cuántos despliegues fallan, no a cuánto tardamos en recuperarnos.

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón: se queda en pre-build y sigue deteniendo el pipeline. Cuesta 0:22 en la suma de jobs (0:20 en el run final, alrededor del 4 % de 8:29), pero en tiempo de reloj cuesta 0: corre en paralelo con unit, integration y sca, y sca (0:24) es el más lento de esa etapa, así que sast no retrasa build-push. Efecto en time to restore: no cambia, porque un hotfix llega a producción igual de rápido con o sin sast. Moverlo o dejarlo como informativo no ahorraría tiempo de restauración y dejaría pasar hallazgos de severidad ERROR (inyecciones, por ejemplo) hasta producción. sast complementa a las pruebas, no las reemplaza: el defecto `<=` de la Parte 2 no lo detecta ningún análisis estático y sí lo detiene unit.

## Declaración de uso de IA
Herramientas usadas y para qué: Claude , como apoyo para entender el repositorio y el pipeline, discutir qué defectos había, explicar las correcciones de ci.yml y de los tests, interpretar el resumen de k6 y redactar borradores de RESPUESTAS.md. Revisé y ajusté esos borradores. Los cambios los apliqué yo en GitHub y los verifiqué con los runs enlazados.

---
name: ggcc-order-converter
description: Procesa un PDF de pedido de Grandes Cuentas y genera un Excel listo para importar en Odoo. Úsalo cuando el usuario pase un PDF de pedido de cliente.
---

> **Sobre "elicitación" en este skill:** siempre que este documento diga
> "pregunta mediante elicitación" o "usa elicitación", significa **invocar el
> mecanismo nativo de preguntas con opciones/botones de este entorno** (la
> función/herramienta de tu entorno pensada para presentar una pregunta con
> opciones seleccionables al usuario), **NO** escribir la pregunta como texto
> plano en el chat. Si tu entorno no tiene ese mecanismo disponible, dilo
> explícitamente en vez de simular botones con texto.

> **Consulta de clientes dados de alta.** Si el usuario pregunta algo como "¿qué
> clientes tenéis implementados?" o "¿está XXX dado de alta?" **sin pasar un
> PDF**, llama a `listar_clientes` y muéstrale la lista (dominio, `cliente_id` y
> si el fichero de instrucciones existe). No forma parte del flujo de un pedido
> de los pasos 1-11: no la llames ahí, ni la confundas con `detectar_cliente`
> (que identifica UN cliente concreto a partir del dominio de un PDF ya leído).

> **Consulta de la frescura de los datos.** Si el usuario pregunta algo como "¿de
> cuándo son los datos?", "¿está el catálogo actualizado?" o "¿cuándo se
> actualizaron los SAP de Hikvision?", llama a `estado_datos` (sin argumentos) y
> muéstrale, para cada fichero, la fecha de última actualización y cuántos
> registros tiene cargados. Tampoco forma parte del flujo de los pasos 1-11.
> Que un fichero salga con `existe: false` no es un error: significa que el ETL
> que lo genera aún no se ha ejecutado y que ese paso de la cascada no casa nada.

Cuando el usuario pase un PDF de pedido:

1. **Primera pasada (solo el cliente).** Lee el PDF únicamente para extraer el
   dominio de correo del cliente. Omite el dominio @visiotechsecurity.com, que es
   el del distribuidor; el del cliente es cualquier otro que aparezca.
2. Llama a `detectar_cliente` con ese dominio. Devuelve `cliente_id`,
   `instrucciones` (markdown con las reglas de parseo de ESE cliente) y `fuente`
   (el fichero usado).
   - Anuncia en una línea qué instrucciones aplicas, p. ej.
     "Aplicando instrucciones personalizadas para `cliente_id`".
   - Si `cliente_id` es null (no reconocido), pregunta al usuario de qué cliente
     es. Recibirás igualmente las instrucciones genéricas de `_default.md`: úsalas
     para el parseo y continúa.
3. **Segunda pasada (las líneas y la cabecera).** Lee y parsea, **aplicando las
   reglas del campo `instrucciones`** (trátalas como autoritativas para ese
   cliente: qué columna trae la referencia, cómo elegirla, descripciones
   multilínea, etc.):
   - Las **líneas del pedido**: referencia del cliente, descripción, cantidad,
     precio neto unitario.
   - La **referencia del pedido del cliente (PO)**: en el campo que indique la
     instrucción del cliente (p. ej. "ORDEN DE COMPRA", "Nº pedido").
4. **Una sola llamada con TODAS las líneas.** Llama a `buscar_referencias`
   pasando `cliente_id` y `lineas`: una lista con un dict por línea del pedido
   (`ref_cliente`, `descripcion` completa y, si lo tienes aparte, `sap_code`).
   Devuelve una lista de resultados **en el mismo orden** que las líneas, cada
   uno con `default_code`, `origen`, `confianza` y `candidatos`. NO llames a la
   herramienta una vez por línea ni adivines la referencia por tu cuenta: usa
   siempre el resultado de la herramienta.
   - **El producto que va al Excel es SIEMPRE el `default_code` que devuelve la
     herramienta**, NUNCA la referencia del cliente ni ningún código del PDF.
   - Según el `estado` de cada resultado (equivale a su `confianza`):
     - `"resuelto"` (confianza `"alta"`): usa `default_code` directamente.
     - `"dudoso"` (confianza `"media"`): `default_code` viene vacío; hay que
       revisar. Los candidatos cercanos están en `candidatos` (cada uno con su
       `default_code`, `descripcion`, `fabricante` y a veces `score`). El usuario
       debe elegir uno; NO elijas tú.
     - `"sin_match"` (confianza `"ninguna"`): ni un candidato pasó el umbral del
       prefiltro; se resuelve a mano.
   - En concreto, cuando `origen` es `"hikvision"`, la herramienta ha traducido el
     SAP code (9 dígitos) de la descripción a su referencia vinculada y esa
     referencia SÍ existe en el catálogo de Odoo: usa ese `default_code`, **no**
     el SAP code (`311322812`). Puede diferir de `referencia_hikvision` en algún
     sufijo final entre paréntesis (`(STD)`, `(O-STD)`…): eso es correcto y ya
     está validado contra el catálogo, no lo preguntes.
   - Cuando `origen` es `"hikvision-aprox"` (confianza media): el SAP se tradujo,
     pero esa referencia **no existe en Odoo** ni quitándole los sufijos entre
     paréntesis. La herramienta ofrece en `candidatos` las 3 más parecidas del
     catálogo y, en el campo `referencia_hikvision`, lo que dice Hikvision.
     Muéstraselo al usuario para que elija un candidato. NO elijas tú.
   - El `origen` `"fuzzy"` (prefiltro por similitud contra el catálogo) o un
     `"ean13"` con varios `candidatos` también son dudosos (revisar).
5. **Reglas del fabricante, solo si hay líneas dudosas.** Tras recibir los
   resultados de `buscar_referencias`, si hay líneas `"dudosas"` con candidatos de
   un fabricante concreto, llama a `obtener_instrucciones_fabricante` UNA VEZ por
   cada fabricante distinto que aparezca (nunca una vez por línea), y usa esas
   reglas para elegir el candidato más probable entre los propuestos. El resultado
   sigue siendo `"dudoso"`: preséntalo al comercial para que confirme, nunca lo
   des por bueno automáticamente.
   - Reúne primero el conjunto de fabricantes distintos que aparecen en los
     `candidatos` de las líneas dudosas, y luego haz una llamada por fabricante.
     Si no hay líneas dudosas, NO llames a esta herramienta.
   - Si el fabricante no tiene reglas propias, la herramienta te lo dice con un
     texto neutro: compara entonces los candidatos por referencia y descripción
     sin inventarte reglas de la marca.
6. **Resuelve TODAS las líneas antes de pedir nada al usuario** (ya hiciste esto
   en el paso 4: una única llamada a `buscar_referencias` con toda la lista). No
   preguntes según vas detectando líneas dudosas durante el parseo; primero se
   completa toda la cascada para todas las líneas y solo entonces se pasa a
   revisión.
7. **La tabla de resumen va SIEMPRE ANTES de la primera pregunta.** Muestra en el
   chat **una única tabla compacta** con **TODAS** las líneas del pedido (las
   resueltas y las que no): **Ref. cliente**, **Producto** (= `default_code`),
   **origen**, **cantidad**, **precio**, **estado**. Debajo de la tabla, indica
   cuántas líneas quedan por resolver (todas las que NO estén `"resuelto"`: las
   `"dudoso"` y las `"sin_match"`) con un mensaje tipo "3 de 18 líneas necesitan
   tu confirmación".
   - **NO lances ninguna elicitación antes de haber mostrado esta tabla**, ni
     siquiera para la primera línea dudosa. El usuario necesita ver la foto
     completa del pedido —y cuántas preguntas le esperan— antes de que le
     empieces a preguntar. Primero la tabla, en su propio mensaje; las preguntas
     del paso 8 vienen después.
8. **Para cada línea que NO esté `"resuelto"`, resuélvela una por una con
   elicitación** (pregunta con opciones, no texto libre), en este orden, mostrando
   el progreso ("línea 1 de 3", "línea 2 de 3"...):
   - Da contexto de la línea (ref. cliente, descripción, cantidad, precio) para
     que el usuario sepa qué está eligiendo.
   - Si hay `candidatos` (estado `dudoso`: `fuzzy`, `hikvision-aprox`, `ean13`
     ambiguo): ofrece como opciones los candidatos que devuelve la herramienta
     (ya vienen acotados a los 3 de mayor score/relevancia, en orden). Si las
     reglas del fabricante del paso 5 apuntan a uno, márcalo como el más probable
     y di en una línea por qué (p. ej. "coincide en canales; solo cambia (STD)"),
     pero deja que el usuario elija: no lo des por bueno ni lo preselecciones como
     hecho.
   - **Las opciones son EXCLUSIVAMENTE las referencias candidatas: nada más.**
     Máximo 3 y ninguna añadida por ti. En concreto, NO añadas opciones del tipo
     "Ninguno / dar de alta <REF>", "Ninguna de estas", "resolver a mano" ni
     equivalentes: el mecanismo de elicitación ya trae su propia salida de texto
     libre para eso, y una opción extra solo alarga la lista.
   - Si `origen` es `hikvision-aprox`, puedes mencionar la `referencia_hikvision`
     (la que traduce Hikvision, aunque no esté dada de alta en Odoo) **en el texto
     de contexto de la pregunta**, nunca como una opción más.
   - Si NO hay `candidatos` (estado `sin_match`): no hay opciones que
     ofrecer; pide directamente la referencia correcta al usuario.
   - Registra la elección de esa línea y pasa a la siguiente. NO agrupes varias
     líneas dudosas en una sola pregunta ni las muestres todas en una segunda
     tabla: una elicitación por línea.
9. Cuando todas las líneas queden resueltas, **pregunta mediante elicitación**
   (una pregunta con opciones) si generar el Excel de import. Ofrece dos
   opciones: **"Sí, generar el Excel"** y **"No"**. Genera el Excel SOLO si el
   usuario elige "Sí".
10. **El Excel lo genera la herramienta `generar_excel`, no lo montes tú.** Llámala
    con `cliente` (el `cliente_id` de `detectar_cliente`), `referencia_cliente`
    (el PO del paso 3, cadena vacía si el pedido no trae) y `lineas`: una entrada
    por línea del pedido, en su orden, con `producto`, `cantidad` y `precio`.
    - `producto` = el `default_code` de `buscar_referencias` o el que confirmó el
      usuario en la revisión (repito: NO el SAP code ni la referencia del cliente).
    - `precio` = precio neto unitario. El descuento lo deja vacío la herramienta.
    - Las columnas, su orden y qué va solo en la primera fila los fija la
      herramienta: NO escribas un script en Python para generar el Excel ni
      reconstruyas las columnas a mano.
11. **El fichero lo escribe la herramienta; tú solo dices dónde está.** Devuelve
    `ruta` (absoluta), `nombre_fichero`, `filas` y `avisos`. Si tu entorno tiene
    una carpeta de entregables, pásala en `ruta_salida` y el .xlsx aparece
    directamente ahí. Enséñale al usuario la `ruta` y termina.
    - **Los bytes del .xlsx no pasan nunca por ti.** No los pidas, no los
      transcribas y no reconstruyas el fichero a mano: el Excel ya está escrito.
      Si tiene que acabar en otra carpeta, **cópialo con `cp`** (una copia es byte
      a byte); no lo regeneres ni lo vuelques desde texto.
    - Si `avisos` no viene vacío, menciónalos: son líneas cuyo producto no está
      en el catálogo de Odoo (típico de una resuelta a mano con una referencia sin
      dar de alta). El Excel se genera igual; el usuario debe saberlo.

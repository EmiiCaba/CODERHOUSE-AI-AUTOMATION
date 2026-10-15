# Pipeline de Contenido Autónomo con Control de Calidad (HITL)

**Caso:** Clínica Veterinaria y Peluquería Canina
**Herramientas:** n8n · Airtable · OpenAI · LinkedIn

Pipeline que genera automáticamente posts para LinkedIn a partir de una idea cargada en Airtable. Antes de redactar, consulta una base de conocimientos privada (RAG) con información validada de la clínica. El borrador **no se publica solo**: queda en pausa hasta que una persona lo revisa y lo aprueba (Human in the Loop).

![Lienzo del flujo en n8n](img/flujo-n8n.png)

---

## Cómo funciona

El flujo tiene dos ramas sobre el mismo lienzo de n8n, cada una con su propio disparador programado (cada 1 minuto).

### Rama A · Generación con RAG

```
Schedule (1 min)
  → Buscar idea nueva          (Idea Semilla cargada y Estado vacío)
  → Estado: Generando
  → Consultar conocimiento (RAG)
  → Armar contexto             (nodo Code)
  → Redactar post LinkedIn     (OpenAI)
  → PAUSA HITL · Guardar borrador (Estado: En revisión)
```

### Rama B · Filtro HITL y publicación

```
Schedule (1 min)
  → Buscar piezas en revisión
  → FILTRO · ¿Aprobado?
       ├─ true  → Estado: Aprobado → Publicar en LinkedIn → Estado: Publicado
       └─ false → Frenado · Espera validación humana
```

---

## Bases de datos (Airtable)

### Centro de Comando — tabla `Piezas`

Registra cada pieza de contenido y su estado.

| Campo | Tipo | Uso |
|---|---|---|
| Idea Semilla | Texto (principal) | Input mínimo que dispara el flujo |
| Estado | Selección única | Generando / En revisión / Aprobado / Rechazado / Publicado |
| Borrador generado | Texto largo | Post escrito por la IA, listo para auditar |
| Aprobado | Checkbox | Semáforo humano: solo si está marcado se publica |
| Última ejecución | Fecha y hora | Se actualiza en cada paso con `{{$now}}` |

Incluye la vista **Cuadro de mando**, agrupada por Estado.

![Centro de Comando](img/centro-de-comando.png)

### Base de Conocimiento — tabla `Conocimiento`

Base separada con la información que el agente tiene permitido usar. Está dividida de forma atómica: **un tema por registro**.

| Tema | Categoría |
|---|---|
| Servicios de peluquería | Servicios |
| Cuidado del pelaje en primavera | Consejos |
| Frecuencia de baño | Consejos |
| Plan de vacunación del cachorro | Salud |
| Por qué vacunar a tiempo | Salud |
| Desparasitación | Salud |
| Cómo pedir turno | Atención |
| Horarios de atención | Atención |
| Tono de marca | Estilo |

El nodo **Consultar conocimiento (RAG)** lee estos registros y **Armar contexto** los une en un solo bloque (`Tema: Contenido`) que se le pasa a OpenAI como único contexto permitido.

![Base de conocimiento](img/base-conocimiento.png)

---

## Nodo de IA

**Mensaje de sistema:** el modelo actúa como redactor de la clínica y debe usar **exclusivamente** la información del contexto. No puede agregar datos externos ni inventar servicios, precios u horarios; si falta información, no la incluye. Respeta el tono de marca.

**Mensaje de usuario:**

```
IDEA SEMILLA: {{$json.idea}}

CONTEXTO:
{{$json.contexto}}

Redactá un post para LinkedIn con esta estructura:
1) Gancho inicial (1 línea)
2) Desarrollo (3 a 5 líneas)
3) Llamado a la acción para pedir turno
4) 3 hashtags
```

---

## Punto de pausa HITL

El último nodo de la Rama A no publica: escribe el resultado en el Centro de Comando y ahí termina.

| Campo | Valor |
|---|---|
| Borrador generado | `{{$json.message.content}}` |
| Estado | En revisión |
| Aprobado | false |
| Última ejecución | `{{$now.toISO()}}` |

Una persona lee el borrador en Airtable y marca **Aprobado** (o cambia el Estado a **Rechazado**).

## Filtro final

La Rama B solo toma piezas con Estado = *En revisión* (las rechazadas quedan afuera).

| Condición | Resultado |
|---|---|
| `{{$json['Aprobado']}}` es true | Se publica en LinkedIn y el Estado pasa a Publicado |
| Cualquier otro caso | Queda frenado. No llega a ninguna plataforma externa |

### Ciclo de estados

```
Generando → En revisión → Aprobado / Rechazado → Publicado
```

---

## Cómo usarlo

1. Crear en Airtable las dos bases (Centro de Comando y Base de Conocimiento) con los campos descritos arriba.
2. En n8n, ir a **Workflows → Import from File** y elegir el archivo `.json` de este repositorio.
3. Configurar las credenciales de **Airtable**, **OpenAI** y **LinkedIn** en cada nodo.
4. Ajustar en los nodos de Airtable los IDs de base y tabla propios.
5. Activar el workflow.
6. Cargar una Idea Semilla en la tabla `Piezas` y esperar el borrador.
7. Revisarlo y marcar **Aprobado** para que se publique.

## Estructura del repositorio

```
├── README.md
├── pipeline-hitl-veterinaria.json   # Workflow exportado de n8n
├── Pre-entrega-7.pdf                 # Documentación de la entrega
└── img/                              # Capturas
```

---

**Autora:** Emilce Paola Caba · Octubre 2026

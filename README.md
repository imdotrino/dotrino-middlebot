# Dotrino Middlebot

**El problema que resuelve: que no salga información sensible de la empresa
dentro de un prompt a una inteligencia artificial.**

Un agente de IA que corre en la máquina de quien trabaja y **no habla directo con
ninguna inteligencia artificial**: cada prompt pasa antes por un **auditor** —un
clasificador de prompts del lado del usuario— que corre en **otra máquina
designada por el acta**, y que tacha lo sensible, pregunta cuando hay dudas,
detiene lo que no puede salir y deja constancia firmada. *Nada que no deba salir
sale.*

Nace para el caso de empresa (línea [Dotrino
Enterprise](https://dotrino.com/enterprise)), y sirve igual para una persona con
dos máquinas.

Parte del ecosistema [Dotrino](https://dotrino.com/) · MIT.

---

## 1. Las dos piezas

| Pieza | Dónde vive | Qué hace |
|---|---|---|
| **`@dotrino/middlebot`** | La máquina donde trabajas | El agente con el que conversas: **CLI** (`middlebot`) y la **misma UI en web local** (`http://127.0.0.1:7717`, solo loopback). Arma el prompt, ejecuta herramientas locales, muestra la respuesta. **No tiene credenciales de ningún proveedor de IA y no puede llamar a uno.** |
| **`@dotrino/middlebot-auditor`** | **Otra** máquina | Recibe cada prompt, verifica quién lo manda, aplica la política, redacta / pregunta / bloquea, y es **el único** que habla con el proveedor de IA. Guarda la bitácora. |

Ambas máquinas están **designadas en el acta del vault**: cada una lleva un
certificado de dispositivo con su rol (`middlebot` / `middlebot-auditor`) firmado
por el master del perfil. Un dispositivo que no esté en el acta no puede hacerse
pasar por auditor, ni pedirle nada como si fuera el bot.

## 2. Por qué dos máquinas y no un flag

Todo el valor está en la separación. En una sola máquina, el que escribe el
prompt es el mismo que decide si sale, guarda la bitácora y tiene la credencial:
un descuido en esa máquina —o un agente al que se le dio demasiada correa— se
lleva las tres cosas de una vez.

Separado:

- La **credencial del proveedor** nunca toca la máquina de trabajo (vive en el
  auditor, entregada por el vault: `@dotrino/vault/service`, scope
  `vault:secrets:middlebot`).
- La **bitácora** vive en la máquina auditora, no en la auditada.
- La **decisión de dejar salir** la toma un proceso al que el código de la
  máquina de trabajo no llega.
- Si el auditor no está, **no sale nada**: el middlebot espera (la regla del
  ecosistema —sin vault, se espera— aplicada al pie de la letra).

### 2.1. Qué es el auditor, dicho técnicamente

Es **un clasificador de prompts**, exactamente el mismo mecanismo que los
asistentes de IA aplican a cada mensaje antes de procesarlo: entra un texto, sale
una etiqueta y un veredicto. La diferencia no está en el mecanismo, está en
**quién lo opera y para quién**:

- **Corre del lado del usuario**, no del lado del modelo: clasifica *antes* de
  que el prompt salga, no después de que llegó.
- **La política la escribes tú** y va firmada por tu acta. El clasificador
  responde a esas reglas, no a otras.
- **Lo gobierna la red de Dotrino**: identidad y roles por el acta del vault,
  transporte por el proxy, bitácora sellada por el signer. No hay un servicio
  central que vea los prompts — el auditor es una máquina tuya, hablando con
  otra máquina tuya.
- **El veredicto es accionable en el borde**: además de etiquetar, redacta,
  pregunta o detiene, y lo que redacta se recompone en tu máquina.

El clasificador puede ser desde un juego de reglas y patrones hasta un modelo
pequeño corriendo en la máquina auditora; la interfaz —sobre firmado entra,
veredicto firmado sale— es la misma en ambos casos, y eso es lo que se
especifica.

## 3. El camino de un prompt

```
  tu máquina                                     otra máquina tuya
┌───────────────────┐                          ┌──────────────────────┐
│  middlebot        │   sobre firmado, E2E     │  auditor             │        proveedor
│  CLI + web local  ├──── proxy.dotrino.com ──▶│  política + bitácora ├──HTTPS──▶  de IA
│  des-redacta      │◀─────── respuesta ───────┤  (única credencial)  │
└───────────────────┘                          └──────────────────────┘
       sin puertos abiertos en ninguna de las dos
```

1. Escribes en el CLI o en la web local.
2. El middlebot arma el **sobre**: el prompt, los adjuntos incluidos, las
   herramientas que el agente quiere usar y un hash del contexto acumulado. Lo
   firma con su llave de dispositivo (`@dotrino/identity`).
3. Sale por **`@dotrino/proxy-client`** (`sendByPubkey` a la pubkey del auditor),
   cifrado de punta a punta. Ninguna de las dos máquinas abre un puerto.
4. El auditor verifica firma y rol contra el acta, aplica la **política firmada**
   y decide (ver §4).
5. **Solo el auditor** llama al proveedor de IA.
6. La respuesta vuelve por el mismo canal. El middlebot **des-redacta** los
   marcadores localmente y te la muestra completa.
7. Todo queda en la **bitácora** (§5).

## 4. Qué decide el auditor

Cuatro veredictos, y siempre uno:

| Veredicto | Qué pasa |
|---|---|
| **pasa** | El prompt sale tal cual. |
| **pasa redactado** | Lo que la política marca (correos, teléfonos, nombres, rutas, llaves, números de documento, lo que definas) se sustituye por marcadores `⟦r1⟧`, `⟦r2⟧`… **El mapa de vuelta se queda en el middlebot**: el proveedor nunca ve el valor, y tú lees la respuesta con los valores puestos. |
| **pregunta** | Te lo muestra en la máquina auditora —o en el teléfono, vía `@dotrino/notifications`— con el diff de lo que se va a enviar, y espera tu aprobación. Nada sale mientras tanto. |
| **bloquea** | No sale, con el motivo escrito. |

**La política** es un archivo firmado por el master del acta; el auditor
**rechaza** una política sin firma o firmada por otro. Describe: patrones a
redactar, rutas y archivos que no salen nunca, tamaño máximo del sobre,
herramientas permitidas, proveedores permitidos, veredicto por defecto y tope de
gasto por día.

**Modos de arranque**, para poder adoptarlo sin fricción:

- **Solo bitácora** — todo pasa, todo queda registrado. Sirve para ver primero
  qué estabas mandando.
- **Encadenado** (el modo del producto) — manda la política; sin auditor, no hay
  salida.
- **Sin salida** — el auditor apunta a un modelo que corre en tus propias
  máquinas: el prompt no sale de tu casa.

## 5. Bitácora

Por cada prompt, una entrada firmada en el auditor: hash del sobre original,
hash de lo que efectivamente salió, veredicto, regla que lo produjo, proveedor y
modelo, tamaño, respuesta y **sello de tiempo** de `signer.dotrino.com` (prueba
de cuándo existió). Es apendable, consultable desde el middlebot y exportable.
No se puede reescribir sin que las firmas dejen de cuadrar.

## 6. Piezas del ecosistema que usa (ninguna nueva)

| Pilar | Para qué |
|---|---|
| `@dotrino/identity` + vault | Acta, certificados de dispositivo con rol, firma de los sobres y de la política |
| `@dotrino/proxy-client` | Transporte E2E entre bot y auditor, sin puertos abiertos |
| `@dotrino/vault/service` | La credencial del proveedor de IA, en el auditor |
| `@dotrino/store` | Historial de conversación del middlebot (del usuario, cifrado) |
| `dotrino-signer` | Sello de tiempo de cada entrada de la bitácora |
| `@dotrino/notifications` | Aprobar desde el teléfono cuando el veredicto es *preguntar* |
| `@dotrino/topbar` + `@dotrino/support` | La web local y la página pública |

## 7. Lo que **no** promete

Se dice acá y se dice en la página:

- **No garantiza qué hace el proveedor** con lo que sí se le envía. Lo que sale,
  salió.
- **No detecta "todo lo sensible"**: redacta lo que la política describe. Lo que
  la política no describe, pasa — para eso están el veredicto *preguntar* y la
  bitácora.
- **No hay cumplimiento normativo, certificaciones ni auditorías de terceros.**
- Como cualquier app del ecosistema: **ninguna app cuida lo que su dueño decide
  mostrar**. El middlebot hace que enviar sea un acto explícito; no reemplaza la
  decisión.

## 8. Puesta en marcha

1. **Enrolar la máquina auditora** en el acta con rol `middlebot-auditor`
   (código tecleado en esa misma máquina; el cert caduca a los 30 días si nadie
   lo renueva).
2. **Escribir y firmar la política** con el master del acta.
3. **Instalar el middlebot** en los equipos de trabajo (rol `middlebot`): quedan
   enlazados al acta y descubren a su auditor por el acta, sin configuración por
   máquina.
4. **Arrancar en modo *solo bitácora*** para ver el tráfico real, y pasar a
   *encadenado* cuando la política esté afinada.

### Orden de trabajo del repo

| Etapa | Alcance |
|---|---|
| 1 | Transporte bot ↔ auditor por proxy, roles en el acta, modo *solo bitácora* |
| 2 | Política firmada, los cuatro veredictos, redacción con marcadores reversibles |
| 3 | Aprobación desde el teléfono, bitácora sellada y exportable |
| 4 | Web local con la UI completa y proveedores intercambiables (incluido modelo local) |

---

MIT · parte del ecosistema [Dotrino](https://dotrino.com/).

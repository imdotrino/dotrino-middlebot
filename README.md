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
│  middlebot        │ sobre sellado y firmado  │  auditor             │        proveedor
│  CLI + web local  ├──── proxy.dotrino.com ──▶│  política + bitácora ├──HTTPS──▶  de IA
│  des-redacta      │◀─────── respuesta ───────┤  (única credencial)  │
└───────────────────┘                          └──────────────────────┘
       sin puertos abiertos en ninguna de las dos
```

1. Escribes en el CLI o en la web local.
2. El middlebot arma el **sobre**: el prompt, los adjuntos incluidos, las
   herramientas que el agente quiere usar y un hash del contexto acumulado. Lo
   firma con su llave de dispositivo (`@dotrino/identity`).
3. Sale por **`@dotrino/proxy-client`** con **`sendSealed()`** a la pubkey del
   auditor —**nunca `sendByPubkey`**, que no cifra—, y **las dos puntas arrancan
   con `requireSealed: true`**. Así el sobre es ilegible para quien opere el
   proxio. El detalle, y de dónde sale la llave con la que se sella, en §3.1.
   Ninguna de las dos máquinas abre un puerto.
4. El auditor verifica firma y rol contra el acta, aplica la **política firmada**
   y decide (ver §4).
5. **Solo el auditor** llama al proveedor de IA.
6. La respuesta vuelve por el mismo canal y **sellada igual** (§3.1): el
   veredicto del auditor tampoco viaja en claro. El middlebot **des-redacta** los
   marcadores localmente y te la muestra completa.
7. Todo queda en la **bitácora** (§5).

### 3.1. El sobre va SELLADO, y la llave sale del acta

**El transporte no cifra.** `@dotrino/proxy-client` enruta por pubkey y nada más:
lo que sale con `sendByPubkey` viaja **legible para quien opere el proxio**, y el
proxio de producción corre en un VPS alquilado. Para el middlebot eso no sería un
detalle sino el agujero entero, porque el sobre lleva el prompt del usuario — que
es exactamente lo que este producto existe para que no salga. Es la norma del
ecosistema (CONVENCIONES-APPS §4.1), y aquí manda:

- **El sobre se manda con `sendSealed()`**, nunca con `sendByPubkey`.
- **Las dos puntas arrancan con `requireSealed: true`**: la del middlebot y la del
  auditor. Sellar solo de salida no sirve de nada — una punta que acepta texto
  plano se salta el sellado entero, y quien no haya leído nunca nada puede
  **contestar** sin sellar. En un producto cuyo veredicto decide si algo sale, un
  *pasa* falsificado es peor que la fuga que se venía a evitar.
- **Prohibido escribir cifrado propio**: el sellado es el del pilar
  (`wrapForMember` / `openWrap` de `@dotrino/identity/content`, la misma cripto de
  los secretos sellados del vault). Si faltara algo, se extiende el pilar.
- Mínimo `@dotrino/proxy-client` **≥ 0.13.0**, que es donde nace el sellado.

**De dónde sale la llave de cifrado del otro lado.** `sendSealed(to, payload, {
peerEncPub })` **exige** esa llave: el pilar no la descubre solo, y sin ella corta
con `code: 'unsealed'` en vez de mandar en claro. Aquí no hay que inventar nada, y
es la ventaja de que el auditor **no sea un desconocido sino otra máquina del
acta**: al enrolarse, cada aparato publica su llave pública de cifrado y queda
escrita en el acta (`admitMember({ pub, encPub, … })`), así que cada miembro es
`{ pub, encPub, label, cn, caps, cert }`. La llave del otro ya está donde se mira
todo lo demás, y la misma consulta que responde *¿este aparato es el auditor?*
responde *¿con qué llave le sello?*.

Simétrico en los dos lados, y es el patrón que ya usa
[`dotrino-passmanager`](https://github.com/imdotrino/dotrino-passmanager) — el
consumidor del sellado del pilar, que lo resuelve así por las dos puntas:

```js
// Cada lado saca del acta el encPub del otro. Nada nuevo: ya está ahí.
const { members } = await identity.profileMembers()
const peer = members.find(m => samePubkey(m.pub, auditorPubkey))
if (!peer?.encPub) throw new Error('auditor has no encryption key in the acta')

await client.sendSealed([auditorPubkey], envelope, { peerEncPub: peer.encPub })
```

Cada punta abre lo que le sellan con **su propia privada de cifrado** (la que le
dio el enrolamiento; su mitad pública es la que está en el acta), pasada al
cliente como `myEncPrivateKey`.

Tres cosas que no se negocian al implementarlo:

- **Se compara con `samePubkey`, nunca con `===`**: dos formas de la misma llave
  no son la misma cadena.
- **Si el acta no trae `encPub` del auditor, se para y se dice.** No hay repliegue
  a `sendByPubkey`: un repliegue ahí mandaría el prompt en claro justo el día en
  que algo se rompió, que es cuando menos hay que fiarse.
- **`unsealed` y `unreachable` son cosas distintas y hay que distinguirlas por
  `e.code`, no por el texto del error.** «No tengo la llave del auditor» no se
  arregla esperando —hay que enlazar los aparatos—, mientras que «el auditor está
  apagado» sí es esperar, que es lo que el middlebot hace por diseño (§2). Si los
  dos se ven igual desde arriba, el usuario espera para siempre una cosa que nunca
  va a llegar.

**Esto se resuelve entero con el acta y no depende de nada por venir.** Si más
adelante el pilar del transporte aprende a averiguar el `encPub` por su cuenta,
este bloque se encoge hasta desaparecer y mejor — pero no es requisito para
empezar: hoy no existe un `memberEncPub(acta, pub)` en `@dotrino/identity` y cada
consumidor hace el `.find()` a mano, que es justo la forma que tendría lo que el
pilar absorbiera.

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
| `@dotrino/identity` + vault | Acta, certificados de dispositivo con rol, firma de los sobres y de la política, y el `encPub` de cada aparato: la llave con la que se sella (§3.1) |
| `@dotrino/proxy-client` | Transporte entre bot y auditor, sin puertos abiertos. **No cifra por sí solo**: lo de punta a punta lo pone `sendSealed` + `requireSealed` en ambas puntas (§3.1) |
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
   (código tecleado en esa misma máquina). En ese enrolamiento el aparato publica
   su llave de cifrado en el acta, que es la que después sella los sobres (§3.1).
   El certificado no caduca por reloj: muere con el acta, así que sacar al auditor
   del acta lo deja fuera en el acto.
2. **Escribir y firmar la política** con el master del acta.
3. **Instalar el middlebot** en los equipos de trabajo (rol `middlebot`): quedan
   enlazados al acta y descubren a su auditor por el acta, sin configuración por
   máquina.
4. **Arrancar en modo *solo bitácora*** para ver el tráfico real, y pasar a
   *encadenado* cuando la política esté afinada.

### Orden de trabajo del repo

| Etapa | Alcance |
|---|---|
| 1 | Transporte **sellado** bot ↔ auditor por proxy (§3.1), roles en el acta, modo *solo bitácora* |
| 2 | Política firmada, los cuatro veredictos, redacción con marcadores reversibles |
| 3 | Aprobación desde el teléfono, bitácora sellada y exportable |
| 4 | Web local con la UI completa y proveedores intercambiables (incluido modelo local) |

---

MIT · parte del ecosistema [Dotrino](https://dotrino.com/).

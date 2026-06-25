# Cuando los guardias fallan abriendo la puerta

### Un post-mortem sobre `fail-open` vs `fail-closed` en un agente de software autónomo

> **Blog técnico — Entrega final · Coderhouse**
> Autor: **Mimix** ([github.com/MimixGuy](https://github.com/MimixGuy)) · Junio 2026
> 🌐 También publicado vía **GitHub Pages** (ver enlace en la descripción del repositorio).

---

Una de las verdades incómodas de construir software que se ejecuta solo, sin nadie mirando, es esta:
**un control de seguridad que no se prueba en el peor momento puede estar engañándote desde el primer día.**
Este es el relato honesto de un fallo silencioso, cómo lo encontramos, y por qué resultó no estar solo.

---

## 🧭 Contexto

Trabajo sobre un **agente de software autónomo** que funciona 24/7 y toma decisiones automáticas sobre
**datos externos en tiempo real**. Como esas decisiones tienen consecuencias reales, el sistema está
protegido por una serie de **«guardias» de seguridad**: comprobaciones que deben **bloquear** una acción
cuando el dato de entrada falta, está viejo (_stale_) o no se puede verificar.

La idea es simple y se entiende sin saber programar: antes de actuar, el sistema le pregunta a un guardia
_«¿el dato que tengo es de fiar?»_. Si la respuesta no es un sí claro, el guardia debe decir **«no pases»**.
Esos guardias son la red de seguridad de todo el sistema.

## 🔥 Problema

Un guardia decidía si un dato de mercado estaba «fresco» mirando una **marca de tiempo** (un sello que dice
_«última vez que se revisó»_). Si el sello era reciente, daba el dato por bueno.

El detalle fatal: el componente que produce ese dato **actualizaba el sello en cada intento, incluso cuando
rechazaba la lectura** por mala calidad. ¿Por qué? Para que un panel de monitoreo mostrara _«sigo vivo»_.

El resultado fue un **dato muerto disfrazado de fresco**. El guardia, confiando en el sello reciente,
concluía _«el dato es de fiar»_… y dejaba al sistema **actuar sobre información ya inválida**. La red de
seguridad estaba ahí, pero con un agujero por el que no saltaba nunca.

Al tirar del hilo, lo peor: **no era un caso aislado**. Una revisión a fondo encontró **~40 guardias** con el
mismo patrón. Cuando no podían verificar su entrada (un valor nulo, malformado o viejo), **dejaban pasar la
acción** (`fail-open`) en lugar de **bloquearla** (`fail-closed`). Y un guardia que falla abriendo la puerta
no es un guardia: es alguien que solo está presente cuando no pasa nada.

## 🛠️ Acciones (post-mortem constructivo, estilo Google SRE)

El objetivo del post-mortem **no fue buscar culpables, sino la causa raíz y el aprendizaje.**

**Descripción objetiva.** El guardia usaba una marca de _«última revisión»_ como sustituto de _«el dato es
actual y válido»_. La fuente actualizaba esa marca incluso al rechazar lecturas. Dos conceptos distintos
quedaron mezclados en un mismo campo.

**Análisis de causa raíz.** Se confundieron dos preguntas:
- **_Liveness_** (¿el componente está vivo / ejecutándose?) → a eso respondía la marca de tiempo.
- **_Freshness_** (¿el dato es actual y utilizable?) → eso es lo que el guardia _creía_ estar comprobando.

Encima de eso, un patrón cultural: guardias escritos a la defensiva como _«ante la duda, dejar pasar»_ —
exactamente lo contrario de lo que exige un control de seguridad.

**Acciones correctivas y preventivas.**
1. **Arreglo puntual:** el guardia ahora valida la **lectura real** (¿es un valor válido, no uno rechazado?),
   no solo el latido del reloj.
2. **Barrido sistemático:** una búsqueda por palabras clave (`stale`, `fail-open`, `guard`, `freshness`…)
   sacó a la luz **toda la clase** de guardias afectados, no solo el del incidente.
3. **Conversión `fail-open` → `fail-closed`:** ante una entrada que no se puede verificar, **bloquear**.
4. **Reversibilidad:** cada cambio detrás de un _feature flag_ con valor seguro por defecto → un fallo se
   revierte con un solo cambio de configuración, **sin redespliegue**.
5. **Verificación:** cada arreglo comprobado en sintaxis y desplegado bajo un protocolo controlado
   (apagar el camino crítico → cargar el código → reactivar → confirmar latido sano), validado en producción.
6. **Regla documentada permanente:** _«los controles de seguridad fallan CERRADO»_.

## 🎓 Aprendizajes

1. **_Liveness_ ≠ _Freshness_.** _«Está funcionando»_ y _«su salida sirve»_ son preguntas distintas; nunca
   respondas una con la señal de la otra.
2. **Un control de seguridad falla CERRADO.** Si un guardia no puede verificar su entrada, bloquear es seguro;
   dejar pasar es un agujero silencioso.
3. **Un bug rara vez está solo.** Un incidente suele destapar un patrón sistémico: un barrido convirtió
   _1 fallo_ en _~40 encontrados y arreglados_.
4. **La reversibilidad da confianza para shippear seguridad.** Los _feature flags_ permiten desplegar cambios
   defensivos sin miedo: una decisión equivocada está a un interruptor de revertirse.

## 🔗 Evidencia de control de versiones

El trabajo se gestionó con un flujo **rama → Pull Request → merge**, con mensajes de commit descriptivos:

- 📁 **Repositorio:** https://github.com/MimixGuy/blog-tecnico-postmortem
- 📝 **Commits:** https://github.com/MimixGuy/blog-tecnico-postmortem/commits/main
- 🔀 **Pull Request:** https://github.com/MimixGuy/blog-tecnico-postmortem/pulls?q=is%3Apr

El contenido de este blog se integró desde la rama `post/fail-open-postmortem` hacia `main` mediante un
Pull Request revisable, en lugar de empujar directo a `main` — demostrando el flujo de trabajo colaborativo.

## 💬 Reflexión: feedback radicalmente sincero

El feedback radicalmente sincero se nota más cuando **te toca a ti recibirlo o admitir tu propio error.**

**Dándolo hacia mí mismo (retractarme por escrito).** Durante la campaña marqué lo que parecía un bug serio:
una **inconsistencia de dirección** en el camino crítico (un control comprobaba una cosa y la acción parecía
usar otra). Lo reporté formalmente para revisión. Al leer el código a fondo para decidir cómo arreglarlo,
encontré una línea que había pasado por alto: el código **sí reasignaba el valor correctamente**. **No había
bug.** Lo honesto fue **retractarme explícitamente, por escrito** — _«lo leí mal, la línea X lo reasigna, no
hay inconsistencia, me equivoqué»_ — en lugar de borrarlo en silencio o defenderlo para no quedar mal.

**Recibiéndolo.** Antes, al concluir que un cambio de parámetro _«empeoraba»_ los resultados, la respuesta
fue un seco **«¿seguro?»**. En vez de defender mi primera conclusión, **rehíce el análisis desde cero**… y el
empujón tenía razón. El feedback sincero funciona en los dos sentidos: **darlo** (retirar tu propia
afirmación) y **recibirlo** (rehacer el trabajo en lugar de defenderlo).

La lección humana detrás de la técnica: en seguridad y en equipos, **la honestidad incómoda a tiempo evale
más que tener razón**. Un guardia que admite _«no sé si este dato es de fiar»_ y bloquea es más valioso que
uno que finge certeza. Lo mismo aplica a las personas.

---

## ✅ Checklist de la entrega

- [x] **Entrada de blog publicada y accesible públicamente** (este README + GitHub Pages)
- [x] **Documentación clara y estructurada según plantilla** (Contexto · Problema · Acciones · Aprendizajes)
- [x] **Evidencia de control de versiones** (repositorio público + commits + Pull Request)
- [x] **Reflexión sobre feedback radicalmente sincero incluida**

---

<sub>Escrito para audiencias técnicas y no técnicas. El proyecto concreto se describe de forma genérica a
propósito: las lecciones (`fail-open` vs `fail-closed`, _liveness_ vs _freshness_, honestidad a tiempo) son
universales y no dependen del dominio.</sub>

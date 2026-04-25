# RENOVAX Payments — Gateway para DHRU Fusion

Integración lista para usar que permite a cualquier panel **DHRU Fusion** cobrar a
través de **RENOVAX Payments**: crypto (USDT, USDC, EURC, DAI y más), PayPal, Stripe
(tarjetas) y cualquier otro método que RENOVAX agregue — todo desde un único checkout.

Cuando el pago se confirma, RENOVAX Payments envía un webhook firmado a tu DHRU y el
invoice se acredita automáticamente. No se requieren cambios en la base de datos.

---

## Tabla de contenidos

1. [Archivos incluidos](#1-archivos-incluidos)
2. [Requisitos](#2-requisitos)
3. [Configurar métodos de pago en RENOVAX Payments](#3-configurar-métodos-de-pago-en-renovax-payments)
4. [Instalación — lado DHRU](#4-instalación--lado-dhru)
5. [Referencia de configuración](#5-referencia-de-configuración)
6. [Cómo funciona](#6-cómo-funciona)
7. [Referencia de eventos webhook](#7-referencia-de-eventos-webhook)
8. [Eventos soportados y acciones](#8-eventos-soportados-y-acciones)
9. [Firewall y lista blanca WAF](#9-firewall-y-lista-blanca-waf)
10. [Seguridad](#10-seguridad)
11. [Resiliencia ante caídas de conexión](#11-resiliencia-ante-caídas-de-conexión)
12. [Solución de problemas](#12-solución-de-problemas)
13. [Soporte](#13-soporte)

---

## 1. Archivos incluidos

| Archivo | Cópialo a (en tu servidor DHRU) |
| --- | --- |
| `modules/gateways/renovaxpayments.php` | `/modules/gateways/renovaxpayments.php` |
| `renovaxpaymentscallback.php` | `/renovaxpaymentscallback.php` (raíz pública) |

> Solo se necesitan **2 archivos**. No hay ningún archivo de panel admin adicional.

---

## 2. Requisitos

| Requisito | Detalle |
| --- | --- |
| DHRU Fusion | Cualquier versión con acceso al filesystem y la ruta `modules/gateways/` |
| Cuenta RENOVAX Payments | Merchant activo en [payments.renovax.net](https://payments.renovax.net) |
| PHP | 7.4 o superior, con las extensiones `curl` y `hash` habilitadas |
| HTTPS | Obligatorio — RENOVAX Payments solo entrega webhooks a URLs `https://` |

---

## 3. Configurar métodos de pago en RENOVAX Payments

Antes de instalar los archivos de DHRU, tu cuenta merchant en RENOVAX Payments necesita
al menos un **método de pago activo**. Esto es lo que verán tus clientes en el checkout
(wallets crypto, formulario de tarjeta, botón PayPal, etc.).

Ve a **Merchants → (tu merchant) → Payment Methods → Add**.

---

### Criptomonedas (USDT, USDC, EURC, DAI y más)

Hay **un único método Crypto por merchant** que cubre todos los tokens y redes soportadas.
Ve a **Payment Methods → Add → Crypto**.

**Paso 1 — Direcciones de wallet.** Ingresa la dirección receptora para cada familia de
blockchain que quieras aceptar (EVM, Tron, Solana, TON, Stellar, Sui, Aptos).
Solo completa las familias que vayas a usar.

> Son direcciones solo de depósito. RENOVAX Payments monitorea las transacciones entrantes
> y te notifica vía webhook — nunca inicia transferencias.

**Paso 2 — Pares token × red.** Aparecerá una matriz con todos los stablecoins
soportados (USDT, USDC, EURC, DAI, FDUSD, PYUSD, TUSD, USDE, EURT…) y las redes
disponibles según las direcciones que ingresaste. Activa solo las combinaciones que
quieras ofrecer.

Puedes activar o desactivar pares individuales en cualquier momento desde la lista de
métodos, sin deshabilitar el método crypto completo.

**Paso 3 — Tolerancia de pago (opcional).**

| Ajuste | Por defecto |
| --- | --- |
| Tolerancia absoluta (faltante en USD aceptado) | `0.01` |
| Tolerancia porcentual (% del monto del invoice) | `0.5 %` |
| Aceptar pagos de más | habilitado |

---

### Stripe (tarjetas de crédito y débito)

Puedes agregar **múltiples cuentas Stripe** al mismo merchant (p. ej. una por moneda o región).
Ve a **Payment Methods → Add → Stripe** y completa:

| Campo | Notas |
| --- | --- |
| **Label** | Nombre interno, p. ej. `Stripe USD` |
| **Secret key** | `sk_live_...` |
| **Publishable key** | `pk_live_...` |
| **Webhook secret** | `whsec_...` — **obligatorio** |
| **Settlement currency** | Solo si tu cuenta Stripe liquida en una moneda diferente |

**Política de países (opcional):** restringe qué países pueden usar esta cuenta.

- **Modo Block** — acepta todos los países *excepto* los listados.
- **Modo Allow** — acepta *solo* los países listados.

---

### PayPal

Ve a **Payment Methods → Add → PayPal** y completa:

| Campo | Notas |
| --- | --- |
| **Label** | Nombre interno, p. ej. `PayPal USD` |
| **Client ID** | De tu aplicación PayPal |
| **Client Secret** | De tu aplicación PayPal |
| **Webhook ID** | **Obligatorio** |

---

### PIX (Brasil — solo BRL)

Ve a **Payment Methods → Add → PIX** y completa:

| Campo | Notas |
| --- | --- |
| **Label** | Nombre interno, p. ej. `PIX BRL` |
| **Provider** | `efi` o `gerencianet` |
| **Client ID** | De tu cuenta EFI Bank |
| **Client Secret** | De tu cuenta EFI Bank |
| **PIX key** | Tu llave PIX registrada |

El cliente ve un código QR inline — sin redireccionamiento.

---

### Mercado Pago

Ve a **Payment Methods → Add → Mercado Pago** y completa:

| Campo | Notas |
| --- | --- |
| **Label** | Nombre interno, p. ej. `Mercado Pago ARS` |
| **Access token** | De tu cuenta Mercado Pago |
| **Public key** | De tu cuenta Mercado Pago |

---

### Activar y desactivar métodos

Desde **Merchants → Payment Methods**:

- Haz clic en el ícono de **pausa / play** junto a cualquier método para activarlo o
  desactivarlo al instante.
- Para Crypto, también puedes activar/desactivar **pares individuales token × red** sin
  deshabilitar el método crypto completo.
- Los métodos deshabilitados no aparecen en el checkout — los clientes no pueden usarlos.

---

### Cómo ven el checkout los clientes

Cuando un cliente abre el link de pago (`pay_url`) ve una grilla con todos los métodos activos:

1. **Crypto** — lleva a un selector de token/red. El cliente elige (p. ej. USDT en Tron)
   y recibe la dirección exacta + monto a enviar, con QR y deep-links de wallets
   (MetaMask, Phantom, TronLink, etc.).
2. **Stripe** — redirige al formulario de tarjeta hosteado por Stripe.
3. **PayPal** — redirige al flujo hosteado de PayPal.
4. **PIX** — muestra un QR inline, sin redireccionamiento.
5. **Mercado Pago** — redirige al checkout de Mercado Pago.

Si el merchant tiene **múltiples cuentas del mismo tipo**, el cliente ve una pantalla
intermedia para elegir entre ellas antes de continuar.

---

## 4. Instalación — lado DHRU

### Paso 1 — Obtener credenciales de RENOVAX Payments

Necesitas dos valores secretos. Pídele al operador de RENOVAX Payments que:

1. Abra **Merchants → (tu merchant) → Edit → API Tokens**.
2. Haga clic en **Create** y copie el valor del token inmediatamente — **se muestra una sola vez**.
3. En la misma página, copie el **Webhook Secret**.

Guarda ambos valores para los pasos siguientes.

---

### Paso 2 — Registrar la URL del webhook

En la configuración del merchant en RENOVAX Payments, establece la URL del webhook:

```text
https://TU-DOMINIO-DHRU.com/renovaxpaymentscallback.php
```

Reemplaza `TU-DOMINIO-DHRU.com` con tu dominio real.

---

### Paso 3 — Subir los archivos

Copia los dos archivos PHP a tu servidor DHRU en las rutas indicadas arriba.
Asegúrate de que el servidor web pueda leerlos:

```bash
chmod 644 modules/gateways/renovaxpayments.php
chmod 644 renovaxpaymentscallback.php
```

---

### Paso 4 — Activar el gateway en el admin de DHRU

1. Entra al panel de administración de DHRU.
2. Ve a **Configuration → Payment Gateways**.
3. Busca **RENOVAX Payments** en la lista y haz clic en **Edit**.
4. Completa los campos (ver [Referencia de configuración](#5-referencia-de-configuración) abajo).
5. Establece **Status** en **Active** y guarda.

El gateway ya está activo para tus clientes en el checkout.

---

## 5. Referencia de configuración

| Campo | Descripción | Valor por defecto |
| --- | --- | --- |
| **API Base URL** | Endpoint de la API de RENOVAX Payments. No lo cambies salvo indicación. | `https://payments.renovax.net` |
| **Bearer Token** | Token del Paso 1. Pégalo tal cual — sin espacios. | *(requerido)* |
| **Webhook Secret** | Secreto HMAC del Paso 1. Verifica la autenticidad del webhook. | *(requerido)* |
| **Invoice TTL (min)** | Minutos que el link de pago permanece válido antes de expirar. | `15` |

> La **moneda** se hereda del perfil del merchant en RENOVAX Payments.
> No hace falta configurarla aquí.

---

## 6. Cómo funciona

```text
El cliente hace clic en "Pagar"
        │
        ▼
renovaxpayments.php  ──POST /api/merchant/invoices──▶  API de RENOVAX Payments
        │                                                       │
        │◀──────────────────── pay_url ─────────────────────────┘
        │
        ▼
El cliente es redirigido al checkout de RENOVAX Payments
(elige Crypto / PayPal / Stripe / etc.)
        │
        ▼
Pago confirmado (on-chain o por el procesador)
        │
        ▼
RENOVAX Payments envía webhook firmado con HMAC
        │
        ▼
renovaxpaymentscallback.php
  1. Verifica la firma HMAC
  2. Responde 200 OK inmediatamente (cierra la conexión)
  3. Acredita el invoice de DHRU en segundo plano
```

### Qué hace cada archivo

- **`modules/gateways/renovaxpayments.php`** es cargado por DHRU cuando el cliente
  hace clic en el botón de pago. Llama a la API de RENOVAX Payments con el monto del
  invoice, guarda `dhru_invoiceid` en el metadata y devuelve un botón que redirige a `pay_url`.

- **`renovaxpaymentscallback.php`** recibe el webhook, verifica su firma HMAC,
  reescribe los montos del invoice de DHRU (bruto / neto / comisiones) y llama a
  `addpayment()` de DHRU para acreditar los fondos.

- El `dhru_invoiceid` viaja en el `metadata` del invoice, por eso no se necesitan
  columnas ni tablas adicionales en la base de datos.

---

## 7. Referencia de eventos webhook

RENOVAX Payments hace POST de un cuerpo JSON como este a tu URL de callback:

```json
{
  "event_type": "invoice.paid",
  "invoice_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "confirmed",
  "invoice_amount": "100.00",
  "invoice_currency": "USD",
  "amount_received_fiat": "100.00",
  "amount_net_fiat": "97.00",
  "fee": "3.00",
  "tx_hash": "0xabc123...",
  "network": "tron",
  "paid_at": "2026-04-25T10:05:00Z",
  "metadata": {
    "dhru_invoiceid": "42",
    "dhru_email": "cliente@ejemplo.com",
    "dhru_systemurl": "https://tu-dhru.com/"
  }
}
```

### Headers de la petición

| Header | Valor |
| --- | --- |
| `X-Renovax-Signature` | `sha256={hmac}` — HMAC-SHA256 del cuerpo raw con tu `webhook_secret` |
| `X-Renovax-Event-Type` | String del tipo de evento (igual que `event_type` en el body) |
| `X-Renovax-Event-Id` | UUID único por entrega — útil para idempotencia adicional |

---

## 8. Eventos soportados y acciones

| Evento | Condición | Acción |
| --- | --- | --- |
| `invoice.paid` | `status = confirmed` | Reescribe montos y acredita el invoice de DHRU |
| `invoice.overpaid` | `status = confirmed` | Igual que arriba — el cliente pagó de más, se acredita el total recibido |
| `invoice.partial` | `status = confirmed` | Igual que arriba — pago parcial acreditado según lo recibido |
| `invoice.expired` | cualquiera | Reconocido (200) e ignorado — sin acreditaciones |
| Cualquier otro evento | cualquiera | Reconocido (200) e ignorado |

### Cómo se calculan los montos

El callback reescribe el invoice de DHRU para reflejar correctamente lo que el cliente
pagó, incluyendo las comisiones de RENOVAX Payments:

```text
tax      = (invoice.taxrate% × amount_net_fiat) + invoice.fixedcharge + renovax_fee
subtotal = amount_received_fiat − tax
total    = subtotal   ← hace que el invoice se cierre correctamente en DHRU
```

El ítem `AddFunds` se actualiza a `subtotal` para que DHRU acredite la billetera con el
monto neto. Se agrega una nota de auditoría completa al invoice.

---

## 9. Firewall y lista blanca WAF

Dos hosts deben poder comunicarse a través de cualquier firewall, proxy inverso o WAF
que proteja tu servidor DHRU.

### Saliente — tu servidor DHRU → RENOVAX Payments

`renovaxpayments.php` realiza una llamada HTTPS saliente cada vez que un cliente hace clic en Pagar.
Permite TCP 443 hacia:

```text
payments.renovax.net
```

Si tu firewall solo acepta IPs (no hostnames), contacta
[payments.renovax.net/support](https://payments.renovax.net/support) para obtener el
rango de IPs actual de `payments.renovax.net`. Las IPs pueden cambiar; usa el hostname
siempre que sea posible.

### Entrante — RENOVAX Payments → tu servidor DHRU

RENOVAX Payments envía eventos webhook a `renovaxpaymentscallback.php` desde sus propios
servidores. Tu WAF o firewall debe:

1. **Permitir `POST /renovaxpaymentscallback.php`** desde las IPs de RENOVAX Payments.
   Solicita la lista de IPs de egreso actual en
   [payments.renovax.net/support](https://payments.renovax.net/support).

2. **Dejar pasar estos headers sin modificarlos** — si son eliminados o reescritos,
   cada webhook fallará con `401 invalid_signature`:

   | Header | Propósito |
   | --- | --- |
   | `X-Renovax-Signature` | Firma HMAC-SHA256 del cuerpo raw |
   | `X-Renovax-Event-Type` | Tipo de evento (p. ej. `invoice.paid`) |
   | `X-Renovax-Event-Id` | UUID único por entrega |
   | `Content-Type` | Debe llegar como `application/json` |

3. **No almacenar en buffer ni modificar el cuerpo de la petición.** El HMAC se calcula
   sobre los bytes exactos recibidos. Cualquier transformación (reformateo, re-encoding,
   gzip) invalidará la firma.

### Reglas WAF comunes a revisar

| Tipo de regla | Qué verificar |
| --- | --- |
| Rate limiting | Añade las IPs de RENOVAX Payments a la lista blanca en `renovaxpaymentscallback.php` — las tormentas de reintentos pueden disparar límites |
| Límite de tamaño de body | Los webhooks pesan menos de 4 KB; un límite de 1 MB es más que suficiente |
| Validación / normalización JSON | Deshabilítala para este endpoint — el body debe llegar a PHP sin modificar |
| Protección anti-bots / CAPTCHA | Excluye `renovaxpaymentscallback.php` de cualquier challenge de JS o CAPTCHA |
| Bloqueo geográfico | La infraestructura de RENOVAX Payments puede egressar desde múltiples regiones — usa lista blanca de IPs en lugar de reglas por país |

---

## 10. Seguridad

- **Verificación de firma** usa `hash_equals()` (comparación en tiempo constante — protege contra ataques de timing).
- **Idempotencia** — si el invoice de DHRU ya está en estado `Paid` o el ID de transacción ya fue registrado, el webhook se reconoce y se descarta sin acreditar dos veces.
- **Bearer Token** y **Webhook Secret** se almacenan como campos de tipo `password` en DHRU — no se muestran en la UI del admin tras guardar.
- Nunca registres ni expongas el `webhook_secret` en logs, código frontend o mensajes de error.
- Sirve siempre `renovaxpaymentscallback.php` a través de **HTTPS**.

---

## 11. Resiliencia ante caídas de conexión

El callback está diseñado para completarse aunque la conexión de red se corte a mitad:

1. `ignore_user_abort(true)` se establece al inicio — el script sigue ejecutándose aunque el otro extremo se desconecte.
2. `set_time_limit(120)` da suficiente tiempo para que finalicen las operaciones en DB.
3. Tras verificar el HMAC y validar el payload, el callback **envía inmediatamente una respuesta `200 OK`** y cierra la conexión HTTP con RENOVAX Payments.
4. Todas las escrituras en base de datos (chequeo de idempotencia, reescritura del invoice, `addpayment`) ocurren **después** de enviar la respuesta, en segundo plano.

Esto garantiza que RENOVAX Payments recibe su `200` rápidamente y nunca reintentará por un timeout en las operaciones de DB de tu servidor.

> En PHP-FPM (la configuración más común) el flush usa `fastcgi_finish_request()`.
> En mod_php / CGI usa output buffering + `Connection: close` como fallback.

---

## 12. Solución de problemas

### 401 invalid_signature

El `webhook_secret` en DHRU no coincide con el del merchant en RENOVAX Payments.
Ve a la configuración del merchant, copia el secret de nuevo y pégalo en la
configuración del gateway de DHRU **sin espacios al inicio ni al final**.

### 400 missing_dhru_invoiceid

El callback recibió un webhook pero el campo `metadata.dhru_invoiceid` estaba vacío.
Normalmente significa que `modules/gateways/renovaxpayments.php` no fue subido o es
una versión antigua. Vuelve a subir el archivo de este paquete.

### El botón de pago no aparece en el checkout

- Confirma que `modules/gateways/renovaxpayments.php` está en la ruta correcta.
- Verifica que el gateway esté en estado **Active** en el admin de DHRU.
- Asegúrate de que el Bearer Token esté configurado (sin espacios).

### No aparecen métodos de pago en el checkout

El cliente abre el `pay_url` pero no ve ninguna opción de pago. Significa que el merchant
no tiene métodos activos. Vuelve a la [sección 3](#3-configurar-métodos-de-pago-en-renovax-payments)
y agrega al menos uno.

### El invoice no se acredita después del pago

1. Revisa el log de errores de PHP buscando líneas que empiecen con `[renovaxpayments]`.
2. Verifica que la webhook URL en RENOVAX Payments apunte a `https://TU-DOMINIO/renovaxpaymentscallback.php`.
3. Consulta el estado del invoice directamente:

```bash
curl https://payments.renovax.net/api/merchant/invoices/{invoice_id} \
     -H "Authorization: Bearer {tu_token}"
```

### Reintentos automáticos

Si tu servidor responde con `5xx` (o hay timeout), RENOVAX Payments reintenta con
backoff exponencial: **30s → 2min → 10min → 1h → 6h → 24h** (6 intentos en total).
Una respuesta `200` detiene los reintentos de inmediato.

### Dónde encontrar los logs

Los errores se escriben mediante `error_log()` de PHP. La ubicación depende de tu servidor:

- **Apache**: directiva `error_log` en `httpd.conf` o en la configuración del virtual host.
- **Nginx + PHP-FPM**: la configuración `error_log` del pool (normalmente `/var/log/php-fpm/www-error.log`).

---

## 13. Soporte

[payments.renovax.net/support](https://payments.renovax.net/support)

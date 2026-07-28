# Fresko App

Tienda móvil de frutas y verduras frescas para Panamá. App estática 100% (HTML + CSS + JS moderno + Firebase).

Funciona en:
- GitHub Pages
- https://www.freskoapp.com
- Cualquier hosting estático

## Archivos entregados

- `index.html` — App completa (cliente + admin)
- `firestore.rules` — Reglas de seguridad Firestore
- `storage.rules` — Reglas de seguridad Storage
- `README.md` — Esta guía

## Requisitos

- Cuenta Firebase (plan gratuito es suficiente)
- GitHub (para Pages) o hosting estático propio

## 1. Crear el proyecto en Firebase

1. Ve a https://console.firebase.google.com
2. Crea un nuevo proyecto llamado **freskoapp** (o el que prefieras). El Project ID del ejemplo es `freskoapp-c495d`.
3. En el proyecto habilita los productos:
   - **Authentication** → Email/Contraseña
   - **Firestore Database** → Crea en modo producción (reglas las subiremos)
   - **Storage** → Activa

## 2. Configurar dominios autorizados (¡importante!)

En Firebase Console → Authentication → Settings → Authorized domains, agrega **exactamente**:

```
www.freskoapp.com
freskoapp.com
danchacho23-dot.github.io
localhost
```

Sin esto el login fallará en producción.

## 3. Subir a GitHub + GitHub Pages

1. Crea un repositorio nuevo en GitHub (público o privado).
2. Sube los 4 archivos (o al menos `index.html` en la raíz).
3. Ve a Settings → Pages → Source: **Deploy from a branch** → Branch: `main` (o master) /root → Save.
4. Espera 1 minuto. Tu sitio estará en:
   `https://<tu-usuario>.github.io/<repo>/`
5. (Opcional) Configura dominio personalizado en GitHub + Firebase auth domains.

## 4. Instalar las reglas de seguridad

### Opción A (recomendada): Firebase CLI

```bash
npm install -g firebase-tools
firebase login
firebase init
# Elige Firestore y Storage cuando pregunte
# Usa los archivos firestore.rules y storage.rules que te di
firebase deploy --only firestore,storage
```

### Opción B: Consola web (rápida)

- Firestore → Rules → pega todo el contenido de `firestore.rules` → Publish
- Storage → Rules → pega todo el contenido de `storage.rules` → Publish

## 5. Crear el usuario administrador

1. En Firebase Console → Authentication → Users → Add user
2. Usa el email y contraseña que quieras (ej: `admin@freskoapp.com`).
3. **Copia el UID** que Firebase le asignó al usuario.
4. **Actualiza el UID en dos lugares**:
   - Dentro de `index.html` (busca `ADMIN_UID`)
   - Dentro de `firestore.rules` y `storage.rules` (la línea `request.auth.uid == '...'`)
5. Si usas el UID exacto provisto `gDK9Zay7iwSDh2dCdLdzXuR30PB2`, solo funciona si creas la cuenta y ese es el UID resultante (poco común). Lo normal es reemplazar por el UID real.

**Nunca guardes la contraseña en el código.**

## 6. Cargar el catálogo inicial (58 productos)

1. Abre la app (en GitHub Pages o localhost).
2. Haz clic en **Admin** (arriba derecha).
3. Inicia sesión con el email/contraseña del admin.
4. Si la colección `products` está vacía, aparecerá el botón **Cargar catálogo inicial (58 productos)**.
5. Haz clic, confirma y espera. Usa `writeBatch()` internamente.
6. El botón desaparece después de cargar.
7. ¡Listo! Los productos aparecen para todos los visitantes.

**El botón solo funciona si NO hay productos** y solo lo ve el admin.

## 7. Probar desde otro teléfono

- Abre la URL de GitHub Pages o tu dominio.
- Agrega productos al carrito.
- Prueba el buscador y filtros de categoría.
- Ve al carrito → completa nombre, dirección, método de pago y notas.
- Toca "Enviar pedido por WhatsApp" (usa el número 50769829621).
- Verifica que el mensaje incluya todo correctamente.

## 8. Reemplazar / agregar fotos reales

- En modo Admin entra a editar cualquier producto.
- Elige una foto desde tu teléfono (la app la comprime automáticamente a máx 800×800 WebP calidad 0.8).
- Verás progreso y mensajes claros de error.
- La foto se sube a Firebase Storage en `products/<order>.webp` y se guarda la URL pública en el documento.
- Los clientes ven la foto real. Si falla la carga se muestra el emoji.

## 9. Personalizar la tienda

En el panel Admin (sección "Tienda"):
- Nombre, subtítulo
- Horario
- Texto de delivery
- Delivery fee y mínimo de orden
- Interruptor "Abierta / Cerrada"

Los cambios se sincronizan en tiempo real con todos los clientes vía `onSnapshot`.

## 10. Revisar la consola del navegador

Abre DevTools (F12) → Console. Verás:

- Mensajes claros de Firebase (login, permisos, red, guardado, borrado, subida)
- Al final: `Tests internos: X/X pasaron`

Pruebas incluidas:
- 58 productos
- Banana = orden 1
- Jengibre = orden 58
- Piña no existe
- Número de WhatsApp correcto
- Cálculo de total con delivery
- Lógica de ocultos y agotados

## 11. Estructura de datos (resumen)

**products/{productId}**
```js
{
  id: string,
  name: string,
  category: "Frutas" | "Verduras" | "Tubérculos" | "Hierbas" | "Otros",
  price: number,
  unit: string,
  emoji: string,
  photo: string | null,   // URL de Storage o null
  description: string,
  visible: boolean,
  available: boolean,
  order: number
}
```

**settings/store**
```js
{
  name: string,
  subtitle: string,
  hours: string,
  delivery: string,
  deliveryFee: number,
  minimumOrder: number,
  isOpen: boolean
}
```

**orders/{orderId}** (creado automáticamente al enviar un pedido por WhatsApp)
```js
{
  items: [{ id, name, unit, qty }],   // sin precio: los empleados leen esto
  customerName: string,
  address: string,
  paymentMethod: string,
  notes: string | null,
  storeOpenAtOrder: boolean,
  status: "sent" | "fulfilled" | "cancelled",
  createdAt: Timestamp
}
```
Cualquier visitante puede **crear** un pedido (así queda un registro aunque el cliente no complete el envío por WhatsApp). Dueño y empleados pueden **leer** pedidos y cambiar `status`; solo el dueño puede borrarlos.

**orders/{orderId}/financials/summary** (mismo momento de creación, en subcolección aparte)
```js
{
  items: [{ id, name, price, qty, lineTotal }],
  subtotal: number,
  deliveryFee: number,
  total: number,
  createdAt: Timestamp
}
```
Solo el **dueño** puede leer esta subcolección — los montos en dólares nunca están en el documento principal del pedido, así que una cuenta de empleado no puede obtenerlos bajo ninguna circunstancia (no es solo que la interfaz los oculte).

**staff/{uid}** (opcional — define quién es dueño/empleado además del `ADMIN_UID` original)
```js
{
  role: "owner" | "employee"
}
```
El `ADMIN_UID` configurado en el código siempre es dueño, incluso sin un documento aquí. Para agregar más cuentas (otro dueño, o empleados), crea el usuario en Authentication, copia su UID y crea un documento en `staff` con ese UID como ID del documento. Ver sección "Cuentas de empleados" más abajo.

## Seguridad implementada

- Clientes: solo lectura de `products/*` y `settings/store`
- Solo el **dueño** puede escribir en productos, settings y subir/borrar en Storage `/products/*`
- **Dueño y empleados** pueden leer y actualizar el estado de `orders/*`, pero **solo el dueño** puede leer los montos en `orders/*/financials/*`
- Todas las demás rutas bloqueadas
- Reglas completas listas para copiar

## Cuentas de empleados (opcional)

Por defecto solo existe la cuenta dueño (`ADMIN_UID`). Para dar acceso a un trabajador que vea y marque pedidos como entregados, **sin ver montos en dólares**:

1. Firebase Console → Authentication → Users → Add user. Crea el email/contraseña del empleado.
2. Copia el UID que Firebase le asignó.
3. Firestore Database → Data → colección `staff` → Add document. Usa ese UID como **ID del documento** (no como campo), y agrega el campo `role` con valor `employee` (string).
4. El empleado ahora puede tocar **Admin**, iniciar sesión, y ver el panel — mostrará solo "Pedidos recientes" (sin Tienda ni Productos), con nombre/dirección/artículos por pedido y botones para marcar Entregado/Cancelar, pero nunca un monto en $.

Para dar a alguien más el rol de dueño completo, repite el mismo proceso pero con `role: owner`.

## Notas importantes

- Cada pedido se guarda en Firestore (`orders` + `orders/{id}/financials`) al momento de tocar "Enviar por WhatsApp", antes de abrir el enlace de wa.me. Así el pedido queda registrado aunque el cliente cierre WhatsApp sin enviarlo. Revísalos en el panel Admin (sección "Pedidos recientes") con cualquier cuenta de dueño o empleado.
- Carrito se guarda solo en localStorage del navegador.
- Al recargar productos (onSnapshot) se sincroniza: quita ocultos/borrados, actualiza precios, bloquea checkout si hay agotados.
- No hay cuentas de cliente, ni pagos online (solo WhatsApp).
- No existe "Piña" en el catálogo.
- Todo usa `try/catch`, botones se deshabilitan durante operaciones, mensajes claros al usuario.
- Mobile-first, 2 columnas en teléfono, diseño verde fresco.

## Troubleshooting rápido

- Login no funciona → revisa dominios autorizados y que el UID en código coincida con el usuario.
- No aparece botón "Cargar catálogo" → ya hay productos o no estás logueado como admin.
- Las imágenes no cargan → revisa Storage rules y que el archivo exista.
- Cambios no se ven en clientes → espera 1-2s (onSnapshot) o revisa consola.
- Error de permisos → UID incorrecto en rules.

## Licencia

Uso libre para Fresko. ¡Mantén el espíritu fresco!

---

Desarrollado siguiendo exactamente la especificación entregada. 4 archivos completos, cero dependencias externas, cero build tools.

---
stoplight-id: ixe0xipl5zmpd
---

# v4_login

`v4_login` es el microservicio de Metacontratas encargado de la autenticación y gestión de los usuarios. A través de este microservicio se obtienen los JWTs (Access Tokens) necesarios para poder realizar llamadas a los otros microservicios de la aplicación.

Este microservicio actúa como intermediario con Keycloak, que es el proveedor de identidad utilizado por la plataforma. En concreto `v4_login` se encarga de:

- Validar las credenciales de los usuarios (email y contraseña).
- Redirigir a los usuarios a sus proveedores de identidad si están federados.
- Gestionar el refresco de tokens y la persistencia de sesiones.
- Controlar los flujos de autenticación mediante código de autorización.
- Gestionar la suplantación de identidad, permitiendo a ciertos usuarios autenticarse en nombre de otros (por ejemplo, administradores o técnicos de soporte).

---

## Conexión con Keycloak

La generación de los tokens de autenticación para Metacontratas se ha implementado de la siguiente forma:

- Un usuario que necesita un token hace una llamada a un endpoint de v4_login.
- v4_login hace las comprobaciones necesarias sobre esa llamada (por ejemplo: comprueba que el usuario y la contraseña son correctos, que el usuario puede obtener dicho token, etc.) → capa AutenticacionService.java
- v4_login calcula un mapa de claims a incluir en el Access Token que se generará→ capa ClaimsService.java
- v4_login realiza una llamada al servidor Keycloak pidiéndole un token para un determinado usuario y pasándole el mapa de claims adicionales a incluir calculado e el paso anterior.
- Keycloak devuelve el token generado a v4_login que lo pasa al usuario.

---

## Tipos de tokens

Hasta ahora, el Backend solo generaba un JWT con un tiempo de expiración prolongado. Sin embargo, con la integración de **Keycloak**, ahora es posible utilizar una variedad de tokens de autenticación siguiendo el estándar, que incluye:

- **Access Tokens**: Permiten acceder a recursos protegidos después de la autenticación y tienen un tiempo de vida limitado. Son equivalentes al JWT único que generaba el Backend anteriormente.

- **Refresh Tokens**: Permiten obtener nuevos Access Tokens sin que el usuario tenga que autenticarse de nuevo, es decir, sirven para "refrescar" los Access Tokens.

- **ID Tokens**: Verifican la identidad del usuario autenticado y proporcionan información sobre él. Se utilizan en situaciones donde es necesario confirmar la identidad del usuario, como al cerrar una sesión en Keycloak.

- **Action Tokens**: Están diseñados para realizar acciones sensibles y específicas, como restablecer una contraseña o aceptar una cita, entre otras.

> Todos los tokens mencionados anteriormente son JWTs firmados con una clave privada y verificados con una clave pública.  
> Por esta razón, **no se recomienda utilizar el término "JWT" en el código**, sino referirse a ellos como `"Access Token"`, `"Refresh Token"`, etc.

Además, es importante tener en cuenta que **en cada petición de tokens a Keycloak se obtiene un conjunto de tokens**:  
`Access Token`, `Refresh Token`, `ID Token`.

En el caso de los **Action Tokens**, solo se recibe el correspondiente `Action Token`, ya que no se refrescan.

Metacontratas no emite siempre el mismo “tipo de Access Token” con el mismo contenido.  
El contenido de cada token depende del contexto en el que el usuario se encuentra dentro de la aplicación.

Por ejemplo, un Access Token obtenido al iniciar sesión **no tendrá el mismo contenido** que uno obtenido al acceder a un entorno específico.

### Personalización de tokens

Metacontratas **solo incluye claims adicionales** en:

- Access Tokens
- Action Tokens

Los **Refresh Tokens** y **ID Tokens** se utilizan tal y como los emite Keycloak.

### Token de login

Access Token que obtiene un usuario tras autenticarse con sus credenciales (usuario/contraseña o proveedor de identidad).  
Se utiliza cuando el usuario aún no ha accedido a un entorno (pantalla de tarjetas de acceso).

- **Claims**: `ClaimsService#claimsLogin`
- **Se emite en los endpoints**:
  - `/login/v1/usuarios/autenticar`
  - `/login/v1/usuarios/autenticar-codigo-autorizacion`
  - `/login/v1/usuarios/{idUsuario}/autenticar`

### Token de entorno

Access Token que se genera al seleccionar un acceso a un entorno concreto (al pulsar en una tarjeta).  
Incluye más información específica del entorno: permisos, ID de entorno, ID de empresa, etc.

- **Claims**: `ClaimsService#claimsEntorno`
- **Se emite en el endpoint**:
  - `/login/v1/usuarios/{idUsuario}/accesos/{idAcceso}/autenticar`

### Token admin

Access Token para tareas administrativas en un entorno, sin necesidad de acceder mediante una tarjeta.

- **Claims**: `ClaimsService#claimsAdmin`
- **Se emite en el endpoint**:
  - `/login/v1/entornos/{idEntorno}/autenticar`

### Token interno

Access Token utilizado en llamadas internas entre microservicios.  
Incluye el rol especial `dios`.  
No se expone al front, solo se utiliza para comunicaciones internas.

- **Claims**: `ClaimsService#claimsInterno`
- **Se emite en los endpoints**:
  - `/login/v1/private/autenticar`
  - `/login/v1/private/autenticarjwt`

### Token público

Action Token emitido con roles específicos para ejecutar una tarea concreta.  
Se usa para operaciones como aceptar una cita o acceder temporalmente a un recurso.

- **Claims**: `ClaimsService#claimsPublico`
- **Se emite en los endpoints**:
  - `/login/v1/private/generar-jwt-publico`
  - `/login/v1/public/autenticar`

### Identificación del tipo de token

Para comprobar el tipo de un Access Token, basta con revisar el claim:

```json
"metacontratas_token_type": "login" | "entorno" | "admin" | "interno" | ...
```

Este claim se incluye por claridad y facilita el control en el consumo de los tokens por parte del frontend o los servicios internos.

---

## Flujos importantes

Los flujos más relevantes a realizar en este microservicio son los siguientes:

### Flujo 1. Refresco de tokens

Para refrescar un Access Token (sea del tipo que sea) el flujo es el siguiente:

> El usuario debe contar con un Access Token a punto de expirar o expirado y con un Refresh Token.

1. **(front:metalogin)** Llama al endpoint `/login/v1/public/refrescar` incluyendo el Access Token a refrescar y el Refresh Token del usuario.
2. **(back:v4_login)** Refresca el Access Token incluyendo todos los claims adicionales que tenía el expirado, y devuelve un nuevo conjunto de tokens de autenticación.

Existen otros endpoints distintos a `/login/v1/public/refrescar` que también requieren el Refresh Token del usuario. Estos endpoints **no refrescan el Access Token recibido**, sino que **necesitan el Refresh Token para mantener la misma sesión de Keycloak del usuario** y evitar crear sesiones arbitrarias.

### Flujo 2. Autenticación por contraseña

Para obtener un Access Token de login al iniciar sesión utilizando email y contraseña:

1. **(front:metalogin)** Llama al endpoint `/login/v1/usuarios/comprobar-autenticacion` con el email del usuario.
2. **(back:v4_login)** Devuelve una URL de autenticación si el usuario está federado.
3. **(front:metalogin)** Si la URL de autenticación viene vacía (el usuario no está federado), llama al endpoint `/login/v1/usuarios/autenticar` con el email y la contraseña del usuario.
4. **(back:v4_login)** Devuelve un conjunto de tokens de autenticación de login.

Este flujo de autenticación es un estándar implementado por Keycloak y se denomina **Resource Owner Password Flow**.

### Flujo 3. Autenticación por proveedor de identidad

Para obtener un Access Token utilizando la página de inicio de sesión única (SSO) de un proveedor de identidad:

1. **(front:metalogin)** Llama al endpoint `/login/v1/usuarios/comprobar-autenticacion` con el email del usuario.
2. **(back:v4_login)** Devuelve una URL de autenticación si el usuario está federado.
3. **(front:metalogin)** Si la URL no viene vacía, redirige al navegador a dicha URL.

La URL contiene un `Query Param` (`redirect_uri`) con la dirección del front donde Keycloak enviará un **código de autorización** tras iniciar sesión.  
> Ver propiedad `keycloak.authorization-code-flow.redirect-uri` en `v4_login/application.properties`.

4. **(Usuario)** Inicia sesión con su proveedor de identidad.
5. **(Keycloak)** Si el inicio de sesión es exitoso, busca el usuario en la base de datos de `v4_login`, y si no existe lo crea.
6. **(Keycloak)** Redirige al navegador a la URL especificada en `redirect_uri` incluyendo el parámetro `code` con el código de autorización.
7. **(front:metalogin)** Captura el código y llama al endpoint `/login/v1/usuarios/autenticar-codigo-autorizacion`.
8. **(back:v4_metacontratas)** Devuelve un conjunto de tokens de autenticación de login.

Este flujo también es estándar en Keycloak y se denomina **Authorization Code Flow**.

### Flujo 4. Suplantación de un usuario

Para obtener un Access Token suplantando a otro usuario:

> El usuario debe contar con un Access Token de login obtenido por cualquiera de los flujos anteriores.

1. **(front:metalogin)** Llama al endpoint `/login/v1/usuarios/{idUsuario}/autenticar` con el ID del usuario a suplantar.
2. **(back:v4_metacontratas)** Devuelve un conjunto de tokens de autenticación suplantados.

#### Recuperar sesión real desde una sesión suplantada

> El usuario debe contar con un Access Token de login **suplantado**.

1. **(front:metalogin)** Llama al endpoint `/login/v1/usuarios/{idUsuario}/autenticar` con **su propio ID de usuario**.
2. **(back:v4_metacontratas)** Devuelve un conjunto de tokens de autenticación **sin suplantación**.

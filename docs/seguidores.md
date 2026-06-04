# Sistema de seguidores (followers / following)

El endpoint de follow es un **toggle**: una sola llamada sirve tanto para seguir
como para dejar de seguir. El backend comprueba el estado actual y ejecuta la
acción contraria. Requiere JWT.

Los datos almacenados dentro de cada usuario son subdocumentos con
`{id, username, first_name, last_name}` — sin contraseñas ni emails.

---

## Endpoints

| Acción | Método | Ruta |
|---|---|---|
| Seguir / dejar de seguir | POST | `/api/users/follow/<target_user_id>` |
| Mis seguidores y seguidos | GET | `/api/users/me/follows` |

Requieren `Authorization: Bearer <token>`.

---

## Lógica de toggle

```
¿current_user ya sigue a target_user?
  NO  →  agrega target en following de current  +  current en followers de target  →  action = "followed"
  SÍ  →  elimina target de following de current +  current de followers de target  →  action = "unfollowed"
```

No se puede seguir a uno mismo: el backend devuelve 400.

---

## Estructura en MongoDB

Después del primer follow, los documentos de usuarios contienen:

```jsonc
// Documento del usuario que sigue (current_user)
{
  "_id": "...",
  "username": "alice",
  // ... otros campos ...
  "following": [
    {
      "id":         "64b1f2c3d4e5f6a7b8c9d0e2",
      "username":   "bob",
      "first_name": "Bob",
      "last_name":  "Smith"
    }
  ]
}

// Documento del usuario seguido (target_user)
{
  "_id": "64b1f2c3d4e5f6a7b8c9d0e2",
  "username": "bob",
  // ... otros campos ...
  "followers": [
    {
      "id":         "64b1f2c3d4e5f6a7b8c9d0e1",
      "username":   "alice",
      "first_name": "Alice",
      "last_name":  "Johnson"
    }
  ]
}
```

---

## Respuesta 200

```jsonc
{
  "ok": true,
  "action": "followed",          // "followed" | "unfollowed"
  "target_user": {
    "id":         "64b1f2c3d4e5f6a7b8c9d0e2",
    "username":   "bob",
    "first_name": "Bob",
    "last_name":  "Smith"
  },
  "followers_count": 5,          // conteo actualizado del target
  "following_count": 3           // conteo actualizado del target
}
```

---

## Errores posibles

| Código | Campo | Mensaje | Causa |
|---|---|---|---|
| 400 | `target_user_id` | Id de usuario inválido | ObjectId malformado |
| 400 | `target_user_id` | No puedes seguirte a ti mismo | `target_user_id` == JWT identity |
| 400 | `database` | Error actualizando follow | Falla al escribir en Mongo |
| 404 | `user` | Usuario no encontrado | `target_user_id` no existe en la DB |
| 401 | — | — | JWT ausente o expirado |

```jsonc
// Formato estándar de error
{
  "ok": false,
  "errors": [{ "field": "user", "message": "Usuario no encontrado" }]
}
```

---

## Uso desde Angular

### 1. Agregar el método al servicio de usuarios

```typescript
// service/users/users.ts

toggleFollow(targetUserId: string): Observable<any> {
  return this.http.post(
    `${this.configService.appConfig.apiUrl}/api/users/follow/${targetUserId}`,
    {}
  );
}
```

### 2. Componente — botón de seguir

```typescript
// user-profile.component.ts

isFollowing = false;
followersCount = 0;
followingCount = 0;
isLoading = false;

loadProfile(userId: string): void {
  // Al cargar el perfil, determina si ya sigues al usuario comparando
  // el id del usuario autenticado contra la lista followers del target.
  // Alternativa: guardar el estado en el store o calcularlo desde /get_user.
}

onToggleFollow(targetUserId: string): void {
  if (this.isLoading) return;
  this.isLoading = true;

  this.userService.toggleFollow(targetUserId).subscribe({
    next: (res) => {
      this.isFollowing      = res.action === 'followed';
      this.followersCount   = res.followers_count;
      this.followingCount   = res.following_count;
      this.isLoading        = false;
    },
    error: (err) => {
      console.error('Error en follow:', err);
      this.isLoading = false;
    },
  });
}
```

### 3. Template — botón con estado visual

```html
<!-- user-profile.component.html -->

<div class="follow-section">
  <button
    mat-raised-button
    [color]="isFollowing ? '' : 'primary'"
    [disabled]="isLoading"
    (click)="onToggleFollow(targetUser.id)"
  >
    <mat-icon>{{ isFollowing ? 'person_remove' : 'person_add' }}</mat-icon>
    {{ isFollowing ? 'Dejar de seguir' : 'Seguir' }}
  </button>

  <span class="count-label">
    {{ followersCount }} seguidores · {{ followingCount }} siguiendo
  </span>
</div>
```

---

---

## 2. Mis seguidores y seguidos — `GET /api/users/me/follows`

Devuelve ambas listas completas del usuario autenticado de una sola llamada.

### Respuesta 200

```jsonc
{
  "ok": true,
  "followers_count": 2,
  "following_count": 3,
  "followers": [
    { "id": "...", "username": "alice", "first_name": "Alice", "last_name": "Johnson" },
    { "id": "...", "username": "carol", "first_name": "Carol", "last_name": "White" }
  ],
  "following": [
    { "id": "...", "username": "bob",   "first_name": "Bob",   "last_name": "Smith"  },
    { "id": "...", "username": "dave",  "first_name": "Dave",  "last_name": "Brown"  },
    { "id": "...", "username": "eve",   "first_name": "Eve",   "last_name": "Davis"  }
  ]
}
```

Si el usuario aún no sigue a nadie ni tiene seguidores, ambas listas llegan vacías (`[]`).

### Ejemplo en Angular

```typescript
// users.ts — servicio
getMyFollows(): Observable<any> {
  return this.http.get(`${this.configService.appConfig.apiUrl}/api/users/me/follows`);
}
```

```typescript
// componente de perfil propio
followers: any[] = [];
following: any[] = [];
followersCount = 0;
followingCount = 0;

loadFollows(): void {
  this.userService.getMyFollows().subscribe({
    next: (res) => {
      this.followers      = res.followers;
      this.following      = res.following;
      this.followersCount = res.followers_count;
      this.followingCount = res.following_count;
    },
  });
}
```

```html
<!-- tabs seguidores / seguidos -->
<mat-tab-group>
  <mat-tab [label]="'Seguidores (' + followersCount + ')'">
    <mat-list>
      <mat-list-item *ngFor="let u of followers" [routerLink]="['/users', u.id]">
        <span matListItemTitle>{{ u.username }}</span>
        <span matListItemLine>{{ u.first_name }} {{ u.last_name }}</span>
      </mat-list-item>
    </mat-list>
  </mat-tab>

  <mat-tab [label]="'Siguiendo (' + followingCount + ')'">
    <mat-list>
      <mat-list-item *ngFor="let u of following" [routerLink]="['/users', u.id]">
        <span matListItemTitle>{{ u.username }}</span>
        <span matListItemLine>{{ u.first_name }} {{ u.last_name }}</span>
      </mat-list-item>
    </mat-list>
  </mat-tab>
</mat-tab-group>
```

---

## Flujo típico en el frontend

```
1. El usuario abre el perfil de otro usuario
         ↓
2. GET /api/users/<id>   →   recibe datos del perfil
         ↓
3. Determina si ya lo sigue (consulta followers del target o estado en store)
         ↓
4. Muestra botón "Seguir" o "Dejar de seguir"
         ↓
5. Usuario hace clic  →  POST /api/users/follow/<id>
         ↓
6. Actualiza isFollowing, followersCount y followingCount desde la respuesta
         ↓
7. El botón cambia de estado sin recargar la página
```

---

## Cómo saber si el usuario autenticado ya sigue a alguien

`GET /api/users/<user_id>` devuelve el campo `is_following` directamente:

```jsonc
{
  "ok": true,
  "user": {
    "id": "64b1f2c3d4e5f6a7b8c9d0e2",
    "username": "bob",
    "first_name": "Bob",
    "last_name": "Smith",
    "age": 28,
    "avatar_base64": null,
    "avatar_mime": null,
    "is_following": true   // ← true si ya lo sigues, false si no
  }
}
```

`is_following` es siempre `false` cuando se consulta el propio perfil.

### Uso en el componente

```typescript
loadProfile(userId: string): void {
  this.userService.getUserById(userId).subscribe({
    next: (res) => {
      this.profile      = res.user;
      this.isFollowing  = res.user.is_following;
      this.followersCount = res.user.followers_count ?? 0;  // si lo expones en el futuro
    },
  });
}
```

Tras ejecutar el toggle, actualiza el estado local con la respuesta:

```typescript
onToggleFollow(): void {
  this.userService.toggleFollow(this.profile.id).subscribe({
    next: (res) => {
      this.isFollowing    = res.action === 'followed';
      this.followersCount = res.followers_count;
    },
  });
}
```

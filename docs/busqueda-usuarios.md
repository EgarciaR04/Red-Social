# Búsqueda y perfil público de usuarios

Estas APIs permiten buscar usuarios por username y consultar el perfil público
de cualquier usuario a partir de su id. Ambas requieren JWT.

Los datos expuestos son **solo públicos**: id, username, nombre, edad y avatar.
No se expone email ni estado de cuenta.

---

## Endpoints

| Acción | Método | Ruta |
|---|---|---|
| Buscar usuarios | GET | `/api/users/search?q=<término>` |
| Perfil público por id | GET | `/api/users/<user_id>` |

Requieren `Authorization: Bearer <token>`.

---

## 1. Búsqueda de usuarios — `GET /api/users/search`

### Query params

| Param | Requerido | Descripción |
|---|---|---|
| `q` | Sí | Substring del username. Mínimo 1 carácter, máximo 50 |
| `limit` | No | Máx resultados (default: 20, máximo: 50) |

### Respuesta 200

```jsonc
{
  "ok": true,
  "total": 2,
  "users": [
    {
      "id":            "64b1f2c3d4e5f6a7b8c9d0e1",
      "username":      "john_doe",
      "first_name":    "John",
      "last_name":     "Doe",
      "age":           25,
      "avatar_base64": "iVBORw0KGgoAAAANSUhEUg...",  // null si no tiene avatar
      "avatar_mime":   "image/png"                   // null si no tiene avatar
    },
    {
      "id":            "64b1f2c3d4e5f6a7b8c9d0e2",
      "username":      "johnny_walker",
      "first_name":    "Johnny",
      "last_name":     "Walker",
      "age":           30,
      "avatar_base64": null,
      "avatar_mime":   null
    }
  ]
}
```

### Ejemplo en Angular

```typescript
// user.service.ts
searchUsers(query: string, limit = 20): Observable<any> {
  const params = { q: query, limit: String(limit) };
  return this.http.get(`${this.apiUrl}/api/users/search`, { params });
}
```

```typescript
// componente
results: any[] = [];

onSearch(term: string): void {
  if (term.trim().length === 0) { this.results = []; return; }
  this.userService.searchUsers(term).subscribe({
    next: res => this.results = res.users,
  });
}
```

```html
<!-- template — lista de resultados con avatar -->
<mat-list>
  <mat-list-item *ngFor="let user of results" (click)="selectUser(user)">
    <img *ngIf="user.avatar_base64"
         [src]="'data:' + user.avatar_mime + ';base64,' + user.avatar_base64"
         class="avatar-sm" />
    <mat-icon *ngIf="!user.avatar_base64">account_circle</mat-icon>
    <span>{{ user.username }}</span>
    <span class="secondary">{{ user.first_name }} {{ user.last_name }}</span>
  </mat-list-item>
</mat-list>
```

---

## 2. Perfil público por id — `GET /api/users/<user_id>`

Llamar después de que el cliente seleccione un usuario de los resultados
de búsqueda. Devuelve el mismo conjunto de campos públicos.

### Respuesta 200

```jsonc
{
  "ok": true,
  "user": {
    "id":            "64b1f2c3d4e5f6a7b8c9d0e1",
    "username":      "john_doe",
    "first_name":    "John",
    "last_name":     "Doe",
    "age":           25,
    "avatar_base64": "iVBORw0KGgoAAAANSUhEUg...",
    "avatar_mime":   "image/png"
  }
}
```

### Ejemplo en Angular

```typescript
// user.service.ts
getUserById(userId: string): Observable<any> {
  return this.http.get(`${this.apiUrl}/api/users/${userId}`);
}
```

```typescript
// flujo completo: buscar → seleccionar → obtener perfil
selectUser(user: any): void {
  this.userService.getUserById(user.id).subscribe({
    next: res => {
      this.selectedUser = res.user;
      // aquí puedes navegar al perfil o abrir un panel lateral
    },
  });
}
```

```html
<!-- perfil del usuario seleccionado -->
<div *ngIf="selectedUser" class="user-profile">
  <img *ngIf="selectedUser.avatar_base64"
       [src]="'data:' + selectedUser.avatar_mime + ';base64,' + selectedUser.avatar_base64"
       alt="Avatar" />
  <mat-icon *ngIf="!selectedUser.avatar_base64">account_circle</mat-icon>
  <h2>{{ selectedUser.username }}</h2>
  <p>{{ selectedUser.first_name }} {{ selectedUser.last_name }}</p>
  <p>Edad: {{ selectedUser.age }}</p>
</div>
```

---

## Errores posibles

| Código | Campo | Mensaje | Causa |
|---|---|---|---|
| 400 | `q` | El término de búsqueda no puede estar vacío | `q` vacío o ausente |
| 400 | `q` | El término de búsqueda es demasiado largo | `q` > 50 caracteres |
| 400 | `user_id` | Id de usuario inválido | ObjectId malformado |
| 404 | `user` | Usuario no encontrado | Id válido pero sin documento |

```jsonc
// Formato estándar de error
{
  "ok": false,
  "errors": [{ "field": "user", "message": "Usuario no encontrado" }]
}
```

---

## Flujo típico en el frontend

```
1. Usuario escribe en un <input> de búsqueda
          ↓
2. GET /api/users/search?q=john   →   lista de resultados
          ↓
3. Usuario hace clic en un resultado
          ↓
4. GET /api/users/<id>            →   perfil completo del usuario
          ↓
5. Mostrar perfil / habilitar interacción futura
   (mensajes, seguir, etc.)
```

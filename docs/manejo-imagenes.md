# Manejo de imágenes — posts, comentarios y foto de perfil

Las imágenes se almacenan como **Base64 embebido** en el documento del post o comentario.
Son opcionales en todos los endpoints de creación.

---

## Endpoints

| Elemento | Método | Ruta |
|---|---|---|
| Publicación | POST | `/api/posts/` |
| Comentario / subcomentario | POST | `/api/posts/<post_id>/comments` |

Requieren `Authorization: Bearer <token>` en el header.

---

## Restricciones del servidor

| Restricción | Valor |
|---|---|
| Tamaño máximo | **5 MB** (imagen decodificada) |
| Tipos aceptados | `image/png`, `image/jpeg`, `image/jpg`, `image/gif`, `image/webp` |
| Límite global del body | 12 MB |

---

## Formato A — multipart/form-data (recomendado con `<input type="file">`)

El archivo va bajo el campo `image`. Los demás campos van como strings normales en el `FormData`.
**No establecer `Content-Type` manualmente** — Angular lo arma con el boundary correcto.

### Servicio Angular

```typescript
// post.service.ts
createPost(content: string, file?: File): Observable<any> {
  const fd = new FormData();
  fd.append('content', content);
  if (file) {
    fd.append('image', file);          // campo requerido por el backend
  }
  return this.http.post(`${this.apiUrl}/api/posts/`, fd);
}

createComment(postId: string, content: string, file?: File, parentId?: string): Observable<any> {
  const fd = new FormData();
  fd.append('content', content);
  if (parentId) fd.append('parent_comment_id', parentId);
  if (file)     fd.append('image', file);
  return this.http.post(`${this.apiUrl}/api/posts/${postId}/comments`, fd);
}
```

### Componente Angular

```typescript
// en el componente
selectedFile: File | null = null;

onFileSelected(event: Event): void {
  const input = event.target as HTMLInputElement;
  this.selectedFile = input.files?.[0] ?? null;
}

submitPost(): void {
  this.postService.createPost(this.content, this.selectedFile ?? undefined)
    .subscribe({ next: res => console.log(res) });
}
```

```html
<!-- template -->
<input type="file" accept="image/png,image/jpeg,image/gif,image/webp" (change)="onFileSelected($event)" />
<button (click)="submitPost()">Publicar</button>
```

---

## Formato B — application/json con Base64

Usar cuando ya se tiene el archivo como Base64 o no se usa `<input type="file">` directamente.

### Helper para convertir File a Base64

```typescript
// image.utils.ts
export function fileToBase64(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload  = () => {
      // El resultado tiene prefijo "data:image/png;base64,XXXX" — lo eliminamos
      const result = reader.result as string;
      resolve(result.split(',')[1]);
    };
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}
```

### Servicio Angular

```typescript
import { fileToBase64 } from './image.utils';

async createPostWithJson(content: string, file?: File): Promise<void> {
  const body: any = { content };
  if (file) {
    body.image_base64 = await fileToBase64(file);
    body.image_mime   = file.type;               // p.ej. "image/png"
  }
  this.http.post(`${this.apiUrl}/api/posts/`, body).subscribe();
}
```

---

## Respuesta de la API

Todos los objetos `post` y `comment` incluyen tanto la imagen del contenido como el avatar del autor:

```jsonc
{
  "ok": true,
  "post": {
    "id":                   "...",
    "author_id":            "...",
    "author_username":      "john_doe",
    "author_avatar_base64": "iVBORw0KGgoAAAANSUhEUg...",  // null si el autor no tiene avatar
    "author_avatar_mime":   "image/png",                  // null si el autor no tiene avatar
    "content":              "Mi publicación",
    "image_base64":         "iVBORw0KGgoAAAANSUhEUg...",  // null si el post no tiene imagen
    "image_mime":           "image/jpeg",                 // null si el post no tiene imagen
    ...
  }
}
```

Lo mismo aplica a cada objeto `comment` y `reply`:

```jsonc
{
  "id":                   "...",
  "author_id":            "...",
  "author_username":      "jane_doe",
  "author_avatar_base64": "...",   // null si el autor no tiene avatar
  "author_avatar_mime":   "image/png",
  "content":              "Buen post!",
  "image_base64":         null,    // null si el comentario no tiene imagen
  "image_mime":           null,
  ...
}
```

---

## Mostrar la imagen en el template

```html
<img
  *ngIf="post.image_base64"
  [src]="'data:' + post.image_mime + ';base64,' + post.image_base64"
  alt="Imagen del post"
/>
```

El mismo patrón aplica para `comment.image_base64` / `comment.image_mime`.

Para mostrar el avatar del autor en un post o comentario:

```html
<!-- Avatar del autor -->
<img
  *ngIf="post.author_avatar_base64"
  [src]="'data:' + post.author_avatar_mime + ';base64,' + post.author_avatar_base64"
  alt="Avatar"
/>
<mat-icon *ngIf="!post.author_avatar_base64">account_circle</mat-icon>
<span>{{ post.author_username }}</span>
```

---

## Manejo de errores

Cuando la imagen no pasa la validación, el servidor devuelve 400 con esta estructura:

```json
{
  "ok": false,
  "errors": [
    { "field": "image", "message": "La imagen supera el tamaño máximo permitido (5 MB)" }
  ]
}
```

Mensajes posibles en `message`:

| Mensaje | Causa |
|---|---|
| `Tipo de imagen no permitido. Permitidos: [...]` | MIME no aceptado |
| `La imagen supera el tamaño máximo permitido (5 MB)` | Archivo mayor a 5 MB |
| `Base64 inválido: ...` | String Base64 corrupto |
| `La imagen está vacía tras decodificar` | Archivo vacío |
| `Falta el tipo de imagen (image_mime)` | Se envió `image_base64` sin `image_mime` (solo formato JSON) |

---

## Foto de perfil (avatar)

El avatar se actualiza a través del mismo endpoint de actualización de usuario.
Es opcional: se puede actualizar junto con otros campos o de forma independiente.

### Endpoint

| Método | Ruta |
|---|---|
| PUT / PATCH | `/api/users/update_me` |

Requiere `Authorization: Bearer <token>`.

### Formato A — multipart/form-data (recomendado)

El archivo va bajo el campo `avatar`.

```typescript
// user.service.ts
updateAvatar(file: File): Observable<any> {
  const fd = new FormData();
  fd.append('avatar', file);   // campo requerido por el backend
  return this.http.patch(`${this.apiUrl}/api/users/update_me`, fd);
}

// Actualizar avatar junto con otros campos
updateProfile(data: { first_name?: string }, file?: File): Observable<any> {
  const fd = new FormData();
  Object.entries(data).forEach(([k, v]) => v !== undefined && fd.append(k, String(v)));
  if (file) fd.append('avatar', file);
  return this.http.patch(`${this.apiUrl}/api/users/update_me`, fd);
}
```

```html
<input type="file" accept="image/png,image/jpeg,image/gif,image/webp" (change)="onAvatarSelected($event)" />
```

```typescript
onAvatarSelected(event: Event): void {
  const file = (event.target as HTMLInputElement).files?.[0];
  if (file) this.userService.updateAvatar(file).subscribe();
}
```

### Formato B — JSON con Base64

```typescript
import { fileToBase64 } from './image.utils';   // helper descrito más arriba

async updateAvatarJson(file: File): Promise<void> {
  this.http.patch(`${this.apiUrl}/api/users/update_me`, {
    avatar_base64: await fileToBase64(file),
    avatar_mime:   file.type,
  }).subscribe();
}
```

### Respuesta

```jsonc
{
  "ok": true,
  "user": {
    "username":      "john_doe",
    "email":         "john@example.com",
    "avatar_base64": "iVBORw0KGgoAAAANSUhEUg...",   // null si no tiene avatar
    "avatar_mime":   "image/png",                   // null si no tiene avatar
    ...
  }
}
```

### Mostrar el avatar en el template

```html
<!-- Con avatar -->
<img
  *ngIf="user.avatar_base64"
  [src]="'data:' + user.avatar_mime + ';base64,' + user.avatar_base64"
  alt="Foto de perfil"
/>

<!-- Placeholder si no tiene avatar -->
<mat-icon *ngIf="!user.avatar_base64">account_circle</mat-icon>
```

### Errores específicos del avatar

| Mensaje | Causa |
|---|---|
| `Tipo de imagen no permitido. Permitidos: [...]` | MIME no aceptado |
| `La imagen supera el tamaño máximo permitido (5 MB)` | Archivo mayor a 5 MB |
| `Base64 inválido: ...` | String Base64 corrupto |

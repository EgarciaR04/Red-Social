# Publicaciones — creación, consulta y modificación

## Endpoints

| Operación | Método | Ruta | Auth |
|---|---|---|---|
| Crear publicación | POST | `/api/posts/` | JWT |
| Listar feed (todos) | GET | `/api/posts/` | JWT |
| Mis publicaciones | GET | `/api/posts/me` | JWT |
| Publicaciones de un usuario | GET | `/api/posts/user/<user_id>` | JWT |
| Obtener publicación con comentarios | GET | `/api/posts/<post_id>` | JWT |
| Actualizar publicación | PATCH | `/api/posts/<post_id>` | JWT (solo autor) |
| Eliminar publicación | DELETE | `/api/posts/<post_id>` | JWT (solo autor) |

Todos los endpoints requieren `Authorization: Bearer <token>`.

---

## Imágenes en publicaciones

Las imágenes se almacenan como **Base64 embebido** directamente en el documento del post. Cada post puede tener **de 0 a 5 imágenes**.

| Restricción | Valor |
|---|---|
| Máximo de imágenes por post | **5** |
| Tamaño máximo por imagen | **5 MB** |
| Tipos aceptados | `image/png`, `image/jpeg`, `image/jpg`, `image/gif`, `image/webp` |

Las imágenes se devuelven como un array `images` dentro del objeto post:

```jsonc
"images": [
  { "base64": "iVBORw0KGgoAAAANSUhEUg...", "mime": "image/png" },
  { "base64": "R0lGODlhAQABAIAAAAAAAP...", "mime": "image/gif" }
]
```

---

## Crear publicación — `POST /api/posts/`

### Formato A — multipart/form-data (recomendado con imágenes)

El campo `images` puede repetirse para enviar varias imágenes.

```typescript
// post.service.ts
createPost(content: string, files: File[] = []): Observable<any> {
  const fd = new FormData();
  fd.append('content', content);
  files.forEach(f => fd.append('images', f));   // mismo campo, varias veces
  return this.http.post(`${this.apiUrl}/api/posts/`, fd);
}
```

### Formato B — JSON con Base64

```typescript
import { fileToBase64 } from './image.utils';

async createPost(content: string, files: File[] = []): Promise<void> {
  const images = await Promise.all(
    files.map(async f => ({ base64: await fileToBase64(f), mime: f.type }))
  );
  this.http.post(`${this.apiUrl}/api/posts/`, { content, images }).subscribe();
}
```

### Campos del body

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `content` | string (1–5000) | Sí | Texto de la publicación |
| `media_urls` | string[] (máx 10) | No | URLs externas de media |
| `images` | PostImage[] (máx 5) | No | Imágenes embebidas `{base64, mime}` |

### Respuesta `201`

```jsonc
{
  "ok": true,
  "post": {
    "id":                   "682b...",
    "author_id":            "681a...",
    "author_username":      "john_doe",
    "author_avatar_base64": "iVBORw0KGgo...",
    "author_avatar_mime":   "image/png",
    "content":              "Mi publicación con imágenes",
    "media_urls":           [],
    "images": [
      { "base64": "iVBORw0KGgo...", "mime": "image/png" },
      { "base64": "R0lGODlhAQAB...", "mime": "image/gif" }
    ],
    "created_at":      "2026-05-06T10:30:00-06:00",
    "updated_at":      null,
    "reactions_count": { "like": 0, "love": 0, "haha": 0, "wow": 0, "sad": 0, "angry": 0 },
    "comments_count":  0
  }
}
```

---

## Consultar publicaciones

### Feed general — `GET /api/posts/`

```typescript
getPosts(page = 1, pageSize = 20): Observable<any> {
  return this.http.get(`${this.apiUrl}/api/posts/`, {
    params: { page, page_size: pageSize }
  });
}
```

### Mis publicaciones — `GET /api/posts/me`

```typescript
getMyPosts(page = 1, pageSize = 20): Observable<any> {
  return this.http.get(`${this.apiUrl}/api/posts/me`, {
    params: { page, page_size: pageSize }
  });
}
```

### Publicaciones de un usuario — `GET /api/posts/user/<user_id>`

```typescript
getUserPosts(userId: string, page = 1, pageSize = 20): Observable<any> {
  return this.http.get(`${this.apiUrl}/api/posts/user/${userId}`, {
    params: { page, page_size: pageSize }
  });
}
```

### Parámetros de paginación

| Parámetro | Default | Máximo |
|---|---|---|
| `page` | 1 | — |
| `page_size` | 20 | 100 |

### Respuesta de listas `200`

```jsonc
{
  "ok": true,
  "posts": [ /* array de objetos post */ ],
  "pagination": {
    "page": 1, "page_size": 20, "total": 47, "total_pages": 3
  }
}
```

### Publicación con comentarios — `GET /api/posts/<post_id>`

```typescript
getPost(postId: string): Observable<any> {
  return this.http.get(`${this.apiUrl}/api/posts/${postId}`);
}
```

---

## Actualizar publicación — `PATCH /api/posts/<post_id>`

Solo el autor. Todos los campos son opcionales.

### Reglas de imágenes en el update

| Escenario | Qué enviar |
|---|---|
| Reemplazar con nuevas imágenes | `images: [{base64, mime}, ...]` (JSON) o archivos `images` (multipart) |
| Eliminar todas las imágenes | `images: []` (JSON) |
| No tocar las imágenes | No enviar el campo `images` |

> Para multipart, no hay forma de "no tocar imágenes" si se suben archivos — cualquier
> archivo en `images` reemplaza la lista completa. Si no se sube ningún archivo, las
> imágenes existentes no cambian.

### Formato A — multipart/form-data

```typescript
updatePost(
  postId: string,
  params: { content?: string; files?: File[]; clearImages?: boolean }
): Observable<any> {
  const fd = new FormData();
  if (params.content !== undefined)  fd.append('content', params.content);
  if (params.files?.length)          params.files.forEach(f => fd.append('images', f));
  return this.http.patch(`${this.apiUrl}/api/posts/${postId}`, fd);
}
```

### Formato B — JSON

```typescript
// Solo actualizar texto
updateText(postId: string, content: string): Observable<any> {
  return this.http.patch(`${this.apiUrl}/api/posts/${postId}`, { content });
}

// Reemplazar imágenes
async replaceImages(postId: string, files: File[]): Promise<void> {
  const images = await Promise.all(
    files.map(async f => ({ base64: await fileToBase64(f), mime: f.type }))
  );
  this.http.patch(`${this.apiUrl}/api/posts/${postId}`, { images }).subscribe();
}

// Eliminar todas las imágenes
clearImages(postId: string): Observable<any> {
  return this.http.patch(`${this.apiUrl}/api/posts/${postId}`, { images: [] });
}

// Actualizar URLs de media
updateMediaUrls(postId: string, urls: string[]): Observable<any> {
  return this.http.patch(`${this.apiUrl}/api/posts/${postId}`, { media_urls: urls });
}
```

### Campos del body

| Campo | Tipo | Descripción |
|---|---|---|
| `content` | string (1–5000) | Nuevo texto de la publicación |
| `media_urls` | string[] (máx 10) | Reemplaza la lista completa. `[]` la vacía. |
| `images` | PostImage[] (máx 5) | Reemplaza la lista de imágenes. `[]` elimina todas. |

### Respuesta `200`

```jsonc
{
  "ok": true,
  "post": {
    "id":       "682b...",
    "content":  "Texto actualizado",
    "images":   [],
    "updated_at": "2026-05-06T11:00:00-06:00"
    // ...resto de campos
  }
}
```

### Errores posibles

| Status | Campo | Causa |
|---|---|---|
| 400 | `body` | Sin campos para actualizar |
| 400 | `content` | Vacío o fuera de rango |
| 400 | `images[i]` | Imagen inválida (mime, tamaño, base64 corrupto) |
| 400 | `images` | Más de 5 imágenes |
| 401 | — | Token JWT ausente o expirado |
| 403 | `auth` | No es el autor del post |
| 404 | `post` | Post no encontrado |

---

## Eliminar publicación — `DELETE /api/posts/<post_id>`

```typescript
deletePost(postId: string): Observable<any> {
  return this.http.delete(`${this.apiUrl}/api/posts/${postId}`);
}
```

---

## Mostrar imágenes en el template

```html
<!-- Galería de imágenes del post -->
<ng-container *ngFor="let img of post.images">
  <img
    [src]="'data:' + img.mime + ';base64,' + img.base64"
    alt="Imagen del post"
  />
</ng-container>
```

---

## Helper para convertir File a Base64

```typescript
// image.utils.ts
export function fileToBase64(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload  = () => resolve((reader.result as string).split(',')[1]);
    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}
```

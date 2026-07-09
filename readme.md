Juan Francisco Bermudez Padilla, 00218123

## Indicaciones

Recientemente, se utilizó AI para crear un sistema de gestion de una biblioteca, el cual ha generado varios errores, su trabajo es arreglarlo. Dado el siguiente caso de uso, explique y/o resuelva cada problema según se le pida.

---

## Consideraciones

La libreria crea automaticamente un correo con los nombres de la persona

---

## Problemas

### 1. Filtro por autor y género (10%)

QA ha reportado que el endpoint para obtener los libros puede filtrar por **autor** y por **género**, o por cualquiera de los dos de manera individual.

Actualmente:

- Filtrar únicamente por autor funciona correctamente.
- Filtrar únicamente por género funciona correctamente.
- Filtrar por **autor y género al mismo tiempo** provoca que el servidor falle.

**Instrucción:** Explique la causa del problema y resuélvalo.

---

### 2. Error al volver a prestar un libro (10%)

Un usuario reportó que al pedir prestado el libro **The Selfish Gene**, devolverlo e intentar pedirlo prestado nuevamente, el servidor falla.

**Instrucción:** Explique la causa del problema y resuélvalo.

---

### 3. Cantidad de libros por género (10%)

Existe un endpoint que devuelve la cantidad de libros disponibles por género. Sin embargo, actualmente dicho endpoint falla.

**Instrucción:** Explique la causa del problema y resuélvalo.

---

### 4. Error al consultar un libro por ID (10%)

Un miembro del equipo de frontend reporta que la siguiente llamada falla:

```http
GET /books?id=ed16ed1e-7017-4697-a08a-d28c09a74acf
```

**Instrucción:** Explique la causa del problema.

---

### 5. Error al crear un libro (10%)

QA ha reportado que el siguiente payload enviado al endpoint `POST /books` provoca un error:

```json
{
  "title": "Clean Code",
  "author": "Robert C. Martin",
  "genre": "classic",
  "isbn": "978-0132350884",
  "available": true,
  "availableCount": 5
}
```

**Instrucción:** Explique la causa del problema.

---

### 6. Devolución de libros no prestados (20%)

QA ha reportado que un usuario es capaz de devolver libros que nunca ha solicitado en préstamo.

**Instrucción:**

- Confirme si este comportamiento es realmente posible.
- Si es posible, explique la causa y resuelva el problema.
- Si no es posible, explique por qué, haciendo referencia al código correspondiente.

---]

**Respuestas**
---

Problema 1

El error ocurre por dos bugs combinados en BookRepository y BookService:

El método findByAuthorAndGenre(String author, String genre) declara el parámetro genre como String, pero el campo genre en la entidad Book es un enum Genre (@Enumerated(EnumType.STRING)). Al construir la consulta derivada (b.genre = ?2), Hibernate no puede bindear un String contra una propiedad de tipo enum, y lanza una excepción de tipo.
Además, en BookService.getAllBooks, la llamada invierte los argumentos: bookRepository.findByAuthorAndGenre(genre, author) cuando la firma espera (author, genre).

---

Problema 2

En MovementService.createMovement, la rama de BORROWING actualiza correctamente availableCount y, si llega a 0, marca available = false. Sin embargo, la rama de RETURN solo incrementa availableCount, sin volver a poner available = true.
Con "The Selfish Gene" (availableCount = 1 inicial): al prestarlo baja a 0 y available pasa a false. Al devolverlo, availableCount sube a 1, pero available queda en false. Al intentar prestarlo de nuevo, if (!book.isAvailable()) es true y lanza RuntimeException("Book is not available"), aunque ya hay ejemplares disponibles.

---

Problema 3

En data.sql, un libro X se inserta con genre = NULL. En BookService.getGenresAvailable(), el ciclo ejecuta book.getGenre().name() para cada libro sin verificar si es null. Al llegar a ese libro, se lanza un NullPointerException, que el GlobalExceptionHandler responde como error 500.

---

Problema 4

BookController expone @GetMapping("/{id}"), es decir, el id se espera como path variable (/books/{id}), no como query param. La llamada GET /books?id=ed16ed1e-... no hace match con esa ruta; en su lugar coincide con el @GetMapping sin path de getAllBooks(author, genre), cuyo método solo declara @RequestParam para author y genre — el parámetro id de la URL se ignora silenciosamente.
Como resultado, el endpoint responde 200 OK con la lista completa de libros (findAll()) en lugar del libro puntual. El frontend espera deserializar un único objeto Book pero recibe un arreglo List<Book>, lo que provoca el fallo reportado del lado del cliente. La llamada correcta debería ser GET /books/ed16ed1e-7017-4697-a08a-d28c09a74acf.

---

Problema 5

El payload envía "genre": "classic" en minúsculas. En BookService.createBook:
javabook.setGenre(Genre.valueOf(dto.getGenre()));
Enum.valueOf es case-sensitive, y las constantes del enum Genre están en mayúsculas (CLASSIC, CRIME, etc.). Como "classic" no coincide exactamente con "CLASSIC", se lanza IllegalArgumentException: No enum constant ... Genre.classic, devuelto como error 500.
Nótese la inconsistencia con updateBook, que sí normaliza con dto.getGenre().toUpperCase() antes de convertir — createBook carece de esa normalización.

---

Problema 6

el comportamiento es posible. En MovementService.createMovement, la rama de RETURN solo valida que existan el Lector (por email) y el Book (por ISBN); en ningún momento verifica que exista un movimiento previo de BORROWING sin devolver para ese lector y ese libro. MovementRepository ni siquiera define una consulta para buscar movimientos por lector/libro, por lo que el servicio no tiene forma de comprobar el historial. Con esto, cualquier lector registrado puede "devolver" cualquier libro existente, incrementando availableCount artificialmente sin haberlo prestado nunca.
Solución: agregar una consulta que obtenga el último movimiento de ese lector para ese libro y validar que sea un BORROWING pendiente antes de aceptar el RETURN.

Optional<Movement> findTopByLector_IdAndBook_IdOrderByTimestampDesc(UUID lectorId, UUID bookId);

Movement lastMovement = movementRepository
        .findTopByLector_IdAndBook_IdOrderByTimestampDesc(lector.getId(), book.getId())
        .orElse(null);

if (lastMovement == null || lastMovement.getType() != MovementType.BORROWING) {
    throw new RuntimeException("This lector has not borrowed this book");
}

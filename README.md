# SimpsonsApp — Análisis de Errores (2do Parcial Parte Práctica)


A continuación se detallan los 18 errores encontrados en el código, con su ubicación, tipo y solución propuesta.

---

## Error 1 — Sintaxis: `init` fuera de la clase con `return` ilegal

**Archivo:** `app/src/main/java/com/example/simpsonsapp/domain/model/Episode.kt`
**Líneas:** 13–15
**Tipo:** Sintaxis

**Código con error:**
```kotlin
init {
    return Episode;
}
```

**Descripción del problema:** El bloque `init` está ubicado fuera del cuerpo del `data class Episode`, después de su llave de cierre. En Kotlin los bloques `init` deben estar dentro de la clase. Adicionalmente, `return` dentro de un `init` es ilegal — estos bloques no retornan valores. El `;` al final es sintaxis Java, no Kotlin.

**Solución:** Eliminar las líneas 13–15 completamente.

---

## Error 2 — Naming: método en snake_case en la interfaz

**Archivo:** `app/src/main/java/com/example/simpsonsapp/domain/repository/EpisodeRepository.kt`
**Línea:** 8
**Tipo:** Naming / Arquitectura

**Código con error:**
```kotlin
fun get_episodes(): Flow<PagingData<Episode>>
```

**Descripción del problema:** El método usa snake_case, violando las convenciones de nomenclatura de Kotlin que exigen camelCase para funciones. Los otros tres métodos de la misma interfaz (`getEpisodesBySeason`, `getAvailableSeasons`, `getEpisodeById`) usan camelCase correctamente. Este nombre además no coincide con el que declara la implementación, rompiendo el contrato de la interfaz.

**Solución:** Renombrar a `getEpisodes()` y actualizar todos los lugares que lo referencien.

---

## Error 3 — Compilación: `override` sin método correspondiente en la interfaz

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/repository/EpisodeRepositoryImpl.kt`
**Línea:** 23
**Tipo:** Compilación

**Código con error:**
```kotlin
override fun getEpisodes(): Flow<PagingData<Episode>>
```

**Descripción del problema:** La interfaz `EpisodeRepository` declara `get_episodes()` y la implementación declara `override fun getEpisodes()`. Son identificadores distintos. El compilador no puede resolver el `override` porque no existe ningún método llamado `getEpisodes()` en la interfaz. El proyecto no compila.

**Solución:** Corregir el nombre en la interfaz a `getEpisodes()` (Error 2). Una vez alineados, el `override` resuelve correctamente.

---

## Error 4 — Compilación: import faltante para `SimpsonsApi`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/repository/EpisodeRepositoryImpl.kt`
**Línea:** 18
**Tipo:** Compilación

**Código con error:**
```kotlin
private val simpsonsApi: SimpsonsApi,
```

**Descripción del problema:** `SimpsonsApi` está definida en el package `com.example.simpsonsapp.data.remote`. `EpisodeRepositoryImpl` está en `com.example.simpsonsapp.data.repository`. En Kotlin los tipos de otros packages deben importarse explícitamente. El archivo no contiene `import com.example.simpsonsapp.data.remote.SimpsonsApi`, por lo que el compilador no puede resolver el tipo y falla.

**Solución:** Agregar al bloque de imports:
```kotlin
import com.example.simpsonsapp.data.remote.SimpsonsApi
```

---

## Error 5 — Runtime crash: falta `.baseUrl()` en Retrofit

**Archivo:** `app/src/main/java/com/example/simpsonsapp/di/DataModule.kt`
**Líneas:** 34–37
**Tipo:** Runtime crash

**Código con error:**
```kotlin
return Retrofit.Builder()
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

**Descripción del problema:** Retrofit requiere obligatoriamente una base URL al construirse. Sin ella lanza `IllegalStateException: Base URL required` en tiempo de ejecución al iniciar la app, antes de realizar cualquier llamada a la API.

**Solución:** Agregar `.baseUrl()` antes de `.addConverterFactory()`:
```kotlin
return Retrofit.Builder()
    .baseUrl("https://thesimpsonsapi.com/")
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

---

## Error 6 — Arquitectura: URL absoluta hardcodeada en anotación `@GET`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/remote/EpisodeRemoteMediator.kt`
**Línea:** 106
**Tipo:** Arquitectura

**Código con error:**
```kotlin
@GET("https://thesimpsonsapi.com/api/episodes")
```

**Descripción del problema:** En Retrofit la anotación `@GET` debe recibir únicamente el path relativo del endpoint. La URL base pertenece al builder de Retrofit en el módulo de inyección de dependencias. Tener la URL absoluta aquí hace imposible cambiar de entorno sin tocar múltiples archivos, y entra en conflicto directo con la ausencia de base URL en `DataModule` (Error 5).

**Solución:** Usar solo el path relativo:
```kotlin
@GET("api/episodes")
```
Y definir la base URL en `DataModule` (complementa Error 5).

---

## Error 7 — Arquitectura (SRP): `SimpsonsApi` definida dentro de `EpisodeRemoteMediator.kt`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/remote/EpisodeRemoteMediator.kt`
**Líneas:** 105–110
**Tipo:** Arquitectura

**Código con error:**
```kotlin
interface SimpsonsApi {
    @GET("https://thesimpsonsapi.com/api/episodes")
    suspend fun getEpisodes(@Query("page") page: Int): EpisodesResponse
}
```

**Descripción del problema:** La interfaz `SimpsonsApi` (contrato con la API REST) está definida dentro del mismo archivo que `EpisodeRemoteMediator` (lógica de paginación remota). Son dos responsabilidades completamente distintas conviviendo en un solo archivo, violando el Principio de Responsabilidad Única. Esto además dificulta encontrar la definición de la API al leer el proyecto.

**Solución:** Mover `SimpsonsApi` a su propio archivo: `data/remote/SimpsonsApi.kt`.

---

## Error 8 — Arquitectura: entidades Room declaradas como `class` en lugar de `data class`

**Archivos:**
- `app/src/main/java/com/example/simpsonsapp/data/local/entity/EpisodeEntity.kt` — línea 7
- `app/src/main/java/com/example/simpsonsapp/data/local/entity/RemoteKeyEntity.kt` — línea 7

**Tipo:** Arquitectura

**Código con error:**
```kotlin
class EpisodeEntity(...)
class RemoteKeyEntity(...)
```

**Descripción del problema:** Las entidades Room son modelos de datos. Declararlas como `class` en lugar de `data class` significa que no tienen implementaciones automáticas de `equals()`, `hashCode()`, `copy()` ni `toString()`. Esto hace que las comparaciones entre instancias fallen silenciosamente, los tests sean poco confiables, y el comportamiento general sea impredecible en estructuras como sets o maps.

**Solución:** Declarar ambas como `data class`:
```kotlin
data class EpisodeEntity(...)
data class RemoteKeyEntity(...)
```

---

## Error 9 — Arquitectura: URL base de la API hardcodeada en la capa UI

**Archivo:** `app/src/main/java/com/example/simpsonsapp/components/Components.kt`
**Línea:** 43
**Tipo:** Arquitectura / DRY

**Código con error:**
```kotlin
model = "https://thesimpsonsapi.com${episode.imagePath}",
```

**Descripción del problema:** La URL base `"https://thesimpsonsapi.com"` está hardcodeada directamente en un Composable de la capa de UI. Esta misma URL ya aparece en `EpisodeRemoteMediator.kt` (línea 106) y debería vivir en un único lugar (constante o configuración). Tener la URL de la API dispersa en la capa de presentación viola la separación de capas de la arquitectura MVVM y hace imposible cambiar el entorno sin buscar y editar múltiples archivos.

**Solución:** El campo `imagePath` del modelo de dominio debería llegar con la URL completa desde el repositorio, o definir la base URL como constante en `data/remote/` y referenciarla desde ambos lugares.

---

## Error 10 — Lógica Compose: side effect en el cuerpo de un Composable

**Archivo:** `app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt`
**Líneas:** 51–53
**Tipo:** Lógica / Compose

**Código con error:**
```kotlin
if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
    viewModel.refreshSeasons()
}
```

**Descripción del problema:** Llamar a `viewModel.refreshSeasons()` directamente en el cuerpo de un `@Composable` hace que se ejecute en cada recomposición. Esto viola las reglas fundamentales de Compose: los side effects deben estar dentro de effect handlers como `LaunchedEffect`. Puede provocar bucles infinitos de recomposición y comportamiento impredecible.

**Solución:** Envolver en `LaunchedEffect` con la key correspondiente:
```kotlin
LaunchedEffect(episodes.loadState.refresh) {
    if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
        viewModel.refreshSeasons()
    }
}
```

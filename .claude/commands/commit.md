---
description: Crea un commit siguiendo el formato de Conventional Commits
model: haiku
---

Tu tarea es crear un commit de git siguiendo estrictamente el formato de **Conventional Commits**.

## Formato del mensaje

```
<tipo>(<alcance opcional>): <descripción corta en español, en minúsculas, sin punto final>

<cuerpo opcional explicando el "por qué", no el "qué">
```

Tipos permitidos:

- `feat`: nueva funcionalidad
- `fix`: corrección de un bug
- `docs`: cambios solo en documentación
- `style`: cambios de formato que no afectan el comportamiento (espacios, comas, etc.)
- `refactor`: cambio de código que no corrige un bug ni añade una funcionalidad
- `test`: se añaden o corrigen tests
- `chore`: tareas de mantenimiento (dependencias, configuración, build, etc.)
- `perf`: cambios que mejoran el rendimiento

## Pasos a seguir

1. Ejecuta en paralelo:
   - `git status` para ver los archivos sin trackear.
   - `git diff` para ver los cambios staged y unstaged.
   - `git log --oneline -10` para revisar el estilo de commits recientes del repositorio.
2. Analiza todos los cambios (staged y no staged) y determina el tipo de Conventional Commit más adecuado y, si aplica, el alcance (por ejemplo el paquete o módulo afectado: `api`, `web-admin`, `web-empleados`, `web-clientes`, `web-shared`).
3. Redacta un mensaje de commit conciso en español que explique el motivo del cambio, no una lista literal de archivos modificados.
4. No incluyas archivos que puedan contener secretos (`.env`, credenciales, claves, etc.). Si detectas alguno, avisa al usuario en vez de incluirlo.
5. Muestra al usuario el mensaje de commit propuesto y pregúntale si le parece correcto antes de continuar. Si el usuario propone cambios, ajusta el mensaje y vuelve a confirmarlo hasta que lo apruebe. No sigas con los pasos siguientes sin esa confirmación explícita.
6. Añade al stage solo los archivos relevantes al cambio (evita `git add -A` o `git add .` si hay archivos ajenos a la tarea).
7. Crea el commit usando un HEREDOC para asegurar el formato correcto del mensaje.
8. Ejecuta `git status` después del commit para confirmar que se creó correctamente.

## Reglas importantes

- Nunca uses `git commit --amend` salvo que el usuario lo pida explícitamente; crea siempre un commit nuevo.
- Nunca uses `--no-verify` ni omitas hooks de pre-commit.
- Nunca hagas `git push` a menos que el usuario lo solicite explícitamente.
- Si el hook de pre-commit falla, corrige el problema, vuelve a hacer stage y crea un commit nuevo (no reintentes con `--amend`).
- Si no hay cambios para commitear, informa al usuario en vez de crear un commit vacío.

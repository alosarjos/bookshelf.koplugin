# Cómo publicar una release

No hay automatización en CI para esto (`.github/workflows/ci.yml` solo hace
lint/test, no publica) -- el proceso es manual, reconstruido aquí a partir
del historial de commits `chore(release): ...` y de las releases ya
publicadas en GitHub. Sigue estos pasos en orden.

## 1. Bump de versión

Edita `_meta.lua`, campo `version`, a la nueva versión (semver: `MAJOR.MINOR.PATCH`).
Debe coincidir exactamente con el tag del paso 4 -- si no coincide, el
comprobador de actualizaciones del plugin muestra un "Update available"
perpetuo tras instalar.

## 2. Traducciones

1. Regenera `locale/bookshelf.pot` a partir de las cadenas `_()`/`T()` del código.
2. `msgmerge` sobre los 13 `.po` completos (`bg_BG`, `de`, `en_GB`, `es`, `fr`,
   `hu`, `it`, `ja`, `pt_BR`, `pt_PT`, `uk`, `vi`, `zh_CN`, `zh_TW`).
3. Traduce a mano todo lo que quede *fuzzy* o sin traducir. **No dejes fuzzy
   sin resolver**: el runtime lee el `.po` directo e ignora el flag fuzzy, así
   que una traducción fuzzy incorrecta se publica tal cual. Revisa también las
   entradas *obsolete* del propio `.po` cuando el emparejamiento de
   `msgmerge` falla (ha pasado con renombrados de cadenas: pares mal
   asociados que un `msgid` sin cambios no habría marcado).
4. `en_GB` se mantiene deliberadamente parcial: solo las cadenas donde cambia
   la ortografía británica (colour, etc.).
5. Valida: `for f in locale/*.po; do msgfmt --check -o /dev/null "$f"; done`
   -- cero errores (los warnings de cabecera tipo `Last-Translator` ausente
   son inofensivos y ya existen en el repo).

## 3. Comprobaciones de CI en local

Las mismas que corre `.github/workflows/ci.yml`:

```sh
# Sintaxis Lua bajo LuaJIT (el runtime real de KOReader)
find . -name '*.lua' -not -path './.git/*' -not -path './.claude/*' -print0 \
  | xargs -0 -n1 luajit -b /dev/null 2>&1 | grep -v '^$'

# Suite de tests bajo Lua 5.4 estándar (no LuaJIT: los tests mockean módulos
# de KOReader y una FFI real chocaría con las cdefs que faltan)
for f in tests/_test_*.lua; do echo "== $f =="; lua5.4 "$f"; done

# Traducciones (ver paso 2.5)
for f in locale/*.po; do msgfmt --check -o /dev/null "$f"; done
```

Si algún test falla, antes de bloquear la release comprueba si es una
regresión real: reproduce el mismo test contra el último tag publicado
(`git worktree add /tmp/check vX.Y.Z && lua5.4 /tmp/check/tests/_test_foo.lua`).
Si ya fallaba antes de tus cambios, es un fallo preexistente y no bloquea --
pero repórtalo.

> **Conocido a fecha de v5.0.0**: `tests/_test_book_repository.lua` (2
> aserciones sobre agrupación por idioma/rating) y `tests/_test_stack_display.lua`
> (crashea con `attempt to concatenate a nil value`) fallan de forma
> preexistente, confirmado ya roto en v4.7.0. Si siguen rotos, no son un
> nuevo problema; si se han arreglado, borra este aviso.

## 4. Tag y push

```sh
git tag -a vX.Y.Z -m "vX.Y.Z"
git push origin vX.Y.Z
```

## 5. Construir el zip distribuible

`git archive` respeta el `export-ignore` de `.gitattributes` (excluye
`tests/`, `tools/`, `docs/`, `.github/`, `*.pot/`, etc. -- solo entra lo que
el plugin necesita en runtime). La carpeta interior **debe** llamarse
`bookshelf.koplugin`: KOReader identifica el plugin por el nombre del
directorio.

```sh
git archive --format=zip --prefix=bookshelf.koplugin/ -o bookshelf.koplugin.zip vX.Y.Z
```

## 6. Publicar la release en GitHub

```sh
gh release create vX.Y.Z bookshelf.koplugin.zip \
  --title vX.Y.Z \
  --notes-file notas.md
```

Formato de las notas (en español, es el idioma que usa este fork en sus
releases): empieza con `Release del fork \`alosarjos/bookshelf.koplugin\`, en
vX.Y.Z.`, seguido de una lista de cambios agrupados por tema (no hace falta
listar cada commit -- para una minor/patch basta con las 3-6 líneas más
relevantes; para una major agrupa por área: nuevas features grandes primero,
luego fixes). Usa `git log v<anterior>..v<nueva> --oneline` como material en
bruto.

## Prerrequisitos que conviene comprobar antes de empezar

- `git status` limpio y rama `master` sincronizada con `origin/master`.
- `origin` apunta a `git@github.com:alosarjos/bookshelf.koplugin.git` (este
  fork, no el repo original).

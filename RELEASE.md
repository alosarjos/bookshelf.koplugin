# Cómo publicar una release

## Automatizado (happy path)

`.github/workflows/sync-and-release.yml` automatiza todo el proceso de abajo
para el caso normal: se ejecuta solo cada lunes (y también a mano desde
Actions → "Sync upstream & release" → "Run workflow", opcionalmente indicando
`upstream_tag` o unas `notes` concretas). Cuando corre:

1. Busca el último tag de `AndyHazz/bookshelf.koplugin` (o el indicado a mano).
2. Si ya está fusionado en `master`, no hace nada.
3. Si no, hace `git merge --no-ff` de ese tag. Usa las traducciones tal como
   vienen de upstream -- **no** regenera `.pot`/`.po` ni traduce nada a mano
   (eso solo pasa en el proceso manual, paso 3 de más abajo).
4. Corre el mismo gate que CI (`.github/actions/checks`): sintaxis LuaJIT,
   suite de tests, validación de `.po`.
5. Si todo pasa, empuja el merge a `master`, lee la versión de `_meta.lua`
   (ya fusionada) y crea el tag `vX.Y.Z`.
6. Construye el zip con `git archive` y publica la release en GitHub con
   `gh release create`, con notas auto-generadas a partir de `git log`.

**Solo cubre el happy path.** Si el merge tiene conflictos, si `_meta.lua` no
trae versión, o si algún check falla, el workflow aborta sin tocar `master`
ni crear ningún tag, y abre un Issue en el repo pidiendo intervención manual
(dedupe: no repite el issue si ya hay uno abierto para ese mismo tag). En ese
caso, sigue el proceso manual completo descrito a partir de aquí -- es el
mismo que reconstruye el resto de este documento a partir del historial de
commits `chore(release): ...` y de las releases ya publicadas en GitHub.

## Proceso manual

Sigue estos pasos en orden.

## 1. Sincronizar con upstream (si aplica)

Cuando upstream (`AndyHazz/bookshelf.koplugin`) ha publicado una versión
nueva y hay que traerla a este fork antes de re-publicar:

```sh
git fetch upstream --tags
git fetch upstream master
```

El tag de la release de upstream es la referencia a mezclar -- **no**
`upstream/master`: se ha dado el caso (v5.0.3) de que el tag más reciente de
upstream no era alcanzable desde ningún branch remoto suyo (probablemente
taguean desde una rama de PR que luego borran). Comprueba primero cuál manda:

```sh
git merge-base --is-ancestor upstream/master vX.Y.Z && echo "el tag es más nuevo"
```

Mezcla el tag, no el branch:

```sh
git merge --no-ff vX.Y.Z -m "Merge upstream vX.Y.Z into fork"
```

Antes de resolver conflictos a mano, localiza el ancestro común real y
compara qué ficheros tocó cada lado -- así sabes de antemano si un fix propio
del fork y un cambio de upstream van a chocar en la misma zona de un fichero
o simplemente coexisten:

```sh
git merge-base master vX.Y.Z            # ancestro común
git diff --stat <ancestro> master       # qué tocó el fork desde ahí
git diff --stat <ancestro> vX.Y.Z       # qué tocó upstream desde ahí
```

Si un mismo fichero aparece en ambas listas, mira el diff completo de ambos
lados sobre ese fichero antes de fusionar: casi siempre son zonas distintas
del fichero (merge limpio), pero si upstream ha seguido evolucionando
exactamente lo mismo que ya arregló el fork (p. ej. el mismo bug de color),
puede que su versión ya incluya y mejore el fix del fork -- revisa el mensaje
del commit de upstream, no asumas que hay que quedarse con el lado del fork.

Tras la fusión, sigue por el paso 2 (bump de versión) -- normalmente ya viene
resuelto por el propio merge si upstream tocó `_meta.lua`, pero compruébalo.
Los pasos de traducciones y CI en local aplican igual.

> **Trampa -- tags de upstream chocando con el tag de la release propia**:
> `git fetch upstream --tags` trae también SUS tags de versión (`v5.0.2`,
> `v5.0.3`, ...) al repo local. Si la nueva versión de este fork usa el mismo
> número (caso normal: este fork no re-numera, solo re-empaqueta la misma
> versión de upstream con sus propios fixes encima), el tag `vX.Y.Z` YA
> EXISTE localmente apuntando al commit de upstream, no al commit de merge de
> este fork. `git tag -a vX.Y.Z ...` falla ahí ("el tag ya existe") -- y si
> ese `tag` va encadenado con `&&`/`;` seguido de `git push origin vX.Y.Z`,
> el `push` SÍ se ejecuta y empuja a `origin` el tag equivocado (el de
> upstream, apuntando a un commit que ni siquiera es ancestro de tu release).
> Antes de taguear, comprueba SIEMPRE:
>
> ```sh
> git tag -l vX.Y.Z            # ¿existe ya?
> git rev-parse vX.Y.Z^{}      # ¿a qué commit apunta?
> ```
>
> Si apunta a un commit que no es tu commit de release, bórralo primero
> (`git tag -d vX.Y.Z`) y créalo de nuevo apuntando a tu HEAD. Si ya se
> empujó el equivocado a `origin`: `git push --delete origin vX.Y.Z` antes de
> publicar el correcto. (Nota aparte: `git fetch origin --tags` puede además
> BORRAR localmente tags que no existan en `origin` si `fetch.pruneTags` está
> a `true` -- no es destructivo de por sí, solo confuso si esperabas verlos;
> se recuperan re-haciendo `git fetch upstream --tags`.)

## 2. Bump de versión

Edita `_meta.lua`, campo `version`, a la nueva versión (semver: `MAJOR.MINOR.PATCH`).
Debe coincidir exactamente con el tag del paso 5 -- si no coincide, el
comprobador de actualizaciones del plugin muestra un "Update available"
perpetuo tras instalar.

## 3. Traducciones

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

Si el paso 1 (sincronizar con upstream) no introdujo ninguna cadena `_()`/`T()`
nueva -- compruébalo con `git diff <ancestro> vX.Y.Z -- '*.lua'` buscando
líneas añadidas que contengan `_(` o `T(` --, este paso no requiere tocar
nada: el `.pot`/`.po` siguen siendo válidos tal cual.

## 4. Comprobaciones de CI en local

Las mismas que corre `.github/workflows/ci.yml`:

```sh
# Sintaxis Lua bajo LuaJIT (el runtime real de KOReader)
# OJO: `luajit -b <entrada> <salida>` -- con xargs -n1 el fichero se añade
# como ÚLTIMO argumento, así que "xargs -n1 luajit -b /dev/null" ejecuta
# luajit -b /dev/null <fichero> y SOBREESCRIBE cada .lua con bytecode
# compilado de una entrada vacía. Usa -I{} para poner el fichero primero.
find . -name '*.lua' -not -path './.git/*' -not -path './.claude/*' -print0 \
  | xargs -0 -I{} luajit -b {} /dev/null

# Suite de tests bajo Lua 5.4 estándar (no LuaJIT: los tests mockean módulos
# de KOReader y una FFI real chocaría con las cdefs que faltan)
for f in tests/_test_*.lua; do echo "== $f =="; lua5.4 "$f"; done

# Traducciones (ver paso 3.5)
for f in locale/*.po; do msgfmt --check -o /dev/null "$f"; done
```

Si algún test falla, antes de bloquear la release comprueba si es una
regresión real: reproduce el mismo test contra el último tag publicado
(`git worktree add /tmp/check vX.Y.Z && lua5.4 /tmp/check/tests/_test_foo.lua`).
Si ya fallaba antes de tus cambios, es un fallo preexistente y no bloquea --
pero repórtalo.

> **Conocido a fecha de v5.0.3**: `tests/_test_book_repository.lua` (2
> aserciones sobre agrupación por idioma/rating) y `tests/_test_stack_display.lua`
> (crashea con `attempt to concatenate a nil value`) fallan de forma
> preexistente, confirmado ya roto en v4.7.0. Si siguen rotos, no son un
> nuevo problema; si se han arreglado, borra este aviso.

## 5. Tag y push

Primero asegúrate de que `master` (con el merge del paso 1 ya incluido, si
aplicaba) está empujado a `origin` -- el tag debe colgar de un commit que
`origin` ya tiene:

```sh
git push origin master
```

Luego el tag (ver la trampa del paso 1 sobre tags de upstream con el mismo
número antes de ejecutar esto):

```sh
git tag -a vX.Y.Z -m "vX.Y.Z"
git push origin vX.Y.Z
```

## 6. Construir el zip distribuible

`git archive` respeta el `export-ignore` de `.gitattributes` (excluye
`tests/`, `tools/`, `docs/`, `.github/`, `*.pot/`, etc. -- solo entra lo que
el plugin necesita en runtime). La carpeta interior **debe** llamarse
`bookshelf.koplugin`: KOReader identifica el plugin por el nombre del
directorio.

```sh
git archive --format=zip --prefix=bookshelf.koplugin/ -o bookshelf.koplugin.zip vX.Y.Z
```

## 7. Publicar la release en GitHub

```sh
gh release create vX.Y.Z bookshelf.koplugin.zip \
  --repo alosarjos/bookshelf.koplugin \
  --title vX.Y.Z \
  --notes-file notas.md
```

> **Trampa -- `gh` apunta al repo equivocado por defecto**: en un clon con
> remoto `upstream` además de `origin`, `gh` sin `--repo` puede resolver el
> repo por defecto al ORIGINAL (`AndyHazz/bookshelf.koplugin`), no a este
> fork -- compruébalo con `gh repo view --json nameWithOwner`. Un
> `gh release create` sin `--repo` explícito intenta crear la release en el
> repo equivocado: si upstream ya tiene una release con ese mismo número de
> versión falla con "a release with the same tag name already exists", lo
> cual al menos avisa -- pero no cuentes con que siempre falle así de claro
> ni con que no tengas permisos de escritura allí. Pasa **siempre**
> `--repo alosarjos/bookshelf.koplugin` explícito en cualquier comando `gh`
> de este proceso (`release create`, `release view`, `release list`, etc.).

Formato de las notas (en español, es el idioma que usa este fork en sus
releases): empieza con `Release del fork \`alosarjos/bookshelf.koplugin\`, en
vX.Y.Z.`, seguido de una lista de cambios agrupados por tema (no hace falta
listar cada commit -- para una minor/patch basta con las 3-6 líneas más
relevantes; para una major agrupa por área: nuevas features grandes primero,
luego fixes). Usa `git log v<anterior>..v<nueva> --oneline` como material en
bruto. Si la release incluye una sincronización con upstream (paso 1), dilo
explícitamente ("Sincroniza los cambios de upstream (vA.B.C y vD.E.F) sobre
este fork") antes de la lista de cambios.

## Prerrequisitos que conviene comprobar antes de empezar

- `git status` limpio y rama `master` sincronizada con `origin/master`.
- `origin` apunta a `git@github.com:alosarjos/bookshelf.koplugin.git` (este
  fork, no el repo original) -- y cualquier comando `gh` de este proceso
  lleva `--repo alosarjos/bookshelf.koplugin` explícito (ver trampa del
  paso 7).

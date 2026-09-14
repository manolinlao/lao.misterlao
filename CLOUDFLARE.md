# Lao y Cloudflare Tunnel

## Arquitectura

Lao se ejecuta localmente en el puerto `5174` y se publica mediante el túnel
existente de Cloudflare llamado `badshot-home`.

```text
https://lao.misterlao.com
        |
        v
Cloudflare Tunnel: badshot-home
        |
        v
http://localhost:5174
```

BadShot continúa usando el mismo túnel en `localhost:5173`.

## Configuración de Vite

El archivo `vite.config.ts` debe incluir:

```ts
server: {
  port: 5174,
  strictPort: true,
  allowedHosts: ['lao.misterlao.com'],
}
```

`allowedHosts` permite que Vite acepte peticiones que llegan desde el dominio
público a través de Cloudflare.

## Configuración del túnel

El archivo del túnel está fuera del proyecto:

```text
~/.cloudflared/config.yml
```

Debe tener una entrada para Lao antes de la regla final `404`:

```yaml
  - hostname: lao.misterlao.com
    service: http://localhost:5174
```

La regla final debe mantenerse al final:

```yaml
  - service: http_status:404
```

No se debe crear un túnel nuevo: Lao reutiliza `badshot-home`.

## Arranque local

### 1. Arrancar Lao

Desde la carpeta del proyecto:

```bash
cd /Users/manolin/MANOLIN/dvlp/lao
npm run dev
```

La aplicación debe estar disponible en:

```text
http://localhost:5174
```

### 2. Validar la configuración del túnel

El orden correcto de las opciones es:

```bash
cloudflared tunnel --config ~/.cloudflared/config.yml ingress validate
```

No escribir `\~`: la tilde debe ir sin barra invertida para que el shell la
expanda correctamente.

### 3. Crear la ruta DNS

Este comando se ejecuta una vez:

```bash
cloudflared tunnel route dns badshot-home lao.misterlao.com
```

Si indica que el registro ya existe, no es necesario repetirlo.

### 4. Arrancar el túnel

En otra Terminal:

```bash
cloudflared tunnel --config ~/.cloudflared/config.yml run badshot-home
```

Después, Lao estará disponible en:

```text
https://lao.misterlao.com
```

Para que funcione desde Internet deben estar ejecutándose tanto Lao como
`cloudflared`.

## Reiniciar después de cambiar la configuración

Si se modifica `config.yml` o `vite.config.ts`:

1. Detener el proceso correspondiente con `Ctrl + C`.
2. Volver a ejecutar `npm run dev` para Lao.
3. Volver a ejecutar el comando `cloudflared tunnel ... run badshot-home`.

## Migración al miniPC

Cuando Lao se mueva al miniPC:

1. Instalar `cloudflared` en el miniPC.
2. Copiar `config.yml` al directorio `~/.cloudflared/` del usuario que
   ejecutará el túnel.
3. Copiar también el archivo de credenciales `.json` del túnel.
4. Actualizar `credentials-file` en `config.yml` para que apunte a la ruta del
   miniPC.
5. Ejecutar Lao en `localhost:5174`.
6. Ejecutar el mismo túnel `badshot-home` en el miniPC.
7. Detener `cloudflared` en el Mac para evitar tener dos conectores activos
   innecesariamente.

No es necesario volver a crear el túnel ni el registro DNS.

## Seguridad

- No subir `config.yml` ni el archivo de credenciales `.json` a Git.
- No compartir el contenido del archivo `.json`.
- El túnel debe ejecutarse solo en el equipo que aloja la aplicación.

# Contenedores muy lentos

Cuando linkeamos nuestros archivos locales con el contenedor mediante el comando

```
-v local/path:/container/path
```

Esto hará que el contenedor se comporte demasiado lento porque Docker en Windows trabaja sobre el subsistema de Linux (WSL).  
El problema ocurre cuando montamos archivos que viven en el filesystem de Windows (NTFS) dentro de Linux (ext4 virtualizado).

En este caso **los archivos no se copian**, pero Docker y WSL deben usar una capa de traducción entre ambos filesystems, lo que genera una degradación importante del rendimiento, sobre todo en proyectos con muchos archivos (por ejemplo, `node_modules`, watchers o hot reload).

Por esta razón el contenedor puede sentirse extremadamente lento dependiendo del tamaño del proyecto.

Trataré de solucionarte este problema en los siguientes pasos:

<br/>

---

## Índice

[1. Comprobar que tengas al menos una Distro WSL2 instalada](#1-comprobar-que-tengas-al-menos-una-distro-wsl2-instalada)

[2. Comprobar versión WSL e instalar una Distro (si no tienes una)](#2-comprobar-versión-wsl-e-instalar-una-distro-si-no-tienes-una)

[3. Usar tu Distro WSL2 en Docker Desktop](#3-usar-tu-distro-wsl2-en-docker-desktop)

[4. Copia tu proyecto local dentro del filesystem de tu Distro](#4-copia-tu-proyecto-local-dentro-del-filesystem-de-tu-distro)

[5. Abrir VSCode en la nueva ubicación de tu proyecto dentro de WSL](#5-abrir-vscode-en-la-nueva-ubicación-de-tu-proyecto-dentro-de-wsl)

[6. Copiar la ruta de tu proyecto en WSL](#6-copiar-la-ruta-de-tu-proyecto-en-wsl)

[7. Docker run!](#7-docker-run)

[8. Errores comunes](#errores-comunes)

---
<br/>

## 1. Comprobar que tengas al menos una Distro WSL2 instalada

Aunque Docker Desktop utilice WSL2 internamente, **es obligatorio tener al menos una Distro de Linux instalada** (Ubuntu, Debian, etc.).  
Docker Desktop necesita una distro WSL2 activa para poder montar volúmenes y exponer correctamente sus servicios internos.

En la interfaz de Docker Desktop ve hacia:

**Configuraciones > Resources > WSL Integration**

![image](https://user-images.githubusercontent.com/33007720/216810893-0c28ddb0-85e6-47d2-8f51-ed297884a0f5.png)

Si te aparece un mensaje como este:

_You don’t have any WSL 2 distro. Please convert a WSL 1 distro to WSL 2_

Significa que **no tienes ninguna distro WSL2 disponible**, por lo que Docker no podrá funcionar correctamente.

Continúa con el siguiente paso para instalar o convertir una distro a WSL2.  
Si **no** te aparece este mensaje, puedes ir directamente al paso:  
[4. Copia tu proyecto local hacia la carpeta de tu Distro](#4-copia-tu-proyecto-local-hacia-la-carpeta-de-tu-distro)


## 2. Comprobar versión WSL e instalar una Distro (si no tienes una)

Abre tu __PowerShell de Windows__ y escribe el siguiente comando:

```
wsl -l -v
```
Se mostrará un listado de las Distros que tienes instaladas.  
La que esté marcada con un signo de asterisco (__*__) es la que se utiliza por defecto.

> ⚠️ **Nota importante**  
> Aunque no planees usar la distro directamente, **Docker Desktop requiere al menos una Distro WSL2 instalada** para funcionar correctamente.

![image](https://user-images.githubusercontent.com/33007720/216811119-54046f10-f426-4e72-bf02-41e8f008128a.png)

Para actualizar tu Distro basta con escribir lo siguiente:

```bash
# wsl --set-version NombreDeTuDistro NUMERO
# por ejemplo para ubuntu
wsl --set-version Ubuntu 2
```

_Este paso está de más, pero si deseas instalar otra Distro para usar con Docker te apoyo con los siguientes comandos:_

```bash
# instalar otra Distro. Ejemplo: Ubuntu
wsl --install -d Ubuntu

# Finalizar Distro anterior
wsl --terminate Debian

# Seleccionar por default Distro Nueva 
wsl --set-default Ubuntu

# Actualizar tu Distro Nueva a la versióna WL2
wsl --set-version Ubuntu 2
```

## 3. Usar tu Distro WSL2 en Docker Desktop

Nuevamente, en la interfaz de Docker Desktop ve hacia  
**Configuraciones > Resources > WSL Integration**.

En este punto ya no deberías ver el mensaje solicitando actualizar o instalar una Distro WSL2.

__NOTA__: Si el mensaje persiste después de haber realizado los pasos anteriores, reinicia Docker Desktop.  
En algunos casos puede quedarse en estado **"stopping"** indefinidamente, por lo que puede ser necesario cerrarlo desde el **Administrador de tareas**.


Puedes aceptar el __checkbox__ o seleccionar tu distro para usar con Docker Desktop. En mi caso yo lo tengo así y me funciona sin problemas:
![image](https://user-images.githubusercontent.com/33007720/216812065-ba3fa736-10aa-474b-9b65-f06f92b09c90.png)

__NOTA 2__: Si no ves tu nueva Distro presiona el boton que dice __Refresh__.

## 4. Copia tu proyecto local dentro del filesystem de tu Distro

Este paso es clave para mejorar el rendimiento.  
El proyecto debe vivir **dentro del filesystem de la Distro WSL2** y no en una ruta de Windows (`C:\`).


  - En tu __explorador de carpetas__ de windows dirígete hacia la barra de navegación y escribe ésta dirección:

    ```
    \\wsl$
    ```

    Verás las carpetas de tus Distros:

    ![image](https://user-images.githubusercontent.com/33007720/216812631-a7720eb5-0800-4dba-8b7c-23c2cad73df0.png)

    En mi caso opté por utilizar Ubuntu como distro por default. Así que entra a la carpeta de tu Distro, en este caso
    __Ubuntu > Home > TuNombreUsuario__.

  - Crea una nueva carpeta en donde guardarás tus proyectos con Node para tener una mejor organización de los mismos.
    Nombra tu carpeta de acuerdo a tus proyectos como "dev", "desarrollo", "mis-proyectos", etc...

    ![image](https://user-images.githubusercontent.com/33007720/216812905-e8bc7b80-ceb2-44cf-bec8-35484f6e490a.png)

  - Arrastra o copia y pega la carpeta de tu proyecto __en windows__ hacia la carpeta para proyectos que acabamos de crear en nuestra __Distro__
    __NOTA__: Asegúrate de **NO copiar** la carpeta `node_modules` ni el archivo `yarn.lock`.  
Estos serán generados dentro del contenedor al instalar las dependencias.

    ![image](https://user-images.githubusercontent.com/33007720/216813260-10752b93-223c-4c6a-a1bf-0deceda9d0bb.png)


## 5. Abrir vscode en la nueva ubicación de tu proyecto dentro de WSL

Hay dos formas, la más fácil es haciendo "click derecho" en tu carpeta del proyecto y en el menú desplegable seleccionar "Abrir con ... code"

![image](https://user-images.githubusercontent.com/33007720/216814048-d5a0529d-7dff-4ca9-9141-7970e2e41dfb.png)

<hr/>

La otra forma es desde la terminal de tu Distro, en mi caso al instalar la distro de Ubuntu, windows instaló su terminal en forma de aplicación de windows y lo que hice fue anclarla a mi barra de tareas.


![image](https://user-images.githubusercontent.com/33007720/216814181-31074391-2197-4dfa-acf2-e9589115d6a5.png)

<hr/>

Otra opción es instalando la aplicacion "__Windows Terminal__" desde la __Tienda de Aplicaciones__ de Microsoft:

![image](https://user-images.githubusercontent.com/33007720/216814306-af8b0782-2c5d-4877-8470-611812429816.png)

Una vez instalada, para abrir la terminal de nuestra Distro seleccionamos en la "flecha" al final de la pestaña para seleccionar la terminal de nuestra Distro.

![image](https://user-images.githubusercontent.com/33007720/216813997-f5a72e1d-dddc-43eb-bd3c-d004ea90f93a.png)


## 6. Copiar la ruta de tu proyecto en WSL

Nuevamente hay dos formas para hacerlo:

  - Entrar a la carpeta del proyecto (el de la Distro) y haciendo "click derecho" en la barra de navegación y copiar (o Copiar dirección):
  
    ![image](https://user-images.githubusercontent.com/33007720/216815066-d8595786-e7bd-4103-b376-33cc7997b51d.png)
    
  - O también escribiendo el comando __pwd__ dentro de la terminal integrada de __vscode__:

    ![image](https://user-images.githubusercontent.com/33007720/216814965-9fb27148-25be-49f0-8c95-2ab851933ba0.png)


## 7. Docker run!

Para evitar problemas con rutas y permisos, se recomienda ejecutar este comando **desde la terminal de tu Distro WSL2**.
Si ejecutas el comando desde PowerShell de Windows, asegúrate de usar una ruta válida dentro del filesystem de WSL.

Te facilito el comando de la clase en donde reemplazrás __tu-dir__ por la dirección que acabas de copiar.

NOTA: Asegúrate de borrar el contenedor anterior si tiene el mismo nombre.

```
docker container run `
--name nest-app `
-w /app `
-p 80:3000 `
-v tu-dir:/app `
node:18-alpine3.16 `
sh -c "yarn install && yarn start:dev"
```

en mi caso este fue el comando exacto:

```
docker container run `
--name nest-app `
-w /app `
-p 80:3000 `
-v \\wsl.localhost\Ubuntu\home\irra\desarrollo\nest-graphql:/app `
node:18-alpine3.16 `
sh -c "yarn install && yarn start:dev"
```

Cuando puedas acceder al localhost, intenta volver a modificar un archivo, el contenedor ahora si debería detectar el cambio, pero dependiendo de la velocidad de tu procesador y algunos otros factores como la RAM, puede que demore algunos segundos. 

## Errores comunes

### 1. `can't access specified distro mount service: stat /run/guest-services/distro-services/docker-desktop.sock`
- **Causa:** No tienes ninguna Distro WSL2 instalada o activa.
- **Solución:** Instala una Distro (Ubuntu, Debian, etc.) y actívala en Docker Desktop → **Settings > Resources > WSL Integration**. Luego mueve tu proyecto dentro del filesystem de la Distro y ejecuta Docker desde la terminal WSL.

### 2. Contenedor extremadamente lento al montar proyecto desde `C:\...`
- **Causa:** Montar archivos de Windows (NTFS) en Docker a través de WSL genera latencia.
- **Solución:** Copia tu proyecto al filesystem de la Distro WSL2 (`/home/usuario/...`) y ejecuta Docker desde allí.

### 3. `stopping` o congelado al reiniciar Docker Desktop
- **Causa:** Docker Desktop no detecta correctamente la Distro WSL2.
- **Solución:** Reinicia Docker Desktop. Si queda en estado `stopping`, ciérralo desde el Administrador de tareas y vuelve a abrirlo.

> 💡 **Tip:** Siempre verifica que tu Distro esté en WSL2 con:
> ```powershell
> wsl -l -v
> ```

<hr/>

<br/>

__Espero que te haya servido. Es un placer ayudarte.__


### Referencias:

https://forums.docker.com/t/docker-resources-you-dont-have-any-wsl-2-distro-please-convert-a-wsl-1-distro-to-wsl-2/99100

https://stackoverflow.com/questions/62154016/docker-on-wsl2-very-slow

https://human-se.github.io/rails-demos-n-deets-2020/demo-wsl/

https://forums.docker.com/t/docker-too-slow-to-pick-up-changes-in-local-filesystem/7878/2



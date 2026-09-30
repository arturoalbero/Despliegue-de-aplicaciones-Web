# Uso de Docker como terminal de Linux

## Docker exec

Un contenedor no es una máquina virtual, se trata de un proceso aislado. Sin embargo, podemos interactuar con él y conectarnos a él, incluso emplearlo como si fuera una terminal de Linux. Esto nos permite personalizar el contenedor a nuestro gusto y, finalmente, podemos convertirlo en una imagen que compartir con nuestros compañeros u otras personas interesadas.

Tenemos dos formas de lanzar el terminal del contenedor de forma interactiva. La primera es en el momento de lanzarlo, con una instrucción así:
```bash
docker run --name miLinux -it alpine:latest sh
```
En esta instrucción, estamos ejecutando el contenedor `miLinux` con la imagen `alpine:latest` en modo iterativo `-it`. Además, le estamos diciendo que ejecute la instrucción `sh`, que significa `shell` (Terminal). Si cerramos el contenedor y lo volvemos a lanzar, no aparecerá de nuevo el terminal.

Para ejecutar instrucciones en un contenedor activo de Docker tenemos que emplear el comando `docker exec`. En este caso:
```bash
docker exec -it miLinux sh
```

Igual que antes, ejecutamos de manera iterativa (`exec` y `-it`) en el contenedor activo `miLinux` la instrucción `sh`, que nos permite acceder al terminal. Algunas distribuciones de linux utilizan `sh`shell, otras `bash` y otras ambos. Es cuestión de probar. Por ejemplo, para una de ubuntu podríamos hacer:

```bash
# Crear un contenedor que se mantiene vivo en segundo plano
docker run -dit --name lab ubuntu:24.04

# Abrir una shell dentro de él
docker exec -it lab bash
```

Cabe destacar que `-dit` es la fusión de la instrucción `-d`(detached) y `-it` (del modo iterativo, que combina en realidad la `i` de input y la `t` de terminal). Sin embargo, el primer docker run no nos va a meter en el terminal, porque no se ejecuta el comando `bash`. Otras opciones útiles de `exec` son:

```bash
docker exec -it -u root lab bash          # loguea con un usuario concreto
docker exec -it -w /etc lab bash          # establece un directorio de trabajo inicial
docker exec -e VAR=valor lab env          # añade una variable de entorno
docker exec lab ls /etc                   # ejecuta un comando puntual, sin shell interactiva
```

### Comandos de linux

Para sacarle partido al uso de un contenedor como un terminal de linux, conviene repasar algunos de los comandos que se pueden usar, tanto en bash como en shell:

| Categoría | Ubuntu (bash) | Alpine (sh) |
|---|---|---|
| Navegación | `pwd`, `ls -la`, `cd`, `tree`* | `pwd`, `ls -la`, `cd`, `tree`* |
| Archivos | `cp`, `mv`, `rm -r`, `mkdir -p`, `touch`, `ln -s` | `cp`, `mv`, `rm -r`, `mkdir -p`, `touch`, `ln -s` |
| Ver/editar texto | `cat`, `less`, `head`, `tail -f`, `nano`*, `vi`*, `grep`, `echo "x" > archivo` | `cat`, `less`, `head`, `tail -f`, `nano`*, `vi`, `grep`, `echo "x" > archivo` |
| Buscar | `find / -name "*.conf"`, `which`, `whereis` | `find / -name "*.conf"`, `which` |
| Permisos | `chmod`, `chown`, `id`, `whoami`, `su`, `useradd` | `chmod`, `chown`, `id`, `whoami`, `su`, `adduser` |
| Procesos | `ps aux`*, `top`*, `kill`, `pkill`* | `ps`, `top`, `kill`, `pkill` |
| Red | `ip a`*, `ping`*, `curl`*, `wget`*, `ss -tlnp`* | `ip a`, `ping`, `wget`, `curl`*, `netstat -tlnp` |
| Sistema | `uname -a`, `cat /etc/os-release`, `df -h`, `env` | `uname -a`, `cat /etc/os-release`, `df -h`, `env` |
| Paquetes | `apt update`, `apt install -y paquete`, `apt remove paquete` | `apk update`, `apk add paquete`, `apk del paquete` |
| Instalar el kit | `apt update && apt install -y nano tree curl wget iputils-ping iproute2 procps` | `apk add --no-cache bash nano tree curl iproute2 procps` |

\* *Estos componentes no vienen instalados en la imagen base; hay que instalarlos.*


## Configuración de un contenedor de Docker para trabajar con él

Para poder trabajar con un contenedor hay que asegurarse de que se pueda acceder a su contenido mediante el `volume binding` y, además, que los puertos estén mapeados correctamente. Si bindeamos los directorios de trabajo, podremos ver dicho contenido en nuestro escritorio, para así poder usar nuestras herramientas favoritas de edición, como VS Code o IntelliJ. Docker no permite montar directamente el raíz. Primero creamos en nuestra máquina los directorios de trabajo:

```bash
mkdir -p trabajo/web trabajo/nginx
```
Después, ejecutamos el docker run donde mapeamos los puertos que necesitemos y bindeamos nuestras carpetas de trabajo a las carpetas necesarias del contenedor:

```sh
docker run -dit --name lab-alpine \
  -p 8080:80 \
  -v "$(pwd)/trabajo/web:/var/www" \
  -v "$(pwd)/trabajo/nginx:/etc/nginx" \
  alpine:3.20
```

`$(pwd)` es una variable de docker, que se sustituye por `./`. Si no te funcionara `$(pwd)` (ya que depende del SO), puedes emplear `./` (siempre que no uses una versión antigua de Docker).

## Creación de una imagen a partir de un contenedor

Cuando el contenedor está como queremos, lo "congelamos" en una imagen con `docker commit`:

```bash
# Salir del contenedor (opcional, se puede hacer con él en marcha). 
# Congelamos lab y lo llamamos mi-ubuntu-nginx, asignándole el número de versión 1.0
docker commit lab mi-ubuntu-nginx:1.0

# Comprobamos que la imagen se haya creado correctamente
docker images

# Ahora podemos crear contenedores nuevos a partir de ella
docker run -dit --name lab2 -p 8081:80 mi-ubuntu-nginx:1.0
```
Cabe destacar lo siguiente:

- La imagen guarda el sistema de archivos (paquetes instalados, configuraciones, archivos creados fuera de volúmenes).
- No guarda el contenido de volúmenes/bind mounts ni el mapeo de puertos (-p): hay que volver a indicarlos en cada docker run.
- Se puede fijar el comando de arranque o los puertos expuestos al hacer el commit:
```bash
  docker commit -c 'CMD ["nginx","-g","daemon off;"]' -c 'EXPOSE 80' lab mi-nginx:1.0
```
- Con `docker history mi-nginx:1.0` puedes ver las capas.
- Para compartirla puedes usar: `docker tag` + `docker push`, o `docker save -o imagen.tar` y `docker load -i imagen.tar`.

> **Advertencia:** `docker commit` es útil para aprender y experimentar, pero no es reproducible por otros usuarios: nadie sabe exactamente qué comandos se ejecutaron. La forma más profesional escrear un Dockerfile: todo lo que se hizo (apt install, configurar, CMD) se traduce en instrucciones (RUN, COPY, CMD).

> **ACTIVIDAD**: Para comprobar que has adquirido los conocimientos, crea un contenedor de alpine linux, usa docker exec para acceder a él, instálale nginx y mariadb, asegúrate de que funcione y congélalo mediante un commit.
>
> Después, usa la imagen que has creado para crear un nuevo contenedor, esta vez con los directorios de trabajo bindeados y los puertos correspondientes (para la base de datos y para el servidor web) correctamente mapeados.
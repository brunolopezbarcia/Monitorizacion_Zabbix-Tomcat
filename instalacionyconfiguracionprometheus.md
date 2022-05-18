# Como instalar y configurar prometheus con apache

## Prerequisitos

* Tener instalado httpd.
* Tener instalado grafana.
* Tener instalado wget

## Instalación

Lo primero que debemos de hacer es crear un usuario para el servicio el nombre que le pondremos sera `prometheus`

```bash
useradd -m -s /bin/false prometheus
```

Podemos comprobar que esta bien creado con  el comando `id prometheus`

Una vez creado el usuario debemos de crear los directorios de instalacion del programa.

```bash
mkdir /etc/prometheus
mkdir /var/lib/prometheus
```

Ahora pondremos como propietario de la carpeta `/var/lib/prometheus` al usuario _prometheus_

```bash
chown prometheus /var/lib/prometheus
```

Ahora debemos de descargarnos el _.tar_ de prometheus.

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.14.0/prometheus-2.14.0.linux-amd64.tar.gz -P /tmp
```

Ahora debemos de ir a a la carpeta donde lo hemos descargado la cual es `/tmp`

Una vez en la carpeta debemos de extraer el archivo _.tar_ que acabamos de descargar. 

```bash
tar -zxpvf prometheus-2.14.0.linux-amd64.tar.gz
```

Ahora entraremos en la carpeta que se acaba de extraer, y copiaremos los binarios a la carpeta `/usr/local/bin`

```bash
cd /tmp/prometheus-2.14.0.linux-amd64
cp prometheus /usr/local/bin
cp promtool /usr/local/bin
```

Una vez copiados los binarios deberemos de crear un archivo de configuracion para prometheus.

```bash
nano /etc/prometheus/prometheus.yml
```

Le añadiremos el siguiente contenido:

```bash
# Global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute. 
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute. 
  scrape_timeout: 15s  # scrape_timeout is set to the global default (10s).
# A scrape configuration containing exactly one endpoint to scrape:# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['IP_SERVIDOR_A_MONITORIZAR:9090']
```

Ahora deberemos de agregar unas reglas al firewall

```bash
firewall-cmd --add-port=9090/tcp --permanent
firewall-cmd --reload
```

Ahora deberemos de crear un servicio para poder iniciar y para prometheus

```bash
nano /etc/systemd/system/prometheus.service
```

Le añadiremos el siguiente contenido:

```bash
[Unit]
Description=Prometheus Time Series Collection and Processing Server
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

Ahora ejecutaremos los siguientes comandos:

```bash
systemctl daemon-reload
systemctl start prometheus
systemctl enable prometheus
```

Para entrar a la interfaz de prometheus desde el navegador pondremos:  `http://IP_SERVER_PROMETHEUS:9090`


## Instalacion de node_exporter 

Node_exporter es una utilidad que permite monitorizar muchos parametros de los sistemas Linux.

Lo primero que debemos de hacer como en el caso de prometheus es añadir un usuario.

```bash
useradd -m -s /bin/false node_exporter
```

Una vez añadido el usuario descargaremos el archivo _.tar_.

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz
```
Extraeremos el archivo _.tar_ con el siguiente comando:

```bash
tar -zxpvf node_exporter-0.18.1.linux-amd64.tar.gz
```

Como con prometheus debemos de copiar los archivos binarios en la ruta `/usr/local/bin` y ponerle como usuario propietario al usuario `node_exporter`

```bash
cp node_exporter-0.18.1.linux-amd64/node_exporter /usr/local/bin
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Ahora debemos de crear un servicio para node_exporter.

```bash
nano /etc/systemd/system/node_exporter.service
```

Tendra el siguiente contenido:

```bash
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Y ejecutaremos los siguientes comandos para iniciar y habilitar el servicio.

```bash
systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter
```

Y añadiremos una excepcion en el firewall.

```bash
firewall-cmd --add-port=9100/tcp  --permanent
firewall-cmd --reload
```

Ahora en el archivo de configuracion de prometheus(`/etc/prometheus/prometheus.yml`) añadiremos lo siguiente despues de todo lo que tenemos, se debe de tener en cuenta las tabulaciones y los espacios; debe de quedar a la misma altura que en el caso anterior.

```bash
- job_name: 'node_exporter'
  static_configs:
   - targets: ['localhost:9100']
```

Ahora haremos un restart de prometheus y ya tendremos configurado prometheus para monitorizar linux. 

```bash
systemctl restart prometheus
```

En caso de querer añadir mas de una maquina a monitorizar la configuracion quedaria tal que así(solo debemos de modificar la parte del node_exporter.):

```bash
 - job_name: 'node_exporter'
   static_configs:
   - targets: ['IP_SERVIDOR_1:9100']
   - targets: ['IP_SERVIDOR_2:9100']
```

Y volveremos a hacer un restart de prometheus.

# Integrar prometheus con grafana.

En la interfaz grafica de grafana debemos de ir a la configuracion y añadir un nuevo *_Data Source_*, añadiremos un nuevo data source, y elejimos el servicio prometheus. Ahora nos saldra una panatalla como esta:

![](/img/addsource.png)

Donde pone URL debemos de poner la direccion que utilizamos para entrar a la interfaz web de prometheus, en este caso `http://IP_SERVER_PROMETHEUS:9090`. Ahora solo deberemos de bajar hasta abajo de todo y darle a _Save & Test_.

Ahora solo debes de crear un nuevo dashboard o importar uno de los dashboards creados por la comunidad. En nuestro caso importamos el dashboard con id `3662` y ya tendrias configurado prometheus para monitorizar un sistema linux.


# Configuracion Prometheus para monitorizar Apache

## Prerequsitos

* Tener instalado httpd.
* Tener instalado grafana.
* Tener instalado wget.
* Tener instalado prometheus.
* Tener instalado curl.


Lo primero que debemos de hacer es descargar el archivo _.tar_ de *apache_exporter*, lo haremos con el siguiente comando:

```bash
curl -s https://api.github.com/repos/Lusitaniae/apache_exporter/releases/latest|grep browser_download_url|grep linux-amd64|cut -d '"' -f 4|wget -qi -
```

Una vez descargado debemos de extraer el archivo tar y copiar sus binarios a `/usr/local/bin`

```bash
tar xvf apache_exporter-*.linux-amd64.tar.gz
sudo cp apache_exporter-*.linux-amd64/apache_exporter /usr/local/bin
sudo chmod +x /usr/local/bin/apache_export
```

Ahora crearemos el servicio para apache_exporter.

```bash
sudo vim /etc/systemd/system/apache_exporter.service
```

El archivo tendra el siguiente contenido:

```bash
[Unit]
Description=Prometheus
Documentation=https://github.com/Lusitaniae/apache_exporter
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/apache_exporter \
  --insecure \
  --scrape_uri=http://localhost/server-status/?auto \
  --telemetry.address=0.0.0.0:9117 \
  --telemetry.endpoint=/metrics

SyslogIdentifier=apache_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```


Ahora deberemos de crear el archivo de argumentos:

```bash
sudo vim /etc/sysconfig/apache_exporter
```

Con el siguiente contenido:

```bash
ARGS="--insecure --scrape_uri=http://localhost/server-status/?auto --telemetry.address=0.0.0.0:9117 --telemetry.endpoint=/metrics"
```

Ahora debemos de iniciar y habilitar el servicio.

```bash
sudo systemctl daemon-reload
sudo systemctl start apache_exporter.service
sudo systemctl enable apache_exporter.service
```


Ahora podemos comprobar que esta activo de dos maneras:

```bash
# Primera manera:
systemctl status apache_exporter.service

#Segunda manera:
sudo ss -tunelp | grep 9117

# Deberia aparecer algo asi: 
# tcp    LISTEN     0      128                   :::9117                 :::*      users:(("apache_exporter",1970,6)) ino:1823474168 sk:ffff880341cd7800
```

Ahora debemos de añadir en el archivo de configuracion de prometheus el _job_ de *apache_exporter*

```bash
# Apache Servers
  - job_name: apache1
    static_configs:
      - targets: ['SERVER_IP_MONITORIZAR:9117']
        labels:
          alias: server1-apache
```

Y reiniciaremos el servicio `sudo systemctl restart prometheus`

Por ultimo debemos de añadir el dashboard a grafana, en este caso no es necesario añadir un nuevo data source debido a que ya teniamos añadido el source de Prometheus. En nuestro caso importamos un dashboard con el id `3894`
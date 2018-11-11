# Replica Set - MongoDB

En el siguiente documento voy a explicar todos los pasos a seguir para montar un **Replica Set de MongoDB** formado por 3 nodos.

En este ejemplo todos los servidores van a estar corriendo bajo Ubuntu Server 18.04 y MongoDB v4.0.4.

## ¿Qué es un Replica Set?

La finalizad de un Replica Set es la de proporcionar una **Alta Disponibilidad** de nuestras bases de datos MongoDB.
La idea consiste en tener corriendo varias instancias de mongo con el fin de que la información se replique entre ellas, de tal forma que si el nodo primario se cae, pueda ser remplazado automáticamente por otro.

![Esquema de red para un cluster de RabbitMQ y HAProxy](https://github.com/RafaMunoz/Replica-Set-MongoDB/blob/master/img/mongodb-replica-set.png)



# Esquema de Red

En el siguiente esquema podemos ver los diferentes servidores o nodos que van a formar nuestro sistema con sus correspondientes Hostname y Direcciones IP.

![Esquema de Red Replica Set](https://github.com/RafaMunoz/Replica-Set-MongoDB/blob/master/img/esquema_red_replica_set.png)



# Configuración basica

En este apartado veremos las configuraciones básicas que se tiene que realizar en cada servidor teniendo en cuenta sus direcciones IP y su Hostname.

Actualizamos el listado de los repositorios y a continuación actualizaremos los paquetes.

	sudo apt update
	sudo apt upgrade



Le pondremos a cada servidor una dirección IP fija a través de Netplan.
Lo primero será comprobar si ya disponemos de algún archivo de configuración:

    ls /etc/netplan


En caso de que no dispongamos de ningún archivo usaremos el comando `sudo netplan generate` y si ya disponemos de uso procedemos a editarlo.

     sudo nano /etc/netplan/50-netcfg.yaml


Un ejemplo de la configuración  de unos de los servidores puede ser la siguiente:

	network:
	ethernets:
	    enp0s3:
	        addresses: [192.168.1.210/24]
	        dhcp4: no
	        gateway4: 192.168.1.1
	        nameservers:
	            addresses: [8.8.8.8, 8.8.4.4]
	version: 2


> Nota: Netplan utiliza ficheros .YAML y en algún momento puede dar problemas con las indentaciones. Es aconsejable realizar la configuración con espacios en vez de con tabulaciones.



Una vez hemos guardado nuestra configuración la podemos aplicar con el siguiente comando.

    sudo netplan apply 


Si durante la instalación del sistema operativo no hemos configurado el hostname correctamente o hemos partido de un clon de otra máquina virtual, lo cambiaremos por el que corresponda.

    sudo nano /etc/hostname


Por ejemplo para el nodo 1 será:

```
mongodb-01
```



Para que este cambio sea persistente a los reinicios editaremos el siguiente archivo.

```
sudo nano /etc/cloud/cloud.cfg
```



Y pondremos el siguiente setting a *true*.

```
preserve_hostname: true
```



Seguidamente también lo cambiaremos en el archivo host:

    sudo nano /etc/hosts


Además de cambiar su nombre, también tenemos que añadir el Hostname y la Dirección IP de los otros dos nodos.

    192.168.1.210	mongodb-01
    192.168.1.211   mongodb-02
    192.168.1.212   mongodb-03

> Nota: Si después nos vamos a conectar al Replica Set desde un ordenador diferente,por ejemplo a través de Robo 3T, necesitaremos tener también estas rutas en el equipo desde el que nos conectemos.



Deberemos reiniciar los servidores para que cojan los cambios.

    sudo reboot
 

# Instalación de Replica Set

## Instalación de MongoDB
Antes de empezar a configurar el Replica Set necesitamos tener instalado MongoDB de forma independiente en cada servidor.

Necesitamos importar la clave pública utilizada por el sistema de gestión de paquetes.

    sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 9DA31620334BD75D9DCB49F368818C72E52529D4


Importamos el repositorio de MongoDB.

	echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu bionic/mongodb-org/4.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.0.list


Actualizamos el listado del repositorio.

    sudo apt update


Procedemos con la instalación de MongoDB.


    sudo apt-get install -y mongodb-org


## Configuración del Replica Set

Ahora procedemos a realizar todas las configuraciones necesarias para el Replica Set.

Editaremos el archivo **mongod.conf** en todos los servidores.

	sudo nano /etc/mongod.conf


Modificaremos el siguiente contenido con la dirección IP correspondiente de cada uno.

 1. mongodb-01:
  	net:
  	  port: 27017
  	  bindIp: 192.168.1.210

 1. mongodb-02:
  	net:
  	  port: 27017
  	  bindIp: 192.168.1.211

 1. mongodb-03:
  	net:
  	  port: 27017
  	  bindIp: 192.168.1.212



Y en el mismo archivo añadiremos el nombre del Replica Set que sera el mismo para todos (**replica01**).

	replication:
	  replSetName: "replica01"



Habilitamos el servicio **mongod** para que se inicie automáticamente cuando se arranquen los servidores.

    sudo systemctl enable mongod.service


Reiniciamos el servicio mongod para que actualice los cambios realizados anteriormente en el archivo mongod.conf.

	sudo systemctl restart mongod.service


Uno de los nodos de MongoDB se ejecuta como PRIMARIO (**MASTER**), y todos los demás nodos funcionarán como SECUNDARIO (**SLAVE**). Los datos están siempre en el nodo PRIMARIO y los conjuntos de datos se replican en todos los demás nodos SECUNDARIOS.

Para configurar nuestro Replica Set iniciamos la terminal en uno de los nodos.

	mongo 192.168.1.210 


Iniciamos el conjunto de réplicas en el nodo 1 ejecutando el siguiente comando.

	rs.initiate()


Posteriormente añadimos los otros dos nodos al Replica Set.

	rs.add("mongodb-02")
	rs.add("mongodb-03")



Podemos comprobar el estado del RPS ejecutando:

	rs.status()
> En el caso de que en el nodo 1 nos haya puesto la dirección IP en vez de el nombre, podemos cambiarlo ejecutando los siguientes comando:
>
> ```
> cfg = rs.conf()
> cfg.members[0].host = "mongodb-01:27017"
> rs.reconfig(cfg)
> ```



Y el siguiente comando nos dirá cual de los nodos es el MASTER.

	rs.isMaster()


# Probando Alta Disponibilidad

Una vez tenemos configurado nuestro Replica Set solo nos queda realizar pruebas para ver si esto realmente funciona.

Hemos utilizado [Robo 3T](https://robomongo.org/) como IDE ya que permite conexiones a Replica Set y es muy intuitivo.

> Tenemos creada una base de datos llamada BD_Prueba y dos colecciones
> Col_01 y Col_02.



En el primer caso vemos como tenemos todos los nodos levantados, mongodb-01 como PRIMARIO y mongodb-02 y mongodb-03 como nodos SECUNDARIOS. 

![Robot 3T Replica Set All nodos UP](https://github.com/RafaMunoz/Replica-Set-MongoDB/blob/master/img/RPST_all_nodes_up.png)



Ahora procedemos a parar el servicio de mongodb-01.

    sudo service mongod stop
Si refrescamos la conexión desde nuestro IDE apreciamos como podemos seguir conectándonos y accediendo a nuestro datos pero en este caso el nodo 1 mongodb-01 esta sin conexión, mongodb-02 a pasado a ser el nodo PRIMARIO y mongodb-03 sigue estando como SECUNDARIO. 

![Robot 3T Replica Set nodo1 DOWN](https://github.com/RafaMunoz/Replica-Set-MongoDB/blob/master/img/RPST_node1_down.png)




# Seguridad

Todo servicio que instalemos tiene que tener cierta seguridad ante posibles intrusos y mas si pasan a ser entornos de producción. 



## Keyfile

Para tener la autenticación activa y que los diferentes nodos puedan comunicarse entre si, necesitamos crear un Keyfile.

Creamos el Keyfile en uno de los nodos.
	openssl rand -base64 756 > mongo-keyfile


Lo siguiente será crear la carpeta donde lo vamos a almacenar y moverlo a dicha carpeta.

	sudo mkdir /opt/mongo
	sudo mv mongo-keyfile /opt/mongo



Le damos permisos para que lo pueda leer mongo.

	sudo chmod 400 /opt/mongo/mongo-keyfile
	sudo chown mongodb:mongodb /opt/mongo/mongo-keyfile



Esta clave necesitamos copiarla a los otros dos nodos. 
Lo copiaremos a través del siguiente comando donde deberemos poner el nombre de usuario del servidor y la dirección IP del destino.

	sudo rsync -av /opt/mongo/mongo-keyfile USUARIO@DIRECCION_IP:/opt/mongo/

> Nota: Será necesario tener la carpeta /opt/mongo creada antes de copiar la clave y posteriormente asignar los permisos al archivo.



## Usuario

También es importante tener en nuestras bases de datos uno o varios usuarios para restringir el acceso y que no pueda acceder todo el mundo, ya que cuando lo instalamos por defecto nos viene sin esta protección.

En este caso vamos a crear un usuario con todos los permisos pero lo correcto seria crear usuarios con diferentes roles para cada persona que vaya a utilizar la base de datos. En el siguiente enlace puedes ver todos los roles que tiene MongoDB: [Roles MongoDB](https://docs.mongodb.com/manual/reference/built-in-roles/)

Lo primero sera conectarnos a la terminar de mongo del nodo que este iniciado como **PRIMARIO**.

	mongo 192.168.1.210


Seleccionamos la base de datos **admin**.

	use admin


Creamos el usuario **admin** con contraseña **admin123**.

	db.createUser({user:"admin", pwd:"admin123", roles:[{role:"root", db:"admin"}]})


Probamos a conectarnos en cualquier nodo.

	mongo 192.168.1.210 -u admin -p --authenticationDatabase admin


Ahora deberemos habilitar la autenticación desde el archivo mongod.conf. Esto lo realizaremos en todos los nodos.

	sudo nano /etc/mongod.conf


En el apartado #security añadimos las siguientes lineas.

	. . .
	security:
	  keyFile: /opt/mongo/mongo-keyfile
	. . . 



Reiniciamos el servicio mongod.

	sudo service mongod restart


# Conexión

Si estamos desarrollando nuestra propia aplicación en Python, Java, PHP... podemos utilizar la siguiente cadena de conexión:

	mongodb://admin:admin123@mongodb-01:27017,mongodb-02:27017,mongodb-03:27017/

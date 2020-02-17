# Cluster de alta disponibilidad
## Definición
Un cluster HA (High Availability) es un sistema orientado a ofrecer y ganartizar servicios en Alta Disponibilidad. Se basa en máquinas redundantes que asumen el servicio cuando algún componente del sistema falla.

Un cluster HA debe ser capaz de detectar cualquier fallo, de hardware o de software, reiniciar la aplicación en otro nodo y mantener el servicio.

## Esquema
En un cluster HA hay que eliminar todos los SPOF (Single Point of Failure), mediante redundancia a todos los niveles.
- Hardware
- Almacenamiento
- Redes

## Conceptos
- **Recurso**: normalmente asociado a un servicio que se quiere poner a prueba de fallos. El recurso pertenece al cluster, no al nodo. 
- **Heartbeat**: Pulso o latido mediante el que se comunican los nodos(por conexión dedica y cifrada).

- **Split brain**: Cuando se pierde la comunicación entre nodos y toman decisiones por su cuenta.
- **Quorum**: Mecanismo para prevenir splir brain.
- **Stonith**: Shoot The Other Node In The Head-> se utiliza sobre un nodo que no responde para asegurar que no esté accediendo a los datos y que se puedan corromper.

## Terminología
- **crm**: cluster resource manager. Software encargado de  la gestión de los recursos del clúster.
- **cib**: cluster information base. Formato para la configuración de los recursos (XML).
- **ocf**: open cluster framework Conjunto de estándares para clúster que definen las APIs que desarrollan las funciones del clúster.
- **message leyer**: capa en la que trabaja la aplicación encargada de controlar la comunicación entre los nodos (hearbeat).
- **VIP**: virtual IP.

## Software
- Hearbeat (Linux-HA): latidos desde el sistema.
- Pacemaker: aplicación dedicada a los latidos.
- OpenAIS: restricciones.
- pcs: cliente que gestiona el cluster (la información que otorga Pacemaker y OpenAIS) (Por ejemplo: pvs status).

## Práctica
### Levantar escenario
~~~
paloma@coatlicue:~/escenarios-HA/03-HA-IPFailover$ vagrant up
paloma@coatlicue:~/escenarios-HA/03-HA-IPFailover$ cd ansible/
paloma@coatlicue:~/escenarios-HA/03-HA-IPFailover/ansible$ ansible-playbook -b site.yaml
~~~

### Creación del cluster
Se va a configurar el cluster manualmente. 
Para ello se instalan las siguientes aplicacinoes en nodo1 y nodo2:
~~~
vagrant@nodo1:~$ sudo apt install pacemaker pcs
~~~

Se cambia la contraseña, en ambos nodos:
~~~
vagrant@nodo1:~$ sudo passwd hacluster
New password: 
Retype new password: 
passwd: password updated successfully
~~~

Se añaden las siguientes líneas en /etc/hosts de ambos nodos para que se conozcan:
~~~
10.1.1.101    nodo1
10.1.1.102    nodo2
~~~

Se eleminan los cluster anteriormente creados, si los hubiera:
~~~
vagrant@nodo2:~$ sudo pcs cluster destroy
Shutting down pacemaker/corosync services...
Killing any remaining services...
~~~

Se autentifican los nodos que van a pertenecer al cluster:
~~~
vagrant@nodo2:~$ sudo pcs host auth nodo1 nodo2 -u hacluster -p hacluster
nodo2: Authorized
nodo1: Authorized
~~~
*
*
*
*
*
*
*
*
*
*
*

Y crear un cluster que llamaremos `mycluster`:

    pcs cluster setup mycluster nodo1 nodo2 --start --enable --force

Para asegurar que los datos del cluster no se puedan corromper la propiedad `stonith` está habilitada por defecto, en nuestro caso para que funcione el cluster es necesario deshabilitarla:

    pcs property set stonith-enabled=false

Por último vamos a crear un recurso del tipo `ocf:heartbeat:IPaddr2`que nos permite que el cluster gestione una ipv4 que se asignará a la interface `eth1``de uno de los nodos. Si el nodo que tiene asignado el recurso no funciona, automáticamente se asignará al otro nodo.

    pcs resource create VirtualIP ocf:heartbeat:IPaddr2 ip=10.1.1.100 cidr_netmask=32 nic=eth1 op monitor interval=30s

Comprobamos el estado del cluster:

    pcs status

    pcs status
    Cluster name: mycluster
    Stack: corosync
    Current DC: nodo1 (version 2.0.1-9e909a5bdd) - partition with quorum
    Last updated: Tue Feb  4 14:59:10 2020
    Last change: Tue Feb  4 14:41:22 2020 by hacluster via crmd on nodo1

    2 nodes configured
    1 resource configured

    Online: [ nodo1 nodo2 ]

    Full list of resources:

     VirtualIP	(ocf::heartbeat:IPaddr2):	Started nodo1

    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled

Vemos como el cluster tiene configurado 2 nodos y un recurso, una IP virtual del tipo `ocf::heartbeat:IPaddr2` asignada al `nodo1`.
Si comprobamos la interface `eth1` del `nodo1` lo podemos comprobar:

    ip a show eth1
    3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:4c:68:a8 brd ff:ff:ff:ff:ff:ff
        inet 10.1.1.101/24 brd 10.1.1.255 scope global eth1
           valid_lft forever preferred_lft forever
        inet 10.1.1.100/32 brd 10.1.1.255 scope global eth1
           valid_lft forever preferred_lft forever

## Prueba de funcionamiento

* Edita el fichero `/etc/resolv.conf` de tu equipo y añade como servidor DNS primario el nodo "dns" que tiene la dirección IP `10.1.1.103`
* Comprueba la conectividad con los nodos del cluster con ping Utiliza dig para resolver el nombre `www.example.com`:

        $ dig @10.1.1.103 www.example.com

* Comprueba que la dirección `www.example.com` está asociada a la dirección IP `10.1.1.100`, que en este escenario es la IP virtual que estará asociada en todo momento al nodo que esté en modo maestro.
* Accede a uno de los nodos del clúster y ejecuta la instrucción `pcs status`. Comprueba que los dos nodos están operativos y que el recurso
VirtualIP está funcionando correctamente en uno de ellos.
* Haz ping a `www.example.com` desde la máquina anfitriona y comprueba la tabla arp. Podrás verificar que la dirección MAC asociada a la dirección IP `10.1.1.100` coincide con la del nodo maestro en estos momentos.
* Para el nodo maestro (supongamos que es `nodo1`):

        $ vagrant halt node2

* Haz ping a `www.example.com` y comprueba que la tabla arp ha cambiado. Ahora la dirección MAC asociada a la dirección IP `10.1.1.100` es la del otro nodo.
* Entra en el nodo maestro y comprueba el estado del clúster con `pcs status`.
* Levanta de nuevo el nodo que estaba parado. Los recursos no van a volver a él porque en la configuración se ha penalizado el movimiento de los recursos, estos tienden a quedarse en el nodo en el que se están ejecutando, no a volver al nodo que era maestro.
* Si queremos que cuando un nodo este levantado el recurso se asigne a este nodo tenemos que crear una restricción de localización de afinidad, por ejemplo cuando el nodo1 este levantado se le asigna el recurso indicamos los siguiente:

        pcs constraint location VirtualIP prefers nodo1=INFINITY

* Podmos ver las restricciones que tenemos asignadas:

        pcs constraint show
        
        Location Constraints:
          Resource: VirtualIP
            Enabled on: nodo1 (score:INFINITY)
        Ordering Constraints:
        Colocation Constraints:
        Ticket Constraints:

* Vuelve a apacgar el `nodo1` comprueba que el recurso se asigna al `nodo2`, vuelve a encender el `nodo1` y comprueba que por la restricción de localización el recurso vuelve a `nodo1`.

* Para eleminar la restricción ejecutamos:

        pcs constraint rm location-VirtualIP-pcmk-1-INFINITY

* Por último accede desde un navegador web a `www.example.com` comprueba al nodo al que esta accediendo, para ese nodo y comprueba que se accede al otro nodo.

## Creación del cluster de forma automática

He creado otro playbook para crear el cluster desde ansible, para ello simplemente:

    $ ansible-playbook -b site_cluster.yaml



Title: Creación y configuración de un escenario Router-NAT con Vagrant y Ansible
Date: 2021-10-10 15:01
Category: Automatización
lang: es
tags: Redes,NAT,Vagrant,Ansible,SSH
summary: En este artículo te explico cómo crear un escenario de Vagrant con dos nodos: un router con NAT conectado con un cliente por una red muy aislada que podrá conectarse a Internet. También prepararemos una receta de ansible que automatice todos los pasos a seguir, y veremos cómo conectarnos por SSH al cliente a través del router.

## Crear el escenario de Vagrant

Vamos a crear un escenario de Vagrant con dos nodos, ambas con un direccionamiento estático, y tendrán las siguientes características:

* El **router** tendrá un total de tres interfaces: la red de mantenimiento de Vagrant, un puente con la red del puente y una red muy aislada, a la que estará conectada la máquina cliente.
* El **cliente** tendrá dos interfaces: la de mantenimiento y la red muy aislada conectada al router.

Si vamos a tener una interfaz "bridge" en el router, lo primero que debemos hacer es crear el puente en nuestro host. Por lo tanto, nos dirigimos al fichero /etc/network/interfaces y lo editamos como root. Añadimos el siguiente contenido (sustituyendo "enp2s0" por nuestra interfaz primaria):
<pre><code class="shell">
#Puente
auto br0
iface br0 inet dhcp
        bridge_ports enp2s0
</code></pre>

<img src="{static}/images/relax.png" alt="Escuchemos un poco de música mientras esperamos" width="150"/>
Guardamos el fichero y reiniciamos el host.

Ahora que ya tenemos nuestro puente, creamos el escenario de Vagrant con la siguiente configuración:
<pre><code class="ruby">
Vagrant.configure("2") do |config|

    config.vm.define :nodo1 do |nodo1|
      nodo1.vm.box = "debian/bullseye64" 
      nodo1.vm.hostname = "servidor" 
      nodo1.vm.synced_folder ".", "/vagrant", disabled: true
      nodo1.vm.network :public_network,
        :dev => "br0",
        :mode => "bridge",
        :type => "bridge" 
      nodo1.vm.network :private_network,
        :libvirt__network_name => "muyaislada",
        :libvirt__dhcp_enabled => false,
        :ip => "10.0.0.1",
        :libvirt__forward_mode => "veryisolated" 
    end
    config.vm.define :nodo2 do |nodo2|
      nodo2.vm.box = "debian/bullseye64" 
      nodo2.vm.hostname = "cliente" 
      nodo2.vm.synced_folder ".", "/vagrant", disabled: true
      nodo2.vm.network :private_network,
        :libvirt__network_name => "muyaislada",
        :libvirt__dhcp_enabled => false,
        :ip => "10.0.0.2",
        :libvirt__forward_mode => "veryisolated" 
    end
  end
</code></pre>

Levantamos las máquinas con "vagrant up". Si entramos en el cliente, deberemos poder hacer ping al router.

## Configuración del escenario router-NAT

Vamos a configurar el escenario que hemos creado en el apartado anterior de manera que cumpla los siguientes objetivos:

* El **router** accederá a Internet por eth1, la interfaz que tenemos en modo puente.
* El **cliente** tendrá acceso a Internet a través de eth1, la interfaz de red aislada conectada con el router, el cual enrutará las peticiones del cliente.

Podemos hacerlo de dos formas: manualmente o con Ansible.

### <img src="{static}/images/manual.png" alt="Lo hacemos nosotros mismos" width="150"/> Método manual

Cambiamos el bit de forwarding para que el router pueda actuar como tal:
<pre><code class="shell">
echo "1" > /proc/sys/net/ipv4/ip_forward
</code></pre>

Entramos en el router y editamos el fichero /etc/network/interfaces, de manera que la interfaz eth0 y eth1 queden de la siguiente manera:
<pre><code class="shell">
# The primary network interface
allow-hotplug eth0
iface eth0 inet dhcp
    post-up ip route del default dev $IFACE || true
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
auto eth1
iface eth1 inet dhcp
    post-up iptables -t nat -A POSTROUTING -s 10.0.0.0 -o eth1 -j MASQUERADE
    #post-up ip route del default dev $IFACE || true
#VAGRANT-END
</code></pre>
De esta manera, eliminamos la ruta por defecto que tiene el router en la interfaz eth0. En su lugar, será eth1 el que enrute las peticiones provenientes de la red 10.0.0.0 y las mande por el puente. Guardamos el fichero y reiniciamos la máquina.

Por ultimo, salimos del router, entramos en el cliente y añadimos una línea en eth0 para eliminar la ruta por defecto. Ahora las peticiones se enviarán a la nueva puerta de enlace, el router:
<pre><code class="shell">
# The primary network interface
allow-hotplug eth0
iface eth0 inet dhcp
      post-up ip route del default dev $IFACE || true
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
auto eth1
iface eth1 inet static
      address 10.0.0.2
      netmask 255.255.255.0
      gateway 10.0.0.1
</code></pre>

Guardamos el fichero y reiniciamos la máquina.

### <img src="{static}/images/peli.png" alt="Se hace automáticamente" width="150"/> Método Ansible

Para esta ocasión tengo preparada esta maravillosa [receta de Ansible](https://github.com/LaraPruna/Escenario-router-nat-ansible). La receta realiza todos los pasos explicados en el método manual, salvo que de forma automatizada. Tan solo tienes que clonar el repositorio en el mismo directorio donde se encuentre el escenario, editar el fichero hosts para cambiar la dirección IP correspondiente a la interfaz eth0 de cada máquina y ejecutar el comando "ansible-playbook site.yaml".

Para **configurar las máquinas automáticamente desde la creación del escenario**, una vez nos aseguremos de que tenemos la receta de Ansible en el mismo directorio, editamos el fichero Vagrantfile y añadimos al final el siguiente contenido:
<pre><code class="ruby">
    config.vm.provision "ansible" do |ansible|
      ansible.playbook = "site.yaml" 
    end
end
</code></pre>

Al levantar el escenario, tras instalar los nodos, comenzará a aplicarse la receta.

## Conectarse por SSH al cliente pasando por el router

Antes que nada, introducimos nuestra clave publica en ambas máquinas. Podemos hacerlo manualmente (copia y pega), con una receta de Ansible o con el siguiente comando:
<pre><code class="shell">
ssh-copy-id -i ~/.ssh/id_rsa.pub vagrant@IP.de.la.máquina
</code></pre>

El router ya estará listo, pero para entrar por SSH en el cliente a través de eth1 necesitaremos hacer uso del **agente SSH**.

<img src="{static}/images/pensando.png" alt="Pensando..." width="150"/> ¿Qué es eso del agente SSH?

Veámoslo de la siguiente forma:

Queremos entrar en una zona restringida dentro de otra zona restringida. Para ello, accedemos a la primera zona llevándonos con nosotros a nuestro agente, nuestro ayudante. Como tenemos nuestras credenciales, no hay problema, nos dejan pasar.

Imaginemos que, para entrar a la siguiente zona (donde nos pedirán de nuevo las credenciales), necesitamos dejar allí al agente como nuestro representante. No obstante, para que cumpla su labor, él también necesita unas credenciales, una **identidad**.

De hecho, si ejecutamos el comando **ssh-add -L**, veremos todas las identidades que posee el agente en ese momento. Si aún no le hemos dado nada, nos dirán que el agente no dispone de ninguna identidad. No es nadie.

<img src="{static}/images/miedo.png" alt="Qué miedo" width="150"/> Es una persona sin rostro...

Por eso, nos reunimos con él en casa, en el host, y le damos una copia de nuestro carnet y nuestra cara con el comando **ssh-add ~/.ssh/id_rsa**. Ahora el agente y nosotros somos la misma persona.

A continuación, nos llevamos al agente al router, opción que activamos empleando uno de los dos siguientes procedimientos:

* Con el parámetro -A del comando ssh:
<pre><code class="shell">
ssh -A vagrant@IP.Interfaz.eth1.router
</code></pre>

* Añadiendo "ForwardAgent yes" en el fichero /etc/hosts (en el host) y accediendo después con ssh al router sin el parámetro -A:
<pre><code<pre><code class="shell">
Host router
    HostName 192.168.122.37
    ForwardAgent Yes
</code></pre>

Una vez dentro del router, podemos comprobar con el comando "ssh-add -L" que nuestro agente tiene una identidad, por lo que ya podemos dejarlo allí en representación nuestra y entrar por ssh al cliente como lo hacemos normalmente:
<pre><code class="shell">
ssh vagrant@IP.Interfaz.eth1.cliente
</code></pre>

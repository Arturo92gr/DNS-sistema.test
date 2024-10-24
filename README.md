# DNS_sistema.test
 Práctica: sistema.test master y slave

1. Activa solamente la escucha del servidor para el protocolo IPv4.

    En el servidor Tierra:  
        ```
        cd /etc/default  
        sudo nano named
        ``` 
        Se cambia la línea: `OPTIONS="-u bind -4"`  
    Se copia el archivo named a la capeta compartida para añadirlo a la provisión:  
        En máquina: `cp named /etc/files/`  
        En provisión: `cp -v /files/named /etc/default`  

2. Establecer la opción dnssec-validation a yes

    `sudo nano /etc/bind/named.conf.options`  
    Se modifican las siguientes líneas:  
        ```
        dnssec-validation yes;  
        //listen-on-v6 { any; };  
        ```
    Se vuelve a copiar el archivo a la carpeta compartida para añadirlo a la provisión  
        En máquina: `cp /etc/bind/named.conf.options /etc/files/`  
        En provisión: `cp -v /files/named.conf.options /etc/bind`  

3. Los servidores permitirán las consultas recursivas sólo a los ordenadores en la red 127.0.0.0/8 y en la red 192.168.57.0/24, para ello utilizarán la opción de listas de control de acceso o acl.

    Primeramente, realizar una copia de seguridad: `sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.backup`  
    Se edita el archivo named.conf.options de la siguiente manera:  
    ```
    acl confiables {  
        127.0.0.0/8;  
        192.168.57.0/24;  
    };

    options {  
        directory "/var/cache/bind";  

        // forwarders {  
        //      0.0.0.0;  
        // };  

        listen-on port 53 { 192.168.57.103; };  

        recursion yes;  
        allow-recursion { confiables; };  

        //========================================================================  
        // If BIND logs error messages about the root key being expired,  
        // you will need to update your keys.  See https://www.isc.org/bind-keys  
        //========================================================================  
        dnssec-validation yes;  

        //listen-on-v6 { any; };  
    };  
    ```

4. El servidor maestro será tierra.sistema.test y tendrá autoridad sobre la zona directa e inversa.



5. El servidor esclavo será venus.sistema.test y tendrá como maestro a tierra.sistema.test.



6. El tiempo en caché de las respuestas negativas de las zonas (directa e inversa) será de dos horas (se pone en segundos).



7. Aquellas consultas que reciba el servidor para la que no está autorizado, deberá reenviarlas (forward) al servidor DNS 208.67.222.222 (OpenDNS).



8. Se configurarán los siguientes alias:
    a. ns1.sistema.test. será un alias de tierra.sistema.test.



    b. ns2.sistema.test. será un alias de venus.sistema.test.



9. mail.sistema.test. será un alias de marte.sistema.test.



10. El equipo marte.sistema.test. actuará como servidor de correo del dominio de correo sistema.test.



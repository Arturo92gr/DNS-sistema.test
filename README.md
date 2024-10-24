# DNS_sistema.test
 Práctica: sistema.test master y slave

**1. Activa solamente la escucha del servidor para el protocolo IPv4.**  
    En el servidor Tierra:  
        ```
        cd /etc/default  
        sudo nano named  
        ```  
        Se cambia la línea: `OPTIONS="-u bind -4"`   
    Se copia el archivo named a la capeta compartida para añadirlo a la provisión:  
        En máquina: `cp named /etc/files/tierra/`  
        En provisión: `cp -v /files/tierra/named /etc/default`  

**2. Establecer la opción dnssec-validation a yes**  
    En el servidor Tierra:  
    `sudo nano /etc/bind/named.conf.options`  
    Se modifican las siguientes líneas:  
        ```
        dnssec-validation yes;  
        //listen-on-v6 { any; };  
        ```  

**3. Los servidores permitirán las consultas recursivas sólo a los ordenadores en la red 127.0.0.0/8 y en la red 192.168.57.0/24, para ello utilizarán la opción de listas de control de acceso o acl.**  
    En el servidor Tierra:  
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
        // Para tierra:
        listen-on port 53 { 192.168.57.103; };
        // Para venus:
        // listen-on port 53 { 192.168.57.102; };
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

    Para comprobar si es correcta la configuración:  
    `named-checkconf /etc/bind/named.conf.options`  
    Por último, se reinicia el servidor y se comprueba su estado:  
    ```
    systemctl restart named  
    systemctl status named  
    ```
    Se copia el archivo a la carpeta compartida y se añade a la provisión  
        En máquina: `cp /etc/bind/named.conf.options /etc/files/tierra/`  
        En provisión: `cp -v /files/tierra/named.conf.options /etc/bind`

**4. El servidor maestro será tierra.sistema.test y tendrá autoridad sobre la zona directa e inversa.**  
    En el servidor Tierra:  
    Primeramente, realizar una copia de seguridad: `sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.backup`  
    Se edita el archivo: `sudo nano /etc/bind/named.conf.local` para añadir ambas zonas.  
    Se indica el nombre de la zona, el tipo y el archivo:  
    ```
    //  
    // Do any local configuration here  
    //  
    // Consider adding the 1918 zones here, if they are not used in your  
    // organization  
    //include "/etc/bind/zones.rfc1918";
    zone "solarsystem.es" {  
        type master;  
        file "/var/lib/bind/solarsystem.es.dns";  
    };
    zone "57.168.192.in-addr-arpa" {  
        type master;  
        file "/var/lib/bind/solarsystem.es.rev";  
    };  
    ```
    Para la zona directa se crea el archivo con el nombre indicado anteriormente en el siguiente directorio: `sudo nano /var/lib/bind/solarsystem.es.dns`  
    Se edita el archivo de la siguiente forma:
    ```
    ;
    ; solarsystem.es
    ;
    $TTL 86400
    @ IN SOA debian.solarsystem.es. admin.solarsystem.es. (
        2024102401  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400 )     ; Negative Cache TTL
    ;
    @                           IN      NS      debian.solarsystem.es.
    debian.solarsystem.es.      IN      A       192.168.57.103
    ```
    Comprobación: `named-checkzone solarsystem.es /var/lib/bind/solarsystem.es.dns`
    Para la zona inversa se crea el archivo con el nombre indicado anteriormente en el siguiente directorio: `sudo nano /var/lib/bind/solarsystem.es.rev`  
    Se edita el archivo de la siguiente forma:
    ```
    ;  
    ; 57.168.192  
    ;  
    $TTL 86400  
    @ IN SOA debian.solarsystem.es. admin.solarsystem.es. (  
        2024102401  ; Serial  
        3600        ; Refresh  
        1800        ; Retry  
        604800      ; Expire  
        86400 )     ; Negative Cache TTL  
    ;  
    @       IN      NS      debian.solarsystem.es.  
    103     IN      PTR     debian.solarsystem.es.  
    ```
    Comprobación: `sudo named-checkzone 57.168.192.in-addr.arpa /var/lib/bind/solarsystem.es.rev`  
    Reinicio del servidor DNS: `sudo systemctl restart bind9`  
    Se copia los archivos a la carpeta compartida y se añaden a la provisión  
        En máquina:  
        ```
        cp /etc/bind/named.conf.local /etc/files/tierra/  
        cp /var/lib/bind/solarsystem.es.dns /etc/files/tierra/  
        cp /var/lib/bind/solarsystem.es.rev /etc/files/tierra/
        ```  
        En provisión:  
        ```
        cp -v /files/tierra/named.conf.local /etc/bind  
        cp -v /files/tierra/solarsystem.es.dns /var/lib  
        cp -v /files/tierra/solarsystem.es.rev /var/lib
        ```

**5. El servidor esclavo será venus.sistema.test y tendrá como maestro a tierra.sistema.test.**  
    En el servidor Venus:  
    Se edita el archivo: `sudo nano /etc/bind/named.conf.local`  
    Se indica el nombre de la zona, el tipo, servidor maestro y el archivo:  
    ```
    //
    // Do any local configuration here
    //
    // Consider adding the 1918 zones here, if they are not used in your
    // organization
    //include "/etc/bind/zones.rfc1918";
    zone "solarsystem.es" {
        type slave;
        masters { 192.168.57.103; };
        file "/var/lib/bind/solarsystem.es.dns";
    };
    zone "57.168.192.in-addr-arpa" {
        type slave;
        masters { 192.168.57.103; };
        file "/var/lib/bind/solarsystem.es.rev";
    };
    ```  
    `sudo systemctl restart bind9`  
    En el servidor Tierra:  
    `sudo nano /etc/bind/named.conf.local`
    Se añade el servidor esclavo y se activa la notificación de cambios:
    ```
    //
    // Do any local configuration here
    //  
    // Consider adding the 1918 zones here, if they are not used in your
    // organization
    //include "/etc/bind/zones.rfc1918";
    zone "solarsystem.es" {
        type master;
        file "/var/lib/bind/solarsystem.es.dns";  
        allow-transfer { 192.168.57.102; };
        notify yes;
    };   
    zone "57.168.192.in-addr-arpa" {
        type master;
        file "/var/lib/bind/solarsystem.es.rev";
    };
    ```
    Se añade otro servidor DNS para la zona: `sudo nano /var/lib/bind/solarsystem.es.dns`  
    ```
    ;                                                                                              ;
    ; solarsystem.es
    ;
    $TTL 86400
    @ IN SOA debian.solarsystem.es. admin.solarsystem.es. (
            2024102401  ; Serial
            3600        ; Refresh
            1800        ; Retry
            604800      ; Expire
            86400 )     ; Negative Cache TTL
    ;
    @                               IN      NS      debian.solarsystem.es.
    debian.solarsystem.es.          IN      A       192.168.57.103
    solarsystem.es                  IN      NS      dns1.solarsystem.es
    dns1.solarsystem.es             IN      A       192.168.57.102
    ```
    `sudo systemctl restart bind9`  
    Se copia los archivos a la carpeta compartida y se añaden a la provisión  
        En máquina tierra:  
        ```
        cp /etc/bind/named.conf.local /etc/files/tierra/  
        cp /var/lib/bind/solarsystem.es.dns /etc/files/tierra/  
        ```  
        En máquina venus:  
        ``` 
        cp /etc/bind/named.conf.local /etc/files/venus/  
        ```  
        En provisión venus:  
        ```
        cp -v /files/venus/named.conf.local /etc/bind  
        ```

**6. El tiempo en caché de las respuestas negativas de las zonas (directa e inversa) será de dos horas (se pone en segundos).**  
    En Tierra se editan los archivos `solarsystem.es.dns` y `solarsystem.es.rev` en la siguiente línea:  
        `7200 )      ; Negative Cache TTL`  
    Se reinicia bind para aplicar cambios: `sudo systemctl restart bind9`  
    Se copian los archivos a la carpeta compartida:
        ```
        cp /var/lib/bind/solarsystem.es.dns /etc/files/tierra/  
        cp /var/lib/bind/solarsystem.es.rev /etc/files/tierra/  
        ```  

**7. Aquellas consultas que reciba el servidor para la que no está autorizado, deberá reenviarlas (forward) al servidor DNS 208.67.222.222 (OpenDNS).**  
    En Tierra:
        `sudo nano /etc/bind/named.conf.options`  
    Se descomentan y editan las siguiente líneas:
        ```
        forwarders {
            208.67.222.222;
        };
        ```  
    Se copia el archivo a la carpeta compartida:
        `cp /etc/bind/named.conf.options /etc/files/tierra/`  

**8. Se configurarán los siguientes alias:**  
    **a. ns1.sistema.test. será un alias de tierra.sistema.test.**  
    `sudo nano /var/lib/bind/solarsystem.es.dns`
    Se incrementa el número de serie para notificar la acutalización y se añade el alias:
    ```
    ;
    ; solarsystem.es
    ;
    $TTL 86400
    @ IN SOA debian.solarsystem.es. admin.solarsystem.es. (
        2024102402  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        7200 )     ; Negative Cache TTL
    ;
    @                               IN      NS      debian.solarsystem.es.
    debian.solarsystem.es.          IN      A       192.168.57.103
    solarsystem.es                  IN      NS      dns1.solarsystem.es
    dns1.solarsystem.es             IN      A       192.168.57.102
    ns1.sistema.test                IN      CNAME   debian.solarsystem.es.
    ```
    **b. ns2.sistema.test. será un alias de venus.sistema.test.**  
    `sudo nano /var/lib/bind/solarsystem.es.dns`
    Se incrementa el número de serie para notificar la acutalización y se añade el alias:
    ```
    ;
    ; solarsystem.es
    ;
    $TTL 86400
    @ IN SOA debian.solarsystem.es. admin.solarsystem.es. (
        2024102403  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        7200 )     ; Negative Cache TTL
    ;
    @                               IN      NS      debian.solarsystem.es.
    debian.solarsystem.es.          IN      A       192.168.57.103
    solarsystem.es                  IN      NS      dns1.solarsystem.es
    dns1.solarsystem.es             IN      A       192.168.57.102
    ns1.sistema.test                IN      CNAME   debian.solarsystem.es.
    ns2.sistema.test                IN      CNAME   dns1.solarsystem.es.  
    ```
    Se reinicia el servicio:
        `sudo systemctl restart bind9`
    Se copia el archivo a la carpeta compartida:
        `cp /var/lib/bind/solarsystem.es.dns /etc/files/tierra/`

**9. mail.sistema.test. será un alias de marte.sistema.test.**  
    `sudo nano /var/lib/bind/solarsystem.es.dns`
    Se incrementa el número de serie para notificar la acutalización y se añade el registro A y CNAME para marte:
    ```
    ;
    ; solarsystem.es
    ;
    $TTL 86400
    @ IN SOA debian.solarsystem.es. admin.solarsystem.es. (
        2024102404  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        7200 )     ; Negative Cache TTL
    ;
    @                               IN      NS      debian.solarsystem.es.
    debian.solarsystem.es.          IN      A       192.168.57.103
    solarsystem.es                  IN      NS      dns1.solarsystem.es
    dns1.solarsystem.es             IN      A       192.168.57.102
    ns1.sistema.test                IN      CNAME   debian.solarsystem.es.
    ns2.sistema.test                IN      CNAME   dns1.solarsystem.es.  
    marte.sistema.test              IN      A       192.168.57.104
    mail.sistema.test               IN      CNAME   marte.sistema.test
    ```
    Se reinicia el servicio:
        `sudo systemctl restart bind9`
    Se copia el archivo a la carpeta compartida:
        `cp /var/lib/bind/solarsystem.es.dns /etc/files/tierra/`

**10. El equipo marte.sistema.test. actuará como servidor de correo del dominio de correo sistema.test.**  
    
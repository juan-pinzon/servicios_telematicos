## DNS Maestro y esclavo
### Configuración DNS Maestro
Se modifica el archivo _/etc/named.conf_ colocando la ip del mismo servidor3 (192.168.60.5) , la del servidor2 (192.168.60.4) y la ip raíz 192.168.60.0.

Además, se configura la zona para el dominio amazonas.com de tipo master en /etc/named.rfc1912.zones
```bash
zone "amazonas.com" IN {
	type master;
	file "amazonas.com.zone";
};
```
Luego se configura  el archivo de la zona en cuestión el cual será /var/named/amazonas.com.zone
```bash
$ORIGIN amazonas.com.
$TTL 86400
@  IN  SOA  amazonas.com. servidor3.amazonas.com. (
		123312  ; serial
		1D  ; refresh
		1H  ; retry
		1W  ; expire
		3H )  ; minimum
@  IN  NS  servidor3.amazonas.com.
@  IN  MX  10 servidor3.amazonas.com.

amazonas.com.  IN  A  192.168.60.5
servidor3  IN  A  192.168.60.5
www  IN  CNAME  servidor3
servidor2  IN  A  192.168.60.4
```
Se pueden hacer las comprobaciones necesarias para saber que todo está ok con los comandos
_named-checkconf_
_named-checkzone_
Si todo está ok entonces se procede a encender o restaurar el servicio de _named_
_service named start|restart_

### Configuración DNS esclavo
Se modifica el archivo _/etc/named.conf_ colocando en el aceptación {any;} y en el allow-update igualmente.
Para la configuración de /etc/named.rfc1912.zones se coloca:
```bash
zone "amazonas.com" IN {
	type slave;
	file "slaves/amazonas.com.zone";
	masters {192.168.60.5;};
};
```
Donde en masters se coloca la ip del maestro. Se debe reiniciar entonces el servicio de named. Se debe verificar que en la ruta /var/named/slaves/ se haya sincronizado el archivo amazonas.com.zone. Ya con esto está funcionando el DNS maestro y esclavo.

[Configuración detallada del servidor DNS maestro-esclavo - programador clic (programmerclick.com)](https://programmerclick.com/article/2888736057/)

[FTP en CentOS 7 II: Seguridad del servicio – Luigi Guarino (wordpress.com)](https://luigiasir.wordpress.com/2017/11/02/ftp-en-centos-7-ii-seguridad-del-servicio/)

[5.9. Port Forwarding Red Hat Enterprise Linux 7 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-port_forwarding)

## FTP seguro
Para esto se deben crear las llaves pública y privada, luego en la configuración vsftpd se coloca la siguiente configuración:
**ssl_enable=YES :** Directiva que habilita el uso de SSL  
**allow_anon_ssl=YES :** Habilitar SSL con usuarios anónimos  
**force_local_data_ssl=YES :** Obliga a usar certificado SSL para traferir datos.  
**force_local_logins_ssl=YES:** Obliga a usar certificado SSL para autentificar usuarios locales.  
**ssl_tlsv1=YES :** Habilita el uso de SSLv1  
**ssl_sslv2=YES:** Habilita el uso de SSLv1  
**ssl_sslv3=YES:** Habilita el uso de SSLv1  
**rsa_cert_file=/etc/vsftpd/vsftpd.pem:** Ubicación del certificado SSL  
**rsa_private_key_file=/etc/vsftpd/vsftpd.pem:** Ubicación de la llave privada SSL
**pasv_min_port=10090**
**pasv_max_port=10100**

## Aprovisionamiento
Para aprovisionar la máquina mientras se crea debemos tener algo como lo siguiente:
```text
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure("2") do |config|
	config.vm.define :firewall do |firewall|
		firewall.vm.box = "bento/centos-7.9"
		firewall.vm.network :private_network, ip: "209.191.200.3"
		firewall.vm.network :private_network, ip: "192.168.60.3"
		firewall.vm.hostname = "firewall"
		firewall.vm.provision "shell", inline:<<-SHELL
		  service NetworkManager stop
		  chkconfig NetworkManager off
		  echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
		  service firewalld start
		  chkconfig firewalld on
		SHELL
	end
	
	config.vm.define :servidor2 do |servidor2|
		servidor2.vm.box = "bento/centos-7.9"
		servidor2.vm.network :private_network, ip: "192.168.60.4"
		servidor2.vm.hostname = "servidor2"
		servidor2.vm.provision "shell", inline:<<-SHELL
		  yum install -y bind-utils bind-libs bind-*
		  yum install -y vsftpd
		SHELL
	end

	config.vm.define :servidor3 do |servidor3|
		servidor3.vm.box = "bento/centos-7.9"
		servidor3.vm.network :private_network, ip: "192.168.60.5"
		servidor3.vm.hostname = "servidor3"
		servidor3.vm.provision "shell", inline:<<-SHELL
		  yum install -y bind-utils bind-libs bind-*
		SHELL
	end
end
```
En donde en la configuración de provision se colacan los comandos necesarios

## Rich rules
Utilizando Rich rules para acivar el puerto 80 para el servicio httpd se usa lo siguiente:
```bash
firewall-cmd --add-rich-rule='rule family="ipv4" source address="209.191.200.3" port port=80 protocol=tcp accept'
```

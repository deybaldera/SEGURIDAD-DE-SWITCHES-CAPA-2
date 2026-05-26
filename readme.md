# SEGURIDAD DE SWITCHES CAPA 2
switchport mode access -> Este comando hace que la interfaz o puerto no transporte varias vlan, solo trabajará como un puerto normal de acceso. 
Si el comando anterior lo complementamos con un switchport port-security evitamos ataques de MAC flooding, con este comando podemos controlar que dispositivos pueden conectarse a ese puerto en especifico. 

```
Switch(config)# interface fa0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport port-security
````

Algo muy importante que debemos recordar es que el comando switchport port-security solo se puede usar cuando el puerto esta en mode Access. 

Comando de verificación: 
```Switch# show port-security interface ```
Muestra la configuración actual de seguridad del puerto o interfaz.

Mas comandos: 

Significa que el puerto solo aceptara hasta 3 direcciones MAC diferentes en este puerto.
----------------------------------------------------------------------------------------
```
Switch(config-if)# switchport port-security maximum 3
```
Aprende automáticamente las MAC conectadas y las guarda como permitidas.
----------------------------------------------------------------------------------------
```
Switch(config-if)# switchport port-security mac-address sticky
```
La MAC segura expirará después de 10 minutos.
----------------------------------------------------------------------------------------
```
S1(config-if)# switchport port-security aging time 10
```
La MAC expirará solo si NO hay actividad, no hay trafico.
----------------------------------------------------------------------------------------
```
S1(config-if)# switchport port-security aging type inactivity
```

Este grupo de comandos es muy útil en lugares donde cambia dispositivos frecuentemente. Así no se tiene que borrar la MAC manualmente. 
```
Switch(config-if)# switchport port-security violation {protect | restrict | shutdown}
Tenemos 3 combinaciones de comandos
````
Este sirve para elegir qué acción tomará el switch cuando ocurra una violación de PortSecurity 
```
- switchport port-security violation protect
/MAC invalida = bloqueada silenciosamente

- switchport port-security violation restrict
/MAC invalida = bloqueada + registrada 

- switchport port-security violation shutdown
/Violación = puerto apagado/

```

# ATAQUE DE SALTO DE VLAN 
Un ataque de salto ocurre cuando un usuario mal intencionado, conectado a un puerto del switch se las ingenia para saltarse esa barda y meter tráfico directamente en otra vlan privada, espiando datos o atacando servidores ocultos. 
1. Suplantación de Enlace Troncal (Trunk Spoofing) 
--------------------------------------------------
En este método el atacante se aprovecha de la configuración por defecto de los switches Cisco y un protocolo llamado DTP (Dynamic Trunking Protocol). 
- Los puertos de un switch por defecto están en modo dinamico automatico, entonces si se conecta un host este puerto actúa como puerto de acceso pero si se conecta con un switch actúa como enlace troncal (trunk) y todos sabemos que este enlace trunk lo que hace es trasportar el tráfico de todas las VLANs. El atacante conecta su laptop y usando una herramienta de software especializado envía mensajes DTP falsos simulando ser otro switch que quiere armar un enlace troncal, el switch se traga el engaño y convierte el puerto del atacante en trunk, de esta manera el atacante tiene acceso a todas las VLANs de la red. 
2. Doble Etiquetado (Doublé Tagging)
------------------------------------
Este método es diferente, mas bien es mas astuto, se aprovecha de como funciona la VLAN Nativa que por defecto es la VLAN 1.
Hablemos un poco de teoría, sabemos que cuando los switches mandan información entre VLANs llevan una etiqueta para identificarse de que Vlan vienen (Tag 802.1Q), sin embargo el trafico de la Vlan nativa viaja sin etiqueta. 
- El atacante que esta en la Vlan 1 envia un paquete de de doble etiqueta a un switch, oculta la segunda etiqueta (objetivo) y coloca como presentación la primera que es la Vlan 1, el primer switch que recibe este paquete de confunde, ve la vlan 1 como es la nativa le quita esa etiqueta para mandar el paquete por el enlace troncal al siguiente Switch. Al quitarle la primera etiqueta queda al descubierto la segunda etiqueta (vlan10), cuando el segundo switch recibe el paquete solo ve vlan10, así que lo entrega feliz mente a las computadoras de la VLAN 10. El atacante ha saltado de la 1 a la 10. 
Nota. Este ataque es de "un solo sentido" (el atacante puede enviar datos, pero las respuestas no regresarán a él por el mismo camino), pero es ideal para lanzar ataques de denegación de servicio (DoS) o infectar servidores.

## COMANDOS
```
S1(config)# interface range fa0/1 - 16
S1(config-if-range)# switchport mode Access
```
- Evita el ataque de Trunk Spoofing. Si un atacante conecta su laptop a cualquiera de estos puertos e intenta negociar un enlace troncal usando DTP, el switch lo ignorará por completo. El puerto nació para ser de acceso y morirá siendo de acceso.


```
S1(config)# interface range fa0/17 - 20
S1(config-if-range)# switchport mode access
S1(config-if-range)# switchport access vlan 1000
S1(config-if-range)# shutdown
```
- Si un atacante se cuela físicamente en la oficina y conecta un cable a la pared en un puerto que estaba libre, no logrará nada. El puerto está apagado de raíz, y si por algún error de administración se encendiera, el atacante caería en la "zona muerta" (VLAN 1000) sin comunicación con el resto de la empresa.



```
S1(config)# interface range fa0/21 - 24
S1(config-if-range)# switchport mode trunk
S1(config-if-range)# switchport nonegotiate
S1(config-if-range)# switchport trunk native vlan 999
```
- Switchport mode trunk: Configura el puerto de forma estática como un enlace troncal. Ya no depende de que DTP "adivine" qué hay del otro lado.

- Switchport nonegotiate: Apaga por completo el protocolo DTP en este enlace. El switch deja de enviar mensajes automáticos intentando negociar. Esto previene que software malicioso intente interactuar con el protocolo.

- Switchport trunk native vlan 999: Modifica la VLAN nativa. Por defecto en todos los switches del mundo es la VLAN 1. Al cambiarla a la VLAN 999 (otra VLAN ficticia y sin uso), se neutraliza por completo el ataque de Doble Etiquetado (Double Tagging), ya que el switch no procesará las etiquetas externas diseñadas para la VLAN por defecto.

---
title: 1. Command and Controls over DNS 🏷️
layout: default
has_toc: false
nav_order: 1
parent: Hunting in Network Environments 🥋 

---

## Command and Controls over DNS 🏷️
---

Vamos a ver un ejemplo de detección de la táctica de Comando y control al DNS.

{: .detail }
>Para recrear la simulación he utilizado el índice de packetbeat-* para buscar posibles C2 a través de DNS el 3 de julio de 2023. Además, del índice winlogbeat-* para correlacionar las consultas DNS y así identificar el proceso malicioso que lo genera. 

El C2 a través de DNS, o más precisamente Comando y Control a través de DNS, es una técnica utilizada por los adversarios donde se utilizan los protocolos DNS para establecer un canal de Comando y Control. En esta técnica, los adversarios pueden disfrazar sus comunicaciones de C2 como consultas y respuestas DNS típicas, eludiendo las medidas de seguridad de la red. Dado esto, buscaremos patrones de consulta DNS inusuales basados en lo siguiente:

- Alto recuento de subdominios únicos.
- Solicitudes DNS inusuales basadas en tipos de consulta (MX, CNAME, TXT).

Vamos a realizar una consulta KQL para listar todas las consultas DNS, exluyendo la busqueda de DNS inversa:

**`network.protocol: dns Y NO [dns.question.name](http://dns.question.name/): *arpa`**


![ELK-1.png](https://i.postimg.cc/c4yYMKWQ/ELK-1.png)



Se puede observar que un dominio inusual **(golge[.]xyz)** consultó 2191 subdominios únicos, lo que puede indicar una posible actividad de C2 a través de DNS proveniente de **WKSTN-1**. 

Veamos más en profundidad el ataque: Haremos una consulta sobre el dominio y el host comprometido.

**`network.protocol: dns Y NO [dns.question.name](http://dns.question.name/): *arpa Y dns.question.registered_domain: golge.xyz Y [host.name](http://host.name/): WKSTN-1`**

![ELK-2.png](https://i.postimg.cc/mZFCHYvL/ELK-2.png)


Se pueden observar varias caracteristicas de actividad sospehosa:

{: .importance }
>- Utilización de subdominios hexadecimales.
>- Consultas TXT y MX inusuales para el dominio golge.xyz.
>- Consultas CNAME que podrían ser utilizadas para enmascarar la verdadera dirección de los servidores involucrados.
>- La presencia de múltiples consultas a este dominio inusual en diferentes tipos de registros DNS en un corto periodo de tiempo
>- La omisión de nombres de dominio inversos.


![ELK-3.png](https://i.postimg.cc/yd2RGxZC/ELK-3.png)

Ahora que tenemos suficiente información, podemos correlacionar esta actividad en winlogbeat-* para identificar el proceso que ejecuta las solicitudes DNS utilizando la siguiente consulta KQL:

![ELK-4.png](https://i.postimg.cc/T1mbvxVL/ELK-4.png)


Basado en los resultados, parece que todas las conexiones a 167[.]71[.]198[.]43:53 son generadas por nslookup.exe. 

Para continuar con la correlación de eventos, utilicemos "View surrounding documents" para ver los eventos posteriores relacionados con esta actividad.


![ELK-5.png](https://i.postimg.cc/02YSt85h/ELK-5.png)

Con ello podemos confirmar la sospecha de C2 a través de DNS. S

iguiendo el enfoque de un cazador de amenazas, el siguiente paso de esta investigación es identificar los eventos generados por el proceso principal de nslookup.exe que estableció el C2 a través de DNS. Esto permitirá rastrear los eventos antes de que se estableciera una conexión C2 exitosa. Además, se observarán los comandos posteriores ejecutados por el proceso principal, ya que se espera que se ejecuten comandos remotos una vez que se haya confirmado que la conexión C2 está activa.

![ELK-6.png](https://i.postimg.cc/XvPMT670/ELK-6.png)


{: .importance }
> También se puede identificar un tráfico DNS inusual con el campo de network.bytes. Las consultas DNS suelen ser cortas,, pero como hemos visto anteriorme, el subdominio se utilizó para manejar una cadena hexadecimal larga para la conexión C2. Por ello, podemos utilizar también el tamaño de solicitud/respuesta para determinar posibles anomalías dentro del tráfico DNS.

---
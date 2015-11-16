# Session 1 - APACHE FLUME

## Características:

  -  50 data-center
  -  Varios archivos de logs /var/log/operaciones.log por cada data-center
  -  Se rota diariamente hacia operaciones-<fecha>.log
  -  Log Expresion: [TIMESTAMP] [TIPO_OPERACION] [DATOS_OPERACION]
  -  Tipo Operacion: VENTA|PEDIDO
  -  Tamaño Maximo fichero: 300 Bytes x 60 x 24
  -  Máquinas fr-central-1 y fr-central-2 para recolectar hacia un HDFS central.
  -  Los puertos 10000 a 10100 abiertos
  -  La URI del HDFS es hdfs://fr-hdfs:9000.
  -  Separada primero por tipo de operación y luego por fecha del evento.
  -  El sistema deberá tener en cuenta que el sistema HDFS puede estar parado durante un periodo máximo de tres horas.
  -  El sistema tiene que implementar un mecanismo de failover en la comunicación desde las sedes a las máquinas recolectoras.

## Arquitectura Utilizada

### Server Agent

Este agente se encarga de la captura y envío del log hacia las máquinas recolectoras. He utilizado un 'source' de tipo 'spooldir' para que el evento se produzca cada vez que se cree un nuevo archivo de log.
Este agente tiene un solo canal de tipo 'file' para que el sistema sea fiable en caso de una caida de máximo 3 horas ya que durante ese tiempo se producirían 10.800 eventos y es demasiada carga para un canal en memoria.
La clave de este agente se basa en la agrupación de 'sinks' en modo balanceador para además de aligerar el proceso de recepción de logs en las máquinas recolectoras, crear un sistema tolerante a fallos. He decidido utilizar una agrupación de tipo 'load_balance' en vez de 'fail_tolerance' porque tal y como explica la documentación de flume, configurando un 'backoff' y 'maxTimeOut' obtenemos un sistema tolerante a fallos y optimizado de manera que utilizamos ambas máquinas recolectoras en vez de solo una si hubiesemos configurado un tipo 'fail_tolerance'
De esta manera enviamos (avro_sink) siguiendo el protocolo round_robin a ambas máquinas (fr-central-1:10000, fr-central-2:10001) y si alguna de ellas falla se envía a la otra de manera que el sistema no se detendría.


### Fr-Central Agent

Este agente se lanza en las máquinas recolectoras. Su configuración se basa en un source de tipo 'avro_source' que recoge los eventos enviados por las máquinas servidoras, un interceptor extrae la información necesaria de ellos y un 'sink' de tipo 'hdfs' los almacena seguidamente en el sistema siguiendo la ruta especificada.
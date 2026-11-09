# MQTT: hola mundo con broker, publicador, dos suscriptores y QoS

Demostracion del estilo **MQTT** (Message Queuing Telemetry Transport) para la exposicion de estilos arquitectonicos de API.

 Autores:
- Jhojan Stiven Aragón Ramírez
- Yader Ibraldo Quiroga Torres
 - Kevin Emmanuel Tovar Lizarazo
## Que es MQTT (resumen)

- **Modelo de comunicacion:** publicacion y suscripcion. Los clientes nunca hablan entre si; todos hablan con un **broker** (intermediario) que reenvia cada mensaje a quien este suscrito al **topico**.
- **Transporte:** TCP (puerto 1883, o 8883 con TLS). Tambien puede ir sobre WebSocket.
- **Formato de datos:** el broker no interpreta el contenido (payload): son bytes. En esta demo usamos JSON.
- **Calidad de servicio (QoS):** 0 = como maximo una vez, 1 = al menos una vez, 2 = exactamente una vez.

## Que hace la demo

```
publisher.py --PUBLISH saludos/hola-------------->  +-----------------+ --saludos/#--------------> subscriber.py (sub-saludos)
             --PUBLISH sensores/sala/temperatura->  | Mosquitto :1883 | --sensores/+/temperatura-> subscriber.py (sub-sensores)
                                                    +-----------------+
```

1. Un **broker** Mosquitto en Docker.
2. Un **publicador** que envia un saludo a `saludos/hola` y una lectura de temperatura a `sensores/sala/temperatura`.
3. **Dos suscriptores** a topicos distintos (con comodines `#` y `+`): cada uno recibe solo lo suyo.
4. El **efecto de QoS**:
   - En el log del broker se ve el intercambio de paquetes de cada nivel.
   - Con una sesion persistente, si el suscriptor esta apagado, el broker **guarda** los mensajes QoS 1/2 y **descarta** los QoS 0.
5. Extra: un **mensaje retenido**, que un suscriptor nuevo recibe apenas se conecta.



## Requisitos previos

- Docker con Docker Compose (para el broker; no hay que instalar Mosquitto).
- Python 3.10 o superior.
- Opcional: [MQTTX](https://mqttx.app/) como cliente grafico.

## Instalacion

```bash
git clone https://github.com/git-init-ud/mqtt-hola-mundo.git
cd mqtt-hola-mundo
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env             # Windows: copy .env.example .env
```

Dependencias:

| Paquete | Para que sirve |
| --- | --- |
| `paho-mqtt` | Cliente MQTT oficial de Eclipse; conecta, publica y se suscribe al broker. |
| `python-dotenv` | Carga las variables del archivo `.env`, para no dejar direcciones ni puertos fijos en el codigo. |

## Variables de entorno

| Variable | Valor por defecto | Descripcion |
| --- | --- | --- |
| `MQTT_HOST` | `localhost` | Direccion del broker |
| `MQTT_PORT` | `1883` | Puerto del broker (tambien lo usa `docker-compose.yml`) |
| `TOPIC_GREETING` | `saludos/hola` | Topico del saludo |
| `TOPIC_SENSOR` | `sensores/sala/temperatura` | Topico de la lectura del sensor |

## Ejecucion (4 terminales)

```bash
# Terminal 1: broker y su log
docker compose up -d
docker compose logs -f mosquitto

# Terminal 2: suscriptor de saludos
python subscriber.py --id sub-saludos --topic "saludos/#"

# Terminal 3: suscriptor de sensores
python subscriber.py --id sub-sensores --topic "sensores/+/temperatura"

# Terminal 4: publicador
python publisher.py
```

Resultado esperado: `sub-saludos` imprime solo el saludo y `sub-sensores` solo la temperatura.

Opciones del publicador: `--qos 0|1|2`, `--burst N`, `--retain`, `--topic saludo|sensor|ambos`, `--message "texto"`.

## Demo de QoS

### A. Intercambio de paquetes por nivel

```bash
python publisher.py --topic saludo --qos 0
python publisher.py --topic saludo --qos 1
python publisher.py --topic saludo --qos 2
```

En el log del broker (terminal 1):

| QoS | Paquetes entre publicador y broker | Garantia |
| --- | --- | --- |
| 0 | `PUBLISH` | Como maximo una vez: se envia y se olvida |
| 1 | `PUBLISH` -> `PUBACK` | Al menos una vez: puede llegar duplicado |
| 2 | `PUBLISH` -> `PUBREC` -> `PUBREL` -> `PUBCOMP` | Exactamente una vez |

Detalle importante: el QoS se negocia por tramo. Si el publicador envia con QoS 2 pero el suscriptor se suscribio con QoS 1, recibe el mensaje con `qos=1`. El QoS efectivo es el minimo entre los dos.

### B. Que pasa si el suscriptor esta apagado

```bash
# 1. Suscriptor con sesion persistente; luego detenerlo con Ctrl+C
python subscriber.py --id sub-persistente --topic "saludos/#" --qos 2 --persistent

# 2. Con el suscriptor apagado
python publisher.py --topic saludo --burst 5 --qos 0 --message "QoS0"
python publisher.py --topic saludo --burst 5 --qos 1 --message "QoS1"

# 3. Volver a encenderlo
python subscriber.py --id sub-persistente --topic "saludos/#" --qos 2 --persistent
```

Llegan **solo los 5 mensajes "QoS1"**, y el suscriptor muestra `sesion previa recuperada: True`. Los QoS 0 se perdieron porque el broker no los encola.

### C. Mensaje retenido

```bash
python publisher.py --topic saludo --retain --message "Ultimo saludo"
python subscriber.py --id sub-nuevo --topic "saludos/#"     # lo recibe al instante con retain=True
```

## Presentación

- [Ver la presentación en Canva](https://canva.link/fo95adq8urtj65f)
- [Descargar la presentación en PDF](./Slides/Protocolo%20MQTT.pdf)

## Fuentes

1. HiveMQ Team. (2026, 9 de febrero). *[Introducing the MQTT Protocol -
   MQTT Essentials: Part 1](https://www.hivemq.com/blog/mqtt-essentials-part-1-introducing-mqtt/)*.
   HiveMQ Blog.
2. HiveMQ Team. (2026, 9 de febrero). *[What is MQTT Quality of Service (QoS)
   0, 1, & 2? - MQTT Essentials: Part 6](https://www.hivemq.com/blog/mqtt-essentials-part-6-mqtt-quality-of-service-levels/)*.
   HiveMQ Blog.
3. Eclipse Foundation. (2024). *[Client Module - Eclipse Paho MQTT Python
   Documentation](https://eclipse.dev/paho/files/paho.mqtt.python/html/client.html)*.
4. EMQ Technologies. (2026). *[MQTTX Documentation](https://mqttx.app/docs)*.
5. OASIS. (2019). *[MQTT Version 5.0](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)*.
6. AWS. (s. f.). *[Use AWS IoT Core MQTT broker with standard MQTT
   libraries](https://aws.amazon.com/blogs/iot/use-aws-iot-core-mqtt-broker-with-standard-mqtt-libraries/)*.
7. Anthropic. (s. f.). *[Claude](https://claude.ai/)*. Herramienta utilizada como
   apoyo para la redacción y revisión del material.

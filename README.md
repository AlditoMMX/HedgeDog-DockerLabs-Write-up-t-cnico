# HedgeDog-DockerLabs-Write-up-t-cnico
Análisis de superficie de ataque, acceso inicial y escalada de privilegios en Linux

## 1. Introducción

En este laboratorio de DockerLabs se analiza la máquina **HedgeDog**, con el objetivo de identificar su superficie de ataque, obtener acceso inicial al sistema y evaluar las posibilidades de escalada de privilegios.

El proceso se desarrolla mediante una metodología progresiva de reconocimiento, enumeración de servicios, análisis de configuraciones locales y validación de permisos administrativos.

La resolución pone de manifiesto dos debilidades principales: la exposición de información que facilita la identificación de usuarios y una configuración excesivamente permisiva de sudo, que permite encadenar cambios de identidad hasta alcanzar privilegios de root.

### Objetivos del análisis:

  - Identificar los servicios expuestos y recopilar información relevante.
  - Validar el acceso inicial mediante SSH en el entorno de laboratorio.
  - Examinar usuarios, directorios y permisos locales.
  - Identificar y demostrar una cadena de escalada de privilegios.
  - Documentar los hallazgos y proponer medidas correctivas.

## 2. Alcance y metodología
El análisis se realizó sobre la máquina vulnerable proporcionada por DockerLabs.

| Parámetro | Descripción |
|-----------|-------------|
| Objetivo              | 172.17.0.2                  |
| Servicios de interés  | SSH y HTTP                  |
| Puerto SSH            | 22/tcp                      |
| Puerto HTTP           | 80/tcp                      |
| Objetivo final        | Obtener privilegios de root |
| Técnicas principales  | Reconocimiento, enumeración y escalada de privilegios  |

Todas las pruebas descritas corresponden al entorno de laboratorio autorizado.

## 3. Resumen ejecutivo

El reconocimiento inicial permitió identificar dos servicios accesibles: SSH en el puerto 22 y HTTP en el puerto 80.

Durante la enumeración web se encontró la cadena tails, que proporcionó una pista para identificar un usuario del sistema. Posteriormente, las pruebas de autenticación SSH permitieron obtener acceso inicial a dicha cuenta dentro del laboratorio.

Una vez establecida la sesión, la revisión de los permisos de sudo reveló una configuración que permitía a tails ejecutar comandos como sonic sin contraseña. A su vez, sonic disponía de permisos para ejecutar comandos como cualquier usuario, incluido root.

La cadena de explotación quedó establecida de la siguiente manera:

<img width="2027" height="776" alt="image" src="https://github.com/user-attachments/assets/0ac272eb-613d-4227-a9c2-c3fec8a3048c" />

## 4. Reconocimiento remoto

**4.1. Descubrimiento de puertos**

La primera fase consistió en identificar los puertos TCP abiertos del objetivo.

```
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
```

El escaneo identificó dos puertos abiertos:

| Puerto  | Servicio | Relevancia |
|---------|----------|------------|
| 22/tcp  | SSH      | Posible vector de acceso remoto |
| 80/tcp  | HTTP     | Superficie web susceptible de enumeración |

Estos resultados permitieron delimitar las siguientes comprobaciones a los servicios SSH y HTTP.

<img width="584" height="411" alt="image" src="https://github.com/user-attachments/assets/13383e45-ecad-4afa-9558-909009f29f7a" />

**4.2. Identificación de versiones**

Para obtener información adicional sobre los servicios detectados, se realizó un segundo escaneo:

```
sudo nmap -sCV -p 22,80 172.17.0.2
```

Los resultados relevantes fueron:

  - **SSH:** OpenSSH 9.6p1 sobre Ubuntu.
  - **HTTP:** Apache httpd 2.4.58.

La identificación de versiones permite orientar la enumeración posterior, aunque no implica por sí misma que los servicios sean vulnerables.

<img width="586" height="326" alt="image" src="https://github.com/user-attachments/assets/5bd86878-00b4-4a2a-9297-599ec94898f1" />

## 5. Enumeración del servicio HTTP

**5.1. Descubrimiento de contenido**

Se utilizó Feroxbuster para identificar recursos accesibles en el servidor web:

```
feroxbuster -u http://172.17.0.2 \
  -w /usr/share/wordlists/dirb/common.txt
```

La enumeración identificó la página principal y el recurso index.html, ambos accesibles mediante respuestas HTTP satisfactorias.

<img width="587" height="460" alt="image" src="https://github.com/user-attachments/assets/a6e01c0c-d387-4fa0-8b7b-2a9bbd5cbfe6" />

**5.2. Análisis de la página principal**

A continuación, se examinó la respuesta del servidor:

```
curl -i http://172.17.0.2/
```

El contenido reveló una cadena especialmente relevante:

```
tails
```

Para confirmar el resultado, se inspeccionó también el contenido de index.html:

```
curl -s http://172.17.0.2/index.html | xxd
```

La salida hexadecimal confirmó que el archivo contenía el texto tails.

Este dato se utilizó como hipótesis para continuar la enumeración del servicio SSH y comprobar si correspondía a una cuenta válida del sistema.

<img width="430" height="258" alt="image" src="https://github.com/user-attachments/assets/87331d11-a084-4d7a-870f-fae48855ea8b" />

## 6. Acceso inicial mediante SSH

Con la información obtenida del servicio HTTP, se comprobó la respuesta del servicio SSH al intentar autenticar al usuario identificado.

```
ssh tails@172.17.0.2
```

El servidor solicitó una contraseña, lo que permitió continuar con las pruebas de autenticación previstas en el laboratorio.

<img width="571" height="136" alt="image" src="https://github.com/user-attachments/assets/08342f65-61a1-46d4-8a25-a54b5fe99090" />

Posteriormente, se utilizó Hydra junto con una lista de contraseñas derivada de rockyou.txt para comprobar la resistencia de la autenticación frente a intentos automatizados.

<img width="834" height="425" alt="image" src="https://github.com/user-attachments/assets/1e72f192-e30c-4835-838d-450402f6d516" />

El resultado fue la identificación de credenciales válidas para el usuario tails en este entorno.

<img width="696" height="386" alt="image" src="https://github.com/user-attachments/assets/1c3f7cd2-1e53-4be1-85cd-6252a1d26814" />

> [!NOTE]
> Evita publicar contraseñas recuperadas en texto claro si no son necesarias para comprender el procedimiento. Si se incluyen en una versión pública, deben identificarse expresamente como credenciales exclusivas del laboratorio.

**6.1. Validación de la sesión**

Con las credenciales obtenidas, se estableció una sesión SSH y se verificó la identidad efectiva:

```
ssh tails@172.17.0.2
whoami
```

El sistema devolvió:

```
tails
```

Con ello quedó confirmado el acceso inicial y comenzó la fase de enumeración local.

<img width="533" height="258" alt="image" src="https://github.com/user-attachments/assets/49c2bed6-37e7-41d4-b2c7-8cb3f96b18cf" />

## 7. Enumeración local del sistema

**7.1. Identidad y contexto de ejecución**

Una vez dentro del sistema, se recopilaron datos básicos del usuario y del entorno:

```
whoami
id
hostname
```

La información obtenida confirmó que la sesión se ejecutaba como tails, sin pertenencia a grupos adicionales en la salida observada.

<img width="375" height="125" alt="image" src="https://github.com/user-attachments/assets/bcc0dcd2-022a-4cd6-87ce-f64f86e1ba40" />

**7.2. Revisión de usuarios locales**

Se examinó el archivo /etc/passwd para identificar cuentas con shell interactiva:

```
cat /etc/passwd
```

Entre las cuentas relevantes se encontraron:

  - ubuntu
  - sonic
  - tails

La presencia de varias cuentas permitió ampliar el análisis de permisos y relaciones entre usuarios.

<img width="555" height="432" alt="image" src="https://github.com/user-attachments/assets/c498d861-4ea1-4b27-a5fa-3db925f8b560" />

**7.3. Inspección de directorios personales**

Se revisó la estructura de los directorios personales:

```
ls -la /home
```

La inspección mostró directorios asociados a los usuarios identificados. Sin embargo, el acceso directo al contenido de /home/sonic fue denegado desde la cuenta tails.

Este resultado evidenció una restricción de acceso entre usuarios, por lo que el siguiente paso consistió en comprobar si existían permisos administrativos que permitieran cambiar de identidad de forma autorizada dentro del laboratorio.

<img width="374" height="123" alt="image" src="https://github.com/user-attachments/assets/f3a0105d-3667-48b8-8f01-12b068c12356" />

## 8. Identificación de permisos excesivos en sudo

**8.1. Revisión de privilegios de tails**

La enumeración de permisos administrativos se realizó mediante:

```
sudo -l
```

El resultado relevante fue:

```
(sonic) NOPASSWD: ALL
```

Esta regla permitía al usuario tails ejecutar cualquier comando como sonic sin necesidad de introducir una contraseña.

El hallazgo constituye el primer eslabón de la escalada: una cuenta sin privilegios administrativos propios podía adoptar la identidad de otro usuario con una autorización excesivamente amplia.

<img width="428" height="70" alt="image" src="https://github.com/user-attachments/assets/d4016569-421b-4556-919a-c3fe7d5140a2" />

**8.2. Movimiento de tails a sonic**

La configuración identificada permitió iniciar una shell con la identidad de sonic:

```
sudo -u sonic /bin/bash
```

La identidad efectiva se verificó mediante:

```
whoami
```

Resultado:

```
sonic
```

Este paso confirmó que la configuración de sudo permitía el movimiento entre usuarios sin necesidad de conocer la contraseña de sonic.

<img width="340" height="70" alt="image" src="https://github.com/user-attachments/assets/873ce247-1079-4f99-988b-8de985909ff0" />

## 9. Escalada de privilegios a root

**9.1. Enumeración de permisos como sonic**

Una vez obtenida la sesión de sonic, se repitió la comprobación de privilegios:

```
sudo -l
```

La salida mostró:

```
(ALL) NOPASSWD: ALL
```

A diferencia de la regla anterior, esta autorización permitía ejecutar comandos como cualquier usuario del sistema, incluido root, sin solicitar una contraseña.

La configuración representaba una vía directa para alcanzar el máximo nivel de privilegios del sistema.

<img width="427" height="74" alt="image" src="https://github.com/user-attachments/assets/54988458-a501-46ef-a997-f5211bca207c" />

**9.2. Obtención y verificación de root**

Se ejecutó el siguiente comando:

```
sudo /bin/bash
```

A continuación, se verificó la identidad efectiva:

```
whoami
```

Resultado:

```
root
```

La comprobación confirmó la finalización de la escalada de privilegios.

<img width="342" height="97" alt="image" src="https://github.com/user-attachments/assets/b8440157-edb2-45a8-bf3a-3c43be385e93" />

## 10. Análisis de la causa raíz

Tras obtener privilegios de root, se inspeccionó la configuración de sudo para identificar las reglas responsables de la escalada.

Las entradas relevantes fueron:

**tails ALL=(sonic) NOPASSWD:ALL**\
**sonic ALL=(ALL) NOPASSWD:ALL**

La primera regla permitía a tails ejecutar cualquier comando como sonic. La segunda concedía a sonic capacidad de ejecución como cualquier usuario, incluido root.

La combinación de ambas reglas hizo posible encadenar dos cambios de identidad y pasar de una cuenta inicial sin privilegios elevados a una shell con control completo del sistema.

<img width="1875" height="839" alt="image" src="https://github.com/user-attachments/assets/a0762418-7590-47f5-8adc-60bc2f177774" />

## 11. Resumen de la cadena de explotación

La resolución puede resumirse en cinco etapas:

  1. **Reconocimiento:** identificación de los puertos 22 y 80.
  2. **Enumeración web:** descubrimiento de la cadena tails.
  3. **Acceso inicial:** obtención de una sesión SSH como tails.
  4. **Movimiento entre usuarios:** ejecución de comandos como sonic mediante permisos de sudo.
  5. **Escalada final:** obtención de una shell root mediante los privilegios asignados a sonic.

<img width="2172" height="724" alt="image" src="https://github.com/user-attachments/assets/9526741e-4059-4afa-8dae-0b9a10172a06" />

## 12. Medidas de mitigación

**12.1. Aplicación del principio de mínimo privilegio**

Las reglas identificadas deberían sustituirse por autorizaciones limitadas a los comandos estrictamente necesarios para las funciones de cada cuenta.

No se recomienda conceder permisos generales mediante NOPASSWD: ALL. Las reglas de sudoers deben revisarse periódicamente y validarse con herramientas como visudo.

**12.2. Protección del servicio SSH**

Para reducir el riesgo de accesos mediante contraseñas débiles o intentos automatizados, se recomienda:

  - Priorizar la autenticación mediante claves SSH.
  - Deshabilitar la autenticación por contraseña cuando la operativa lo permita.
  - Aplicar controles de frecuencia y bloqueo ante intentos repetidos.
  - Restringir el acceso al servicio mediante firewall y listas de control.
  - Supervisar los registros de autenticación.

**12.3. Reducción de la exposición de información**

El servidor web no debería publicar información interna que facilite la identificación de cuentas del sistema.

Aunque la cadena tails no constituye por sí sola una vulnerabilidad crítica, en este laboratorio sirvió como punto de partida para orientar las pruebas posteriores.

<img width="1942" height="809" alt="image" src="https://github.com/user-attachments/assets/7ca5d940-4e16-4460-8325-0663157d504a" />

## 13. Conclusiones

El laboratorio HedgeDog demuestra cómo varias debilidades de configuración pueden combinarse para comprometer un sistema Linux.

La información expuesta por el servicio HTTP facilitó la identificación de un usuario, mientras que los permisos excesivos de sudo permitieron encadenar cambios de identidad hasta obtener privilegios de root.

El hallazgo principal no fue una vulnerabilidad aislada de una versión concreta de software, sino una configuración de autorización que concedía capacidades superiores a las necesarias.

Las principales lecciones del laboratorio son:

  - La enumeración debe realizarse de forma sistemática, incluso cuando un servicio parece no contener información relevante.
  - Las pistas obtenidas en un servicio pueden orientar la investigación de otros servicios.
  - La revisión de sudo -l es una comprobación esencial durante la enumeración local de Linux.
  - Los permisos NOPASSWD: ALL pueden crear rutas directas de escalada de privilegios.
  - Una cadena de permisos entre usuarios puede tener un impacto equivalente al compromiso completo del sistema.

**Cadena final validada:**

```
HTTP → tails → sonic → root
```

Con esta resolución queda documentado tanto el procedimiento de explotación como la causa raíz que hizo posible el compromiso del laboratorio.

> [!CAUTION]
> **Laboratorio realizado con fines educativos en un entorno autorizado de DockerLabs.**

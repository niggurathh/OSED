# OSED
Notas sobre el examen OSED - Español

--Esta información es solamente para recalcar puntos importantes--

## Protecciones

¿Como realizar bypass del DEP?

Dado que el DEP (Data Execution Prevention) como su nombre lo dice, evita que ejecutemos codigo en zonas donde no se debería ejecutar codigo, en este caso, nuestra shell code. 
Para ello es necesario utilizar ROP, pero, ¿qué es ROP? 

El ROP es una técnica que se basa en encontrar y utilizar secuencias de instrucciones ya existentes en la memoria del programa, conocidas como "gadgets". Cada uno de estos gadgets termina en una instrucción de retorno (ret), y el atacante manipula la pila de ejecución para enlazar estos gadgets de manera que se ejecute el código deseado. 
Al utilizar código ya presente en el sistema, el ROP puede evadir ciertas medidas de seguridad que solo verifican la integridad o la procedencia del código nuevo o modificado.
La idea es que, a pesar de que la ejecución directa de código puede estar bloqueada por el DEP, se pueden utilizar las instrucciones ya presentes en la memoria para realizar acciones y obtener un shell.

Una vez que comprendemos que es el ROP, podemos finalizar utilizando estas instrucciones para llamar a una funcion de las API de windows.

## Llamadas a funciones de Windows para evitar DEP

WriteProcessMemory().  Esto te permitirá copiar la Shellcode en otra ubicación (que sea ejecutable), por lo que puedes saltar y ejecutar la Shellcode. La ubicación de destino debe tener permisos de escritura y de ejecución.

Después de construir toda tu cadena ROP para crear los parámetros, el resultado final sea algo así:
![image](https://github.com/user-attachments/assets/374a5ae0-51f0-4159-bcd7-d2c1baad9f70)

Esta tabla muestra donde es posible utilizar las llamadas.
![image](https://github.com/user-attachments/assets/0e9c6ebb-9dee-4c7d-8702-3fa80b37ec4c)

Un dato importante es que cuando desees utilizar uno de los métodos de la API de Windows, tendrás que configurar la pila con los parámetros correctos para esa función primero.

Para esto, nos enfocaremos exclusivamente en WriteProcessMemory
(https://learn.microsoft.com/es-mx/windows/win32/api/memoryapi/nf-memoryapi-writeprocessmemory?redirectedfrom=MSDN)

Se utiliza de la siguiente forma:

```
BOOL WINAPI WriteProcessMemory(
 __in HANDLE hProcess,
 __in LPVOID lpBaseAddress,
 __in LPCVOID lpBuffer,
 __in SIZE_T nSize,
 __out SIZE_T *lpNumberOfBytesWritten
); 
```

Esta función requiere de 6 parámetros en la pila

![image](https://github.com/user-attachments/assets/f8266c8b-e91b-410d-8758-e310022e2ccb)

## Explicación de técnicas
### RET directo
En un típico exploit de RET directo. Sobreescribes EIP directamente con un valor arbitrario (o, para ser más precisos, EIP se sobrescribe cuando se activa la ultima parte de una función que utiliza un EIP guardado sobrescrito. Cuando eso sucede, lo más probable es ver que se controlan datos de la memoria en el lugar donde ESP apunta. Así que, si no fuera por el DEP, pondrías un puntero a "JMP ESP" (! Pvefindaddr j esp) y saltar a tu Shellcode.
Cuando DEP está habilitado, no podemos hacer eso. En lugar de saltar a ESP (sobreescribir EIP con un puntero que saltaría a ESP), tenemos que llamar al primer Gadget de ROP, ya sea directamente en EIP o haciendo que EIP retorne a ESP. Los Gadgets se deben configurar de una manera determinada para que formen una cadena y devuelvan un Gadget para el próximo Gadget sin tener que ejecutar el código directamente desde la pila. 

### SEH
En un exploit de SEH, las cosas son un poco diferentes. Sólo controlas el valor en EIP cuando el SE Handler es llamado mediante la activación de una violación de acceso, por ejemplo. En exploits de SEH, sobrescribirás SEH con un puntero a POP/POP/RET, lo que te hará aterrizar en el próximo SEH, y ejecutar las instrucciones en ese lugar. Cuando DEP está habilitado, no podemos hacer eso. Podemos llamar al p/p/r, pero cuando aterriza, comenzaría a ejecutar el código de la pila. Y no podemos ejecutar código en la pila, Tenemos que construir una cadena de ROP, y evitar la cadena o desactivar el DEP. Esta cadena se coloca en la pila como parte del Payload del exploit. Así, en el caso de un exploit de SEH, tenemos que encontrar una manera de regresar a nuestro Payload (en la pila) en lugar de llamar a una secuencia POP POP RET.

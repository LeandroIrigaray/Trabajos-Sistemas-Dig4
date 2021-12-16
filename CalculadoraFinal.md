# Trabajos-Sistemas-Dig4

; SE DEFINEN LOS REGISTROS ; 
;==============================================================================================================================; 
.DEF DATA = R16 
.DEF AUX = R17 
.DEF TEMP = R18 
.DEF RL = R19 
.DEF RH = R20 
 
.EQU CLK = PORTB5 
.EQU DIN = PORTB3 
.EQU LOAD = PORTB2 
;===============================================================================================================================; 
; SEGMENTO DE DATOS ; 
;===============================================================================================================================; 
.DSEG 
A: .BYTE 1 
B: .BYTE 1 
SIGNO_RESTA: .BYTE 1 
UNIDAD: .BYTE 1 
DECENA: .BYTE 1 
CENTENA: .BYTE 1 
UNIDAD_MIL: .BYTE 1 
DECENA_MIL: .BYTE 1 
POSICION: .BYTE 1 
;===============================================================================================================================; 
; SEGMENTO DE CODIGO ; 
;===============================================================================================================================; 
.CSEG 
;===============================================================================================================================; 
; INICIALIZAR SP ; 
;===============================================================================================================================; 
.MACRO SET_STACK 
LDI R16, LOW(RAMEND) 
OUT SPL, R16 
LDI R16, HIGH(RAMEND) 
OUT SPH, R16 
.ENDMACRO 
;===============================================================================================================================; 
; SE DEFINE EL ORIGEN DE MEMORIA ; 
;===============================================================================================================================; 
.ORG 0x0000 
JMP RESET 
;===============================================================================================================================; 
; VECTOR DE INTERRUPCION: RECEPCION USART ; 
;===============================================================================================================================; 
.ORG 0x0024 
JMP USART_RX 
;===============================================================================================================================; 
; SE DEFINE POR SEGURIDAD ; 
;===============================================================================================================================; 
.ORG 0x0034 
RETI 
;===============================================================================================================================; 
; INICIO DEL PROGRAMA ; 
;===============================================================================================================================; 
RESET: 
;Inicializa el SP 
SET_STACK 
 
;Se define el puerto C0 y C1 como entrada 
LDI R16,(0<<DDC0)|(0<<DDC1) 
OUT DDRC, R16 
 
;Se habilita la resistencia pull-up en PC0 y PC1 
;LDI R17, (1<<PC0)|(1<<PC1) 
;OUT PORTC, R17 
NOP 
NOP 
NOP 
 
; Habilita los pines CLK (13), DIN (11) y LOAD (10) 
LDI R16, (1<<CLK)|(1<<DIN)|(1<<LOAD) 
OUT DDRB, R16 
 
; Habilita el SPI, modo maestro 
LDI R17, (1<<SPE)|(1<<MSTR)|(1<<SPR1) 
OUT SPCR, R17 
 
; Modo de decodificacion a "Codigo fuente-B" 
LDI AUX, 0x09 
LDI TEMP, 0xFF 
RCALL MAX7219_writeData 
 
; El limite de escaneo se ejecuta desde 0 
LDI AUX, 0x0B 
LDI TEMP, 4 
RCALL MAX7219_writeData 
 
; Modo de intensidad 
LDI AUX, 0x0A 
LDI TEMP, 0x0D 
RCALL MAX7219_writeData 
 
; Modo encendido 
LDI AUX, 0x0C 
LDI TEMP, 1 
RCALL MAX7219_writeData 
 
;Inicializa la USART 
RCALL USART_INIT 
 
;Se habilitan globalmente las interrupciones 
SEI 
 
;Borra los numeros del display 
RCALL BORRAR 
 
LOOP: 
 
;Limpia los registros donde se guarda el resultado 
CLR RH 
CLR RL 
 
;Configuro ADC 
LDI R16, (0<<REFS1)|(1<<REFS0)|(1<<ADLAR)|(0<<MUX3)|(0<<MUX2)|(0<<MUX1)|(0<<MUX0) 
STS ADMUX, R16 
 
RCALL INICIA 
RCALL POTENCIOMETRO_A 
 
 
;Configuro ADC 
LDI R16, (0<<REFS1)|(1<<REFS0)|(1<<ADLAR)|(0<<MUX3)|(0<<MUX2)|(0<<MUX1)|(1<<MUX0) 
STS ADMUX, R16 
 
RCALL INICIA 
LDS AUX, ADCH 
STS B, AUX 
LDS RL, B 
RJMP LOOP 
 
 
INICIA: 
;Se activa el ADC y empieza la conversi?n Anal?gica Digital AVR 
LDI R16, (1<<ADEN)|(0<<ADSC)|(0<<ADATE)|(0<<ADIE)|(1<<ADPS2)|(1<<ADPS1)|(1<<ADPS0) 
STS ADCSRA, R16 
 
;Empieza a convertir 
LDS R17, ADCSRA 
ORI R17, (1<<ADSC) 
STS ADCSRA, R17 
 
ESPERA: 
LDS R17, ADCSRA 
SBRC R17, ADSC 
RJMP ESPERA 
 
 
RET 
 
POTENCIOMETRO_A: 
LDS AUX, ADCH 
STS A, AUX 
LDS RH, A 
 
RET 
 
;===============================================================================================================================; 
; INTERRUPCION DE RECEPCION ; 
;===============================================================================================================================; 
USART_RX: 
RCALL LEER 
RETI 
 
;===============================================================================================================================; 
; FUNCIONES ; 
;===============================================================================================================================; 
; - CONVERTIR 
; - USART_INIT 
; - USART_TRANSMIT 
; - USART_MOSTRAR 
;===============================================================================================================================; 
; Convierte en decenas de mil, unidades de mil, centenas, decenas y unidades ; 
;===============================================================================================================================; 
CONVERTIR: 
;Pone en cero a todos los registros 
CLR R21 
STS DECENA_MIL, R21 
CLR R22 
STS UNIDAD_MIL, R22 
CLR R23 
STS CENTENA, R23 
CLR R24 
STS DECENA, R24 
CLR R25 
STS UNIDAD, R25 
 
DECENAS_MIL: 
LDI TEMP, 0x27 ;carga el valor decimal 10000 (0x2710 en hexadecimal) en TEMP:AUX (r18:r17) 
LDI AUX, 0x10 
 
CP RL, AUX ;comparar RH:RL con 10000 
CPC RH, TEMP 
 
BRCS UNIDADES_MIL ;si es menor de 10000 ir a UNIDADES_MIL 
 
SUB RL, AUX ;restar 10000 de RH:RL 
SBC RH, TEMP 
 
INC R21 
STS DECENA_MIL, R21 ;incrementar DECENA_MIL 
 
CP RL, AUX ;comparar RH:RL con 10000 
CPC RH, TEMP 
 
BRCC DECENAS_MIL ;si es mayor de 10000, volver a restar 1000 
 
UNIDADES_MIL: 
LDI TEMP, 0x03 ;cargar el valor decimal 1000 (0x03E8 en hexadecimal) en TEMP:AUX (r18:r17) 
LDI AUX, 0xE8 
 
CP RL, AUX ;comparar RH:RL con 1000 
CPC RH, TEMP 
 
BRCS CENTENAS ;si es menor de 1000 ir a CENTENAS 
 
SUB RL, AUX ;restar 1000 de RH:RL 
SBC RH, TEMP 
 
INC R22 
STS UNIDAD_MIL, R22 ;incrementar UNIDAD_MIL 
 
CP RL, AUX ;comparar RH:RL con 1000 
CPC RH, TEMP 
 
BRCC UNIDADES_MIL ;si es mayor de 1000, volver a restar 1000 
 
CENTENAS: 
LDI TEMP, 0x00 ;cargar el valor decimal 100 (0x0064 en hexadecimal) en TEMP:AUX (r18:r17) 
LDI AUX, 0x64 
 
CP RL, AUX ;comparar RH:RL con 100 
CPC RH, TEMP 
 
BRCS DECENAS ;si es menor de 100 ir a DECENAS 
 
SUB RL, AUX ;restar 100 de RH:RL 
SBC RH, TEMP 
 
INC R23 
STS CENTENA, R23 ;incrementar CENTENA 
 
CP RL, AUX ;comparar RH:RL con 100 
CPC RH, TEMP 
 
BRCC CENTENAS ;si es mayor de 100, volver a restar 100 
 
DECENAS: 
CPI RL, 10 ;comparar RL con 10 
BRLO UNIDADES ;si es menor, ir a UNIDADES 
SUBI RL, 10 ;si es mayor restar 10 de RL 
INC R24 
STS DECENA, R24 ;incrementar DECENA 
CPI RL, 10 ;comparar RL con 10 
BRSH DECENAS ;si es mayor de 10, volver a restar 10 
UNIDADES: 
CPI RL, 1 ;comparar RL con 1 
BRLO SALIR ;si es menor, ir a SALIR 
SUBI RL, 1 ;si es mayor restar 1 de RL 
INC R25 
STS UNIDAD, R25 ;incrementar UNIDAD 
CPI RL, 1 ;comparar RL con 1 
BRSH UNIDADES ;si es mayor de 1, volver a restar 1 
 
SALIR: 
RET 
 
;===============================================================================================================================; 
; Inicializa la USART a 9600 baudios y 16MHz ; 
;===============================================================================================================================; 
USART_INIT: 
;Set baud rate to UBRR0 
LDI R16, 0x67 
CLR R17 
 
STS UBRR0H, R17 
STS UBRR0L, R16 
 
;Enable receiver and transmitter (y tambien se activa la recepcion por interrupcion) 
LDI R16, (1<<RXEN0)|(1<<TXEN0)|(1<<RXCIE0) 
STS UCSR0B, R16 
 
;Set frame format: 8data, 2stop bit 
LDI R16, (1<<USBS0)|(3<<UCSZ00) 
STS UCSR0C, R16 
 
RET 
;===============================================================================================================================; 
; Envia un dato al monitor serie ; 
;===============================================================================================================================; 
USART_TRANSMIT: 
;Wait for empty transmit buffer 
LDS R17, UCSR0A 
SBRS R17, UDRE0 
RJMP USART_TRANSMIT 
 
;Put data (r16) into buffer, sends the data 
STS UDR0, R16 
RET 
 
;===============================================================================================================================; 
; Recibe un dato del monitor serie ; 
;===============================================================================================================================; 
USART_RECEIVE: 
;Wait for data to be received 
LDS R17, UCSR0A 
SBRS R17, RXC0 
RJMP USART_RECEIVE 
;Get and return received data from buffer 
LDS R16, UDR0 
RET 
 
 
;===============================================================================================================================; 
; Recibe un dato del monitor serie ; 
;===============================================================================================================================; 
LEER: 
 
RCALL USART_RECEIVE 
 
;Si el caracter recibido es 'A' o 'a', mostrar el valor de A 
CPI DATA, 65 ;A 
BREQ CHAR_A 
CPI DATA, 97 ;a 
BREQ CHAR_A 
 
;Si el caracter recibido es 'B' o 'b', mostrar el valor de B 
CPI DATA, 66 ;B 
BREQ CHAR_B 
CPI DATA, 98 ;b 
BREQ CHAR_B 
 
;Si el caracter recibido es '+', mostrar la suma entre A y B 
CPI DATA, 43 ;+ 
BREQ CHAR_SUMA_JMP ;Se realiza esto de los JMP 
RJMP CHAR_SUMA_NO_JMP ;debido a que surgio un error 
CHAR_SUMA_JMP: ;Error:Relative branch out of reach
RJMP CHAR_SUMA 
 
CHAR_SUMA_NO_JMP: 
;Si el caracter recibido es '-', mostrar la resta entre A y B 
CPI DATA, 45 ;- 
BREQ CHAR_RESTA_JMP 
RJMP CHAR_RESTA_NO_JMP 
CHAR_RESTA_JMP: 
RJMP CHAR_RESTA 
 
CHAR_RESTA_NO_JMP: 
;Si el caracter recibido es '*', mostrar el producto entre A y B 
CPI DATA, 42 ;* 
BREQ CHAR_MULTIPLICACION_JMP 
RJMP CHAR_MULTIPLICACION_NO_JMP 
CHAR_MULTIPLICACION_JMP: 
RJMP CHAR_MULTIPLICACION 
 
CHAR_MULTIPLICACION_NO_JMP: 
;Si el caracter recibido es '/', mostrar la division entera entre A y B 
CPI DATA, 47 ;/ 
BREQ CHAR_DIVISION_JMP 
RJMP CHAR_DIVISION_NO_JMP 
CHAR_DIVISION_JMP: 
RJMP CHAR_DIVISION 
 
CHAR_DIVISION_NO_JMP: 
;Si el caracter recibido es '&', mostrar la logica AND entre A y B 
CPI DATA, 38 ;& 
BREQ CHAR_AND_JMP 
RJMP CHAR_AND_NO_JMP 
CHAR_AND_JMP: 
RJMP CHAR_AND 
 
CHAR_AND_NO_JMP: 
;Si el caracter recibido es '|', mostrar la logica OR entre A y B 
CPI DATA, 124 ;| 
BREQ CHAR_OR_JMP 
RJMP CHAR_OR_NO_JMP 
CHAR_OR_JMP: 
RJMP CHAR_OR 
 
CHAR_OR_NO_JMP: 
;Si el caracter recibido es '^', mostrar la logica XOR entre A y B 
CPI DATA, 94 ;^ 
BREQ CHAR_XOR_JMP 
RJMP CHAR_XOR_NO_JMP 
CHAR_XOR_JMP: 
RJMP CHAR_XOR 
 
CHAR_XOR_NO_JMP: 
RET 
 
CHAR_A: 
LDI DATA, 65 ;Muestra la letra 'A' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDI DATA, 61 ;Muestra el signo '=' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDS RL, A ;Carga el valor de 'A' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 10 ;Realiza un salto de linea 
RCALL USART_TRANSMIT 
RET 
 
CHAR_B: 
LDI DATA, 66 ;Muestra la letra 'B' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDI DATA, 61 ;Muestra el signo '=' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDS RL, B ;Carga el valor de 'B' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 10 ;Realiza un salto de linea 
RCALL USART_TRANSMIT 
RET 
 
CHAR_SUMA: 
LDS RL, A ;Carga el valor de 'A' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDI DATA, 43 ;Muestra el caracter '+' 
RCALL USART_TRANSMIT 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDS RL, B ;Carga el valor de 'B' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDI DATA, 61 ;Muestra el signo '=' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
CLR RH 
CLR RL 
 
LDS RH, A 
LDS RL, B 
 
RCALL SUMA 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 10 ;Realiza un salto de linea 
RCALL USART_TRANSMIT 
RET 
 
CHAR_RESTA: 
LDS RL, A ;Carga el valor de 'A' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDI DATA, 45 ;Muestra el caracter '-' 
RCALL USART_TRANSMIT 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDS RL, B ;Carga el valor de 'B' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDI DATA, 61 ;Muestra el signo '=' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
CLR RH 
CLR RL 
 
LDS RH, A 
LDS RL, B 
 
RCALL RESTA 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 10 ;Realiza un salto de linea 
RCALL USART_TRANSMIT 
 
CLR R21 
STS SIGNO_RESTA, R21 
 
RET 
 
CHAR_MULTIPLICACION: 
LDS RL, A ;Carga el valor de 'A' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDI DATA, 42 ;Muestra el caracter '*' 
RCALL USART_TRANSMIT 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDS RL, B ;Carga el valor de 'B' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDI DATA, 61 ;Muestra el signo '=' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
CLR RH 
CLR RL 
 
LDS RH, A 
LDS RL, B 
 
RCALL MULTIPLICACION 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 10 ;Realiza un salto de linea 
RCALL USART_TRANSMIT 
RET 
 
CHAR_DIVISION: 
LDS RL, A ;Carga el valor de 'A' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDI DATA, 47 ;Muestra el caracter '/' 
RCALL USART_TRANSMIT 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDS RL, B ;Carga el valor de 'B' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDI DATA, 61 ;Muestra el signo '=' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDS RL, B ;Carga el valor de 'B' al registro RL 
CPI RL, 0 
 
BREQ INDETERMINADO_JMP 
RJMP INDETERMINADO_NO_JMP 
INDETERMINADO_JMP: 
RJMP INDETERMINACION 
RET 
INDETERMINADO_NO_JMP: 
 
CLR RH 
CLR RL 
 
LDS RH, A 
LDS RL, B 
 
RCALL DIVISION 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 10 ;Realiza un salto de linea 
RCALL USART_TRANSMIT 
RET 
 
CHAR_AND: 
LDS RL, A ;Carga el valor de 'A' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDI DATA, 38 ;Muestra el caracter '&' 
RCALL USART_TRANSMIT 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDS RL, B ;Carga el valor de 'B' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDI DATA, 61 ;Muestra el signo '=' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
CLR RH 
CLR RL 
 
LDS RH, A 
LDS RL, B 
 
RCALL LOGICA_AND 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 10 ;Realiza un salto de linea 
RCALL USART_TRANSMIT 
RET 
 
CHAR_OR: 
LDS RL, A ;Carga el valor de 'A' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDI DATA, 124 ;Muestra el caracter '|' 
RCALL USART_TRANSMIT 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDS RL, B ;Carga el valor de 'B' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDI DATA, 61 ;Muestra el signo '=' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
CLR RH 
CLR RL 
 
LDS RH, A 
LDS RL, B 
 
RCALL LOGICA_OR 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 10 ;Realiza un salto de linea 
RCALL USART_TRANSMIT 
RET 
 
CHAR_XOR: 
LDS RL, A ;Carga el valor de 'A' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDI DATA, 94 ;Muestra el caracter '^' 
RCALL USART_TRANSMIT 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
 
LDS RL, B ;Carga el valor de 'B' al registro RL 
CLR RH 
 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
LDI DATA, 61 ;Muestra el signo '=' 
RCALL USART_TRANSMIT 
LDI DATA, 32 ;Realiza un espacio 
RCALL USART_TRANSMIT 
CLR RH 
CLR RL 
 
LDS RH, A 
LDS RL, B 
 
RCALL LOGICA_XOR 
RCALL CONVERTIR ;Llama a la funcion que convierte los valores de hexadecimal a decimal 
 
MOV DATA, RL ;Mueve el valor convertido al registro DATA 
RCALL USART_MOSTRAR ;Llama a la funcion que muestra el numero en decimal 
 
LDI DATA, 10 ;Realiza un salto de linea 
RCALL USART_TRANSMIT 
RET 
 
 
INDETERMINACION: 
LDI DATA, 73 ;Muestra la letra 'I' 
RCALL USART_TRANSMIT 
LDI DATA, 110 ;Muestra la letra 'n' 
RCALL USART_TRANSMIT 
LDI DATA, 100 ;Muestra la letra 'd' 
RCALL USART_TRANSMIT 
LDI DATA, 101 ;Muestra la letra 'e' 
RCALL USART_TRANSMIT 
LDI DATA, 116 ;Muestra la letra 't' 
RCALL USART_TRANSMIT 
LDI DATA, 101 ;Muestra la letra 'e' 
RCALL USART_TRANSMIT 
LDI DATA, 114 ;Muestra la letra 'r' 
RCALL USART_TRANSMIT 
LDI DATA, 109 ;Muestra la letra 'm' 
RCALL USART_TRANSMIT 
LDI DATA, 105 ;Muestra la letra 'i' 
RCALL USART_TRANSMIT 
LDI DATA, 110 ;Muestra la letra 'n' 
RCALL USART_TRANSMIT 
LDI DATA, 97 ;Muestra la letra 'a' 
RCALL USART_TRANSMIT 
LDI DATA, 100 ;Muestra la letra 'd' 
RCALL USART_TRANSMIT 
LDI DATA, 111 ;Muestra la letra 'o' 
RCALL USART_TRANSMIT 
 
LDI DATA, 10 ;Realiza un salto de linea 
RCALL USART_TRANSMIT 
 
RCALL BORRAR 
 
RET 
 
;===============================================================================================================================; 
; Convierte en formato ASCII y muestra el dato por el monitor serie ; 
;===============================================================================================================================; 
USART_MOSTRAR: 
 
RCALL BORRAR 
;Si el numero es negativo, primero muestra el signo menos (-) 
LDS R21, SIGNO_RESTA 
CPI R21, -1 
BREQ DIGITO_MENOS 
 
DIGITOS: 
;Si la decena de mil es igual a cero no la muestra 
LDS R21, DECENA_MIL 
CPI R21, 0 
BRNE DIGITO_CINCO 
 
 
;Si la unidad de mil es igual a cero no la muestra 
LDS R22, UNIDAD_MIL 
CPI R22, 0 
BRNE DIGITO_CUATRO 
 
;Si la centena es igual a cero no la muestra 
LDS R23, CENTENA 
CPI R23, 0 
BRNE DIGITO_TRES 
 
;Si la decena es igual a cero no la muestra 
LDS R24, DECENA 
CPI R24, 0 
BRNE DIGITO_DOS 
 
;Si la unidad es mayor o igual a cero, si la muestra 
LDS R25, UNIDAD 
CPI R25, 0 
BRSH DIGITO_UNO 
 
DIGITO_MENOS: 
 
LDS AUX, POSICION 
LDI TEMP, 10 
RCALL MAX7219_writeData 
 
LDI DATA, 45 
RCALL USART_TRANSMIT 
 
RJMP DIGITOS 
 
DIGITO_CINCO: 
 
LDI AUX, 0x05 
LDS TEMP, DECENA_MIL 
RCALL MAX7219_writeData 
 
LDS DATA, DECENA_MIL 
RCALL MUESTRA 
 
DIGITO_CUATRO: 
 
LDI AUX, 0x04 
LDS TEMP, UNIDAD_MIL 
RCALL MAX7219_writeData 
 
LDS DATA, UNIDAD_MIL 
RCALL MUESTRA 
 
DIGITO_TRES: 
 
LDI AUX, 0x03 
LDS TEMP, CENTENA 
RCALL MAX7219_writeData 
 
LDS DATA, CENTENA 
RCALL MUESTRA 
 
DIGITO_DOS: 
 
LDI AUX, 0x02 
LDS TEMP, DECENA 
RCALL MAX7219_writeData 
 
LDS DATA, DECENA 
RCALL MUESTRA 
 
DIGITO_UNO: 
 
LDI AUX, 0x01 
LDS TEMP, UNIDAD 
RCALL MAX7219_writeData 
 
LDS DATA, UNIDAD 
RCALL MUESTRA 
RET 
MUESTRA: 
;Carga el valor decimal 48 en aux2 
LDI TEMP, 48 
ADD DATA, TEMP 
RCALL USART_TRANSMIT 
RET 
 
;===============================================================================================================================; 
; FIN DE FUNCIONES ; 
;===============================================================================================================================; 
 
 
;==============================================================================================================================; 
; OPERACIONES ARITMETICAS ; 
;==============================================================================================================================; 
; - SUMA ; 
; - RESTA ; 
; - MULTIPLICACION ; 
; - DIVISION ; 
; - LOGICA AND ; 
; - LOGICA OR ; 
; - LOGICA XOR ; 
;===============================================================================================================================; 
; Realiza la suma entre los registros AH y AL, y guarda el resultado en AH:AL ; 
;===============================================================================================================================; 
SUMA: ; 
CLR AUX ; 
CLR TEMP ; 
MOV AUX, RH ; 
CLR RH ; 
ADD RL, AUX ; 
ADC RH, TEMP ; 
RET ; 
;===============================================================================================================================; 
; Realiza la resta entre dos valores AH y AL, y los almacena en AL ; 
;===============================================================================================================================; 
RESTA: ; 
CP RH, RL ; 
BRLO NEGATIVA ; 
SUB RH, RL ; 
MOV RL, RH ; 
; 
RJMP SALIDA ; 
NEGATIVA: ; 
SUB RL,RH ; 
CLR RH ; 
LDI R21, -1 ; 
STS SIGNO_RESTA, R21 ; 
; 
SALIDA: ; 
CLR RH ; 
CPI RL, 99 ; 
BRPL _4 ; 
; 
CPI RL, 9 ; 
BRPL _3 ; 
; 
CPI RL, 10 ; 
BRLO _2 ; 
; 
RET ; 
; 
_4: ; 
LDI AUX, 0x04 ; 
STS POSICION, AUX ; 
RET ; 
; 
_3: LDI AUX, 0x03 ; 
STS POSICION, AUX ; 
RET ; 
; 
_2: LDI AUX, 0x02 ; 
STS POSICION, AUX ; 
RET ; 
; 
;===============================================================================================================================; 
; Realiza la multiplicacion entre los registros AH y AL, y guarda el resultado en AH:AL ; 
;===============================================================================================================================; 
MULTIPLICACION: ; 
MUL RH, RL ; 
MOV RH, r1 ; 
MOV RL, r0 ; 
RET ; 
;===============================================================================================================================; 
; Realiza la division entera entre los registros AH y AL, y guarda el resultado en AL ; 
;===============================================================================================================================; 
DIVISION: ; 
CLR AUX ;cociente ; 
COCIENTE: ; 
INC AUX ; 
SUB RH, RL ; 
BRCC COCIENTE ; 
DEC AUX ; 
; 
CLR RH ; 
MOV RL, AUX ; 
RET ; 
;===============================================================================================================================; 
; Realiza la logica AND entre los registros AH y AL, y guarda el resultado en AL ; 
;===============================================================================================================================; 
LOGICA_AND: ; 
AND RL, RH ; 
CLR RH ; 
RET ; 
;===============================================================================================================================; 
; Realiza la logica OR entre los registros AH y AL, y guarda el resultado en AL ; 
;===============================================================================================================================; 
LOGICA_OR: ; 
OR RL, RH ; 
CLR RH ; 
RET ; 
;===============================================================================================================================; 
; Realiza la logica XOR entre los registros AH y AL, y guarda el resultado en AL ; 
;===============================================================================================================================; 
LOGICA_XOR: ; 
EOR RL, RH ; 
CLR RH ; 
RET ; 
;===============================================================================================================================; 
; FIN DE OPERACIONES ARITMETICAS ; 
;===============================================================================================================================; 
 
 
spiSendByte: 
; Start transmission of data (r16) 
OUT SPDR, r16 
Wait_Transmit: 
; Wait for transmission complete 
IN R16, SPSR 
SBRS R16, SPIF 
RJMP Wait_Transmit 
RET 
 
 
MAX7219_writeData: 
CBI PORTB, LOAD 
MOV R16, AUX 
RCALL spiSendByte 
//RCALL SPI_MasterTransmit 
MOV R16, TEMP 
RCALL spiSendByte 
//RCALL SPI_MasterTransmit 
SBI PORTB, LOAD 
RET 
 
 
BORRAR: 
 
LDI AUX, 0x05 
LDI TEMP, 15 
RCALL MAX7219_writeData 
 
LDI AUX, 0x04 
LDI TEMP, 15 
RCALL MAX7219_writeData 
 
LDI AUX, 0x03 
LDI TEMP, 15 
RCALL MAX7219_writeData 
 
LDI AUX, 0x02 
LDI TEMP, 15 
RCALL MAX7219_writeData 
 
LDI AUX, 0x01 
LDI TEMP, 15 
RCALL MAX7219_writeData 
 
RET

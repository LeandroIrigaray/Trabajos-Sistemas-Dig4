# Trabajos-Sistemas-Dig4
#Vivero1

#define F_CPU 16000000UL
#define BAUD 9600
#define MYUBRR0 F_CPU/16/BAUD-1
#include "I2C.h"
#include "LCD_I2C.h"
#include <avr/io.h>				// input/output
#include <avr/interrupt.h>		// interruptions
#include <avr/eeprom.h>         // libreria eeprom

#define T 84
#define H 72
#define S 86
#define L 76

#define addrTsp 0x00
#define addrTHys 0x01
#define addrHAsp 0x02
#define addrHAhys 0x03
#define addrHSsp 0x04
#define addrHShys 0x05
#define addrVVmax 0x06
#define addrVVmin 0x07
#define addrEmax 0x08
#define addrEmin 0x0A

uint8_t Tsp=22;
uint8_t THys=5;
uint8_t HAsp=70;
uint8_t HAhys=10;
uint8_t HSsp=45;
uint8_t HShys=10;
uint8_t VVmax=30;
uint8_t VVmin=3;
uint16_t Emax=850;
uint8_t Emin=120;

uint16_t  Temperature = 0;
uint16_t  Humidity = 0;
uint16_t  Speed = 0;
uint16_t  Light = 0;

void esperarTX();
void enviarADC(int);
void eeprom_init(void);

void Configuration(){

	UCSR0A=0;                                 //Modo Normal Velocidad
	UCSR0B |= (1<<TXEN0)|(1<<RXEN0);          //Habilito Tx y Rx
	UCSR0B |= (1<<RXCIE0);                    //Interrupcion Recepcion Completa
	UCSR0C=(1<<UCSZ01)|(1<<UCSZ00);           //8 Bits
	UBRR0=103;                                //9600 Baudios
	
	//Configure PB4 as output
	DDRB = (1<<DDB4);//--------------------------------------------- Red Heating
	//Configure PB5 as output
	DDRB = (1<<DDB5);		//-------------------------------------- Blue IrrigationPump
	//Configure PD3 and PD4 as output
	DDRD = (1<<DDD3)|(1<<DDD4);	//---------------------- Yellow Aeration - Green Humidifier
	//Disable digital part of PC0, PC1, PC2 and PC3
	DIDR0 |= (1<<ADC0D)|(1<<ADC1D)|(1<<ADC2D)|(1<<ADC3D);
	//Enable pull-up resistance on PD0
	PORTD = (1<<PORTD0);
	//Configure PD1 as output and PD0 as input
	DDRD = (1<<DDD1)|(0<<DDD0);
	sei();
}

void eeprom_init(){
	if(1){
		eeprom_write_byte (addrTsp, Tsp);
		eeprom_write_byte (addrTHys,THys);
		eeprom_write_byte (addrHAsp, HAsp);
		eeprom_write_byte (addrHAhys, HAhys);
		eeprom_write_byte (addrHSsp, HSsp);
		eeprom_write_byte (addrHShys, HShys);
		eeprom_write_byte (addrVVmax,VVmax);
		eeprom_write_byte (addrVVmin, VVmin);
		eeprom_write_byte (addrEmax, Emax);
		eeprom_write_byte (addrEmin, Emin);
	}
}

void esperarTX(){
	while(!(UCSR0A&(1<<UDRE0))); //USART Data Register Empty
}

// TRANSMISION POR USART
void enviarADC(int datoEnviar){
	//UDR0=datoEnviar/1000+48;
	//esperarTX();
	//datoEnviar%=1000;
	UDR0=datoEnviar/100+48;
	esperarTX();
	datoEnviar%=100;
	UDR0=datoEnviar/10+48;
	esperarTX();
	UDR0=datoEnviar%10+48;
	esperarTX();
}

// TRANSMISION POR USART
void UsartPutChar(unsigned char data){
	// Esperamos que se vacie el buffer de transmision:
	while (!(UCSR0A & (1<<UDRE0)));

	// Ponemos el dato en el buffer de transmision:
	UDR0 = data;
}

// TRANSMISION POR USART
void UsartPutString(char *data)
{
	while (*data)
	{
		UsartPutChar(*data++);
	}
}

void timer_init()
{
	TCCR1A=0;          //Normal Mode
	TCCR1B=(1<<CS12);  //Prescaler 256
	TCCR1C=0;
	TIMSK1=(1<<TOIE1);  //Timer, Overflow Interrupt Enable
	TCNT1=3036;
}

void adc_init(void)
{
	ADCSRA=(1<<ADATE)|(1<<ADIE); //Auto Trigger and ADC Interrupt Enable
	ADCSRA=(1<<ADPS2)|(1<<ADPS0); //Division Factor 32
	ADCSRB=(1<<ADTS2)|(1<<ADTS1); //Timer/Counter1 Overflow
	ADCSRA |=(1<<ADEN); // ADC Enable
}

void IrrigationPump(){//----------------- Blue
	if (Emin < Light && Light < Emax) {
		
		
		if (Humidity < HSsp - HShys){
			PORTB |= (1<<PORTB5);
		}
		if (Humidity > HSsp + HShys){
			PORTB &= ~(1<<PORTB5);
		}
	}
}

void Aeration(){//------------------------ Yellow
	if (VVmin < Speed && Speed < VVmax) {
		
		
		if (Temperature > Tsp + THys){
			PORTD |= (1<<PORTD4);
		}
		if (Temperature < Tsp - THys){
			PORTD &= ~(1<<PORTD4);
		}
	}
}

void Heating(){//------------------------- Red
	if (PORTD &= ~(1<<PORTD4)) {
		
		
		if (Temperature < Tsp - THys){
			PORTD |= (1<<PORTB4);
		}
		if (Temperature > Tsp + THys){
			PORTD &= ~(1<<PORTB4);
		}
	}
}

void Humidifier(){//------------------------- Green
	if (Humidity < HAsp - HAhys){
		PORTD |= (1<<PORTD3);
	}
	if (Humidity > HAsp + HAhys){
		PORTD &= ~(1<<PORTD3);
		
	}
}

int main(void)
{
	Configuration();
	adc_init();
	timer_init();
	eeprom_init();
	i2c_init();
	i2c_start(); 
	lcd_init();
	IrrigationPump();
	Aeration();
	Heating();
	Humidifier();

	Tsp = eeprom_read_byte(addrTsp);
	THys = eeprom_read_byte(addrTHys);
	HAsp = eeprom_read_byte (addrHAsp);
	HAhys = eeprom_read_byte (addrHAhys);
	HSsp = eeprom_read_byte(addrHSsp);
	HShys = eeprom_read_byte (addrHShys);
	VVmax = eeprom_read_byte(addrVVmax);
	VVmin = eeprom_read_byte (addrVVmin);
	Emax = eeprom_read_byte(addrEmax);
	Emin = eeprom_read_byte(addrEmin);
	
	while(1)
	{
		//eeprom_update_byte(Variable1, alfa);
		eeprom_update_byte(addrTsp, Tsp);
		eeprom_update_byte(addrTHys, THys);
		eeprom_update_byte(addrHAsp, HAsp);
		eeprom_update_byte(addrHAhys, HAhys);
		eeprom_update_byte(addrHSsp, HSsp);
		eeprom_update_byte(addrHShys, HShys);
		eeprom_update_byte(addrVVmax, VVmax);
		eeprom_update_byte(addrVVmin, VVmin);
		eeprom_update_byte(addrEmax, Emax);
		eeprom_update_byte(addrEmin, Emin);
	}
}

int adc_read(int channel) //â€€Analog Channel Selection
{
	if(channel==0)
	ADMUX=0b01000000;
	if(channel==1)
	ADMUX=0b01000001;
	if(channel==2)
	ADMUX=0b01000010;
	if(channel==3)
	ADMUX=0b01000011;
	
	ADCSRA |=(1<<ADSC); //Start Conversion
	
	while(ADCSRA&(1<<ADSC))
	{
	}
	return (ADCW/10.23);
}

void Error(){
	UsartPutString("----------------------------------------------\n");
	UsartPutString("-------- ERROR EN EL INGRESO DE DATOS --------\n");
	UsartPutString("----------------------------------------------\n");
	UsartPutString("- Las opciones validas son:			-\n");
	UsartPutString("-T: Temperatura		-\n");
	UsartPutString("-H: Humedad		-\n");
	UsartPutString("-V: Velocidad del viento -\n");
	UsartPutString("-L: Luxometro            -\n");
	UsartPutString("---------------------------------------------\n\n");
}

ISR(TIMER1_OVF_vect)
{
	Temperature=adc_read(0);
	Humidity=adc_read(1);
	Speed=adc_read(2);
	Light=adc_read(3);
	
	//UsartPutString("SE MIDIO");
	TCNT1=3036;
}

ISR(USART_RX_vect){
	switch (UDR0){
		case T:
		case 't':
		UsartPutString("Temperatura: ");
		esperarTX();
		enviarADC(Temperature);
		UsartPutString("Â°C\n");
	    lcd_cmd(0x80);
		lcd_msg("Temperatura");
		lcd_cmd(0xC3);                   //Function to Send String to LCD
		lcd_dwr(Temperature);       //Function to send data to LCD

		break;
		
		case H:
		case 'h':
		UsartPutString("Humedad: ");
		esperarTX();
		enviarADC(Humidity);
		UsartPutString("%\n");
		lcd_cmd(0x80);
		lcd_msg("Humedad");
		lcd_cmd(0xC3);                   //Function to Send String to LCD
		lcd_dwr(Humidity);       //Function to send data to LCD

		break;

		case S:
		case 's':
		UsartPutString("Velocidad: ");
		esperarTX();
		enviarADC(Speed);
		UsartPutString("m/s\n");
	    lcd_cmd(0x80);
		lcd_msg("Velocidad");
		lcd_cmd(0xC3);                   //Function to Send String to LCD
		lcd_dwr(Speed);       //Function to send data to LCD

		break;

		case L:
		case 'l':
		UsartPutString("Luminosidad: ");
		esperarTX();
		enviarADC(Light);
		UsartPutString("lux\n");
		lcd_cmd(0x80);
		lcd_msg("Luminosidad");
		lcd_cmd(0xC3);                   //Function to Send String to LCD
		lcd_dwr(Light);       //Function to send data to LCD

		break;

		default:
		Error();
		break;
	}
}
